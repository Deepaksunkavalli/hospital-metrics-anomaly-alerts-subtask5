# Alert Lifecycle & Aggregation Flow (Subtask 5)

## 1) Signal Generation (Detection Layer)
Inputs: metric value, baseline/forecast, residual, anomaly score, model type  
Output: anomaly event (per metric per period)

## 2) Data Quality Gate
Checks:
- feed delay/missing
- missingness %
- duplicates / sudden zeros
- unit mismatch
Decision:
- If severe issue → create **Data Quality Incident** and stop operational alert
- Else → continue

## 3) Scoring & Severity Assignment
Compute:  
I (importance), M (magnitude), P (persistence), C (correlation), Q (quality)  
Priority = weighted score (0–100)  
Map to severity: Low/Medium/High/Critical

## 4) Incident Aggregation
Group anomaly events into incidents using:
- unit/department match
- time window overlap
- cluster category (Capacity, Flow, Infection, Staffing, Devices, Finance/Coding)
- correlation evidence (C threshold)
Result: one incident with multiple contributing metrics

## 5) Fatigue Controls
- Progressive alerting (confirm via persistence before notifying)
- Cooldown and deduplication (update existing incident)
- Snooze/dismiss with reason capture
- Auto-recalibration trigger for noisy metrics

## 6) Delivery & Routing
Severity determines:
- channel: pager/SMS/email/dashboard/digest
- recipients: on-call ops, unit manager, quality team
- SLA timers: acknowledge/triage/action/resolve  
Escalation if SLA breached or severity increases

## 7) Investigation Support
Incident page provides:
- top contributing metrics + sparkline trends
- baseline comparison (same weekday/seasonality)
- related metrics panel
- recommended RCA checklist
- ownership + notes + action tracking

## 8) Closure & Feedback Loop
Upon closure:
- usefulness rating (1–5)
- severity correctness
- root cause category
- notes  
System uses feedback to:
- recalibrate thresholds
- refine seasonality rules
- adjust correlation clusters and routing policies
- publish alert performance metrics monthly
