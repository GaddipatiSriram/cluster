# CoreDNS External DNS Server

Ansible playbook for deploying CoreDNS as an external DNS server on Kubernetes, providing DNS resolution for the `engatwork.com` domain.

## Overview

This playbook deploys:
- **MetalLB** - LoadBalancer for bare-metal Kubernetes
- **CoreDNS** - DNS server exposed via LoadBalancer IP

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
│              │ (internal)      │                  │  (8.8.8.8)      ││
│              └─────────────────┘                  └─────────────────┘│
└──────────────────────────────────────────────────────────────────────┘
```

## Quick Start

```bash
# Full deployment (MetalLB + CoreDNS)
ansible-playbook deploy-coredns.yml

# Deploy only MetalLB
ansible-playbook deploy-coredns.yml --tags metallb

# Deploy only CoreDNS
ansible-playbook deploy-coredns.yml --tags coredns

# Verify DNS
ansible-playbook deploy-coredns.yml --tags verify
```

## Project Structure

```
coredns/
├── ansible.cfg           # Ansible configuration
├── inventory.ini         # Control plane target
├── deploy-coredns.yml    # Main deployment playbook
├── group_vars/
│   ├── all.yml           # DNS records and configuration
│   └── okd.yml           # OKD-specific DNS records
├── playbooks/
│   └── update-okd-dns.yml  # Update OKD DNS records
└── docs/
    ├── README.md         # Detailed documentation
    └── QA.md             # Questions & Answers
```

## Configuration

### DNS Records

Edit `group_vars/all.yml`:

```yaml
dns_records:
  - ip: "10.10.1.101"
    hostnames:
      - "mgmt-core-01"
      - "k8s"
  - ip: "10.10.2.101"
    hostnames:
      - "gitlab"
```

### OKD DNS Integration

The `playbooks/update-okd-dns.yml` playbook updates DNS for OKD clusters:

```bash
ansible-playbook playbooks/update-okd-dns.yml \
  -e okd_cluster_name=mgmt-devops-okd \
  -e okd_api_vip=10.10.2.50 \
  -e okd_ingress_vip=10.10.2.51 \
  -e okd_dns_state=present   # or 'absent' to remove
```

This creates:
- `api.<cluster>.engatwork.com` → API VIP
- `api-int.<cluster>.engatwork.com` → API VIP
- `*.apps.<cluster>.engatwork.com` → Ingress VIP

## Network Configuration

| Component | IP/Range | Purpose |
|-----------|----------|---------|
| CoreDNS LB | 10.10.1.200 | DNS server endpoint |
| MetalLB Pool | 10.10.1.200-220 | Available LB IPs |
| Upstream DNS | 8.8.8.8, 8.8.4.4 | External resolution |

## Testing

```bash
# Test internal resolution
dig @10.10.1.200 mgmt-core-01.engatwork.com

# Test OKD API
dig @10.10.1.200 api.mgmt-devops-okd.engatwork.com

# Test OKD wildcard
dig @10.10.1.200 console.apps.mgmt-devops-okd.engatwork.com

# Test external forwarding
dig @10.10.1.200 google.com
```

## Kubernetes Resources

```bash
# Check MetalLB
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system

# Check CoreDNS
kubectl get pods -n coredns
kubectl get svc -n coredns
kubectl get configmap -n coredns coredns -o yaml
```

## Troubleshooting

```bash
# Check CoreDNS logs
kubectl logs -n coredns -l app=coredns -f

# Restart CoreDNS
kubectl rollout restart deployment -n coredns coredns
```

## Documentation

See [docs/README.md](docs/README.md) for detailed documentation.
