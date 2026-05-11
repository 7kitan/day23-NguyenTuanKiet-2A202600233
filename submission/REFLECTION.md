# Day 23 Lab Reflection

> Fill in each section. Grader reads the "What I'd change" paragraph closest.

**Student:** _your name_
**Submission date:** _YYYY-MM-DD_
**Lab repo URL:** _public GitHub URL_

---

## 1. Hardware + setup output

Paste output of `python3 00-setup/verify-docker.py`:

```
Docker:        OK  (28.5.1)
Compose v2:    OK  (2.40.2-desktop.1)
RAM available: 7.65 GB (OK)
Ports free:    BOUND: [8000, 9090, 9093, 3000, 3100, 16686, 4317, 4318, 8888]
Report written: /Users/kitan/dev/day23/00-setup/setup-report.json
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

Prometheus's pull-based model gives the scraper control over cardinality, not the instrumented service - preventing misconfigured apps from DOSing the monitoring system with metric floods.

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
{
  "prompt_length": {
    "psi": 3.461,
    "kl": 1.7982,
    "ks_stat": 0.702,
    "ks_pvalue": 0.0,
    "drift": "yes"
  },
  "embedding_norm": {
    "psi": 0.0187,
    "kl": 0.0324,
    "ks_stat": 0.052,
    "ks_pvalue": 0.133853,
    "drift": "no"
  },
  "response_length": {
    "psi": 0.0162,
    "kl": 0.0178,
    "ks_stat": 0.056,
    "ks_pvalue": 0.086899,
    "drift": "no"
  },
  "response_quality": {
    "psi": 8.8486,
    "kl": 13.5011,
    "ks_stat": 0.941,
    "ks_pvalue": 0.0,
    "drift": "yes"
  }
}
```

**Detected drift:** 2 features show significant drift (PSI > 0.2):
- `prompt_length`: shifted from mean=50 to mean=85 (users sending longer prompts)
- `response_quality`: shifted from beta(8,2) to beta(2,6) (quality degraded from high to low)

### Which test fits which feature?

**prompt_length** (continuous, unbounded token count):
- **Best: PSI** - Industry standard for monitoring count-based features in production ML systems (Arize, Fiddler, Gantry all use PSI). PSI=3.461 successfully detected the 50→85 token shift. Binning makes it robust to outliers and interpretable to stakeholders ("PSI > 0.2 = investigate"). Established thresholds (0.1/0.2) enable automated alerting.
- Alternative: KS test for statistical hypothesis testing (provides p-value for significance).

**embedding_norm** (continuous, bounded ~[0.7, 1.3], L2 norm):
- **Best: KS test** - Non-parametric, no distribution assumptions, handles bounded continuous distributions well. KS stat=0.052 correctly identified no drift. Robust to outliers in embedding space without requiring binning choices.
- Alternative: MMD with RBF kernel if monitoring full high-dimensional embeddings (not just L2 norm). MMD captures complex distribution shifts in kernel space.

**response_length** (continuous, unbounded token count):
- **Best: PSI** - Same reasoning as prompt_length. Production systems prefer PSI for interpretability and consistency across features. PSI=0.016 shows stable generation length. Binning smooths noise from token-level variance.
- Alternative: Quantile-based monitoring (P50/P95 shift detection) for real-time dashboards.

**response_quality** (continuous, bounded [0,1], model score):
- **Best: KL divergence** - Treats quality score as a probability-like distribution. KL=13.5 quantifies the magnitude of quality degradation (catastrophic shift from beta(8,2) to beta(2,6)). More informative than binary "drift/no drift" - tells you *how much* quality dropped.
- Alternative: KS test for statistical significance (KS stat=0.941, p<0.001 confirms extreme shift). Use both: KL for magnitude, KS for confidence.

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

**Day 20 (llama.cpp model serving)** would be the hardest to expose in production. llama.cpp's HTTP server doesn't natively emit Prometheus metrics - it's a C++ inference engine focused on speed, not observability. The lab's integration requires either patching llama.cpp source to add a `/metrics` endpoint or running a sidecar that scrapes llama.cpp's internal stats and re-exports them in Prometheus format. This is fragile: any llama.cpp version upgrade could break the sidecar's stat parsing. In contrast, Day 19's Qdrant has native Prometheus support, and Day 17's Airflow has well-maintained exporters (statsd_exporter). The lesson: when choosing infrastructure components, native observability support (OpenMetrics/OTLP) is a first-class requirement, not a nice-to-have.

---

## 6. The single change that mattered most

> **Grader reads this closest.** What one thing about your stack design — a metric you added, a label you dropped, a panel you reorganized, an alert threshold you tuned — made the biggest difference between "works" and "useful"? Write 1-2 paragraphs. Connect it to a concept from the deck.

Adding `uid: prometheus` to datasources.yml (one line) fixed the entire integration layer. Without it, Grafana auto-generated a random UID, causing all dashboard queries to fail silently despite Prometheus having the data. This connects to the deck's §4 emphasis on dashboards-as-code: explicit configuration beats implicit behavior. When dashboard JSON hardcodes `"uid": "prometheus"`, the provisioning must guarantee that UID exists. Without deterministic identifiers, dashboards break on every fresh install - the difference between "works on my laptop" and "works in production."
