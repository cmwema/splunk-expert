---
name: splunk-machine-learning
description: >-
  Splunk Machine Learning Toolkit (MLTK) and ML-based SPL — all tasks. Trigger
  for: MLTK, | fit, | apply, | summary, | score, MLflow integration, anomaly
  detection (anomalydetection, IsolationForest, LOF, DBSCAN), forecasting
  (StateSpaceForecast, ARIMA, LLP), clustering (KMeans, Birch, SpectralClustering),
  classification (RandomForestClassifier, LogisticRegression, SVM, DecisionTree),
  regression (LinearRegression, ElasticNet, RandomForestRegressor), feature
  engineering in SPL, model persistence (savemodel, loadmodel), MLTK Containers,
  custom MLflow models, Smart Monitoring, Predictive Analytics, Splunk AI Assistant.
  Default: Splunk Enterprise 9.x, MLTK 5.x.
---

# Splunk Machine Learning

Router only. Read `references/machine-learning.md` for all MLTK tasks.

## Routing

| Task | Action |
|---|---|
| Anomaly detection — | fit / | apply | Read `references/machine-learning.md` |
| Forecasting — StateSpaceForecast / ARIMA | Read `references/machine-learning.md` |
| Classification / regression | Read `references/machine-learning.md` |
| MLflow model import / MLTK Containers | Read `references/machine-learning.md` |
| SPL-native stats anomaly (without MLTK) | Defer to `splunk-spl` |

## Shared conventions

- Split data: train on historical data with explicit time range; apply on live data — never mix train/apply windows.
- Always `| summary` after `| fit` to inspect feature importance, R², and error metrics before deploying.
- Persist models with `into <model_name>` and reload with `| apply <model_name>` — do not retrain on every search run.
- Schedule retraining as a saved search (`cron_schedule`) when concept drift is expected.
- Feature engineering: normalize numeric fields with `| eval z = (x - mean) / stdev`; encode categoricals with `| eval cat = case(...)` before `| fit`.
- Anomaly detection: prefer `| anomalydetection` for simple outlier flagging; MLTK `IsolationForest` for multivariate.
- Forecasting: `StateSpaceForecast` handles seasonality automatically; use `future_timespan` to bound predictions.
- MLTK Containers: required for GPU-accelerated models and custom Python environments — run as a separate container reachable from the search head.
- Never expose raw MLTK model output to end-users without a threshold and human-readable label.

## Output format

Fenced `splunk` for SPL with MLTK commands. Include train + apply + score phases. ⚠️ callouts for data leakage, concept drift, and model staleness.
