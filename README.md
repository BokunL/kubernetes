# Kubernetes Single-Node Setup with kubeadm

## Prerequisites

### Extend disk partition and filesystem
```bash
sudo growpart /dev/sda 2
sudo resize2fs /dev/sda2
```
> Extends the partition to use all unallocated disk space, then resizes the ext4 filesystem to fill it — no reboot needed.

---

## 1. Disable Swap
```bash
sudo swapoff -a
sudo sed -i '/\sswap\s/s/^/#/' /etc/fstab
```
> Kubernetes requires swap to be off. The `sed` command comments out the swap entry in `/etc/fstab` so it stays disabled after reboot.

---

## 2. Load Kernel Modules
```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
```
> `overlay` is used by containerd for container filesystems. `br_netfilter` enables iptables rules to see bridged traffic. The config file persists them across reboots.

---

## 3. Configure Sysctl Networking
```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```
> Allows iptables to process bridged IPv4/IPv6 traffic and enables IP forwarding — both required for Kubernetes pod networking.

---

## 4. Install and Configure containerd
```bash
sudo apt-get update -q && sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```
> Installs containerd as the container runtime. `SystemdCgroup = true` is required so containerd and kubelet use the same cgroup driver.

---

## 5. Install kubeadm, kubelet, kubectl
```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update -q && sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
> Adds the official Kubernetes apt repository and installs v1.32. `apt-mark hold` prevents unintended upgrades.

---

## 6. Initialize the Cluster
```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```
> Bootstraps the control plane. The `--pod-network-cidr` matches Flannel's default range.

---

## 7. Set Up kubeconfig
```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
> Copies the admin kubeconfig to your home directory so `kubectl` works as a regular user.

---

## 8. Install Flannel CNI
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```
> Deploys the Flannel network plugin so pods can communicate across the cluster.

---

## 9. Untaint Control-Plane (single-node)
```bash
kubectl taint nodes --all node-role.kubernetes.io/control-plane-
```
> By default, the control-plane node rejects workload pods. This removes the taint so regular pods can be scheduled on the single node.

---

## Verify
```bash
kubectl get nodes
kubectl get pods -A
```
