# Kubernetes Bootstrap Playbook

Ansible playbook for bootstrapping a vanilla Kubernetes cluster using kubeadm on Rocky Linux/Fedora VMs.

## Overview

This playbook creates a production-ready Kubernetes cluster with:
- **kubeadm** - Cluster initialization
- **containerd** - Container runtime
- **Flannel** - Pod networking (CNI)
- **Helm** - Package manager
- **metrics-server** - Resource metrics

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    MGMT_CORE VLAN (10.10.1.0/24)                │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  mgmt-core-01   │  │  mgmt-core-02   │  │  mgmt-core-03   │  │
│  │  Control Plane  │  │  Control Plane  │  │  Control Plane  │  │
│  │  + Worker       │  │  + Worker       │  │  + Worker       │  │
│  │  10.10.1.101    │  │  10.10.1.102    │  │  10.10.1.103    │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│           │                   │                   │              │
│           └───────────────────┴───────────────────┘              │
│                          │                                       │
│                    Flannel CNI                                   │
│                   (10.244.0.0/16)                                │
└─────────────────────────────────────────────────────────────────┘
```

## Quick Start

```bash
# Full cluster bootstrap
ansible-playbook bootstrap-k8s.yml

# Run specific phases
ansible-playbook bootstrap-k8s.yml --tags prereq
ansible-playbook bootstrap-k8s.yml --tags runtime
ansible-playbook bootstrap-k8s.yml --tags k8s-install
ansible-playbook bootstrap-k8s.yml --tags init-cluster
ansible-playbook bootstrap-k8s.yml --tags cni
ansible-playbook bootstrap-k8s.yml --tags join-workers
ansible-playbook bootstrap-k8s.yml --tags post-install
```

## Project Structure

```
k8s-bstrp/
├── ansible.cfg           # Ansible configuration
├── bootstrap-k8s.yml     # Main playbook (7 phases)
├── update-flannel.yml    # Flannel CNI update playbook
├── inventory/
│   └── hosts.ini         # Node definitions
├── group_vars/
│   └── all.yml           # Cluster configuration
├── tasks/                # Reusable task files
├── vars/                 # Additional variables
└── docs/
    ├── README.md         # Detailed documentation
    └── QA.md             # Questions & Answers
```

## Configuration

Edit `group_vars/all.yml`:

```yaml
# Kubernetes version
k8s_version: "1.31"

# Network configuration
pod_network_cidr: "10.244.0.0/16"
service_cidr: "10.96.0.0/12"

# Control plane endpoint
control_plane_endpoint: "10.10.1.101:6443"
```

## Playbook Phases

| Phase | Tag | Description |
|-------|-----|-------------|
| 1 | `prereq` | Configure prerequisites (swap, kernel modules, sysctl) |
| 2 | `runtime` | Install containerd and CNI plugins |
| 3 | `k8s-install` | Install kubeadm, kubelet, kubectl |
| 4 | `init-cluster` | Initialize control plane with kubeadm |
| 5 | `cni` | Install Flannel network plugin |
| 6 | `join-workers` | Join worker nodes to cluster |
| 7 | `post-install` | Install Helm, metrics-server |

## Components Installed

| Component | Version | Purpose |
|-----------|---------|---------|
| Kubernetes | 1.31.x | Container orchestration |
| containerd | 1.7.x | Container runtime |
| Flannel | 0.26.x | Pod networking (CNI) |
| Helm | 3.16.x | Package manager |
| metrics-server | latest | Resource metrics |

## Network Configuration

| Network | CIDR | Purpose |
|---------|------|---------|
| Node Network | 10.10.1.0/24 | MGMT_CORE VLAN |
| Pod Network | 10.244.0.0/16 | Flannel overlay |
| Service Network | 10.96.0.0/12 | ClusterIP services |

## Accessing the Cluster

After deployment, kubeconfig is saved locally:

```bash
# Set kubeconfig
export KUBECONFIG=/root/learning/cluster/kubeconfig

# Verify cluster
kubectl get nodes
kubectl get pods -A
```

## Verification

```bash
# Check nodes
kubectl get nodes -o wide

# Check system pods
kubectl get pods -n kube-system

# Check Flannel
kubectl get pods -n kube-flannel

# Check metrics
kubectl top nodes
```

## Troubleshooting

```bash
# Check kubelet logs
journalctl -u kubelet -f

# Check containerd
systemctl status containerd

# Reset cluster (start over)
kubeadm reset -f
```

## Documentation

See [docs/README.md](docs/README.md) for detailed documentation.
