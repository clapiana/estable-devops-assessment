# Part 2 – Incident Response

**Scenario:** At 2AM, Estable Pay transactions fail in APAC region. Other regions are fine. Logs show DB timeouts.

---

## 1. Troubleshooting

The fact that it's region-specific and showing DB timeouts narrows it down a lot. I'd focus on APAC database connectivity and capacity, not global issues.

Steps I'd follow:
1. Check for recent changes — any deployment or Terraform apply near 2AM? This is the most common root cause in my experience
2. Application logs — look at Wallet and Payment Gateway pods. What kind of timeout errors? Are pods restarting, OOMKilled?
3. CosmosDB metrics — RU consumption (is it throttled?), server-side latency, replication lag to APAC region, connection count
4. Network — test connectivity from pod to CosmosDB private endpoint. Check NSG rules, Private Endpoint DNS resolution, VNet peering status
5. Azure Service Health — is there a platform incident affecting APAC?

Most likely cause: CosmosDB RU throttling (HTTP 429 errors manifesting as timeouts from the app's perspective). I've seen this happen before — the app doesn't handle 429 retries properly and just times out.

---

## 2. Azure Tools/Logs

In order of how I'd check them:

1. **Azure Monitor Alerts** — see if anything already fired for APAC resources
2. **Application Insights Live Metrics** — real-time failure rate and dependency calls
3. **Log Analytics** — KQL query to confirm throttling:

```kql
AzureDiagnostics
| where ResourceProvider == "MICROSOFT.DOCUMENTDB"
| where statusCode_s == "429"
| where TimeGenerated > ago(1h)
| summarize ThrottledRequests=count() by bin(TimeGenerated, 1m)
```

4. **CosmosDB Metrics blade** — RU consumption vs provisioned, latency per region
5. **Container Insights** — pod health, restart count
6. **Activity Log** — any recent config changes to APAC resources

---

## 3. Immediate Mitigation

Depends on what we find:

**If it's CosmosDB throttling** — scale up RU/s or switch to autoscale:
```bash
az cosmosdb sql database throughput migrate \
  --account-name cosmosdb-estable \
  --name estable-db \
  --resource-group rg-estable-prod \
  --throughput-type autoscale
```

**If the regional issue is hard to fix quickly** — redirect APAC traffic to another region through Front Door:
```bash
az afd origin update \
  --resource-group rg-estable-global \
  --profile-name fd-estable \
  --origin-group-name default \
  --origin-name apac-origin \
  --enabled-state Disabled
```

**If it's a pod or connection issue** — rolling restart:
```bash
kubectl rollout restart deployment/wallet -n wallet --context aks-apac
```

---

## 4. Long-Term Preventive Measures

- Enable CosmosDB autoscale so we don't hit throttling again
- Set proactive alerts at 70% RU consumption, not when it's already failing
- Add circuit breaker pattern in the app so it degrades gracefully instead of hammering a struggling DB
- Set up synthetic monitoring from APAC locations — test the full flow every 5 min, alert before users notice
- Progressive deployments: staging → US → EU → APAC, with bake time between each region
- Document runbooks for common incidents and link them in the alert descriptions
- Run chaos engineering drills with Azure Chaos Studio — simulate DB outages and network failures in staging
