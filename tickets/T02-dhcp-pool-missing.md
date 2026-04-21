# Ticket #T02: "My PC got a weird 169 address and nothing works"

**[← Back to lab overview](../README.md)**

**Affected users:** PC1 and PC2 (both VLAN 20 Staff)
**Severity:** Sev-B (multiple users in one VLAN, business impact)
**Packet Tracer file:** [`packet-tracer-files/SOHO-Lab-01-t02.pkt`](../packet-tracer-files/SOHO-Lab-01-t02.pkt)

---

## Reported symptom

> *"My computer says 'limited connectivity' at the bottom of the screen, and my IP address starts with 169."*

## Clarifying questions I asked

- Is anyone else in your area affected? (Yes, the coworker next to them has the same issue.) This immediately raises severity.

## Diagnosis

### 1. `ipconfig /all` on PC1: confirm the reported 169 address

In Packet Tracer's simulation, all address fields show `0.0.0.0` after a failed DHCP renew. (In real Windows this would be an APIPA address `169.254.x.x`; PT shows `0.0.0.0` instead. Both indicate the same underlying problem: no DHCP response received.)

![PC1 no IP](../screenshots/t02-01-symptom-pc1-no-ip-ipconfig.png)

### 2. `ipconfig /release` then `ipconfig /renew`: try to re-acquire

Renew times out with `DHCP request failed.`

![renew fails](../screenshots/t02-02-pc1-renew-fail.png)

### 3. Rule out a PC-specific issue: test PC2 (same VLAN)

PC2 also fails release/renew with the same `DHCP request failed.` output. Two PCs in the same VLAN failing DHCP → the problem is upstream, not on the PCs themselves.

![PC2 also no IP](../screenshots/t02-03-pc2-also-no-ip.png)

### 4. Check Router0's DHCP pool configuration

On Router0, `show ip dhcp pool` shows only `VLAN10-IT` and `VLAN30-GUEST`. `VLAN20-STAFF` is missing entirely.

![Router0 show ip dhcp pool](../screenshots/t02-04-router0-show-ip-dhcp-pool-missing.png)

## Root cause

The DHCP pool for VLAN 20 was removed from Router0's configuration. Without a pool, the router ignores DHCPDISCOVER broadcasts from VLAN 20 clients, and they fall into the no-lease state.

## Fix

Recreate the pool on Router0:

```
enable
configure terminal
ip dhcp pool VLAN20-STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 203.0.113.1
 domain-name soho.local
end
write memory
```

![fix: DHCP pool re-added](../screenshots/t02-05-fix-router0-readd-pool.png)

## Verification

PC1 renew succeeds with `192.168.20.11`; pings to gateway and external both succeed.

![verify: PC1 renew success](../screenshots/t02-06-verify-pc1-renew-success.png)
![verify: PC1 pings](../screenshots/t02-07-verify-pc1-pings.png)

---

**[← Back to lab overview](../README.md)**
