# cluster — k8s bootstrap + authoritative DNS

The middle layer of the homelab build: turns Fedora VMs from `ovirt-setup/` into Kubernetes clusters via kubeadm, and runs the authoritative CoreDNS for `*.engatwork.com`.

## How this fits

Four sibling repos build on each other:

| Repo | Layer | Where |
|---|---|---|
| **`ovirt-setup/`** | Hypervisor + VMs + pfSense networks + Kea DHCP | `/root/learning/ovirt-setup/` |
| **`cluster/`** (this repo) | kubeadm bootstrap + authoritative CoreDNS | `/root/learning/cluster/` |
| **`okd/`** | OKD agent-based installer (separate path; targets the OKD cluster only) | `/root/learning/okd/` |
| **`cplanes/`** | GitOps content (ArgoCD AppSets + Helm charts) — what runs *inside* the clusters | `/root/learning/cplanes/` |

`cluster/` produces a working k8s cluster with Flannel + MetalLB + (optionally) a labeled cluster Secret registered with ArgoCD on OKD. After Phase 8 of `bootstrap-k8s.yml`, the cluster's labels drive ApplicationSet selectors in `cplanes/argo-applications/sre/` and platform services start reconciling automatically.

## Subprojects

This repo contains two playbook trees:

| Path | Purpose |
|---|---|
| **`k8s-bstrp/`** | Generic kubeadm cluster bootstrap (8 phases). One `bootstrap-k8s.yml` handles every cluster type via `-i inventory/<name>.ini`. |
| **`mgmt-core-coredns/`** | Authoritative CoreDNS deployed on the mgmt-core cluster (MetalLB VIP `10.10.1.200`). Serves `*.engatwork.com` to every VM in the homelab. |

## Architecture — CoreDNS plane (mgmt-core)

```
┌─────────────────────────────────────────────────────────────────────┐
│                  mgmt-core Kubernetes cluster                        │
│                       VLAN 10 — 10.10.1.0/24                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                    │
│  │mgmt-core-01│  │mgmt-core-02│  │mgmt-core-03│  CP + workers       │
│  │10.10.1.100 │  │10.10.1.101 │  │10.10.1.102 │  (3-node cluster)   │
│  └────────────┘  └────────────┘  └────────────┘                    │
│                                                                      │
│  Authoritative CoreDNS for engatwork.com                             │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │  MetalLB VIP: 10.10.1.200                                     │   │
│  │  pfSense forwards engatwork.com → here                        │   │
│  │  Records: api.mgmt-devops-okd, *.apps.<cluster>, internal     │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

Other clusters bootstrapped here get their pods, services, and ingress; their cluster Secret lands in OKD's argocd-sre + argocd-devops namespaces during Phase 8.

## Project structure

```
cluster/
├── README.md                                     this file
├── docs/
│   └── eks-bootstrap-argo.md                     AWS-recommended EKS patterns (teaching contrast)
├── mgmt-core-coredns/                            Authoritative DNS, mgmt-core only
│   ├── ansible.cfg
│   ├── inventory.ini                             targets mgmt-core-01
│   ├── deploy-coredns.yml                        main playbook
│   ├── group_vars/
│   │   ├── all.yml                               DNS records (engatwork.com zone)
│   │   └── okd.yml                               OKD-specific records
│   ├── playbooks/
│   │   └── update-okd-dns.yml                    add/remove an OKD cluster's API + ingress records
│   └── docs/
└── k8s-bstrp/                                    Generic kubeadm bootstrap (any cluster)
    ├── ansible.cfg
    ├── bootstrap-k8s.yml                         main 8-phase playbook
    ├── update-flannel.yml                        rebuild Flannel with patched pod_cidr
    ├── group_vars/
    │   ├── all.yml                               version pins, CNI choice, ArgoCD instances
    │   ├── mgmt_core.yml
    │   ├── mgmt_devops.yml
    │   ├── mgmt_forge.yml
    │   ├── mgmt_observability.yml
    │   ├── mgmt_workload.yml                     single-node workload cluster (ARC + ephemeral)
    │   └── dev_{web,apps,data}.yml               rebuild templates (VMs currently deleted)
    ├── inventory/
    │   └── <one .ini per cluster>
    ├── tasks/
    │   ├── register-argocd.yml                   Phase 8 — labels + cluster Secret in argocd-{sre,devops}
    │   └── update-flannel-cluster.yml
    ├── vars/
    │   └── flannel-update.yml                    pod_cidr per cluster (avoids 10.244/16 collision)
    └── docs/
```

## Quick start

### Bootstrap a Kubernetes cluster

```bash
cd k8s-bstrp

# Use the right inventory + (implicit) group_vars/<cluster_vars_file>.yml
ansible-playbook -i inventory/mgmt-forge.ini bootstrap-k8s.yml

# 8 phases:
#   0. Load cluster-specific vars
#   1. Prereqs on all nodes (swap, kernel modules, sysctl, SELinux)
#   2. containerd + CNI plugins + runc
#   3. kubelet/kubeadm/kubectl install
#   4. kubeadm init on first control plane
#   5. CNI install (Flannel — pod_cidr patched per cluster)
#   6. Join workers   (skipped if single_node_cluster: true)
#   7. Post-install (metrics-server, Helm, untaint master if single-node)
#   8. Register cluster with ArgoCD on OKD (gated by argocd_register)
```

Tag-based partial runs:

```bash
ansible-playbook -i inventory/mgmt-forge.ini bootstrap-k8s.yml --tags prereq
ansible-playbook -i inventory/mgmt-forge.ini bootstrap-k8s.yml --tags init-cluster
ansible-playbook -i inventory/mgmt-forge.ini bootstrap-k8s.yml --tags phase8     # rerun ArgoCD registration only
```

### Deploy / update authoritative CoreDNS (mgmt-core only)

```bash
cd mgmt-core-coredns

# Re-render the Corefile from group_vars/all.yml + group_vars/okd.yml
ansible-playbook deploy-coredns.yml

# Add/remove a specific OKD cluster's API + ingress records
ansible-playbook playbooks/update-okd-dns.yml \
    -e okd_cluster_name=mgmt-devops-okd \
    -e okd_api_vip=10.10.2.50 \
    -e okd_ingress_vip=10.10.2.51 \
    -e okd_dns_state=present
```

## Cluster topology — current state

The kubeadm-bootstrapped clusters this repo manages. (OKD is installed via `okd/` repo, not here.)

| Cluster | VLAN | Subnet | Pod CIDR | VMs | Status | Hosts |
|---|---|---|---|---|---|---|
| **mgmt-core** | 10 | 10.10.1.0/24 | 10.244.0.0/16 | .100-.102 (3) | ✅ live | Authoritative CoreDNS for `*.engatwork.com` |
| **mgmt-storage** | 13 | 10.10.4.0/24 | 10.252.0.0/16 | .100-.102 (3) | ✅ live | Rook-Ceph server (RBD/CephFS/RGW) — no other workloads |
| **mgmt-forge** | 14 | 10.10.5.0/24 | 10.250.0.0/16 | .103-.105 (3) | ✅ live | Vault, Keycloak today; planned: Harbor, SonarQube, GitLab, CNPG, Strimzi/Kafka, Redis, Apicurio |
| **mgmt-observability** | 12 | 10.10.3.0/24 | 10.246.0.0/16 | .100-.102 (3) | ✅ live | Prom/Graf/Alertmgr/Loki/Tempo + (future) OpenSearch + Wazuh |
| **mgmt-workload** | 11 | 10.10.2.0/24 | 10.251.0.0/16 | .120 (1) | 📋 planned | Single-node — ARC controller + runners + ephemeral workloads (`single_node_cluster: true`) |
| **mgmt-devops** | 11 | 10.10.2.0/24 | 10.245.0.0/16 | — | 📋 legacy | kubeadm path for VLAN 11 — currently superseded by OKD on the same VLAN. Group_vars retained for fallback. |
| dev-{web,apps,data} | 20-22 | 10.20.{1,2,3}.0/24 | 10.{247,248,249}.0.0/16 | (deleted) | 🗑️ rebuild template | VMs deleted to free RAM; vars/inventory kept for rebuild when 2nd oVirt host arrives |

Pod CIDRs are unique per cluster so the OKD control plane can talk to every workload without overlap. `bootstrap-k8s.yml` Phase 5 downloads the Flannel manifest, rewrites the default `10.244.0.0/16` to the cluster's `pod_network_cidr`, then applies — see `vars/flannel-update.yml` for the canonical mapping.

## Phase 8 — ArgoCD registration

After Phase 7 (post-install), `bootstrap-k8s.yml` runs a localhost play that registers the new cluster with the **two ArgoCD instances on OKD** (`argocd-sre` and `argocd-devops`). It produces:

**On the target cluster:**
- `kube-system/argocd-manager` ServiceAccount
- ClusterRoleBinding → `cluster-admin`
- `kube-system/argocd-manager-token` Secret (long-lived SA token, k8s ≥1.24 pattern)

**On OKD, in each ArgoCD namespace:**
- `<cluster_name>` Secret with label `argocd.argoproj.io/secret-type: cluster`, `stringData.server`, `stringData.config` containing the bearer token + CA, plus all the labels needed for ApplicationSet matching.

The labels merged onto each Secret come from two places (later overrides earlier):

```yaml
# group_vars/all.yml — applied to every cluster
cluster_labels_defaults:
  managed-by: k8s-bstrp
  k8s-version: "1.31.4"

# group_vars/<cluster>.yml — per-cluster overrides
cluster_labels:
  tier: mgmt        # or dev
  role: workload    # or forge / observ / core / storage / web / apps / data
  env: nonprod
  storage: ceph-rbd # or none
```

Those labels drive ApplicationSet `clusters` selectors in `cplanes/argo-applications/sre/`. Pattern A (`tier In [mgmt,dev], role NotIn [storage]`) targets every workload-capable cluster — cert-manager, ingress-nginx, ESO, MetalLB land here automatically. Pattern B (`role: forge`) targets only mgmt-forge for central platform services like Vault and Keycloak.

Disable Phase 8 per-cluster with `argocd_register: false` in `group_vars/<cluster>.yml` — used for foundational clusters (mgmt-core, mgmt-storage) we deliberately keep out of GitOps to avoid bootstrap-circular-dependency failures.

### Phase 8 known gap (tracked)

`bootstrap-k8s.yml` Phase 8 currently runs `kubectl apply -n argocd-sre -f …` against whatever the *current* kubectl context happens to be. Easy to break if a stale context is loaded (e.g., dev-apps from a prior run). Pinning `--context "{{ okd_context | default('admin') }}"` makes the playbook self-contained — fix tracked in `cplanes/forge/todo.md`.

## CoreDNS authoritative records

Served from MetalLB VIP `10.10.1.200`; pfSense forwards `engatwork.com` here.

| Record | IP | Purpose |
|---|---|---|
| `api.mgmt-devops-okd.engatwork.com` | 10.10.2.50 | OKD API VIP (HAProxy on pfSense) |
| `*.apps.mgmt-devops-okd.engatwork.com` | 10.10.2.51 | OKD Ingress VIP |
| `*.apps.mgmt-forge.engatwork.com` | 10.10.5.201 | mgmt-forge ingress-nginx |
| `*.apps.mgmt-observability.engatwork.com` | 10.10.3.201 | mgmt-observability ingress-nginx |
| `*.apps.mgmt-workload.engatwork.com` | 10.10.2.211 | (when deployed) mgmt-workload ingress-nginx |

Cluster nodes themselves (`<cluster>-NN.engatwork.com`) get individual A records from `mgmt-core-coredns/group_vars/all.yml`.

## Cluster details

| Setting | Value |
|---|---|
| Kubernetes version | 1.31.4 (`k8s_version: 1.31`) |
| Container runtime | containerd 1.7.24 + runc 1.2.3 |
| CNI | Flannel 0.26.1 (default; `cni_plugin: calico` toggles to Calico 3.29.1) |
| LoadBalancer | MetalLB (deployed via `cplanes/networking/metallb/`, Pattern A AppSet) |
| Local storage provisioner | (planned — Pattern A AppSet) |
| metrics-server | installed in Phase 7 with `--kubelet-insecure-tls` (lab) |
| Helm | installed in Phase 7, version 3.16.3 |

## Testing DNS

```bash
# From any VM in the network
dig @10.10.1.200 api.mgmt-devops-okd.engatwork.com
dig @10.10.1.200 console.apps.mgmt-devops-okd.engatwork.com
dig @10.10.1.200 keycloak.apps.mgmt-forge.engatwork.com
```

## Cluster cleanup

When you tear down VMs (via `ovirt-setup/playbooks/compute/cleanup_vms.yml`), the cluster Secret in OKD's `argocd-sre` + `argocd-devops` namespaces stays. Until Phase 8 grows a "de-register" hook (tracked task), remove it manually:

```bash
oc --context admin delete secret -n argocd-sre    <cluster_name>
oc --context admin delete secret -n argocd-devops <cluster_name>
```

ArgoCD's ApplicationSets immediately stop generating Applications targeting the deregistered cluster on next reconcile (~3 min).

## Troubleshooting

### k8s issues

```bash
kubectl get nodes                     # everyone Ready?
kubectl get pods -n kube-system       # core daemons up?
kubectl get pods -n kube-flannel      # CNI healthy?
journalctl -u kubelet -f --since "5m ago"
```

### CoreDNS issues

```bash
kubectl logs -n coredns -l app=coredns
kubectl get configmap -n coredns coredns -o yaml
# Sanity test from another VM
dig @10.10.1.200 +short keycloak.apps.mgmt-forge.engatwork.com
```

### Phase 8 silently failed

Symptom: bootstrap completes but `kubectl --context admin get secret -n argocd-sre <cluster>` returns nothing.

Cause: kubectl context wasn't pointed at OKD when Phase 8 ran. Pin it:

```bash
KUBECONFIG=/root/.kube/okd-admin.conf \
    ansible-playbook -i inventory/<cluster>.ini bootstrap-k8s.yml --tags phase8
```

## Documentation

- [`docs/eks-bootstrap-argo.md`](docs/eks-bootstrap-argo.md) — AWS-recommended EKS lifecycle + ArgoCD registration patterns; teaching contrast against this homelab build
- [`k8s-bstrp/docs/README.md`](k8s-bstrp/docs/README.md) — kubeadm phase walkthrough
- [`k8s-bstrp/docs/QA.md`](k8s-bstrp/docs/QA.md) — bootstrap Q&A
- [`mgmt-core-coredns/docs/README.md`](mgmt-core-coredns/docs/README.md) — CoreDNS deep dive
- Cluster topology + plane assignments: [`/root/learning/cplanes/README.md`](../cplanes/README.md)

## Requirements

- Fedora 41 VMs on oVirt (template `fedora-41-mgmt`)
- Ansible 2.14+
- Python 3.9+
- For Phase 8: kubeconfig with `admin` context targeting the OKD cluster

## License

MIT
