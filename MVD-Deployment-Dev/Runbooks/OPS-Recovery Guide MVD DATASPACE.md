# Recovery Guide — MVD DATASPACE

## 1) Objective

Stop everything running in `ns=default` in a controlled way, bring up only the base (storage/DB/Vaults and dependencies), recreate Jobs from IaC by running `./restart_devbox.sh`, and finally bring up `authority/consumer/provider`.

### Operating principles

- Do not apply runtime hotfixes (no `kubectl patch` on Jobs “to make it pass”)
- If a critical Job such as `vault-keygen` fails, the correct action is to fix the definition in the repo (variables/secret/template) and reapply
- This procedure is intended for recovery and logical reset, not for “doing magic” on a broken cluster
- Execute carefully: stopping/starting in Kubernetes has edge cases (stuck pods, PVs, old jobs, etc.)

---

## 2) Preparation and context

### 2.1 Validate the current context and set the namespace

```bash
kubectl config current-context
kubectl config set-context --current --namespace=default
kubectl get ns default
```

### 2.2 Initial status (quick snapshot)

```bash
kubectl get deploy,sts,job
kubectl get pods -o wide
```

---

## 3) Controlled shutdown of the default namespace

Scale to 0 to “turn off” without deleting resources (keeps definitions, PVCs, etc.).

### 3.1 Scale Deployments to 0

```bash
kubectl get deploy -o name | xargs -r -I{} kubectl scale {} --replicas=0
```

### 3.2 Scale StatefulSets to 0

```bash
kubectl get statefulset -o name | xargs -r -I{} kubectl scale {} --replicas=0
```

### 3.3 Verify there are no remaining pods

```bash
watch -n 2 'kubectl get pods'
```

When the list is empty (or only system pods remain outside `default`), stop watch with `Ctrl+C`.

---

## 4) Bring up the “base” (without authority/consumer/provider)

Bring up core components and dependencies, excluding anything starting with `authority-`, `consumer-`, `provider-`.

### 4.1 Bring up CORE StatefulSets

```bash
for s in $(kubectl get statefulset -o jsonpath='{.items[*].metadata.name}' \
  | tr ' ' '\n' | egrep -v '^(authority-|consumer-|provider-)'); do
  kubectl scale statefulset/"$s" --replicas=1
done
```

### 4.2 Bring up CORE Deployments

```bash
for d in $(kubectl get deploy -o jsonpath='{.items[*].metadata.name}' \
  | tr ' ' '\n' | egrep -v '^(authority-|consumer-|provider-)'); do
  kubectl scale deploy/"$d" --replicas=1
done
```

### 4.3 Wait for readiness of critical components (adjust list per environment)

```bash
for ss in azurite azurite-report-storage eventhubs postgresql; do
  kubectl rollout status statefulset/"$ss" --timeout=180s || true
done

for d in broker kafka-proxy-provider kafka-proxy-provider-oauth2 kafkacat; do
  kubectl rollout status deploy/"$d" --timeout=180s || true
done
```

---

## 5) Cleanup leftovers (Completed Jobs and Error pods)

### 5.1 Delete Completed Jobs (success)

```bash
NS=default
kubectl -n "$NS" delete job --field-selector=status.successful=1
```

### 5.2 Delete pods in Error state (if any leftovers remain)

```bash
NS=default
kubectl -n "$NS" get pods --no-headers | awk '$3=="Error"{print $1}' \
  | xargs -r kubectl -n "$NS" delete pod --grace-period=0 --force
```

---

## 6) Explicitly bring up Vaults before recreating Jobs

Operational requirement: vault-keygen-type Jobs depend on Vaults being up.

```bash
kubectl scale statefulset authority-vault consumer-vault provider-vault --replicas=1

for ss in authority-vault consumer-vault provider-vault; do
  kubectl rollout status statefulset/"$ss" --timeout=180s || true
done
```

---

## 7) Recreate Jobs from IaC (no patches)

### 7.1 Delete init/keygen/storage Jobs (if present)

```bash
kubectl get jobs -o name \
  | egrep -i 'db-init|vault-keygen|blob|container|storage|azurite' \
  | xargs -r kubectl delete
```

### 7.2 Run the rebuild script

This step must recreate Jobs and wait for completion as implemented in the repo.

```bash
./restart_devbox.sh
```

### 7.3 Verify Jobs completion

```bash
kubectl get jobs

for j in $(kubectl get jobs -o jsonpath='{.items[*].metadata.name}'); do
  kubectl wait job/"$j" --for=condition=complete --timeout=20m || true
done
```

### 7.4 Policy when `vault-keygen` fails (no hotfix)

If `vault-keygen` ends in Error:

- Do **not** run `kubectl patch` to “inject” values on the fly
- Correct action: fix the Job definition in the repo so that:
    - `VAULT_TOKEN` is obtained from the Secret `<role>-vault/rootToken` (or the standard mechanism defined)
    - rerun `./restart_devbox.sh`

This preserves consistency: the “source of truth” must be the repo/IaC, not manual patches.

---

## 8) Bring up authority / consumer / provider

### 8.1 Scale Deployments with those prefixes

```bash
for d in $(kubectl get deploy -o jsonpath='{.items[*].metadata.name}' \
  | tr ' ' '\n' | egrep '^(authority-|consumer-|provider-)'); do
  kubectl scale deploy/"$d" --replicas=1
done
```

### 8.2 Wait for readiness of some critical ones (adjust to the real ones in the environment)

```bash
for d in authority-identityhub consumer-controlplane provider-dataplane; do
  kubectl rollout status deploy/"$d" --timeout=180s || true
done
```

### 8.3 Final health snapshot

```bash
kubectl get pods | egrep 'CrashLoopBackOff|Error|ImagePullBackOff' || true
kubectl get pods -o wide
kubectl get deploy,sts,job
```
