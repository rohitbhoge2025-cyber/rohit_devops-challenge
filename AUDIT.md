# AUDIT REPORT

This document records the defects identified during the initial review of the inherited Skybyte DevOps challenge repository. Findings are categorized by operational impact area and include the observed issue, production risk, and remediation approach.


# Security Findings

## Container runs as root
- File: `Dockerfile`
- Issue: No `USER` directive is defined, so the container runs as root by default.
- Risk: A compromised application process would gain root privileges inside the container, increasing privilege escalation and container breakout risk.
- Fix: Added a dedicated non-root user and configured the container to run with restricted privileges.

## Unpinned base image
- File: `Dockerfile`
- Issue: The base image uses `python:3.9` without patch-level pinning.
- Risk: Builds become non-deterministic and may pull vulnerable or behaviorally different image versions over time.
- Fix: Replaced the image with a pinned slim Python runtime image.

## Plaintext API token committed in Helm values
- File: `helm/skybyte-app/values.yaml`
- Issue: The API token is hardcoded directly in the Helm values file.
- Risk: Secrets stored in Git may leak through repository history, CI logs, Helm release metadata, or developer access.
- Fix: Removed the secret from Helm values and migrated secret handling to a Kubernetes Secret referenced through `secretKeyRef`.

## Missing container security context
- File: `helm/skybyte-app/templates/deployment.yaml`
- Issue: Deployment lacks hardened container security settings such as `runAsNonRoot`, `readOnlyRootFilesystem`, `allowPrivilegeEscalation`, dropped capabilities, and `seccompProfile`.
- Risk: Containers run with unnecessarily permissive runtime settings, increasing the blast radius of a compromise.
- Fix: Added a restricted security context aligned with Kubernetes restricted pod security recommendations.

## Secret managed directly through Terraform state
- File: `terraform/main.tf`
- Issue: The Kubernetes API token secret is provisioned directly through Terraform variables.
- Risk: Terraform stores secrets in state files, which can expose sensitive values if the backend is not properly secured.
- Fix: Documented state security assumptions for local challenge usage and evaluated production alternatives such as External Secrets and encrypted remote state backends.

# Reliability Findings

## Missing resource requests and limits
- File: `helm/skybyte-app/templates/deployment.yaml`
- Issue: Containers do not define CPU or memory requests and limits.
- Risk: Pods may consume excessive resources, impact neighboring workloads, and reduce scheduler predictability.
- Fix: Added explicit CPU and memory requests/limits and enforced them through policy validation.

## Weak liveness and readiness probes
- File: `helm/skybyte-app/templates/deployment.yaml`
- Issue: Health probes lack thresholds and timing configuration such as `initialDelaySeconds`, `timeoutSeconds`, and `failureThreshold`.
- Risk: Pods may restart unnecessarily or receive traffic before the application is fully ready.
- Fix: Tuned probe behavior using application-aware thresholds.

## Missing graceful shutdown handling
- File: `helm/skybyte-app/templates/deployment.yaml` and application runtime
- Issue: Deployment lacks `terminationGracePeriodSeconds` and the application does not explicitly handle SIGTERM draining behavior.
- Risk: In-flight requests may be interrupted during rolling deployments or node shutdown events.
- Fix: Added graceful SIGTERM handling and configured sufficient termination grace periods.

## Single replica deployment
- File: `helm/skybyte-app/values.yaml`
- Issue: Application replica count is configured as `1`.
- Risk: A single pod failure causes complete service interruption.
- Fix: Retained a single replica for local Kind/Minikube simplicity while documenting that production environments should use multiple replicas.

## Missing Prometheus metrics endpoint
- File: `app/main.py`
- Issue: Application does not expose Prometheus-compatible metrics.
- Risk: Operators cannot observe request rates, latency, or service health trends.
- Fix: Added `/metrics` endpoint exposing request counters and latency histograms.

# Hygiene Findings

## Mutable image tag (`latest`) used
- File: `helm/skybyte-app/values.yaml`
- Issue: Deployment uses the mutable `latest` image tag.
- Risk: Different environments may pull different image versions, making deployments non-deterministic and difficult to roll back safely.
- Fix: Replaced `latest` with immutable versioned image tags.

## Unsafe image pull policy combined with mutable tag
- File: `helm/skybyte-app/values.yaml`
- Issue: `imagePullPolicy: IfNotPresent` is used together with the mutable `latest` tag.
- Risk: Kubernetes may continue using cached images instead of updated builds.
- Fix: Switched to immutable image tags while retaining predictable image pull behavior.

## Pip cache retained inside image
- File: `Dockerfile`
- Issue: Python dependencies are installed without `--no-cache-dir`.
- Risk: Unnecessary package cache increases image size and attack surface.
- Fix: Updated dependency installation to avoid retaining pip cache artifacts.

## Python linting effectively disabled
- File: `.github/workflows/ci.yml`
- Issue: `flake8` excludes the entire application directory and uses `--exit-zero`.
- Risk: Linting failures never fail the CI pipeline, making validation ineffective.
- Fix: Replaced the linting configuration with strict source validation.

## Helm validation failures intentionally ignored
- File: `.github/workflows/ci.yml`
- Issue: `helm lint` failures are suppressed using `|| true`.
- Risk: Invalid Helm charts still produce successful CI runs.
- Fix: Removed failure suppression and added strict validation behavior.

## Terraform validation failures intentionally ignored
- File: `.github/workflows/ci.yml`
- Issue: `terraform validate` failures are ignored using `|| true`.
- Risk: Invalid infrastructure definitions may reach deployment stages undetected.
- Fix: Enabled strict Terraform validation behavior.

## Kubernetes manifests not schema validated in CI
- File: `.github/workflows/ci.yml`
- Issue: Rendered manifests are not validated against Kubernetes schemas.
- Risk: Invalid manifests may pass CI and fail only during deployment.
- Fix: Added `helm template` and `kubeconform` validation.

## No container or filesystem vulnerability scanning
- File: `.github/workflows/ci.yml`
- Issue: CI pipeline lacks Trivy source and image vulnerability scanning.
- Risk: Vulnerable dependencies and container layers may reach production undetected.
- Fix: Added Trivy filesystem and container image scanning with build failure thresholds.

## No Kyverno policy validation in CI
- File: `.github/workflows/ci.yml`
- Issue: CI pipeline does not validate manifests against admission control policies.
- Risk: Security regressions may bypass enforcement until deployment time.
- Fix: Added Kyverno policy checks against rendered manifests during CI execution.

# Documentation Findings

## Repository documentation outdated
- File: `README.md`
- Issue: Documentation does not accurately reflect deployment flow, security posture, observability setup, or operational validation procedures.
- Risk: Operators may deploy or troubleshoot the application incorrectly.
- Fix: Rewrote the README with updated setup instructions, architecture explanation, SLO definition, and operational guidance.

## Misleading secret rotation guidance
- File: `helm/skybyte-app/values.yaml`
- Issue: Comment suggests secure operational secret rotation while the secret itself is committed directly into source control.
- Risk: Creates false confidence around actual secret management practices.
- Fix: Removed plaintext secret storage and documented proper secret lifecycle handling.