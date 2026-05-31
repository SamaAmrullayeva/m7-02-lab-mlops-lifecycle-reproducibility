# Lifecycle: ETA Model (NorthStar Logistics)

```mermaid
flowchart TD
    subgraph Offline["🔵 Offline / Experimentation"]
        A[Raw Data\nS3 partitioned by date] -->|DVC pull → dataset_hash SHA-256| B[Feature Pipeline\nSpark job]
        B -->|features_v{N}.parquet + dataset_hash| C[Experiment Tracking\nMLflow]
        C -->|run_id, params, metrics| D[Training Run\nSageMaker Training Job]
        D -->|model artifact + run_id| E[Evaluation\nfrozen holdout set]
    end

    subgraph Registry["🟡 Model Registry (MLflow)"]
        F[Staging\nmodel:/eta/{run_id}]
        G[Production\nmodel:/eta/production]
        H[Archived\nmodel:/eta/{old_version}]
    end

    subgraph Serving["🟢 Online Serving"]
        I[Canary Deployment\n5% traffic slice]
        J[Full Production\n100% traffic]
    end

    subgraph Monitoring["🔴 Monitoring"]
        K[Metrics Collector\nCloudWatch + Evidently]
        L[Drift Detector\nPSI on feature distributions]
    end

    E -->|"✅ AUTO: MAE ≤ 4.2 min p95\n→ register to Staging"| F
    E -->|"❌ AUTO: fails gate\n→ reject, notify owner"| C

    F -->|"👤 MANUAL: ML Lead approves\nafter slice metrics review"| G
    G -->|"👤 MANUAL: retire on 3rd newer Production\nor policy violation"| H

    G -->|model_uri + deployed_version tag| I
    I -->|"✅ AUTO: canary p95_latency ≤ 150ms\n& error_rate ≤ 0.1% over 1hr"| J
    I -->|"❌ AUTO: canary fails\n→ rollback to prev production"| G

    J -->|request logs, latency, predictions| K
    K -->|drift_signal: PSI > 0.2 on\ntrip_distance or traffic_index| L
    L -->|"⚡ drift alert\n→ triggers retraining run"| A

    K -->|"SLA breach: p95 > 150ms\n→ page on-call"| J
```

## Artifact Glossary

| Artifact | Format | Owner |
|---|---|---|
| `dataset_hash` | SHA-256 of DVC-tracked parquet | Feature Pipeline |
| `run_id` | MLflow UUID | Training Run |
| `model_uri` | `model:/eta/{version}` | MLflow Registry |
| `deployed_version` | Semver tag `eta-{major}.{minor}` | CD Pipeline |
| `drift_signal` | PSI score + feature name | Evidently job |

## Transition Annotations

| Transition | Type | Trigger |
|---|---|---|
| Evaluation → Staging | **Automatic** | Gate metrics pass |
| Staging → Production | **Manual** | ML Lead approval |
| Production → Archived | **Manual** | Policy or version retirement |
| Canary → Full Production | **Automatic** | Canary health check passes |
| Canary → Rollback | **Automatic** | Canary health check fails |
| Monitoring → Retraining | **Automatic** | Drift alert (PSI > 0.2) |
