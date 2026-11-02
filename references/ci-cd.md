# CI/CD & Pipeline Review Lens

Applies to `.gitlab-ci.yml`, GitHub Actions workflows, Jenkinsfiles, Dockerfiles, deploy scripts, and infrastructure-as-code. The organising question: **can this pipeline reliably stop bad code from reaching production, and does it do so fast enough that people don't route around it?**

## Contents

1. Gating and correctness
2. Speed and feedback loops
3. Secrets and supply chain
4. Containers and images
5. Deployment and recovery
6. Infrastructure as code
7. Maintainability

---

## 1. Gating and correctness

- **Failures that don't fail the build.** `|| true`, `continue-on-error: true`, `allow_failure: true`, `set +e`, or a test step whose exit code is swallowed by a wrapper script. A quality gate that cannot go red is theatre — this is typically Critical, since the team believes they are protected and they are not.
- **What actually gates a merge.** Trace it: are tests required to pass before merge, or do they run after? Are they required on the target branch or only on the source? Protected-branch settings are part of the review even though they live outside the repo — if they can't be inspected, list it as an open question rather than assuming.
- **Coverage and quality thresholds** that are set but not enforced, or ratcheted down over time to keep builds green.
- **Missing steps entirely** — no linting, no dependency vulnerability scan, no build verification before deploy.
- **Deploy jobs that can run from any branch**, or manual deploy jobs with no approval requirement.

## 2. Speed and feedback loops

Pipeline duration determines whether the pipeline gets used or worked around.

- **No dependency caching.** Reinstalling `node_modules`, pip packages, or Maven artifacts on every run typically wastes several minutes per job. Check cache keys are correct — a cache key that never invalidates serves stale dependencies, and one keyed on something too volatile never hits.
- **Serial stages that could be parallel.** Lint, unit tests, and type checks have no dependency on each other and should run concurrently.
- **Fail-slow ordering.** Fast checks (lint, typecheck, unit) belong before slow ones (E2E, build, deploy) so feedback arrives in a minute rather than twenty.
- **Repeated work across stages** — building the application in the test stage and again in the deploy stage, rather than building once and passing an artifact forward.
- **Full pipeline on every change** in a monorepo where path-based rules could skip untouched services.
- **Docker layer ordering** that busts the cache unnecessarily (see below).

## 3. Secrets and supply chain

- **Hardcoded secrets** in pipeline files, scripts, or committed `.env` files. Critical, always. Check git history too — a secret removed in a later commit is still exposed and needs rotation, not just deletion.
- **Secrets echoed into logs** by `set -x`, debug flags, or a curl command with credentials in the URL.
- **Unpinned third-party actions.** `uses: some/action@main` executes whatever that repository contains at run time, with access to the pipeline's secrets. Pin to a commit SHA.
- **Unpinned base images.** `FROM node:latest` makes builds non-reproducible and silently changes the runtime.
- **No dependency scanning** — no `npm audit`, `pip-audit`, Dependabot, Renovate, or equivalent. Also check whether a lockfile exists and is committed.
- **Overly broad permissions** — pipeline tokens or cloud roles with write access far beyond what the job needs.

## 4. Containers and images

- **Running as root.** No `USER` directive means the container runs privileged by default.
- **Single-stage builds shipping build tooling.** Compilers, dev dependencies, and source in the production image enlarge the attack surface and the image. Multi-stage builds fix this.
- **Cache-hostile layer order.** Copying the entire source before installing dependencies invalidates the dependency layer on every code change:

```dockerfile
# Dependency layer rebuilds on any source change
COPY . .
RUN npm ci

# Dependency layer is cached until the manifest changes
COPY package*.json ./
RUN npm ci
COPY . .
```

- **No `.dockerignore`**, pulling `.git`, `node_modules`, and local env files into build context.
- **No healthcheck** on long-running service images.

## 5. Deployment and recovery

- **No rollback path.** If a deploy goes wrong, what is the documented recovery? "Redeploy the previous commit" only works if the previous artifact still exists and migrations are reversible.
- **Irreversible database migrations** run automatically as part of deploy, with no separation between expand and contract phases.
- **No smoke test after deploy** — the pipeline reports success on "the deploy command exited zero" rather than "the service is actually serving traffic".
- **Deploy scripts that aren't idempotent**, so a retry after partial failure leaves inconsistent state.
- **No environment parity** — staging differing from production in ways that make staging results uninformative.

## 6. Infrastructure as code

- **State file handling** — local Terraform state, or remote state without locking, which allows concurrent applies to corrupt it.
- **Manual drift.** Resources created by hand in the console and never brought under code; look for signs like resources referenced but not defined.
- **Plan not reviewed before apply**, or apply running automatically on merge without a human seeing the plan output for destructive changes.
- **Secrets in variable files** committed to the repo.
- **Copy-pasted environment definitions** where a module with parameters would prevent staging and production from drifting apart.

## 7. Maintainability

- **Long inline shell blocks** in YAML — hard to test, lint, or run locally. Extracting to a versioned script in the repo makes the pipeline debuggable.
- **Duplication across jobs** that anchors, extends, or reusable workflows would eliminate.
- **No way to run pipeline steps locally**, forcing "commit to test" cycles that pollute history.
- **Magic values** — untagged runner selectors, unexplained timeouts, environment names hardcoded in many places.
- **No pipeline observability** — nobody tracks pipeline success rate or duration over time, so degradation goes unnoticed until it's severe.
