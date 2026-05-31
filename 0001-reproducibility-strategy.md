# ADR 0001: Reproducibility Strategy for NorthStar Models

## Context

NorthStar's ML team currently stores models as files in S3 with ad-hoc names like `eta_v2_FINAL.onnx`. There is no record of which dataset version, code commit, or library environment produced any given artifact. A production incident cannot be reproduced, a failed experiment cannot be re-run, and the team cannot prove to auditors what data trained the model in production.

## Decision

Reproducibility will be enforced across four layers. Each layer has a specific tool and a specific enforcement point; there is no "trust the engineer" step.

**Environment:** Every training job runs inside a versioned Docker image built from a pinned `Dockerfile` in the monorepo. The image digest (`sha256:...`) is recorded as a required lineage field at MLflow registration time. Base images are pulled from ECR, not Docker Hub, to prevent upstream mutations. The CI pipeline refuses to register a model artifact if the image digest field is absent. Python dependencies are pinned in `requirements.txt` using `pip-compile` (pip-tools); no floating versions are permitted in that file.

**Data:** All training datasets are tracked with DVC, backed by S3. The `dataset_hash` (SHA-256 computed by DVC over the dataset content) is a required lineage field. The feature pipeline writes a DVC `.dvc` pointer file as its output artifact; downstream training jobs consume the pointer, not the raw S3 path. This means dataset version is part of the DAG, not ambient state. The frozen evaluation holdout set is a named DVC dataset (`holdout-eta-v1`) that is locked and never retrained on; re-creating it from scratch requires a manual unlock and a new hash.

**Code:** All training runs are triggered from a CI pipeline (GitHub Actions) that captures the exact `git_sha` of the triggering commit. Uncommitted local changes cannot produce a registerable artifact; the CI registration step reads `git status --porcelain` and aborts if the working tree is dirty. The `git_sha` is a required lineage field stored in MLflow.

**Randomness:** A single integer `random_seed` is passed as an explicit training parameter and stored as a required lineage field. It seeds Python's `random`, NumPy's `np.random`, and PyTorch's `torch.manual_seed` via a shared `set_seeds(seed)` utility called at the top of every training entry point. CUDA non-determinism is eliminated by setting `torch.use_deterministic_algorithms(True)` and `CUBLAS_WORKSPACE_CONFIG=:4096:8`; a small throughput penalty is accepted in exchange for reproducibility.

## Alternatives Rejected

- **Rely on S3 object timestamps and manual notes for dataset versioning:** S3 timestamps are mutable (objects can be overwritten silently), and manual notes rot. This is the current state and is precisely what caused the `eta_v2_FINAL.onnx` situation. DVC provides content-addressed hashing that S3 timestamps cannot.

- **Use Conda environments instead of Docker for environment pinning:** Conda solves Python dependencies but does not pin system libraries (GLIBC, CUDA drivers, C extensions). Two engineers on different base OS versions can get different training results even with identical `environment.yml` files. Docker image digests pin the full OS layer.

- **Record only the MLflow run_id and trust MLflow to store everything:** MLflow stores what the training script explicitly logs. If the script doesn't log the dataset hash or git SHA, they're absent. Enforcement must happen at registration time via a plugin hook that validates required lineage fields — not at logging time, where it can be forgotten.

## Consequences

- **Good:** Any registered model artifact can be reproduced by checking out the `git_sha`, pulling the `dvc_remote_url` dataset, and running the training job inside the `base_image_digest` container with the recorded `random_seed`. Incident post-mortems and compliance audits have a clear, automated answer to "what trained this model."
- **Good:** The lineage enforcement hook at registration acts as a hard gate — a training run that cut corners (wrong branch, no DVC pointer, missing seed) simply cannot be promoted to Staging.
- **Bad:** `torch.use_deterministic_algorithms(True)` adds ~10–15% training time overhead and prevents some CUDA kernel optimisations. This is accepted; training runs weekly/monthly, not continuously.
- **Bad:** The team must maintain Docker base images and a private ECR registry. Image build pipelines add ~5 minutes to the CI cycle. For a four-person team this is manageable; at 20 engineers it would need a dedicated image-maintenance rotation.

## Revisit If

Regulatory requirements (e.g., EU AI Act conformity assessment) mandate bit-for-bit reproducibility across hardware generations — at that point, deterministic CUDA alone is insufficient and the team would need to evaluate hardware-locked reproducibility tooling or snapshot-based VM replication for training environments.
