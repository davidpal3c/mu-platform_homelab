BR-TASK-0003 — k3s cluster bootstrap (Phase 0C)
===============================================

**Canonical task spec (scope, acceptance, namespace lock):** `tasks/TASK_0003.md` — always prefer that file if this runbook drifts.

**This file:** operator runbook + build-report scaffold for TASK-0003 under `docs/build-reports/`. Namespace names below match the **Ceiba** ontology lock (`ceiba`, `aluna-context`, `aluna-mira`, `aluna-terra`, `observability`). Do **not** use legacy `opencontext-*` names for new bootstrap work.  
**Node:** `aluna-node-01`  
**Dependency:** TASK-0002 complete and verified  
**Builder role boundary:** infrastructure bootstrap only; no workload deployment, no observability stack rollout, no ingress hardening.

---

Part A — Task reference (from Planner)
--------------------------------------

**Objective:** Install and stabilize single-node k3s on `aluna-node-01`, verify storage-path alignment with existing bind mounts, create baseline namespaces, and run a PVC smoke test.

**In-scope:**

- k3s control-plane install.
- kubeconfig access.
- Namespace bootstrap:
  - `ceiba` (Ceiba / control-plane product namespace)
  - `aluna-context`
  - `aluna-mira`
  - `aluna-terra`
  - `observability`
- DNS/networking baseline checks.
- StorageClass + PVC smoke test (local-path).

**Non-scope:**

- Observability stack deployment (TASK-0004).
- TLS/public ingress hardening (TASK-0005).
- Aluna Context API/workload deployment.
- HA/multi-node design.

**Acceptance summary:** Node Ready, core pods healthy, namespaces exist, bind mounts still point to `/platform/*`, PVC binds and test pod mounts it, RAID unchanged.

---

Part B — Operator runbook (execute on aluna-node-01)
------------------------------------------------------

### B.0 Preconditions and safety checks

1) Confirm current phase prerequisites:

```bash
lsblk -f
findmnt /platform /data /backups
findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd
cat /proc/mdstat
```

Expected:

- `/platform`, `/data`, `/backups` mounted.
- `/var/lib/rancher`, `/var/lib/kubelet`, `/var/lib/containerd` backed by `/platform/*` bind mounts.
- `md0`, `md1`, `md2` all `[UU]`.

2) Check swap status (k3s/Kubernetes requirement):

```bash
swapon --show
free -h
```

If any swap is active, disable and persist:

```bash
sudo swapoff -a
sudo sed -i.bak '/\sswap\s/s/^/#/' /etc/fstab
```

3) Update packages and install prerequisites:

```bash
sudo apt update
sudo apt install -y curl ca-certificates
```

4) Ensure bind targets exist before install:

```bash
sudo mkdir -p /var/lib/rancher /var/lib/kubelet /var/lib/containerd
sudo mkdir -p /platform/rancher /platform/kubelet /platform/containerd
sudo mount -a
```

---

### B.1 Install k3s control plane

Use one deterministic install command.

Decision for this task:

- Disable bundled Traefik now (`--disable traefik`) to avoid conflicting ingress posture before TASK-0005.
- Write kubeconfig readable by non-root operator (`--write-kubeconfig-mode 644`).

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server --disable traefik --write-kubeconfig-mode 644" \
  sh -
```

Check service health:

```bash
sudo systemctl status k3s --no-pager
sudo journalctl -u k3s -n 80 --no-pager
```

---

### B.2 kubectl access and cluster sanity

1) Validate kubeconfig path and access:

```bash
ls -l /etc/rancher/k3s/k3s.yaml
kubectl version --client
kubectl get nodes -o wide
```

If `kubectl` is not found in PATH for user shell:

```bash
sudo ln -sf /usr/local/bin/kubectl /usr/bin/kubectl
```

If kubeconfig env var needed explicitly:

```bash
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

2) Verify core system pods:

```bash
kubectl get pods -A -o wide
kubectl get pods -n kube-system
```

Expected minimum healthy components: `coredns`, `local-path-provisioner`, `metrics-server` (if enabled by default in this k3s build), and control-plane components.

---

### B.3 Verify storage-path discipline remains intact

Run after k3s starts to ensure runtime writes remain on `/platform`.

```bash
findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd
df -h /platform /var/lib/rancher /var/lib/kubelet /var/lib/containerd
```

Expected:

- Sources under `/platform/*`.
- No drift of runtime-heavy paths back to boot RAID `/var`.

---

### B.4 Namespace bootstrap

```bash
kubectl create namespace ceiba --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace aluna-context --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace aluna-mira --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace aluna-terra --dry-run=client -o yaml | kubectl apply -f -
kubectl create namespace observability --dry-run=client -o yaml | kubectl apply -f -
kubectl get ns
```

---

### B.5 StorageClass and PVC smoke test (local-path)

1) Verify default storage class:

```bash
kubectl get sc
```

Expected: `local-path` present (usually default in k3s).

2) Apply smoke-test manifest:

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: smoke-pvc
  namespace: aluna-context
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: local-path
---
apiVersion: v1
kind: Pod
metadata:
  name: smoke-writer
  namespace: aluna-context
spec:
  restartPolicy: Never
  containers:
    - name: writer
      image: busybox:1.36
      command: ["/bin/sh", "-c", "echo task-0003-smoke > /data/probe.txt; sleep 20"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: smoke-pvc
EOF
```

3) Verify PVC and pod lifecycle:

```bash
kubectl get pvc,pv -n aluna-context
kubectl get pod smoke-writer -n aluna-context -w
kubectl logs smoke-writer -n aluna-context
```

Expected:

- PVC status `Bound`.
- Pod reaches `Completed`.

4) Cleanup smoke resources:

```bash
kubectl delete pod smoke-writer -n aluna-context --ignore-not-found
kubectl delete pvc smoke-pvc -n aluna-context --ignore-not-found
```

---

### B.6 Final verification checklist

Run and record outputs:

```bash
kubectl get nodes -o wide
kubectl get pods -A -o wide
kubectl get ns
findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd
kubectl get sc
kubectl get pvc,pv -A
cat /proc/mdstat
```

TASK-0003 passes if:

- Node is `Ready`.
- Core pods healthy.
- Namespaces exist.
- Runtime bind mounts remain correct.
- PVC smoke test binds and mounts.
- RAID remains `[UU]`.

---

### B.7 Rollback / recovery notes

If install fails or needs rollback:

1) Uninstall k3s:

```bash
sudo /usr/local/bin/k3s-uninstall.sh
```

2) Verify mounts and RAID are unaffected:

```bash
findmnt /platform /data /backups
findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd
cat /proc/mdstat
```

3) If bind mounts were altered accidentally, restore from TASK-0002 `fstab` backup and remount:

```bash
sudo cp -a /etc/fstab.pre-task-0002 /etc/fstab
sudo mount -a
```

4) Retry install only after prerequisites pass again.

---

Part C — Build report (fill during execution)
---------------------------------------------

### C.1 Goal

Bootstrap single-node k3s control plane on aluna-node-01 while preserving Phase 0B storage-tier isolation and runtime bind mounts.

### C.2 Files and systems touched

- **Node systems:**
  - `k3s` systemd service and binaries.
  - `/etc/rancher/k3s/k3s.yaml` kubeconfig.
  - Kubernetes API resources:
    - namespaces
    - PVC/PV smoke-test resources.
- **Repo files:**
  - `docs/build-reports/BR-TASK-0003.md` (this report).

### C.3 Decisions made

- k3s install mode: single-node server.
- Bundled Traefik disabled at install (`--disable traefik`) pending ingress strategy in TASK-0005.
- Kubeconfig mode set to readable by operator (`--write-kubeconfig-mode 644`).
- Storage smoke test uses `local-path` provisioner (single-node baseline, not HA claim).

### C.4 Commands executed

Record exact command history used on node, including:

- prereq checks (`lsblk`, `findmnt`, `swapon`, `mdstat`)
- k3s install command
- namespace creation commands
- PVC smoke-test apply/verify/cleanup commands
- rollback commands (if any)

### C.5 Validation results

Record outputs/snapshots for:

- `kubectl get nodes -o wide`
- `kubectl get pods -A -o wide`
- `kubectl get ns`
- `findmnt /var/lib/rancher /var/lib/kubelet /var/lib/containerd`
- `kubectl get sc`
- `kubectl get pvc,pv -A`
- `cat /proc/mdstat`

Result status: `PASS` / `REVISE` / `FAIL`.

### C.6 Risks and open questions

- Resource pressure on single-node control plane under future workloads.
- Firewall/network defaults that may affect API access from remote machine.
- Need to document exact API server exposure policy before TASK-0005.
- Confirm whether disabling Traefik aligns with forthcoming ingress-controller choice.

### C.7 Proposed docs/state updates after success

Update:

- `docs/system-blueprint.md`
  - Add k3s control-plane baseline and namespace model.
  - Add storage/runtime path mapping validation outcome.
- `docs/platform-overview.md`
  - Reflect cluster bootstrap status and operational readiness.
- `context/project-context.json`
  - Record cluster baseline state (k3s installed, namespaces created, storage class baseline).
- `tasks/todo.md`
  - Move TASK-0003 to Complete only after review + doc sync.

### C.8 Next task handoff

Next: `TASK-0004` (observability stack baseline), after TASK-0003 is verified and documentation/state sync is complete.

