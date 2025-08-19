# System_Design_repo
High to low level system design of centralized logging system

# 1) Goals & requirements

**Functional**

* Ingest logs from apps/infra (containers, VMs, functions, LB/WAF, DB, CDN).
* Handle multiple formats (JSON, text), auto-parse, enrich (host, env, trace\_id).
* Search/visualize, build saved views, dashboards, and alerts.
* Multi-tenant (teams/projects) with RBAC.
* Retention tiers (hot/warm/cold), legal hold, export.
* Correlate with traces/metrics (observability).

**Non-functional**

* High throughput (≥ 100k–1M events/sec).
* Low query latency for recent logs (P95 < 2–5s for 95% queries).
* Backpressure & loss protection (durable queues).
* Exactly-once or effectively-once semantics (dedupe keys).
* Compliance: PII redaction, encryption at rest/in transit, audit.

---

# 2) High-level architecture

```
[Sources: apps, pods, nginx, RDS, cloudtrail, lambda]
         │
         ▼
   Log Shippers (daemonset/agent)
   - Fluent Bit / Filebeat / Vector
   - Local buffering, multiline, sampling
         │
         ▼
   Ingestion Layer (HTTP/gRPC + Kafka)
   - REST/OTLP endpoints (rate-limit, auth)
   - Kafka/Pulsar for durable buffering
         │
         ├────────► Stream Processing
         │          - Fluentd/Logstash/Kafka Streams
         │          - Parse, normalize (ECS), redact PII,
         │            enrich (geoip, k8s labels, service, trace_id)
         │
         └────────► Hot Storage (search)
                    - Elasticsearch/OpenSearch/Loki
                    - Hot/Warm nodes + index lifecycle mgmt
                           │
                           ├──► Cold/Archive (cheap)
                           │    - S3/GCS + index manifest
                           │
                           └──► OLAP (optional for analytics)
                                - ClickHouse/BigQuery/Druid
         
   Query & UX
   - Kibana/OpenSearch Dashboards/Grafana
   - Alerting service (rules engine)
   - RBAC & tenancy gateway
```

---

# 3) Component choices (trade-offs)

**Shippers/Agents**

* **Fluent Bit** (fast, tiny, C), **Filebeat** (tight ES integration), **Vector** (rich transforms).
* Must support: multiline (Java stacktraces), local disk buffer, TLS, backoff.

**Transport / Buffer**

* **Kafka** (default): high throughput, partitioning by `tenant_id` or `service`.
* **Pulsar** if you want tiered storage and geo-replication built-in.

**Processing**

* **Fluentd**/**Logstash** (config-driven), **Kafka Streams/Flink** for heavy transforms.
* Responsibilities: parsing, re-timestamping, schema normalization (ECS/OpenTelemetry), redact PII, add k8s metadata, trace\_id extraction.

**Searchable store (hot)**

* **Elasticsearch/OpenSearch**: free text + aggregations; mature ecosystem.
* **Loki**: very cheap for logs-at-scale; labels + object store; queries via LogQL (great with Grafana). Less powerful aggregations vs ES.
* **ClickHouse**: blazing-fast analytics, but full text/regex is trickier.

**Archive**

* S3/GCS/Blob + lifecycle (IA/Glacier). Keep queryability via:

  * ES/OpenSearch “searchable snapshots”
  * Loki’s boltdb-shipper in object storage
  * External query engines (Athena/Trino) for ad-hoc digs.

**Visualization & Alerts**

* **Kibana**/**OpenSearch Dashboards** with ES/OpenSearch.
* **Grafana** for Loki/Tempo/Prometheus stack; alerts via Grafana Alerting/PagerDuty/Slack.

---

# 4) Data model (ECS-style)

Normalize into a predictable schema to make queries/alerts portable:

```json
{
  "@timestamp": "2025-08-20T10:39:12.561Z",
  "message": "GET /payments/123 200 in 37ms",
  "log": { "level": "INFO", "logger": "http" },
  "host": { "name": "ip-10-0-12-34" },
  "service": { "name": "payments-api", "version": "1.12.3" },
  "cloud": { "provider": "aws", "region": "ap-south-1" },
  "kubernetes": {
    "namespace": "prod",
    "pod": { "name": "payments-6c9b7f" },
    "container": { "name": "app" },
    "labels": { "team": "finops" }
  },
  "http": { "request": { "method": "GET" }, "response": { "status_code": 200 } },
  "url": { "path": "/payments/123" },
  "event": { "dataset": "nginx.access" },
  "trace": { "id": "8f7b3e..." },
  "span": { "id": "a91d..." },
  "user": { "id": "u_29381" }
}
```

**Parsing rules**

* Prefer structured JSON logs at source (best latency & accuracy).
* For text logs: define grok/regex parsers once in processing layer; add unit tests.

---

# 5) Tenancy, security, and governance

* **Tenancy model**: add `tenant_id` and `env` labels at ingestion; physically separate topics/indices per tenant+env for isolation.
* **RBAC**: gateway issues scoped JWT; index-level permissions (ES/OpenSearch) or per-tenant namespace labels (Loki).
* **PII controls**: redact via processor (regex/fpe) before indexing; maintain allow/deny lists.
* **Encryption**: TLS everywhere; at-rest via KMS; restrict snapshot buckets.
* **Audit**: log access queries (who searched what), rule changes, exports.

---

# 6) Indexing & lifecycle (for ES/OpenSearch)

**Index design**

* Time-based indices: `logs-<tenant>-<dataset>-yyyy.MM.dd`.
* Shards per index based on daily volume; aim shard size 20–50 GB.
* Use **rollover** at size/time; ILM moves shards from hot→warm→frozen.

**Templates & mappings (snippet)**

```json
{
  "index_patterns": ["logs-*-*"],
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "refresh_interval": "10s",
    "codec": "best_compression"
  },
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "@timestamp": { "type": "date" },
      "message": { "type": "text" },
      "service.name": { "type": "keyword" },
      "log.level": { "type": "keyword" },
      "http.response.status_code": { "type": "integer" },
      "trace.id": { "type": "keyword", "ignore_above": 256 }
    }
  }
}
```

**ILM policy (example)**

* **Hot (7 days)**: fast SSD nodes.
* **Warm (30–90 days)**: slower nodes, force merge to 1 segment.
* **Cold (180–365 days)**: searchable snapshots on object storage.
* **Delete** after retention unless legal hold tagged.

---

# 7) Query, alerting, SLOs

**Typical queries**

* Error rate spike:

  * ES: `log.level:ERROR AND service.name:payments-api AND @timestamp:[now-15m TO now]`
  * Loki: `{service="payments-api", level="error"} |= "" | unwrap ts`
* P95 latency (if logged): aggregate by service & path.
* Security: failed logins by IP, geo anomalies.

**Alerting**

* Rule: `count(level=ERROR) / count(total) > 2% over 5m` per service.
* Delivery: Slack/PagerDuty/Email; include sampled log exemplars.
* **Dedup**: use fingerprint (hash of message template + service + path).

**SLOs (example)**

* Ingestion availability 99.9%
* End-to-end ingest-to-search delay P95 < 30s (hot tier)
* Query P95 < 3s for 1-day time range, < 10s for 7-day

---

# 8) Backpressure, reliability, and cost control

* **At agent**: disk buffer size (e.g., 1–5 GB), max inflight, exponential backoff.
* **At gateway**: rate limit per tenant; shed over-limit traffic with 429 + retry-after.
* **Queue**: Kafka with replication (3x), min.insync.replicas=2, acks=all.
* **Idempotency**: generate `_id` as hash of `tenant_id + ts + source + message` to avoid duplicates on retries.
* **Sampling**: dynamic sampling for noisy debug logs; keep all ERROR/WARN.
* **Cost levers**: structured logs (smaller), drop unneeded fields, longer refresh interval, downsample after 7 days, tier to object storage quickly.

---

# 9) Capacity planning (rough math)

Let:

* V = events/sec, S = avg event size (bytes)
* **Ingress bandwidth** ≈ V × S
* **Daily volume** ≈ V × S × 86,400
* ES hot storage (replica=1, index/segment overhead \~1.3×):

  * **Daily hot storage** ≈ Daily volume × 2 (replicas) × 1.3
* Kafka: plan **retention** of 24–48h; disk ≈ Daily volume × retention\_days × 1.2 overhead.
* Shards: target 20–50 GB/shard/day ⇒ `shards_per_day ≈ (daily hot volume) / 40 GB`.

Example: 150k eps, 400 B/event → 60 MB/s ingress; 5.2 TB/day raw ⇒ \~13.5 TB/day hot with replicas/overhead. Plan nodes accordingly.

---

# 10) APIs (ingestion & search)

**Ingestion (HTTP)**

```
POST /v1/logs
Headers: Authorization: Bearer <token>
Body: [
  { "ts":"2025-08-20T10:39:12Z", "service":"payments", "level":"INFO",
    "message":"charge captured", "trace_id":"...", "fields":{...} },
  ...
]
```

* Validate size (max 5–10 MB), batch up to 500 events.
* Respond with per-event status and dedupe keys.

**Search (proxying to backend)**

```
GET /v1/search?q=service:payments AND level:ERROR&from=now-1h&to=now&limit=200
```

* Enforce tenant scoping, field whitelists, rate limits.

**Admin**

* Create tenant, API keys, ILM policies, export to S3, legal hold.

---

# 11) Kubernetes (EFK/Loki) deployment blueprint

* **DaemonSet**: Fluent Bit on every node with:

  * Inputs: `tail` (container logs), `systemd`.
  * Filters: kubernetes metadata, multiline, redact.
  * Outputs: HTTP to gateway and/or Kafka.
* **StatefulSets**:

  * ES/OpenSearch data nodes (hot/warm), master nodes, ingest nodes.
  * Loki distributor/ingester/querier + memcached + S3 backend.
  * Kafka (with ZooKeeper if not KRaft) or managed (MSK/Confluent).
* **ConfigMaps/Secrets**: parser rules, credentials, CA bundles.
* **ServiceMesh** (optional): mTLS for agents→gateway.
* **Observability**: self-logs to another tenant (“platform”).

---

# 12) Rollout plan

1. **Pilot** one tenant (staging) with 7-day hot retention.
2. Onboard top 3 noisy services; fix log formats to JSON.
3. Set PII rules + sampling policies.
4. Capacity test (burst 2× expected).
5. Enable alerts for ERROR spikes, 5xx from gateways.
6. Expand to all services; enforce budget guardrails & team quotas.

---

# 13) Quick stack presets

**Option A (classic, powerful):**
EFK = **Fluent Bit → Kafka → Logstash → OpenSearch + OpenSearch Dashboards**, archive to S3 via snapshots.

**Option B (cost-efficient, cloud-native):**
**Vector/Fluent Bit → Loki (boltdb-shipper) + S3 → Grafana**, traces in Tempo, metrics in Prometheus.

**Option C (analytics-heavy):**
**Fluent Bit → Kafka → Flink → ClickHouse for analytics + OpenSearch for text search**, dual write.

---

# 14) Common pitfalls (and fixes)

* **Unbounded cardinality labels** (e.g., `user_id` as label in Loki) ⇒ explode memory. Keep high-cardinality fields as log fields, not labels/terms.
* **Tiny shards** in ES ⇒ slow queries. Target 20–50 GB shards; use rollover.
* **Regex-heavy parsing at query time** ⇒ parse on ingest instead.
* **No backpressure** ⇒ agent drops. Always buffer at agent + queue.
* **PII leaks** ⇒ test redaction with sample corpora; add unit tests for parsers.

---
