# CoreDNS - Questions & Answers

## General

### Q: What is MetalLB and why is it needed?

**A:** MetalLB is a LoadBalancer implementation for bare-metal Kubernetes clusters. In cloud environments (AWS, GCP), LoadBalancer services get external IPs automatically. On bare-metal, MetalLB fills this gap by:
- Assigning IPs from a configured pool (10.10.1.200-220)
- Announcing IPs via ARP (Layer 2 mode)
- Routing traffic to the correct pods

Without MetalLB, LoadBalancer services would stay in "Pending" state forever.

### Q: Why deploy CoreDNS externally instead of using the built-in one?

**A:** Kubernetes has CoreDNS for internal service discovery (e.g., `my-service.default.svc.cluster.local`). The external CoreDNS we deployed:
- Serves DNS for the entire infrastructure (all VLANs)
- Resolves `*.engatwork.com` to infrastructure IPs
- Forwards external queries (google.com) to upstream DNS
- Is accessible from outside the cluster

### Q: How does pfSense integrate with CoreDNS?

**A:** pfSense is configured with a Domain Override:
- Domain: `engatwork.com`
- IP: `10.10.1.200` (CoreDNS LoadBalancer)

All VMs using pfSense as their DNS server automatically get `*.engatwork.com` resolved via CoreDNS, while external domains go to pfSense's upstream DNS.

---

## Configuration

### Q: Why use a hosts file instead of zone files?

**A:** CoreDNS supports multiple backends. The hosts plugin was chosen for simplicity:
- Easy to understand and modify
- No zone file syntax to learn
- Managed via Ansible variables
- Sufficient for lab environments

For production, you might use the `file` plugin with proper zone files or integrate with etcd.

### Q: How do I add wildcard DNS records?

**A:** The hosts plugin doesn't support wildcards. For wildcard support, modify the Corefile to use the `template` plugin:

```
engatwork.com:53 {
    template IN A apps {
        match ^.*\.apps\.engatwork\.com\.$
        answer "{{ .Name }} 60 IN A 10.10.1.200"
    }
    hosts /etc/coredns/hosts {
        fallthrough
    }
}
```

### Q: Can I use this for dynamic DNS with ExternalDNS?

**A:** Yes, but it requires additional setup:
1. Deploy ExternalDNS controller
2. Configure CoreDNS with etcd backend
3. ExternalDNS watches Ingress/Service resources
4. Automatically creates DNS records

This is useful when you want DNS records created automatically when deploying apps.

### Q: How do I change the upstream DNS servers?

**A:** Edit `group_vars/all.yml`:
```yaml
upstream_dns:
  - "1.1.1.1"
  - "1.0.0.1"
```
Then re-run the playbook with `--tags coredns`.

---

## Troubleshooting

### Q: What happens if CoreDNS pods restart?

**A:** CoreDNS runs with 2 replicas for high availability. MetalLB maintains the same IP (10.10.1.200) regardless of which pod handles traffic. The LoadBalancer service handles failover automatically.

### Q: Why did the Service fail with "AllocationFailed"?

**A:** MetalLB doesn't allow both `spec.loadBalancerIP` and the annotation `metallb.universe.tf/loadBalancerIPs`. The playbook uses only the annotation to specify the IP.

### Q: CoreDNS pods are CrashLoopBackOff?

**A:** Check the Corefile syntax. The most common issue is missing `health` and `ready` plugins:
```
.:53 {
    forward . 8.8.8.8
    health :8080
    ready :8181
}
```

Check logs:
```bash
kubectl logs -n dns -l app=coredns-external
```

### Q: DNS queries are slow or timing out?

**A:** Check if CoreDNS can reach upstream DNS:
```bash
kubectl exec -n dns -it <coredns-pod> -- nslookup google.com 8.8.8.8
```

Also verify MetalLB speaker pods are running:
```bash
kubectl get pods -n metallb-system
```

### Q: How do I check CoreDNS logs?

**A:**
```bash
kubectl logs -n dns -l app=coredns-external -f
```

### Q: LoadBalancer IP stuck in "Pending"?

**A:** Check MetalLB:
```bash
# Are speakers running?
kubectl get pods -n metallb-system

# Is IP pool configured?
kubectl get ipaddresspool -n metallb-system

# Check service events
kubectl describe svc coredns-external -n dns
```

---

## Integration

### Q: How do I make all VMs use this DNS automatically?

**A:** Configure pfSense DHCP to distribute the DNS server:
1. Services → DHCP Server → Select interface
2. Set DNS Server: `10.10.1.200` (or keep pfSense as DNS with domain override)

Or configure pfSense DNS Resolver to forward `engatwork.com` to CoreDNS (already done).

### Q: Can I add reverse DNS (PTR records)?

**A:** Yes, add a reverse zone to the Corefile:
```
10.in-addr.arpa:53 {
    hosts /etc/coredns/reverse-hosts {
        fallthrough
    }
}
```

And create reverse-hosts entries:
```
101.1.10.10 mgmt-core-01.engatwork.com
```

### Q: How do I integrate with cert-manager for Let's Encrypt?

**A:** For internal certificates, you'd typically use:
1. cert-manager with a self-signed CA, or
2. DNS-01 challenge with your actual domain registrar

For `engatwork.com` to work with Let's Encrypt, you'd need to own the domain and configure DNS-01 validation at your registrar, not on this internal CoreDNS.
