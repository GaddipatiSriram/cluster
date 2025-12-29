# CoreDNS External DNS Server

## Overview

This Ansible playbook deploys CoreDNS as an external DNS server on the Kubernetes cluster, providing DNS resolution for the `engatwork.com` domain across all VLANs. It uses MetalLB to expose CoreDNS with a stable LoadBalancer IP.

## Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                         DNS Resolution Flow                          │
│                                                                      │
│  ┌─────────┐     ┌─────────────┐     ┌─────────────────────────┐    │
│  │   VMs   │────▶│   pfSense   │────▶│  CoreDNS (10.10.1.200)  │    │
│  │ (VLANs) │     │ DNS Forward │     │   on K8s cluster        │    │
│  └─────────┘     └─────────────┘     └───────────┬─────────────┘    │
│                                                   │                  │
│                        ┌──────────────────────────┴──────────┐       │
│                        │                                     │       │
│                        ▼                                     ▼       │
│              ┌─────────────────┐                  ┌─────────────────┐│
│              │ engatwork.com   │                  │  External DNS   ││
│              │ (hosts file)    │                  │  (8.8.8.8)      ││
│              └─────────────────┘                  └─────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

## Components

| Component | Version | Purpose |
|-----------|---------|---------|
| MetalLB | 0.14.8 | LoadBalancer for bare-metal K8s |
| CoreDNS | 1.11.3 | DNS server |
| IP Pool | 10.10.1.200-220 | LoadBalancer IP range |

## Files

```
coredns/
├── deploy-coredns.yml     # Main deployment playbook
├── inventory.ini          # Control plane target
├── group_vars/all.yml     # DNS records and configuration
├── ansible.cfg            # Ansible settings
└── docs/
    ├── README.md          # This file
    └── QA.md              # Questions & Answers
```

## Usage

```bash
cd /root/cluster/coredns

# Full deployment (MetalLB + CoreDNS)
ansible-playbook -i inventory.ini deploy-coredns.yml

# Deploy only MetalLB
ansible-playbook -i inventory.ini deploy-coredns.yml --tags metallb

# Deploy only CoreDNS
ansible-playbook -i inventory.ini deploy-coredns.yml --tags coredns

# Verify DNS
ansible-playbook -i inventory.ini deploy-coredns.yml --tags verify
```

## DNS Records

Current records configured in `group_vars/all.yml`:

| Hostname | IP | Description |
|----------|-----|-------------|
| pfsense.engatwork.com | 192.168.0.101 | pfSense firewall |
| ovirt.engatwork.com | 192.168.0.105 | oVirt Engine |
| k8s.engatwork.com | 10.10.1.101 | K8s API endpoint |
| mgmt-core-01.engatwork.com | 10.10.1.101 | Control plane |
| mgmt-core-02.engatwork.com | 10.10.1.102 | Worker 1 |
| mgmt-core-03.engatwork.com | 10.10.1.103 | Worker 2 |
| dns.engatwork.com | 10.10.1.200 | CoreDNS itself |

## Adding New DNS Records

Edit `group_vars/all.yml`:

```yaml
dns_records:
  - ip: "10.10.2.101"
    hostnames:
      - "gitlab"
      - "git"
  - ip: "10.10.3.101"
    hostnames:
      - "prometheus"
      - "monitoring"
```

Then re-run:
```bash
ansible-playbook -i inventory.ini deploy-coredns.yml --tags coredns
```

## Network Configuration

| Component | IP/Range | Purpose |
|-----------|----------|---------|
| CoreDNS LB | 10.10.1.200 | DNS server endpoint |
| MetalLB Pool | 10.10.1.200-220 | Available LB IPs |
| Upstream DNS | 8.8.8.8, 8.8.4.4 | External resolution |

## Testing DNS

```bash
# Test internal resolution
dig @10.10.1.200 pfsense.engatwork.com

# Test external forwarding
dig @10.10.1.200 google.com

# From a VM using pfSense as DNS
nslookup k8s.engatwork.com
```

## Kubernetes Resources

```bash
# Check MetalLB
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system

# Check CoreDNS
kubectl get pods -n dns
kubectl get svc -n dns
kubectl get configmap coredns-external -n dns -o yaml
```
