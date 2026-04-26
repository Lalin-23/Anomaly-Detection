# SentnelOps Anomaly Detection System

## Overview

A **hybrid rule-based + LLM reasoning system** for detecting anomalous or inefficient infrastructure resources.

The system is designed with three priorities:
- **Determinism** — rule engine always produces results
- **Explainability** — every decision is interpretable
- **Reliability** — graceful fallback when LLM is unavailable

It combines fast heuristic detection with contextual reasoning using the Google Gemini API.

---

## Architecture

The system consists of three main components:

### 1. Data Model

Structured using Python dataclasses:

- `Resource`  
  Contains infrastructure metrics:
  - CPU (avg, p95)
  - Memory usage
  - Network usage
  - Disk usage
  - Error rate
  - Security attributes (internet exposure, identity)

- `AnalysisResult`  
  Standardized output format:
  - anomaly type
  - reason
  - suggested action
  - confidence score
  - optional security note

---

### 2. Rule Engine (Deterministic Layer)

The rule engine performs first-pass anomaly detection using fixed thresholds.

#### Detects:

- **Over-provisioned** → very low CPU usage  
- **CPU saturation** → high CPU usage  
- **CPU spikes** → low average but high p95  
- **Memory pressure** → high memory usage  
- **High network traffic** → excessive network load  
- **Disk saturation** → disk near capacity  
- **High error rate** → system instability  
- **Resource imbalance** → CPU–memory mismatch  

#### Output:
- anomaly type  
- triggered flags  
- base confidence score  

This layer always runs and does not depend on external APIs.

---

### 3. LLM Reasoning Layer (Google Gemini API)

Enhances rule-based output using contextual reasoning.

#### Features:
- Generates better natural language explanations  
- Adjusts confidence score slightly  
- Detects subtle patterns missed by rules  

#### Implementation:
- Uses **direct HTTP calls (`urllib`)**
- No external SDK required  
- Model: `gemini-3-flash-preview`

#### Failure Handling:
- If API fails or key is missing:
  - System falls back to rule-based reasoning  
  - No disruption in execution  

---

### 4. Orchestrator

The `AnomalyDetector`:
- Combines rule engine and LLM  
- Handles fallback logic  
- Supports batch processing  
- Produces final structured output  

---

## Anomaly Types

| Type | Description |
|------|------------|
| `over_provisioned` | Resource underutilized |
| `cpu_saturation` | CPU consistently high |
| `cpu_spikes` | Bursty workload |
| `memory_pressure` | Memory near limit |
| `high_network_traffic` | Abnormal traffic |
| `disk_saturation` | Disk nearly full |
| `high_error_rate` | Elevated failures |
| `resource_imbalance` | CPU-memory mismatch |
| `normal` | No anomaly |

---

## Security Signals

Additional contextual risks detected:

- Internet-facing + identity → **high blast radius**
- High outbound traffic → **possible data exfiltration**
- High error rate on public service → **possible attack**
- High internal network → **possible lateral movement**

---

## Confidence Scoring

```
confidence = rule_confidence + llm_adjustment
```

- LLM adjustment is small and bounded  
- Final score is clipped to `[0.05, 0.99]`  

This avoids overconfidence while allowing refinement.

---

## How to Run

### Requirements

- Python 3.10+
- No external dependencies required

---

### Optional: Enable LLM

Set your Gemini API key in the code:

```python
GEMINI_API_KEY = "your_api_key_here"
```

---

### Run

```bash
python anomaly_detector.py
```

Disable LLM:

```bash
python anomaly_detector.py --no-llm
```

---

## Output

- Printed to console  
- Saved as:

```
sample_outputs.json
```

---

## Example Output

```json
{
  "resource_id": "i-1",
  "is_anomalous": true,
  "anomaly_type": "over_provisioned",
  "reason": "CPU avg=2% and p95=5% indicate very low utilization.",
  "suggested_action": "Downsize instance or enable auto-scaling.",
  "confidence": 0.82,
  "security_note": "Internet-facing with identity attached — high blast radius risk."
}
```

---

## Design Tradeoffs

| Aspect | Decision | Tradeoff |
|--------|--------|---------|
| Determinism | Rule-first | Less adaptive |
| Explainability | LLM layer | Adds latency |
| Cost | LLM optional | Reduced insight without it |
| Simplicity | No ML models | No learned behavior |
| Reliability | Fallback mechanism | Less depth without LLM |

---

## Future Improvements

- Historical baselines (Z-score over time)  
- ML-based anomaly detection (Isolation Forest)  
- Context-aware thresholds (dev vs prod)  
- Real-time streaming pipeline  
- Feedback loop for false positives  
- Severity levels (LOW to CRITICAL)  
- Trend detection (early warnings)  

---

## Summary

This system balances:
- **Speed (rules)**  
- **intelligence (LLM)**  
- **reliability (fallback)**  

making it practical for real-world infrastructure monitoring without requiring large datasets or complex ML pipelines.
