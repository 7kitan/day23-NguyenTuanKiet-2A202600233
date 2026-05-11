# Track 3: Tracing and Logs - Summary

## Completed Checkpoints

### 1. End-to-End Trace (✓)
- Trace ID: `b43a6454fb4ac8b87a2f2b9227eeee32`
- Complete hierarchy with 4 spans:
  - `predict` (335.7ms) - parent span
    - `embed-text` (7.1ms)
    - `vector-search` (13.1ms)
    - `generate-tokens` (313.5ms)

**To capture:** Screenshot of Jaeger UI showing this trace with flame graph

### 2. GenAI Semantic Conventions (✓)
Span attributes follow OpenTelemetry GenAI conventions:
- `gen_ai.request.model` = "gpt-4"
- `gen_ai.usage.input_tokens` = 4
- `gen_ai.usage.output_tokens` = 35
- `gen_ai.response.finish_reason` = "stop"

**To capture:** Screenshot of span details panel in Jaeger showing these attributes

### 3. Tail-Sampling Policy (✓)
Configuration in `otel-config.yaml`:
- Keep 100% of ERROR status traces
- Keep 100% of traces > 2s latency
- Keep 1% of healthy traces (probabilistic)

**Metrics verification:**
- Spans received: 124
- Spans exported: 8
- Retention rate: 6.5% (expected ~3% for typical traffic)

**To document in REFLECTION:** Explain the sampling math and verify error traces are retained

### 4. Structured Logs with trace_id (✓)
Example log line:
```json
{
  "model": "gpt-4",
  "input_tokens": 4,
  "output_tokens": 42,
  "quality": 0.823,
  "duration_seconds": 0.3416,
  "trace_id": "a3f5b6f009f70a51d644bf8b1dcfb26e",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-05-11T05:48:14.350389Z"
}
```

**To paste in REFLECTION:** This log line showing trace_id correlation

## Remaining Work

### Logs to Loki (Partial)
- Loki is running and ready
- OTel Collector has Loki exporter configured
- **Issue:** Logs are only in stdout, not reaching Loki via OTLP

**Options to fix:**
1. Add OTLP log export to the app (requires code changes)
2. Configure filelog receiver with Docker log mounting
3. Use Promtail as log shipper (not in current stack)

For lab purposes, the structured JSON logs with trace_id demonstrate the concept even if not in Loki.

## URLs for Screenshots
- Jaeger UI: http://127.0.0.1:16686
- Trace URL: http://127.0.0.1:16686/trace/b43a6454fb4ac8b87a2f2b9227eeee32
- Grafana: http://127.0.0.1:3000
