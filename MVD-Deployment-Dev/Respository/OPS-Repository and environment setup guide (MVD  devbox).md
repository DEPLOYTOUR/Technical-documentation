# Repository and environment setup guide (MVD / devbox)

## 1) Expected repository path on the remote machine

The remote deployment assumes the repository is already cloned into a fixed path on the remote host, and it executes scripts from there.

This is explicitly shown in the pipeline.

**File:** `devbox-release-deploy.yml`

**Excerpt (path + remote script):**
- `REPO_DIR="/home/ubuntu/DEPLOYTOUR/hiberus/DEPLOYTOUR-connector"`
- `SCRIPT_PATH="${REPO_DIR}/.github/workflows/scripts/redeploy_version_tag.sh"`

**Implication:**
If the repo is not located at:

- `/home/ubuntu/DEPLOYTOUR/hiberus/DEPLOYTOUR-connector`

the remote redeploy step will fail even if everything else is correct.

---

## 2) GHCR credentials (GitHub Container Registry)

The pipeline authenticates to GHCR using:

- `GHCR_USER`
- `GHCR_TOKEN` (secrets)

**Login (conceptual):**

```bash
echo "${GHCR_TOKEN}" | docker login ghcr.io -u "${GHCR_USER}" --password-stdin
```

### What is needed in practice

- `GHCR_TOKEN`: a PAT with permissions to publish packages (minimum: `packages:write`)
- The host where images are built/tagged must have **Docker** available if it needs to push/pull

---

## 3) Machine requirements (to operate Kubernetes + Terraform)

The devbox “reset/start” is performed with `restart_devbox.sh`, which:

- validates connectivity to the cluster
- (optionally) installs/ensures `ingress-nginx`
- ensures the registry secret
- runs Terraform
- waits for critical jobs

Therefore, the machine needs:

- `kubectl` configured (kubeconfig + valid context)
- `terraform`
- `docker` (if tag/push flow to GHCR is enabled)

---

## 4) Kubernetes image access configuration (imagePullSecret)

In the remote redeploy, it enforces that the namespace has a registry secret and it attaches it to the `default` ServiceAccount.

**Conceptual snippet:**

```bash
CRED="ghcr-cred"

kubectl -n "${NS}" create secret docker-registry "${CRED}"   --docker-server=ghcr.io   --docker-username="${GHCR_USER}"   --docker-password="${GHCR_TOKEN}"

kubectl -n "${NS}" patch serviceaccount default   -p '{"imagePullSecrets":[{"name":"ghcr-cred"}]}'
```

### Critical note (for production)

In devbox it’s practical to patch `serviceaccount/default`, but for production it’s usually better to:

- use dedicated ServiceAccounts per app
- define it declaratively (Helm/Terraform) to avoid drift
- avoid dynamic mutations if strict configuration control is required

---

## 5) Important note: Postgres without bash and adjustment in `db_init.sh`

The script is written for `/bin/sh` (not bash), which avoids failures with minimal Postgres images (e.g., Alpine-based).

**Example:**

```sh
#!/bin/sh
set -eu

psql_base="psql -v ON_ERROR_STOP=1 -h ${DB_FQDN} -U ${POSTGRES_USER}"
```

This fits scenarios where a Job executes scripts and the Postgres image does not include bash.
