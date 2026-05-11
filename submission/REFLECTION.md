# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _your name_
**Submission date:** _YYYY-MM-DD_
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
... paste here ...
```

---

## 2. Track 02 — Dashboards & Alerts

### 6 essential panels (screenshot)

Drop `submission/screenshots/dashboard-overview.png`.

### Burn-rate panel

Drop `submission/screenshots/slo-burn-rate.png`.

### Alert fire + resolve

| When | What | Evidence |
|---|---|---|
| _T0_ | killed `day23-app`         | screenshot `alertmanager-firing.png` |
| _T0+90s_ | `ServiceDown` fired   | screenshot `slack-firing.png` |
| _T1_ | restored app              | — |
| _T1+60s_ | alert resolved        | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

_(2-3 sentences)_

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

**Verified trace structure:** Found trace `a2d92f3955c491294e9738989e2df5e8` with 4 spans:
- Parent: `predict` (152ms) with `gen_ai.request.model=gpt-4`
- Child 1: `embed-text` (7ms) with `text.length=17`
- Child 2: `vector-search` (16ms) with `k=5`
- Child 3: `generate-tokens` (126ms) with `gen_ai.usage.input_tokens=4`, `gen_ai.usage.output_tokens=35`, `gen_ai.response.finish_reason=stop`

### Log line correlated to trace

Paste the log line and the trace_id it links to:

```json
{
  "model": "test-model",
  "input_tokens": 4,
  "output_tokens": 11,
  "quality": 0.801,
  "duration_seconds": 0.0944,
  "trace_id": "b978385ae22f5086e40a82970a9c9add",
  "event": "prediction served",
  "level": "info",
  "timestamp": "2026-05-11T06:50:56.716970Z"
}
```

The `trace_id` field enables correlation between logs and traces. In Grafana's Loki datasource, a derived field extracts this value and creates a clickable link to the corresponding trace in Jaeger.

### Tail-sampling math

**Policy configuration:**
1. `keep-errors`: 100% of traces with status_code = ERROR
2. `keep-slow`: 100% of traces with latency > 2000ms
3. `probabilistic-1pct`: 1% of remaining healthy traces

**Calculation for N traces/sec:**

Assuming typical distribution: 1% errors, 1% slow (non-error), 98% healthy fast traces:

```
sampled = N × (P(error) × 1.0 + P(slow ∧ ¬error) × 1.0 + P(healthy) × 0.01)
        = N × (0.01 × 1.0 + 0.01 × 1.0 + 0.98 × 0.01)
        = N × (0.01 + 0.01 + 0.0098)
        = N × 0.0298
        ≈ 3% retention
```

**Cost reduction:** ~97% fewer traces stored vs. retain-everything, while keeping 100% of actionable traces (errors and outliers).

**Buffer requirements:** With `decision_wait: 30s` and `num_traces: 50000`, the collector can handle up to ~1,666 traces/sec before buffer overflow (50,000 / 30 = 1,666).

---

## 4. Track 04 — Drift Detection

### PSI scores

Paste `04-drift-detection/reports/drift-summary.json`:

```json
... paste here ...
```

### Which test fits which feature?

For each of `prompt_length`, `embedding_norm`, `response_length`, `response_quality`, name the test (PSI / KL / KS / MMD) you'd choose in production and why.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

_(2-3 sentences. If you didn't have prior days running, write about which one would be hardest based on the integration scripts.)_

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.
