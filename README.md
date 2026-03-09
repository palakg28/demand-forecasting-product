# 📦 Marketplace Dynamics: Demand Forecasting Product

> **Product Area:** Marketplace Logistics · **Role:** Product Manager · **Timeline:** Jan 2025  
> **Impact:** 20% reduction in service bottlenecks · 98% SLA compliance maintained

---

## 🧭 Overview

Marketplace platforms face a fundamental supply-demand synchronization problem: **customer order volume is unpredictable, but shopper availability must be pre-allocated.** Without foresight, this mismatch creates bottlenecks — long wait times, unmet orders, and SLA breaches.

This project documents the end-to-end product work to scope, design, and ship a **Demand Forecasting Engine** that bridges the gap between ML predictions and operational decision-making.

---

## 🎯 Problem Statement

| Symptom | Root Cause | Business Impact |
|---|---|---|
| Order fulfillment delays (>30 min) | Shopper supply doesn't match demand spikes | Customer churn, NPS drop |
| Manual staffing decisions | No forward-looking demand signal | Reactive ops, high labor cost |
| SLA breaches during peak hours | Bottlenecks at high-demand zones | Contractual penalties, trust erosion |

**Core question:** *How do we give operations teams a 2–4 hour forward view of demand so they can make proactive staffing decisions?*

---

## 🏗️ Product Scope & Strategy

### North Star Metric
**Bottleneck Rate** — % of 30-min windows where order volume exceeds shopper capacity by >15%

**Target:** Reduce bottleneck rate by **20%** within 90 days of launch.

### What We Built

```
┌─────────────────────────────────────────────────────────┐
│              DEMAND FORECASTING ENGINE                  │
│                                                         │
│  Data Inputs         ML Layer          Output Tools     │
│  ───────────         ────────          ────────────     │
│  · Order history  →  Gradient  →  · Ops Dashboard      │
│  · Time of day       Boosting     · Staffing Alerts     │
│  · Weather           (XGBoost)    · Zone Heatmaps       │
│  · Local events   →  LSTM      →  · Slack Notifications │
│  · Shopper GPS       (hourly)     · Shift Rec Engine    │
│  · Zone density                                         │
└─────────────────────────────────────────────────────────┘
```

### What We Explicitly Scoped Out (v1)
- Real-time re-routing of active shoppers (v2)
- Customer-facing ETAs powered by forecasts (v3)
- Automated shift scheduling (v2)

---

## 📐 PRD Highlights

### User Stories

**As an Operations Manager**, I want to see predicted demand by zone for the next 4 hours so I can dispatch the right number of shoppers before bottlenecks occur.

**As a Workforce Planner**, I want shift recommendations generated from forecasted demand so I can reduce both understaffing and overstaffing costs.

**As a Site Reliability Engineer**, I want automated alerts when predicted demand will exceed shopper capacity so I can escalate before SLAs are breached.

### Acceptance Criteria (Abbreviated)
- [ ] Forecast accuracy ≥ 80% (MAPE ≤ 20%) on held-out test data
- [ ] Dashboard loads in < 2 seconds (P95)
- [ ] Alerts fire ≥ 90 minutes before projected bottleneck window
- [ ] Shopper supply recommendations visible per zone, per 30-min block

---

## 🗃️ Data Model & KPI Design

### Core KPIs Defined

```sql
-- Bottleneck Rate: % of time windows where demand > supply capacity
SELECT
    zone_id,
    time_window,
    COUNT(*) FILTER (WHERE predicted_orders > shopper_capacity * 1.15)
        * 1.0 / COUNT(*) AS bottleneck_rate,
    AVG(predicted_orders)   AS avg_predicted_demand,
    AVG(shopper_capacity)   AS avg_supply
FROM demand_forecasts
WHERE forecast_date = CURRENT_DATE
GROUP BY zone_id, time_window
ORDER BY bottleneck_rate DESC;
```

```sql
-- SLA Compliance Rate: orders fulfilled within SLA window
SELECT
    DATE_TRUNC('hour', order_placed_at) AS hour_bucket,
    COUNT(*) FILTER (WHERE fulfilled_at - order_placed_at <= INTERVAL '45 minutes')
        * 1.0 / COUNT(*) AS sla_compliance_rate,
    COUNT(*) AS total_orders
FROM orders
WHERE order_placed_at >= NOW() - INTERVAL '7 days'
GROUP BY hour_bucket
ORDER BY hour_bucket;
```

```sql
-- Forecast vs Actual: measure model accuracy over time
SELECT
    f.zone_id,
    f.forecast_window,
    f.predicted_orders,
    a.actual_orders,
    ABS(f.predicted_orders - a.actual_orders) * 1.0
        / NULLIF(a.actual_orders, 0) AS mape,
    CASE
        WHEN ABS(f.predicted_orders - a.actual_orders) * 1.0
             / NULLIF(a.actual_orders, 0) <= 0.20 THEN 'ACCURATE'
        ELSE 'NEEDS REVIEW'
    END AS accuracy_flag
FROM demand_forecasts f
JOIN actual_order_counts a
    ON f.zone_id = a.zone_id
    AND f.forecast_window = a.time_window
ORDER BY mape DESC;
```

```sql
-- Staffing Efficiency: are we over- or under-staffing?
SELECT
    zone_id,
    shift_date,
    SUM(scheduled_shoppers)     AS total_scheduled,
    SUM(utilized_shoppers)      AS total_utilized,
    ROUND(SUM(utilized_shoppers) * 100.0 / NULLIF(SUM(scheduled_shoppers), 0), 1)
        AS utilization_pct,
    SUM(unmet_orders)           AS total_unmet_orders
FROM staffing_actuals
GROUP BY zone_id, shift_date
ORDER BY shift_date DESC, utilization_pct ASC;
```

---

## 📊 Metrics Framework

```
                      NORTH STAR
                   Bottleneck Rate
                   (Target: -20%)
                        │
          ┌─────────────┼─────────────┐
          │             │             │
    Forecast        SLA          Staffing
    Accuracy     Compliance     Efficiency
    (MAPE ≤ 20%) (≥ 98%)       (Util 85-95%)
          │             │             │
     Model        Alert Lead       Shift
    Retraining       Time         Rec CTR
    Cadence       (≥ 90 min)     (>70%)
```

---

## 🔁 Go-to-Market & Rollout

### Phased Launch Plan

| Phase | Scope | Success Gate | Duration |
|---|---|---|---|
| **Alpha** | 2 pilot zones, ops team only | Forecast MAPE < 25% | 2 weeks |
| **Beta** | 10 zones, workforce planners added | SLA compliance holds at ≥ 97% | 3 weeks |
| **GA** | All zones, Slack alerts live | Bottleneck rate drops ≥ 10% | Ongoing |

### Stakeholder Alignment
- **Operations:** Daily standup integration, heatmap embedded in existing ops console
- **Engineering:** Weekly model performance review, retraining pipeline owned by ML team
- **Finance:** Staffing cost delta tracked monthly vs. pre-launch baseline

---

## 📉 Results (Post-Launch)

| Metric | Baseline | Post-Launch | Change |
|---|---|---|---|
| Bottleneck Rate | 18.4% | 14.7% | **-20.1% ✅** |
| SLA Compliance | 94.2% | 98.1% | **+3.9pp ✅** |
| Avg. Alert Lead Time | N/A | 112 min | **>90 min target ✅** |
| Staffing Utilization | 71% | 88% | **+17pp** |
| Unmet Orders (peak hr) | 340/day | 198/day | **-41.8%** |

---

## 🛠️ Tech Stack

| Layer | Tool |
|---|---|
| Forecasting Model | XGBoost + LSTM ensemble |
| Feature Store | dbt + Snowflake |
| Orchestration | Airflow (retraining pipeline, 6hr cadence) |
| Dashboard | Internal React app + Recharts |
| Alerting | PagerDuty + Slack webhook |
| Monitoring | Grafana (model drift detection) |

---

## 📁 Repository Structure

```
demand-forecasting-product/
├── README.md                   ← You are here
├── prd/
│   └── demand_forecasting_prd_v1.md
├── sql/
│   ├── bottleneck_rate.sql
│   ├── sla_compliance.sql
│   ├── forecast_accuracy.sql
│   └── staffing_efficiency.sql
├── metrics/
│   └── kpi_framework.md
├── dashboard/
│   └── prototype/              ← Interactive dashboard mockup
└── docs/
    ├── rollout_plan.md
    └── stakeholder_brief.md
```

---

## 💡 Key PM Learnings

1. **ML ≠ Product.** The model was only 30% of the work. Translating confidence intervals into human-readable staffing recommendations was the hardest PM challenge.

2. **Alert fatigue is real.** First iteration fired too many low-confidence alerts. We introduced a threshold filter (≥70% confidence + ≥15% supply gap) to protect ops team attention.

3. **Define "accuracy" with ops, not data science.** MAPE of 20% sounds bad to an ML engineer; to an ops manager, being off by 4 shoppers out of 20 is acceptable. Aligning on *operational* accuracy thresholds changed the entire v1 spec.

---

## 🤝 Contributing / Contact

This is a portfolio project demonstrating PM skills in:
- Marketplace logistics & supply/demand systems
- SQL-based KPI design and metrics frameworks  
- ML product integration (translating model outputs to user tools)
- Cross-functional stakeholder alignment

---

*Built to demonstrate product management depth for marketplace and logistics roles.*
