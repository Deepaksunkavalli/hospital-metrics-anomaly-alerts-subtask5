# Appendix: Worked Examples — Priority Scoring & Incident Aggregation (Subtask 5)

This appendix provides three end-to-end examples showing:
1) how a Priority Score is computed using the scoring framework, and
2) how multiple metric-level anomalies aggregate into a single incident.

## Scoring Formula
Priority = 0.30×I + 0.30×M + 0.20×P + 0.15×C + 0.05×Q

Severity mapping:
- 85–100: Critical
- 70–84: High
- 50–69: Medium
- 0–49: Low

---

## Example 1 — ICU Capacity Incident (Critical)

### Scenario
On a given day:
- ICU occupancy % spikes well above expected
- ICU admissions increase
- ICU discharges decrease
This pattern suggests a true capacity incident (not noise).

### Metric-level anomaly events (same day, ICU cluster)
1) ICU occupancy % (z=3.6)
2) ICU admissions/day (z=2.4)
3) ICU discharges/day (z=-2.2)

### Choose the “primary” alert metric for scoring
Primary metric: ICU occupancy %

### Assign scores
**I (Importance):** ICU occupancy is Tier 1 → I = 95  
**M (Magnitude):** |z|=3.6 → M = 85  
**P (Persistence):** last 7 days window, anomalies on 4 of 7 days → P = 100*(4/7)=57.14  
Streak is 3 consecutive days → +15 → P = 72.14  
**C (Correlation):** related ICU metrics anomalous r=2 (admissions + discharges) → C = min(100, 25×2)=50  
**Q (Data Quality):** checks passed → Q = 100

### Compute Priority
Priority = 0.30*95 + 0.30*85 + 0.20*72.14 + 0.15*50 + 0.05*100  
= 28.5 + 25.5 + 14.428 + 7.5 + 5  
= 80.928

### Severity
Priority ≈ 80.93 → **High**

### Escalation logic
Because ICU capacity is Tier 1 and risk is high, apply a safety escalation rule:
- If Tier 1 metric AND ICU occupancy > 95% (policy threshold), 
escalate one level.
Result: **Critical ICU Capacity Incident**

### Delivery & SLA
- Delivery: Pager/SMS + Email + Dashboard
- Ack: 15 minutes; Triage: 1 hour; Action plan: 4 hours; Resolve: 24 hours

### Aggregated incident title
**“ICU Capacity Stress: Occupancy High + Admissions Up + Discharges Down”**

---

## Example 2 — ED Flow Incident (High)

### Scenario
Over 6 hours:
- ED arrivals up
- ED boarding time up
- LWBS up
This indicates throughput breakdown.

Primary metric: ED boarding time (hourly)

### Assign scores
**I:** ED boarding time is Tier 2 → I = 80  
**M:** percentile score 97th percentile → M = 80  
**P:** last 24 hours; anomalies in 6 of 24 → P = 25  
Streak=4 consecutive hours → +15 → P = 40  
**C:** r=2 related metrics (arrivals + LWBS) → C = 50  
**Q:** passed checks → Q = 100

### Compute Priority
Priority = 0.30*80 + 0.30*80 + 0.20*40 + 0.15*50 + 0.05*100  
= 24 + 24 + 8 + 7.5 + 5  
= 68.5

### Severity
Priority = 68.5 → **Medium**  
But apply escalation rule:
- If ED boarding + LWBS both abnormal for ≥3 hours → 
escalate to High (operational safety risk).
Result: **High ED Flow Incident**

### Delivery & SLA
- Delivery: Email + Dashboard (+Teams optional)
- Ack: 1 hour; Triage: 4 hours; Action plan: 1 day; Resolve: 3 days

### Aggregated incident title
**“ED Throughput Breakdown: Boarding Up + LWBS Up + Arrivals Surge”**

---

## Example 3 — Infection Signal (High → Critical depending on persistence)

### Scenario (Daily)
- C. diff positivity rate up
- Isolation orders up
- Broad-spectrum antibiotic starts up

Primary metric: C. diff positivity rate

### Assign scores
**I:** Tier 1 quality metric → I = 92  
**M:** |z|=2.8 → M = 60  
**P:** last 7 days; anomalies 5 of 7 → P=71.43  
Streak=5 → +25 → P=96.43  
**C:** r=2 related metrics anomalous → C=50  
**Q:** passed checks → Q=100

### Compute Priority
Priority = 0.30*92 + 0.30*60 + 0.20*96.43 + 0.15*50 + 0.05*100  
= 27.6 + 18 + 19.286 + 7.5 + 5  
= 77.386

### Severity
Priority ≈ 77.39 → **High**

### Escalation rule
If Tier 1 infection metric persists ≥5/7 days AND correlation r≥2 → 
escalate to Critical (outbreak risk).
Result: **Critical Infection Signal Incident** (policy-driven escalation)

### Delivery & SLA
- Delivery: Pager/SMS (optional per hospital policy) + Email + Dashboard to Infection Prevention team
- Ack: 15 minutes (if escalated to Critical) / 1 hour (if High)
