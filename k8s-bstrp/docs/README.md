# Kubernetes Bootstrap Playbook

## Overview

This Ansible playbook bootstraps a vanilla Kubernetes cluster using kubeadm on Fedora 41 VMs running on oVirt. The cluster consists of 1 control plane node and 2 worker nodes on the MGMT_CORE VLAN (10.10.1.0/24).

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    MGMT_CORE VLAN (10.10.1.0/24)            │
│                                                             │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│  │  mgmt-core-01   │  │  mgmt-core-02   │  │  mgmt-core-03   │
│  │  Control Plane  │  │     Worker      │  │     Worker      │
│  │  10.10.1.101    │  │  10.10.1.102    │  │  10.10.1.103    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘
│           │                   │                   │
│           └───────────────────┴───────────────────┘
│                          │
│                    Flannel CNI
│                   (10.244.0.0/16)
└─────────────────────────────────────────────────────────────┘
```

## Components Installed

| Component | Version | Purpose |
|-----------|---------|---------|
| Kubernetes | 1.31.4 | Container orchestration |
| containerd | 1.7.24 | Container runtime |
| Flannel | 0.26.1 | Pod networking (CNI) |
| Helm | 3.16.3 | Package manager |
| metrics-server | latest | Resource metrics |

## Files

```
k8s-bstrp/
├── bootstrap-k8s.yml      # Main playbook (7 phases)
├── inventory.ini          # Node definitions
├── group_vars/all.yml     # Configuration variables
├── ansible.cfg            # Ansible settings
└── docs/
    ├── README.md          # This file
    └── QA.md              # Questions & Answers
```

## Usage

```bash
cd /root/cluster/k8s-bstrp

# Full cluster bootstrap
ansible-playbook -i inventory.ini bootstrap-k8s.yml

# Run specific phases
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags prereq
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags runtime
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags k8s-install
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags init-cluster
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags join-workers
ansible-playbook -i inventory.ini bootstrap-k8s.yml --tags post-install
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

## Network Configuration

| Network | CIDR | Purpose |
|---------|------|---------|
| Node Network | 10.10.1.0/24 | MGMT_CORE VLAN |
| Pod Network | 10.244.0.0/16 | Flannel overlay |
| Service Network | 10.96.0.0/12 | ClusterIP services |

## Credentials

| Access | Username | Password |
|--------|----------|----------|
| Node SSH | root | unix |
| Node SSH | admin | unix |

## Kubeconfig

The kubeconfig is saved to `/root/cluster/kubeconfig` after deployment:

```bash
export KUBECONFIG=/root/cluster/kubeconfig
kubectl get nodes
```
