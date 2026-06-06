# Part 3 – Security & Compliance

---

## 1. Least-Privilege Access

### Environment isolation

Separate Azure subscriptions per environment — Dev, Staging, Prod. A Management Group above them for shared policies. This is the strongest isolation boundary Azure offers.

### Human access

Using Azure PIM (Privileged Identity Management) for just-in-time access:

- **Dev**: developers get Contributor standing access (it's dev, they need to move fast)
- **Staging**: developers get Reader by default. Need Contributor? Request it via PIM — 2h max, requires justification
- **Prod**: zero standing access for anyone. Senior DevOps gets Reader standing; Contributor requires PIM activation (1h max, needs approval from another senior). SRE on-call gets Contributor via PIM (4h, auto-approved during active incidents since you can't be waiting for approvals at 2AM)

All PIM activations require MFA and a justification. Monthly access reviews auto-revoke anything unused.

### Application access

Workload Identity — each pod gets its own managed identity with scoped permissions. The Wallet MI can read secrets. The Payment Gateway MI can read secrets AND sign with crypto keys. No shared identities, no passwords in the system.

NetworkPolicies at the K8s level enforce that only pods in the Wallet namespace can talk to the Payment Gateway namespace.

---

## 2. Audit Trails & Compliance

Central Log Analytics Workspace with 2 year retention, collecting:
- Azure Activity Log (who changed what in the control plane)
- Azure AD Audit Logs (role assignments, PIM activations)
- Key Vault Diagnostic Logs (every single key/secret access)
- CosmosDB Diagnostic Logs
- AKS Audit Logs (K8s API server calls)
- NSG Flow Logs

For long-term retention (7+ years for compliance): export to Azure Immutable Blob Storage (WORM — write once read many). Can't be tampered with.

Enforcement via Azure Policy — deny resources without encryption, deny public IPs on AKS, require private endpoints, require HTTPS everywhere. Also auto-deploy diagnostic settings so nothing goes unmonitored.

Microsoft Defender for Cloud enabled — specifically Defender for Containers, Key Vault, and CosmosDB.

---

## 3. DDoS & Bot Protection

**Network layer (L3/L4):** Azure DDoS Protection Standard on all public VNets. It's always-on with adaptive tuning.

**Application layer (L7):** Front Door WAF with OWASP managed ruleset plus custom rules:
- Rate limiting: 100 req/min per IP on API endpoints
- Geo-filtering for high-risk regions
- Bot Manager to block known bad bots

**Application level:** per-user rate limiting on sensitive endpoints — withdrawals capped at 5 req/min, regular reads at 60 req/min. CAPTCHA for account creation and after failed login attempts.

**Network isolation:** NSGs default to deny-all. Payment Gateway has zero public exposure. Private Endpoints on all PaaS services eliminate the public attack surface entirely.

---

## 4. Backup & Disaster Recovery

RPO/RTO targets:

| Data | RPO | RTO |
|------|-----|-----|
| Transactions (CosmosDB) | ~0 (multi-region writes) | < 5 min (automatic failover) |
| Crypto keys (Key Vault) | ~0 (replicated) | < 1 min |
| App config (in Git) | 0 | < 30 min (redeploy from repo) |

**Database:** CosmosDB continuous backup with PITR — can restore to any second within the last 30 days. Multi-region replication is always on. I'd also do a daily export to immutable blob storage as an extra safety net.

**Key Vault:** Soft Delete enabled (90 day retention) with Purge Protection on. Nobody can permanently delete a key, not even a subscription admin.

**Infrastructure:** everything is in Terraform so it's reproducible. K8s workloads managed via GitOps (ArgoCD or Flux). Velero for persistent volume backups.

**Testing:** monthly automated restore tests — restore a backup, validate data integrity, tear it down. Quarterly DR drills where we actually measure RTO/RPO against our targets.

All backups encrypted at rest (AES-256) and in transit (TLS 1.2+).
