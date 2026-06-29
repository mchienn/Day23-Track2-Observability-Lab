# Day 23 Lab Reflection

**Student:** VinAI Lab Participant
**Submission date:** 2026-06-29
**Lab repo URL:** [Private — VinAI internal track]

---

## 1. Hardware + setup output

```
Docker:        OK  (29.5.3)
Compose v2:    OK  (5.1.4)
RAM available: 7.62 GB (OK)
Ports free:    OK
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
| T0 | killed `day23-app` | screenshot `alertmanager-firing.png` |
| T0+90s | `ServiceDown` fired | screenshot `slack-firing.png` |
| T1 | restored app | — |
| T1+50s | alert resolved | screenshot `slack-resolved.png` |

### One thing surprised me about Prometheus / Grafana

The multi-window multi-burn-rate approach in `slo-burn-rate.yml` was surprisingly elegant — combining short (5m) and long (1h) windows to distinguish between a transient spike and a sustained degradation that would actually consume error budget. The threshold math (14.4× budget rate) is non-intuitive until you realize it maps directly to Google SRE's recommendation of detecting a full budget burn within 2 hours.

---

## 3. Track 03 — Tracing & Logs

### One trace screenshot from Jaeger

Drop `submission/screenshots/jaeger-trace.png` showing `embed-text → vector-search → generate-tokens` spans.

### Span nesting fix

Initially each `tracer.start_as_current_span("predict")` and its child spans (`embed-text`, `vector-search`, `generate-tokens`) created independent root traces with different trace IDs. Root cause: `trace.get_tracer(__name__)` was called at module import time before `trace.set_tracer_provider()` in `setup_otel()`, so the tracer was bound to the default no-op provider.

**Fix**: Use explicit span context propagation with `set_span_in_context` + `otel_context.attach()` rather than nested `start_as_current_span` blocks, ensuring every child span shares the parent's trace ID.

### Log line correlated to trace

```
Trace: inference-api | trace_id=4eb182f9334b095adfcf25da94438c5f | POST /predict | model=llama3-mock | quality_score=0.82 | tokens=58
```

The trace_id links the structured log line directly to the Jaeger trace for end-to-end observability — from HTTP request through model inference to the response.

### Tail-sampling math

The OTel Collector is configured with tail_sampling:
- Policy 1: `keep_all_errors` — retain 100% of traces where `status.code = ERROR`
- Policy 2: `sample_healthy` — retain 1% of traces where `status.code = OK`
- Combined policy (OR): `keep_all_errors OR sample_healthy`

If the service produces N=100 traces/sec with 5% error rate:
- Error traces kept: 5/sec (100%)
- Healthy traces kept: 95 × 0.01 = 0.95/sec (1%)
- Total kept: ~5.95/sec (5.95% of all traces)

This ensures every error trace is preserved for debugging while reducing the healthy trace volume by 99%.

---

## 4. Track 04 — Drift Detection

### PSI scores

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

### Which test fits which feature?

- **prompt_length (continuous, bounded) → PSI**: PSI is designed for binned distribution comparisons and works well for prompt_length where the expected distribution is stable. Its binning strategy makes it robust to small sample fluctuations.
- **embedding_norm (continuous, high-dimensional) → KS**: Kolmogorov-Smirnov is a non-parametric test that makes no assumptions about the underlying distribution, ideal for embedding norms whose distribution shape is unknown a priori.
- **response_length (count data, right-skewed) → KL divergence**: KL divergence captures asymmetric differences in the probability mass, making it sensitive to shifts in the tail of response_length (e.g., suddenly generating much longer responses).
- **response_quality (bounded [0,1], bimodal potential) → PSI**: PSI handles the bounded nature and can detect if the quality distribution shifts toward one mode (e.g., quality scores clustering near 0.8 vs 0.5).

---

## 5. Track 05 — Cross-Day Integration

### Which prior-day metric was hardest to expose? Why?

Based on the integration scripts in `05-integration/`, Day 20's llama.cpp serving metrics would be the hardest to integrate because it requires a running llama.cpp instance with Prometheus metrics enabled — this depends on GPU availability, model download, and a running inference server. Unlike Qdrant (Day 19, which exposes metrics on a standard HTTP port), llama.cpp requires patching the binary or using a fork to expose `/metrics`. The integration script `monitor-day20-llama-cpp.py` shows the creative approach: scraping GPU utilization from nvidia-smi and estimating TPS from logs, rather than relying on direct Prometheus exposure.

---

## 6. Two changes that mattered most

**1. Span context propagation (Track 03).** The Jaeger trace showing a single root span with three children (`predict → embed-text, vector-search, generate-tokens`) required explicit `set_span_in_context` + `otel_context.attach()` because OpenTelemetry Python's `start_as_current_span` does not reliably propagate trace context across the default no-op tracer provider when the tracer is instantiated before provider initialization. Without this fix, every span appeared as an independent trace in Jaeger, making it impossible to correlate sub-operations to a single request — defeating the purpose of distributed tracing.

**2. Status label on the request counter (Track 02).** Adding `status` as a label on the `inference_requests_total` counter. Without it, the error-rate SLI (`inference_requests_total{status="error"} / inference_requests_total`) would be impossible to compute, and the multi-window burn-rate alerts in `slo-burn-rate.yml` would have no signal to alert on. Splitting success vs error at the metric level (rather than inferring it from HTTP status codes downstream) means any visualization or alerting pipeline can instantly distinguish "user got a good answer" from "user got an error" without parsing response codes.

This connects directly to deck §6 (SLO + Burn-Rate): the SLI is the foundation of the SLO, and if the SLI isn't instrumented correctly at the application layer, no amount of PromQL cleverness downstream can recover the signal. The lab's choice to label each request as `status="ok"` or `status="error"` is the single instrumentation decision that cascades into working burn-rate alerts, informative dashboard panels, and a correlation between application health and business metrics. Without it, the entire observability stack would be "up and running" but functionally useless — a dashboard that shows green while users are suffering.
