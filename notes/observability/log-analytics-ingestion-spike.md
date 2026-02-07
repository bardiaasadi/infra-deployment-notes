# Investigating a Log Analytics Ingestion Spike (AGW Access Logs)

## Executive summary

A sudden Log Analytics ingestion and cost spike was investigated in Application Gateway access logs.

**What mattered:**
- The spike was **volume-driven**, not caused by larger log payloads.
- Traffic was **highly concentrated**: a single public IP repeatedly calling one endpoint.
- The source IP was **owned by us**, not external abuse.
- A configuration change happened around the same time, but was **correlated, not causal**.

This distinction (volume vs. payload size, correlation vs. causation) guided both the investigation and the response.

---

## The problem framing

Log Analytics costs increased sharply over a short time window.

The key questions were:
1. Is ingestion growing because logs are *bigger*, or because there are *more requests*?
2. Is this broad system behavior, or a narrow traffic pattern?
3. Is the source external, or something we own?
4. How do we put a brake on cost **without** losing visibility while root cause is addressed?

---

## Evidence-driven investigation

### Volume vs. size (the first fork in the road)

The first check was whether `_BilledSize` per request changed.

```kusto
AGWAccessLogs
| summarize
    Requests = count(),
    AvgBytesPerRequest = avg(_BilledSize)
  by bin(TimeGenerated, 1d)
| sort by TimeGenerated asc
````

**Result:** average bytes per request stayed effectively flat.
**Conclusion:** ingestion growth was driven by **request volume**, not verbosity.

---

### Narrowing the traffic signature

Next, traffic was grouped by host, URI, status code, and user agent.

```kusto
AGWAccessLogs
| extend
    Host = tostring(column_ifexists("Host", column_ifexists("host_s",""))),
    Uri  = tostring(column_ifexists("RequestUri", column_ifexists("requestUri_s","")))
| summarize
    Requests = count(),
    IngestedMB = sum(_BilledSize) / 1024 / 1024
  by Host, Uri
| sort by IngestedMB desc
| take 20
```

**Result:** ingestion was dominated by a **single endpoint**.

---

### Identifying the source

Filtering to that endpoint and grouping by client IP made the pattern obvious:

```kusto
AGWAccessLogs
| where Uri == "SENSITIVE_ENDPOINT"
| extend ClientIp = tostring(column_ifexists(
    "clientIp_s",
    column_ifexists("ClientIp", column_ifexists("ClientIP",""))
))
| summarize Requests = count() by ClientIp
| sort by Requests desc
```

**Result:** one IP accounted for the majority of traffic.

A quick Azure lookup confirmed the IP belonged to **our own infrastructure**.

---

## What we concluded

1. **This was not log bloat** — it was a traffic amplification issue.
2. **Blast radius was narrow** — one IP, one endpoint.
3. **Ownership was internal** — investigation shifted from security to system behavior.
4. **Timing coincidences can mislead** — configuration changes overlapped but were not the cause.

---

## Mitigation & controls (while root cause was fixed)

### Cost brake: daily ingestion cap

A daily ingestion cap was applied to a **temporary workspace** used for these logs.

**Intent:**
Limit worst-case cost exposure while preserving enough signal to continue analysis.

This is not a permanent solution — it is a **seatbelt**, not a steering wheel.

---

### Tripwire: signature-based alert

A scheduled query alert was added to detect the exact failure signature:

* specific source IP
* specific endpoint
* requests/minute exceeding a threshold

```kusto
AGWAccessLogs
| where TimeGenerated > ago(5m)
| where ClientIp == "PUBLIC_IP_1"
| where Uri == "SENSITIVE_ENDPOINT"
| summarize Requests = count() by bin(TimeGenerated, 1m)
| where Requests > THRESHOLD_RPM
```

**Design choice:** detect *patterns*, not generic spikes, to avoid noise.

---

### Alert hygiene: suppression

To prevent notification spam during an active incident, alert actions were muted for a fixed duration after firing.

**Result:**
The alert continued evaluating every 5 minutes, but notifications were sent **once per incident window**, not continuously.

---

## Why this mattered operationally

* The investigation stayed **data-driven**, not assumption-driven.
* We avoided over-correcting (e.g., disabling logs blindly).
* Cost controls were applied surgically and temporarily.
* The final fix addressed the system behavior, not the symptom.

---

## Takeaway

Log Analytics cost incidents are rarely “logging problems” in isolation.
They are usually **traffic problems wearing a logging bill**.

Separating *volume* from *payload*, and *correlation* from *causation*, is what keeps these incidents calm instead of chaotic.

