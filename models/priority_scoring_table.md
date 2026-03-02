# Priority Scoring Reference Table (Subtask 5)

## A) Dimension Definitions (0–100)

### Metric Importance (I)
Tier 1: 90–100 (patient safety / immediate harm)
Tier 2: 70–89 (major operational disruption)
Tier 3: 40–69 (financial/coding/efficiency)
Tier 4: 10–39 (informational)

### Anomaly Magnitude (M)
**If z-score:**
| |z| range | M |
|---|---:|
| <1 | 0 |
| 1–<2 | 25 |
| 2–<3 | 60 |
| 3–<4 | 85 |
| ≥4 | 100 |

**If percentile p:**
p<90 → 0–30  
90–95 → 40–60  
95–99 → 70–90  
>99 → 95–100  

### Persistence (P)
P = 100 × (k/w) + streak_boost  
streak_boost = +15 if consecutive ≥3, +25 if ≥5 (cap 100)  
Default windows: hourly w=24, daily w=7, weekly w=8

### Correlated Anomalies (C)
C = min(100, 25×r)  
r = # related metrics anomalous in same time window

### Data Quality Confidence (Q)
feed missing/late → Q=0 (Data Quality Incident)  
missing>20% → 20  
duplicate/zero/unit mismatch → 30  
minor completeness issue → 70  
passed checks → 100

---

## B) Final Priority Score Formula
Priority = 0.30×I + 0.30×M + 0.20×P + 0.15×C + 0.05×Q

## C) Severity Mapping
85–100: Critical  
70–84: High  
50–69: Medium  
0–49: Low

## D) Default SLA Targets
Critical: Ack 15m, Triage 1h, Plan 4h, Resolve 24h  
High: Ack 1h, Triage 4h, Plan 1d, Resolve 3d  
Medium: Ack 1d, Triage 2d, Plan 1w, Resolve 2w
