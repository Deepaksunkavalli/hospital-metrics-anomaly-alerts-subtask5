# Subtask 5 — Intelligent Alert Prioritization & Fatigue Prevention
## Hospital Metrics Anomaly Detection System — Alert Management Framework

### Purpose
This document defines a comprehensive alert management framework that converts anomaly signals into actionable incidents while minimizing alert fatigue. It includes: (1) multi-dimensional priority scoring, (2) severity mapping with delivery methods and SLAs, (3) alert aggregation into incidents, (4) fatigue prevention mechanisms, and (5) a closed-loop feedback system for continuous improvement.

---

## 1. Design Goals
1. **Detect early**: Identify emerging operational and quality issues early (capacity crises, infection signals, staffing strain, device failures, coding/billing anomalies).
2. **Reduce false alarms**: Require data quality validation and persistence checks before notifying users.
3. **Prioritize actions**: Rank alerts using impact-oriented scoring.
4. **Prevent fatigue**: Deduplicate, aggregate, throttle, and support user controls (snooze/dismiss).
5. **Enable response**: Provide clear ownership, delivery channels, and SLAs.
6. **Learn continuously**: Use user feedback and outcomes to recalibrate thresholds and models.

---

## 2. Multi-Dimensional Priority Scoring

### 2.1 Inputs
Each anomaly event produced by the detection layer provides:
- `metric_id`, `metric_name`, `department/unit`
- `timestamp`, `time_grain` (hourly/daily/weekly)
- `observed_value`, `expected_value`, `residual`
- `anomaly_score` (e.g., z-score, forecast residual, ML score percentile)
- `model_type` (SPC, forecasting, isolation forest, changepoint, etc.)
- `data_quality_flags` (missing/late/duplicate/unit mismatch)
- `related_metrics_anomalous` (optional, from correlation engine)

### 2.2 Scoring Dimensions (0–100)
We compute a **Priority Score (0–100)** using five dimensions:

1) **Metric Importance (I)** — criticality of the metric to patient safety and operations  
2) **Anomaly Magnitude (M)** — how abnormal the signal is vs expected  
3) **Persistence Duration (P)** — sustained vs transient anomaly  
4) **Correlated Anomalies (C)** — strength of related anomalies (incident-level confidence)  
5) **Data Quality Confidence (Q)** — penalty for low-confidence or broken data feeds

> Note: If a severe data issue is detected (e.g., feed missing/late), route as a **Data Quality Incident** rather than an operational incident.

---

### 2.3 Metric Importance Score (I)
Each metric is assigned a tier in the metric catalog:

- **Tier 1 (I=90–100):** Immediate patient safety / high harm potential  
  Examples: ICU occupancy %, mortality proxies, sepsis bundle compliance, CLABSI/CAUTI indicators
- **Tier 2 (I=70–89):** Major operational flow/capacity risk  
  Examples: ED boarding time, nurse-to-patient ratio, ventilator availability, device downtime
- **Tier 3 (I=40–69):** Financial/coding/efficiency risk  
  Examples: denials rate, charge lag, DRG distribution shift, CMI anomalies
- **Tier 4 (I=10–39):** Informational/low impact

Recommendation: store a single numeric importance score per metric (e.g., ICU occupancy = 95).

---

### 2.4 Anomaly Magnitude Score (M)
Magnitude depends on the detector type. We map to 0–100.

#### A) If detector outputs z-score (|z|)
- |z| < 1 → 0  
- 1 ≤ |z| < 2 → 25  
- 2 ≤ |z| < 3 → 60  
- 3 ≤ |z| < 4 → 85  
- |z| ≥ 4 → 100

#### B) If detector outputs percentile score (p)
- p < 90 → 0–30  
- 90–95 → 40–60  
- 95–99 → 70–90  
- >99 → 95–100  

---

### 2.5 Persistence Score (P)
Anomalies that persist are more actionable than one-off spikes.

Let:
- `w` = window length (# periods)
- `k` = # anomalous periods in last window
- `s` = consecutive streak length (most recent)

Base persistence:
- **P = 100 × (k / w)**

Streak boost:
- +15 if `s ≥ 3`
- +25 if `s ≥ 5`
Cap at 100.

Default windows:
- Hourly metrics: w = 24 (last 24 hours)
- Daily metrics: w = 7 (last 7 days)
- Weekly metrics: w = 8 (last 8 weeks)

---

### 2.6 Correlated Anomalies Score (C)
Correlated anomalies increase confidence that the signal reflects a real operational incident.

Approach: cluster-based correlation sets (simple, deployable).

For each metric, define a related set (examples):
- **ICU cluster:** ICU occupancy, ICU admits, ICU discharges, vent availability, ICU LOS
- **ED cluster:** arrivals, boarding time, LWBS, door-to-doc, admit decision-to-bed
- **Infection cluster:** C. diff positivity, isolation orders, blood cultures, broad antibiotic starts
- **Staffing cluster:** nurse ratio, overtime hours, sick calls, agency %
- **Finance/coding cluster:** DRG shift, CMI, denials, charge lag

Let `r` = number of related metrics anomalous in the same time window.

- C = min(100, 25 × r)
  - r=0 → 0
  - r=1 → 25
  - r=2 → 50
  - r=3 → 75
  - r≥4 → 100

---

### 2.7 Data Quality Confidence (Q)
Data issues are a leading cause of false alerts. We reduce priority when data quality is low.

Rules:
- Critical feed missing/late → **Q = 0** (create Data Quality Incident)
- Missing values > 20% → Q = 20
- Duplicate spikes / sudden zeros / unit mismatch → Q = 30
- Minor anomalies in completeness → Q = 70
- All checks pass → Q = 100

---

### 2.8 Final Priority Score (0–100)
Weighted scoring:

**Priority = 0.30×I + 0.30×M + 0.20×P + 0.15×C + 0.05×Q**

Rationale:
- Importance + Magnitude dominate (most clinically meaningful)
- Persistence reduces noise
- Correlation increases incident confidence
- Data quality prevents false operational alerts

---

## 3. Severity Levels, Delivery Methods, and SLAs

### 3.1 Severity Mapping
| Priority Score | Severity | Interpretation |
|---:|---|---|
| 85–100 | Critical | Immediate operational/quality threat |
| 70–84 | High | Likely true issue needing same-day triage |
| 50–69 | Medium | Monitor; investigate if persists or correlates |
| 0–49 | Low | Informational; log/digest only |

### 3.2 Delivery Channels by Severity
| Severity | Delivery | Audience |
|---|---|---|
| Critical | Pager/SMS + Email + Dashboard | On-call operations lead, nursing supervisor, quality/safety officer |
| High | Email + Dashboard (+Teams/Slack optional) | Unit manager, bed management, quality team |
| Medium | Dashboard + Daily digest email | Ops analysts, department leads |
| Low | Dashboard log only | Reference monitoring |

### 3.3 SLAs and Escalation
| Severity | Acknowledge | Triage | Action Plan | Target Resolution |
|---|---:|---:|---:|---:|
| Critical | 15 min | 1 hr | 4 hrs | 24 hrs |
| High | 1 hr | 4 hrs | 1 day | 3 days |
| Medium | 1 day | 2 days | 1 week | 2 weeks |
| Low | N/A | N/A | N/A | N/A |

Escalation:
- If not acknowledged within SLA → escalate to next role (e.g., unit director / admin on call).
- If severity increases (score jumps across band) → send immediate update even within cooldown.

---

## 4. Alert Aggregation into Incidents

### 4.1 Why Aggregation
Clinicians and administrators respond to **incidents**, not isolated metric blips. Aggregation reduces fatigue and improves actionability.

### 4.2 Aggregation Rules
Group multiple alerts into a single incident when they share:
- Same department/unit (ICU, ED, Med-Surg, OR, etc.)
- Same time window:
  - Hourly: within ±6 hours
  - Daily: within ±2 days
- Same cluster category (Capacity, Flow, Infection, Staffing, Devices, Finance/Coding)
- Correlation evidence (C score ≥ 50 recommended)

### 4.3 Incident Examples
- **ICU Capacity Incident:** ICU occupancy ↑ + ICU admits ↑ + ICU discharges ↓
- **ED Flow Incident:** ED arrivals ↑ + boarding time ↑ + LWBS ↑
- **Infection Signal:** C. diff positivity ↑ + isolation orders ↑ + broad antibiotics ↑
- **Staffing Stress:** nurse ratio ↓ + overtime ↑ + sick calls ↑
- **Coding/Finance Anomaly:** DRG distribution shift + CMI change + denials ↑

### 4.4 Incident Output (What users see)
Each incident includes:
- Title (human-readable)
- Severity level + Priority score
- Impact summary (patients affected / bed-hours / resource risk)
- Contributing metrics list (top 3–7)
- Trend context (last 7 periods vs baseline)
- Suggested investigation checklist
- Owner routing + SLA clock

---

## 5. Alert Fatigue Prevention

### 5.1 Progressive Alerting (Staged Notification)
Stages:
1) Detect (internal signal only)
2) Confirm (persistence rule; e.g., ≥2 of last 3 periods abnormal)
3) Notify (deliver based on severity)

Escalation rule:
- Medium → High if persists 24h (daily) or 6h (hourly), or correlation rises
- High → Critical if sustained + multiple related metrics abnormal

### 5.2 Cooldown and Deduplication
- **Cooldown window:** do not send repeat alerts for same incident within:
  - Critical: 30 minutes
  - High: 2 hours
  - Medium: 24 hours
Unless severity increases or new correlated metrics emerge.

- **Deduplication:** if an incident is already open, append updates rather than create new alerts.

### 5.3 User Controls: Snooze / Dismiss
- Snooze options: 2h, 8h, 24h, 72h
- Dismiss requires reason:
  - Known planned event
  - Data issue
  - Already being handled
  - False alarm
  - Threshold too sensitive
These reasons are captured for model improvement.

### 5.4 Auto-Escalation for Recalibration
Trigger “Recalibration Review” when:
- Alert volume for a metric exceeds a policy threshold (e.g., >3 High/Critical per week), OR
- >40% of alerts are dismissed as false alarms in a month, OR
- Median “usefulness rating” < 3/5

Recalibration actions:
- Adjust thresholds via backtesting
- Improve seasonality/holiday handling
- Fix denominators (rate metrics) and data quality rules
- Switch detector type for that metric (SPC ↔ forecast residuals)

### 5.5 Engagement Tracking
Track by severity and unit:
- Acknowledge rate
- Time-to-acknowledge
- Dismiss/snooze rates
- % incidents closed with root cause category
Low engagement indicates fatigue and triggers governance review.

---

## 6. Feedback Loop and Continuous Improvement

### 6.1 Feedback Collection
Required for High/Critical incidents:
- Usefulness rating (1–5)
- Correct severity? (too high / correct / too low)
- Root cause category:
  - Staffing
  - Capacity/flow
  - Infection/clinical
  - Device/equipment
  - Documentation/coding
  - Data quality
  - Other
- Free-text notes (optional)

### 6.2 Continuous Improvement Cycle
Weekly/Monthly:
1) Analyze alert outcomes by metric and unit
2) Update thresholds using retrospective backtests
3) Refine correlation clusters / related metric mapping
4) Review routing rules and SLAs
5) Publish performance summary:
   - Alerts/week by severity
   - % useful alerts
   - Mean time-to-detect
   - Mean time-to-acknowledge
   - Top noisy metrics and actions taken

Governance:
- Version all rule changes (threshold, scoring weights, routing)
- Maintain audit trail for alert decisions and user actions

---

## 7. Summary
This alert management framework operationalizes anomaly detection into an actionable incident response system by combining multi-dimensional scoring, severity-based delivery and SLAs, intelligent aggregation, fatigue prevention controls, and a feedback-driven improvement loop. It is designed for high-stakes hospital operations where both missed signals and alert overload carry significant cost.
