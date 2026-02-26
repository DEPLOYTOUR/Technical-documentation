# Pipeline guide: `devbox-release-deploy.yml`

## 1) When it runs

This workflow runs when:

- A **Release is published**: `release: types: [published]`
- Or it is run manually: `workflow_dispatch` with an optional `version` input

> Note: in the provided YAML there is **no** `concurrency` configuration.

**Repo path:** `.github/workflows/devbox-release-deploy.yml`

---

## 2) Relevant global variables

Defined under `env:` in the workflow:

- **REGISTRY**: `ghcr.io/deploytour`
- **NAMESPACE**: `default`
- **DEVBOX_COMPONENTS**: multi-line list of components/images that get tagged/promoted

---

## 3) Pipeline jobs

### Job A — `build_publish_promote`

**Goal:** build artifacts, build images, publish `${VERSION}` to GHCR, and promote `${VERSION} -> latest`.

**Steps:**
1. **Checkout**
2. **Resolve VERSION**
   - Uses `github.event.release.tag_name` (Release) or `workflow_dispatch.inputs.version` (manual).
   - The script writes `version=<...>` to `GITHUB_OUTPUT`.
3. **GHCR login**
   - `docker login ghcr.io` using `GHCR_USER/GHCR_TOKEN`.
4. **Build jars**
   - `./gradlew shadowJar`
5. **Build Docker images**
   - `./gradlew dockerize`
6. **Tag & push VERSION**
   - For each component in `DEVBOX_COMPONENTS`: tags local `img:latest` as `ghcr.io/deploytour/img:${VERSION}` and pushes it.
7. **Promote VERSION -> latest**
   - For each component: pulls `ghcr.io/deploytour/img:${VERSION}`, tags it as `:latest`, and pushes.

**Job output:** exposes `version` for the next job.

**Operational notes:**
- `Tag & push VERSION` assumes local images `"<img>:latest"` exist (built by `gradlew dockerize`).
- If `DEVBOX_COMPONENTS` contains an invalid entry (e.g., `all`), it will fail with “No such image …”.

---

### Job B — `redeploy_version`

**Goal:** redeploy on a remote machine via SSH, using the repo already cloned on that machine.

**Steps:**
1. **Checkout** (only for the runner; the real deploy is remote)
2. **SSH (appleboy/ssh-action)** and execute on the remote host:
   - `REPO_DIR="/home/ubuntu/DEPLOYTOUR-connector"`
   - `SCRIPT_PATH="${REPO_DIR}/.github/workflows/scripts/redeploy_devbox_terraform.sh"`
   - `chmod +x` and run the script.

The workflow forwards these env vars to the remote script via `envs:`:
`DEVBOX_COMPONENTS,NAMESPACE,VERSION,REGISTRY,GHCR_USER,GHCR_TOKEN`.

---

## 4) What the remote redeploy does (`redeploy_devbox_terraform.sh`)

**Technical summary:**
- Validates `kubectl`, `terraform`, and kubeconfig.
- Exports `TF_VAR_*` variables and runs:
  - `terraform -chdir=system-tests init -upgrade`
  - `terraform -chdir=system-tests apply -auto-approve -var="image_tag=${VERSION}"`
- Then prints:
  - deployments and their images in `NAMESPACE` (`default`)
  - pods in `default`
  - and filters problematic states (ImagePullBackOff, CrashLoop, etc.)

**Key point:** this redeploy does **not** patch deployments manually nor force `imagePullPolicy`. The actual change must be applied through Terraform (whatever is defined under `system-tests`).

---

## 5) Versioning: how the version is actually applied

### 5.1 What the pipeline does today

- Publishes images with immutable tag `${VERSION}` in GHCR for the listed components.
- Promotes `${VERSION}` to `latest` in GHCR.
- Deploys through Terraform, forcing `-var="image_tag=${VERSION}"` (high priority vs `terraform.tfvars`).

### 5.2 Practical evaluation

- For devbox/MVD environments, keeping `latest` can be acceptable, but `${VERSION}` should be the source of truth.
- The IaC deployment (Terraform) is already aligned with this by passing `image_tag` via CLI.

---

## 6) Flow (text)

1. Release published **or** workflow_dispatch
2. `build_publish_promote`
   - resolve_version → `VERSION`
   - ghcr_login
   - gradle shadowJar
   - gradle dockerize
   - tag_push_version: `img:latest` → `ghcr.io/deploytour/img:${VERSION}`
   - promote_version_to_latest: `ghcr.io/deploytour/img:${VERSION}` → `:latest`
3. `redeploy_version`
   - SSH to remote host
   - run `redeploy_devbox_terraform.sh`
   - terraform apply with `-var="image_tag=${VERSION}"` and print deployment state
