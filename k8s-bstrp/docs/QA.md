# Kubernetes Bootstrap - Questions & Answers

## General

### Q: Why kubeadm instead of k3s or other distributions?

**A:** This is a vanilla Kubernetes installation using the official kubeadm tool. Benefits:
- Standard upstream Kubernetes (no vendor lock-in)
- Full control over components
- Matches production environments
- Official documentation applies directly

### Q: Why Flannel instead of Calico or Cilium?

**A:** Flannel was chosen for simplicity:
- Lightweight VXLAN-based networking
- Simple configuration
- Works well for lab environments
- Can be changed by setting `cni_plugin: "calico"` in variables

### Q: Does Flannel work like a L3 router or L2 switch?

**A:** Flannel is a **Layer 3 overlay network** using VXLAN encapsulation. It's neither a pure router nor switch - it creates a flat L2 network from the pod's perspective while using L3 tunnels between nodes.

**Architecture:**
```
┌─────────────────────────────────────────────────────────────────┐
│                    Flannel Overlay Network                       │
│                     (e.g., 10.245.0.0/16)                       │
├─────────────────────────────────────────────────────────────────┤
│   Node 1 (10.10.2.100)              Node 2 (10.10.2.101)        │
│   ┌─────────────────┐               ┌─────────────────┐         │
│   │ Subnet: 10.245.0.0/24           │ Subnet: 10.245.1.0/24     │
│   │  ┌─────┐ ┌─────┐│               │┌─────┐ ┌─────┐  │         │
│   │  │Pod A│ │Pod B││               ││Pod C│ │Pod D│  │         │
│   │  │.10  │ │.11  ││               ││.10  │ │.11  │  │         │
│   │  └──┬──┘ └──┬──┘│               │└──┬──┘ └──┬──┘  │         │
│   │     └───┬───┘   │               │   └───┬───┘     │         │
│   │      cni0      │               │     cni0       │         │
│   │    (bridge)     │               │   (bridge)     │         │
│   │    flannel.1    │               │  flannel.1     │         │
│   │    (VXLAN)      │               │  (VXLAN)       │         │
│   └────────┬────────┘               └───────┬────────┘         │
│            └──────────── UDP:8472 ──────────┘                   │
│                     (VXLAN Encapsulation)                       │
└─────────────────────────────────────────────────────────────────┘
```

**Traffic flow:**

| Scenario | Path | Method |
|----------|------|--------|
| Same node | Pod A → cni0 bridge → Pod B | L2 switching (no encap) |
| Cross node | Pod A → cni0 → flannel.1 → VXLAN → Node 2 → Pod C | L3 VXLAN tunnel |

**VXLAN encapsulation (cross-node):**
```
Original packet:  [Src: 10.245.0.10 → Dst: 10.245.1.10]
                              ↓
Encapsulated:     [Outer IP: Node1 → Node2] [UDP:8472] [VXLAN] [Original packet]
```

**Summary:**
- **Pod's view**: Flat L2 network - all pods appear on same LAN
- **Implementation**: L3 overlay using VXLAN (UDP encapsulation)
- **Same node**: L2 bridge (cni0) - no encapsulation
- **Cross node**: L3 routing + VXLAN tunnel
- **Subnet allocation**: Each node gets /24 from cluster /16

### Q: Why containerd instead of Docker?

**A:** containerd is the recommended container runtime for Kubernetes:
- Docker support was removed in K8s 1.24+
- containerd is lighter weight
- Direct CRI integration (no dockershim)

---

## Configuration

### Q: Why was swap disabled?

**A:** Kubernetes requires swap to be disabled because:
- kubelet fails to start with swap enabled
- Memory limits become unpredictable with swap
- Fedora 41 uses zram swap which required special handling (blacklisting zram module)

### Q: How do I add more worker nodes?

**A:** Edit `inventory.ini`:
```ini
[k8s_workers]
mgmt-core-02    ansible_host=10.10.1.102    k8s_role=worker
mgmt-core-03    ansible_host=10.10.1.103    k8s_role=worker
mgmt-core-04    ansible_host=10.10.1.104    k8s_role=worker  # New node
```
Then run:
```bash
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags prereq,runtime,k8s-install,join-workers
```

### Q: How do I change the pod network CIDR?

**A:** Edit `group_vars/all.yml` before running the playbook:
```yaml
pod_network_cidr: "10.244.0.0/16"  # Change this
```
Note: Cannot be changed after cluster initialization.

### Q: How do I upgrade Kubernetes version?

**A:** Edit `group_vars/all.yml`:
```yaml
k8s_version: "1.32"
k8s_full_version: "1.32.0"
```
Then follow the official kubeadm upgrade procedure.

---

## Troubleshooting

### Q: Why did get_url fail with SSL errors?

**A:** Fedora 41's Python has SSL compatibility issues with the Ansible `get_url` module. The playbook uses `curl` commands instead for downloading containerd, CNI plugins, and Helm.

### Q: Kubelet keeps failing with "swap on" error?

**A:** Fedora 41 uses zram swap which recreates on boot. The playbook:
1. Disables swap with `swapoff -a`
2. Masks zram systemd services
3. Blacklists the zram kernel module

If it persists, manually run:
```bash
swapoff -a
systemctl mask swap-create@zram0.service
echo "blacklist zram" > /etc/modprobe.d/zram-blacklist.conf
```

### Q: How do I reset the cluster?

**A:** On each node:
```bash
kubeadm reset -f
rm -rf /etc/kubernetes /var/lib/kubelet /var/lib/etcd /etc/cni/net.d
```

### Q: Worker node won't join - token expired?

**A:** Generate a new join command on control plane:
```bash
kubeadm token create --print-join-command
```

### Q: Pods stuck in "Pending" state?

**A:** Check if Flannel is running:
```bash
kubectl get pods -n kube-flannel
```
If not, reinstall CNI:
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/download/v0.26.1/kube-flannel.yml
```

---

## Access & Security

### Q: Where is the kubeconfig?

**A:**
- Ansible controller: `/root/cluster/kubeconfig`
- Control plane node: `/etc/kubernetes/admin.conf`

```bash
export KUBECONFIG=/root/cluster/kubeconfig
kubectl get nodes
```

### Q: How do I access the cluster from another machine?

**A:** Copy the kubeconfig:
```bash
scp root@10.10.1.101:/etc/kubernetes/admin.conf ~/.kube/config
```
Update the server URL if needed:
```bash
sed -i 's/127.0.0.1/10.10.1.101/' ~/.kube/config
```

### Q: How do I create additional users?

**A:** Use kubeadm to create certificates or set up RBAC:
```bash
kubectl create serviceaccount developer -n default
kubectl create clusterrolebinding developer-binding --clusterrole=edit --serviceaccount=default:developer
```
