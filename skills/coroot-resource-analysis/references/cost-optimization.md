# Cost and capacity optimization

Use `coroot_get_costs(lookback_hours=24)` when investigating cost or right-sizing concerns. The official Coroot MCP tool list has no cost tool; without the wrapper, state that cost data is unavailable and fall back to utilization evidence.

## Compare

Where the data supports it:

- Configured resources and actual consumption.
- Utilization over representative workload periods.
- Replicas and workload distribution.
- Idle capacity and saturation.
- CPU throttling and memory pressure.
- Deployment changes and cost spikes.
- Cost per application or node, when provided.

## Right-sizing rules

- Do not recommend reducing resources based only on low average CPU usage. Evaluate peak load, latency, throttling, memory pressure, and availability requirements.
- Do not recommend increasing memory limits without considering node capacity.
- Consider whether raising a CPU limit would merely transfer contention to the node.
- Present recommendations as recommendations; never claim a resource change was made.
