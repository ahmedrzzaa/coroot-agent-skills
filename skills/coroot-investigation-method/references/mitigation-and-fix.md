# Mitigation versus fix

Mitigation restores service now; a fix removes the cause. Report them separately. Never perform a mitigation without explicit authorization from the user, and capture evidence (traces, connection and pool state, log samples, metric snapshots) before anything that resets state.

| Cause | Mitigation (restores service) | Side effects | Fix (removes cause) | Follow-up alert |
|---|---|---|---|---|
| **Connection leak / pool exhaustion** | Restart the service (empties the pool); scale out temporarily | Erases the leaked-connection evidence; problem returns | Release connections on every code path, including errors (see below); enable pool leak detection; set an acquire timeout so callers fail fast; server-side idle-transaction timeout | Pool in-use = max, or waiters > 0; active connections to the database at pool size |
| **Bad deployment** | Roll back | Loses the new version's other changes | Fix the regression, add a test, redeploy | New error patterns after rollout; SLO burn after deploy |
| **Slow query or lock** | Cancel the blocking query (database owner's call) | Aborted work, possible retries | Index, query rewrite, shorter transactions | Lock waiters, top-query time |
| **CPU throttling** | Raise the CPU limit or scale out | Shifts contention to the node | Right-size limits from peaks; optimise hot paths (Profiling tab) | Throttled time per second |
| **Memory leak / OOM** | Restart, temporarily raise the limit | Leak returns; node pressure | Fix the leak (heap profiles) | Sustained RSS growth, OOM kills |
| **Retry storm** | Shed load, open circuit breakers, lower retry counts | Some requests fail fast | Retry budgets, backoff with jitter, propagate deadlines | Outbound rate rising faster than inbound |
| **Traffic spike** | Scale out, rate-limit | Cost; rejected requests | Capacity planning, autoscaling | Inbound rate versus baseline |

## Releasing connections on every path

The pattern is the same in every language: acquire, use, and release in a construct that runs even when the code throws or returns early.

- **Java (JDBC, HikariCP):** try-with-resources on `Connection`, `Statement`, `ResultSet`. HikariCP `leakDetectionThreshold` logs connections held too long; `connectionTimeout` bounds how long callers wait.
- **Go (`database/sql`):** `defer rows.Close()` right after a successful query; always `Commit` or `Rollback` transactions (`defer tx.Rollback()` is safe after commit). With pgx pools, `defer conn.Release()`.
- **Python:** context managers (`with pool.connection() as conn:`), or `try`/`finally` returning the connection.
- **Node.js (`pg`):** `const client = await pool.connect(); try { … } finally { client.release(); }`.
- **PostgreSQL server side:** `idle_in_transaction_session_timeout` ends sessions that sit inside an open transaction.

## Writing the follow-up

- Record the mitigation as mitigation: what was done, when, by whom, and that the cause is not fixed.
- Open a ticket for the fix with the evidence: trace IDs, the relevant metrics, and the code path if known.
- Add an alert on the cause, not only the symptom, and update the runbook with how to recognise and confirm it.
- If the incident recurs, treat it as a known bug with an owner and a deadline.
