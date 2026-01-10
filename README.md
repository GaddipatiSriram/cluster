# Cluster Services

Ansible playbooks for deploying Kubernetes clusters and core services (CoreDNS, monitoring) on oVirt VMs.

## Overview

This project contains automation for:
- **k8s-bstrp**: Kubernetes cluster bootstrap using kubeadm
- **coredns**: CoreDNS deployment for internal DNS resolution

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     Mgmt-Core Kubernetes Cluster                     │
│                        (10.10.1.0/24)                               │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐                 │
│  │ mgmt-core-01│  │ mgmt-core-02│  │ mgmt-core-03│                 │
│  │ 10.10.1.101 │  │ 10.10.1.102 │  │ 10.10.1.103 │                 │
│  │             │  │             │  │             │                 │
│  │ Control     │  │ Control     │  │ Control     │                 │
│  │ Plane +     │  │ Plane +     │  │ Plane +     │                 │
│  │ Worker      │  │ Worker      │  │ Worker      │                 │
│  └─────────────┘  └─────────────┘  └─────────────┘                 │
│                                                                      │
│  Services:                                                          │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                       CoreDNS                                 │   │
│  │                    10.10.1.200 (VIP)                         │   │
│  │                                                               │   │
│  │  Resolves:                                                    │   │
│  │  - *.engatwork.com → Internal services                       │   │
│  │  - api.okd.engatwork.com → 10.10.2.50                        │   │
│  │  - *.apps.okd.engatwork.com → 10.10.2.51                     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

## Project Structure

```
cluster/
├── README.md
├── .gitignore
├── coredns/                 # CoreDNS deployment
│   ├── ansible.cfg
│   ├── inventory.ini
│   ├── deploy-coredns.yml   # Main deployment playbook
│   ├── group_vars/
│   │   ├── all.yml          # DNS records configuration
│   │   └── okd.yml          # OKD-specific DNS records
│   ├── playbooks/
│   │   └── update-okd-dns.yml  # Update OKD DNS records
│   └── docs/
└── k8s-bstrp/               # Kubernetes bootstrap
    ├── ansible.cfg
    ├── inventory/
    │   └── hosts.ini
    ├── group_vars/
    │   └── all.yml          # Cluster configuration
    ├── bootstrap-k8s.yml    # Main bootstrap playbook
    ├── update-flannel.yml   # Flannel CNI update
    ├── tasks/
    └── docs/
```

## Quick Start

### 1. Bootstrap Kubernetes Cluster

```bash
cd k8s-bstrp

# Configure inventory
vi inventory/hosts.ini

# Configure cluster settings
vi group_vars/all.yml

# Bootstrap cluster
ansible-playbook bootstrap-k8s.yml
```

### 2. Deploy CoreDNS

```bash
cd coredns

# Configure DNS records
vi group_vars/all.yml

# Deploy CoreDNS
ansible-playbook deploy-coredns.yml
```

## Subprojects

### [k8s-bstrp](k8s-bstrp/)

Kubernetes cluster bootstrap using kubeadm with:
- HA control plane (3 nodes)
- Flannel CNI networking
- MetalLB load balancer
- Local storage provisioner

### [coredns](coredns/)

CoreDNS deployment for internal DNS:
- Kubernetes-hosted CoreDNS
- Custom zone files for internal domains
- OKD cluster DNS integration
- MetalLB VIP for DNS service

## DNS Configuration

CoreDNS provides DNS resolution for:

| Record | IP | Purpose |
|--------|-----|---------|
| `*.engatwork.com` | Various | Internal services |
| `api.mgmt-devops-okd.engatwork.com` | 10.10.2.50 | OKD API |
| `*.apps.mgmt-devops-okd.engatwork.com` | 10.10.2.51 | OKD Ingress |

### Update OKD DNS Records

```bash
cd coredns
ansible-playbook playbooks/update-okd-dns.yml \
  -e okd_cluster_name=mgmt-devops-okd \
  -e okd_api_vip=10.10.2.50 \
  -e okd_ingress_vip=10.10.2.51 \
  -e okd_dns_state=present
```

## Kubernetes Cluster Details

| Setting | Value |
|---------|-------|
| K8s Version | 1.28+ |
| CNI | Flannel |
| Load Balancer | MetalLB |
| DNS VIP | 10.10.1.200 |
| Pod CIDR | 10.244.0.0/16 |
| Service CIDR | 10.96.0.0/12 |

## Testing DNS

```bash
# From any VM in the network
dig @10.10.1.200 api.mgmt-devops-okd.engatwork.com

# Test wildcard
dig @10.10.1.200 console.apps.mgmt-devops-okd.engatwork.com
```

## Troubleshooting

### Kubernetes Issues
```bash
# Check nodes
kubectl get nodes

# Check system pods
kubectl get pods -n kube-system

# Check CoreDNS pods
kubectl get pods -n coredns
```

### CoreDNS Issues
```bash
# Check CoreDNS logs
kubectl logs -n coredns -l app=coredns

# Check ConfigMap
kubectl get configmap -n coredns coredns -o yaml
```

## Requirements

- Rocky Linux 9 VMs on oVirt
- Ansible 2.14+
- Python 3.9+
- kubectl configured for cluster access

## License

MIT
