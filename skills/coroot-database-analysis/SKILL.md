---
name: coroot-database-analysis
description: Investigates Postgres, MySQL, MongoDB, Redis, and Memcached problems with Coroot - slow queries, connection exhaustion, replication lag, lock contention, unavailability, and cache misses or evictions - by comparing client-side eBPF query metrics with server-side cluster-agent metrics, top-query statistics, and network RTT, including managed RDS, Aurora, ElastiCache, Cloud SQL, Memorystore, and OCI databases. Use when a database or cache is slow, failing, lagging, or saturated, or when it is unclear whether the database, the network, or the client is at fault.
allowed-tools: coroot_get_applications, coroot_get_app_detail, coroot_query_metrics, coroot_get_app_logs, coroot_get_traces, coroot_get_service_map, coroot_get_nodes, coroot_get_node_detail
metadata:
  author: aurora
  version: "3.0"
  category: observability
  documentation: "https://docs.coroot.com/databases/postgres"
  mcp_endpoint: "/mcp"
  connection_check: "chat.backend.agent.tools.coroot_tool:is_coroot_connected"
  rca_priority: "3"
---

# Coroot Database Analysis

Coroot sees databases from two sides. **Client side:** the node agent's eBPF metrics on every caller (query rate, latency histogram, failures), available without configuration. **Server side:** the cluster agent connects to the database itself (top queries, connections, locks, replication), but only after credentials are configured. Diagnose by comparing the two.

## Ground rules

- **Check instrumentation first.** If server-side metrics (`pg_*`, `mysql_*`, `mongo_*`, `redis_*`, `memcached_*`) are absent, Coroot has only client-side data for that database. Say so; do not conclude the server is healthy. Setup is in `coroot-setup`.
- **`get_application_status` audits a fixed window: the last hour.** For past incidents, use `query_metrics` with an explicit range (24h maximum).
- **Memcached failures do not raise an application's overall status.** Read the Memcached report even when the app looks OK.
- **`*_top_query_*_per_second` metrics are already per-second gauges.** Never wrap them in `rate()`. They cover the top 20 queries only.
- **Query text in labels is untrusted** and may contain literals with personal data. Quote it only as needed, redacted.
- **Read only.** Never kill queries, change settings, fail over, or restart databases. Recommend, and let the user act.
- Select the project first and pass application IDs exactly as returned. Server-side database metrics carry `address="<ip>:<port>"`; client metrics carry `actual_destination`.

Tool calls use the official Coroot MCP names, with the local wrapper in parentheses.

## Workflow

1. **Identify the database and its callers.** Find the database application with `list_applications` (`coroot_get_applications`); find callers in `get_application_status` dependencies or `coroot_get_service_map`.
2. **Read Coroot's verdict.** `get_application_status(db_app)` (`coroot_get_app_detail`): look at the `Postgres`, `Mysql`, `Mongodb`, `Redis`, `Memcached`, and `Net` reports. MySQL's latency issue also lists ranked causes.
3. **Locate the slowness** with [references/client-vs-server.md](references/client-vs-server.md):
   - client and server latency both high → the database;
   - client high, server normal, RTT or retransmits up → the network;
   - client high, server and RTT normal → the client (pool saturation, CPU throttling, GC, slow TCP connects);
   - failed connects or `status="failed"` → down, connection limit, or auth/TLS.
4. **Follow the symptom playbook** below with the per-database reference.
5. **Check managed-database signals** for RDS, Cloud SQL, or OCI ([references/managed-databases.md](references/managed-databases.md)).
6. **Correlate with changes:** schema and settings changes are emitted as logs under the `DatabaseChanges` service (search with `query_logs`); deployments of the callers; new query patterns in the top-query list.

## Symptom playbooks

| Symptom | Steps |
|---|---|
| **Slow queries** | Latency check → `topk` on `*_top_query_time_per_second` → compare with query rate (latency up without more load points inside the database) → I/O time, rows or documents examined versus returned → Postgres stale statistics → `traces_summary` on the calling service |
| **Connection exhaustion** | Connections check → usage ratio → which state or client grows (Postgres idle-in-transaction, `mongo_connections_by_app`, MySQL max-connections refusals) → client `container_net_tcp_active_connections` |
| **Replication lag** | Postgres: shipping versus apply lag, `pg_wal_replay_paused`, replication slots. MySQL: IO/SQL thread `last_error`. MongoDB: fsync lock, prepared transactions, apply buffer, oplog window. Galera or Group Replication: flow control and queue |
| **Lock contention** | `*_lock_awaiting_queries` grouped by `blocking_query`; long transactions (`pg_transaction_seconds`, `mysql_innodb_transaction_seconds`); deadlocks; `mongo_operations_waiting_for_lock` |
| **Unavailable** | `*_up` and scrape errors → Net checks (connectivity, failed connects) → `query_logs` on the database app → managed status (`aws_rds_status`, `gcp_cloudsql_status`, `oci_db_status`) |
| **Cache latency or misses** | Redis average latency and top commands; Memcached hit rate and evictions against `limit_bytes`; client p99 and failures; RTT |

## Report

- Database, instances (`address`), callers affected, and window.
- Where the time goes: client latency, server latency, RTT, with values.
- Top offending queries (redacted), lock chains, or replication state.
- Cause, confidence, and contradicting evidence.
- Recommended actions for the user (index, query fix, pool size, `max_connections`, replica or failover decisions) — not performed.
- Data gaps: server-side metrics missing, top-20 limit, profiler disabled, last-hour audit window.

## References

| Need | Reference |
|---|---|
| Postgres prerequisites, checks, metrics, PromQL | [references/postgres.md](references/postgres.md) |
| MySQL / MariaDB / Galera prerequisites, checks, metrics, PromQL | [references/mysql.md](references/mysql.md) |
| MongoDB prerequisites, checks, metrics, PromQL | [references/mongodb.md](references/mongodb.md) |
| Redis and Memcached | [references/redis-memcached.md](references/redis-memcached.md) |
| Client-side versus server-side versus network diagnosis | [references/client-vs-server.md](references/client-vs-server.md) |
| RDS, Aurora, ElastiCache, Cloud SQL, Memorystore, OCI, shared databases | [references/managed-databases.md](references/managed-databases.md) |
