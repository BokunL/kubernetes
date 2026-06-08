# Kubernetes Upgrade: v1.32.13 → v1.33.12

**Date:** 2026-06-08  
**Node:** `bokun-vmware-virtual-platform` (single-node control-plane)  
**From:** `v1.32.13`  
**To:** `v1.33.12`

1. Update apt source  →  change v1.XX to v1.XX+1 in kubernetes.list
2. apt install kubeadm  →  upgrade kubeadm first
3. kubeadm upgrade apply  →  upgrades control plane static pods (apiserver, etcd, scheduler, controller-manager)
4. kubectl drain  →  evict pods from node
5. apt install kubelet kubectl  →  upgrade the node-level binaries
6. systemctl restart kubelet  →  pick up new version
7. kubectl uncordon  →  bring node back into scheduling
---

## Steps Performed

### 1. Add v1.33 apt Repository

Replaced the existing v1.32 Kubernetes package source with the v1.33 source:

```bash
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /" \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
```

> The existing GPG keyring at `/etc/apt/keyrings/kubernetes-apt-keyring.gpg` is shared across minor versions — no new key needed.

---

### 2. Upgrade kubeadm

```bash
sudo apt-mark unhold kubeadm
sudo apt-get install -y kubeadm=1.33.12-1.1
sudo apt-mark hold kubeadm
```

Verify:
```bash
kubeadm version
```

---

### 3. Upgrade Control Plane Components

```bash
sudo kubeadm upgrade apply v1.33.12 --yes
```

This upgraded the following static pod components:
- `etcd` (certificates renewed)
- `kube-apiserver` (certificate renewed)
- `kube-controller-manager` (certificate renewed)
- `kube-scheduler` (certificate renewed)
- `CoreDNS`
- `kube-proxy`

The `kubeadm-config` ConfigMap in `kube-system` was also updated to reflect the new version.

---

### 4. Drain the Node

Evict all non-DaemonSet pods before upgrading kubelet:

```bash
kubectl drain bokun-vmware-virtual-platform --ignore-daemonsets --delete-emptydir-data
```

DaemonSets (flannel, kube-proxy) are ignored and stay running.

---

### 5. Upgrade kubelet and kubectl

```bash
sudo apt-mark unhold kubelet kubectl
sudo apt-get install -y kubelet=1.33.12-1.1 kubectl=1.33.12-1.1
sudo apt-mark hold kubelet kubectl
```

Restart kubelet to pick up the new version:

```bash
sudo systemctl daemon-reload
sudo systemctl restart kubelet
```

---

### 6. Uncordon the Node

Bring the node back into scheduling rotation:

```bash
kubectl uncordon bokun-vmware-virtual-platform
```

---

## Verification

```bash
kubectl get nodes
```

Expected output:
```
NAME                            STATUS   ROLES           AGE   VERSION
bokun-vmware-virtual-platform   Ready    control-plane   ...   v1.33.12
```

---

## Notes

- All three binaries (`kubeadm`, `kubelet`, `kubectl`) are held via `apt-mark hold` to prevent accidental upgrades.
- Kubernetes upgrades must be done **one minor version at a time** (e.g., 1.32 → 1.33, not 1.32 → 1.34).
- For future upgrades, repeat this process: update the apt source URL to the next minor version and follow the same steps.
- CNI (Flannel) did not require changes — it is compatible with v1.33.
