## Lessons Learned from Creating Datadog Metrics & Monitors
# Avoid redundant metrics
If a failure or event is already captured in the Dead Letter Queue (DLQ), there’s usually no need to create an additional metric for it. Rely on the DLQ as the source of truth.

# Instrument actions, not checks
Don’t emit metrics for methods that only check or validate state. Focus on methods that perform meaningful actions—especially when those actions fail.

# Be mindful with high-cardinality tags
Avoid adding tags with many possible values (e.g., organization_id). High-cardinality tags can significantly increase cost and reduce the usefulness of aggregations.

# Use scheduled workers for state tracking
If you need to monitor ongoing conditions or system state, consider using a scheduled job (e.g., a cron-based worker) instead of emitting metrics continuously.

# Leverage tracing for end-to-end visibility
Use traces to capture the full lifecycle and duration of processes like consumers. This provides better insight than metrics alone for performance and bottleneck analysis.
