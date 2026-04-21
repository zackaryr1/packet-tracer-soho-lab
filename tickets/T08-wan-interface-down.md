# Ticket #T08 — "We can reach internal stuff but can't get to the internet"

**[← Back to lab overview](../README.md)**

**Affected users:** All clients (full network)
**Severity:** Sev-A (total external outage)
**Packet Tracer file:** [`packet-tracer-files/SOHO-Lab-01-t08.pkt`](../packet-tracer-files/SOHO-Lab-01-t08.pkt)

---

## Reported symptom

> *"Shared drive works, internal sites work, but nobody can reach anything external. Our ping tests to 203.0.113.1 just time out."*

## Diagnosis

### 1. Test internal reachability

From PC0, `ping 192.168.20.10` (Printer0, cross-VLAN internal) — 4/4 replies. Internal routing is intact. Rules out a total network outage.

![internal pings work](../screenshots/t08-01-symptom-pc0-internal-ping-success.png)

### 2. Test external reachability

`ping 203.0.113.1` — 4/4 timeouts. External is specifically broken.

![external ping fails](../screenshots/t08-02-symptom-pc0-external-ping-fail.png)

### 3. Trace the failing path with `tracert`

`tracert 203.0.113.1` reaches `192.168.10.1` (Router0) on hop 1, then dies. **Tracert dying at the edge router means the problem is *on* the router** — not the LAN, not the PC. Three most likely causes: (a) egress interface is down, (b) no route for the destination, (c) an ACL is dropping traffic on egress.

![tracert dies at router](../screenshots/t08-03-pc0-tracert-dies-at-router0.png)

### 4. Check router interface status

On Router0, `show ip interface brief` — `GigabitEthernet0/1` (the WAN uplink) is **administratively down**. All internal sub-interfaces are up.

![Gi0/1 admin down](../screenshots/t08-04-router0-show-ip-int-brief-gi01-down.png)

### 5. Check the routing table

`show ip route` — no `C  203.0.113.0/30` connected route (because Gi0/1 is down), and no `S*  0.0.0.0/0` default route (the static default's next-hop is unreachable, so the routing table drops it automatically).

![routing table missing default](../screenshots/t08-05-router0-show-ip-route-missing-default.png)

## Root cause

Router0's WAN interface was left administratively shut down after maintenance. The connected route to `203.0.113.0/30` disappeared from the routing table, which in turn invalidated the default route. Internal traffic still worked because it uses the Gi0/0 sub-interfaces (still up), but anything destined outside the internal VLANs had no route.

## Fix

`no shutdown` on Gi0/1:

```
enable
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
write memory
```

![fix: no shutdown Gi0/1](../screenshots/t08-06-fix-router0-no-shutdown-gi01.png)

## Verification

Routing table now includes the connected `203.0.113.0/30` and the default route `S*  0.0.0.0/0 via 203.0.113.1`. PC0 pings external successfully; tracert completes in 2 hops.

![verify: ip route restored](../screenshots/t08-07-verify-router0-show-ip-route-restored.png)

---

**[← Back to lab overview](../README.md)**
