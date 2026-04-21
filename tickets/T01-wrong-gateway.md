# Ticket #T01: "I can't get on the internet or the shared drive"

**[← Back to lab overview](../README.md)**

**Affected user:** PC0 (IT, VLAN 10)
**Severity:** Sev-C (single user, no productivity impact beyond one person)
**Packet Tracer file:** [`packet-tracer-files/SOHO-Lab-01-t01.pkt`](../packet-tracer-files/SOHO-Lab-01-t01.pkt)

---

## Reported symptom

> *"Everything was working yesterday. This morning nothing loads — not Outlook, not the shared drive, not websites."*

## Clarifying questions I asked

- Is anyone else at your row affected? (No, just this PC.)
- Did anything change since yesterday? (User says no, but assumptions like this are rarely reliable.)

## Diagnosis

### 1. `ipconfig /all`: check the basics first

Default Gateway shows `192.168.10.99`, not the expected `192.168.10.1`.

![ipconfig showing wrong gateway](../screenshots/t01-01-symptom-ipconfig.png)

### 2. `ping 192.168.10.99`: verify the configured gateway exists

All 4 packets time out. Confirms that address doesn't exist anywhere on the network.

![ping to fake gateway fails](../screenshots/t01-02-ping-gateway-fail.png)

### 3. `ping 192.168.10.1`: try pinging the *real* gateway

4/4 replies. This is the key diagnostic. Even though the PC has the wrong default gateway configured, the real gateway is still pingable because it's on the same subnet. The PC can reach it via ARP at Layer 2 without consulting its (broken) default gateway.

![ping to real gateway succeeds](../screenshots/t01-03-ping-gateway-real-success.png)

### 4. `ping 203.0.113.1`: test off-subnet reachability

4/4 timeouts. This is the ping that proves the impact of the misconfiguration: off-subnet traffic requires the default gateway, and the PC's default gateway doesn't exist.

![ping off-subnet fails](../screenshots/t01-04-ping-offsubnet-fail.png)

### 5. Compare with a working neighbor

Neighbor's gateway is a real, reachable IP. PC0's is not.

![neighbor comparison](../screenshots/t01-05-neighbor-compare.png)

## Root cause

PC0's default gateway was manually set to `192.168.10.99`, a non-existent address. Same-subnet traffic works because ARP bypasses the default gateway, but anything off-subnet (internet, cross-VLAN servers, printers in other VLANs) has no route out.

## Fix

Reset IP Configuration to DHCP.

![fix: switched to DHCP](../screenshots/t01-06-fix-dhcp-toggle.png)

## Verification

`ipconfig /all` shows Default Gateway now `192.168.10.1`; `ping 203.0.113.1` returns 4/4 replies.

![verify: ping internet succeeds](../screenshots/t01-07-verify-ping-internet.png)

---

**[← Back to lab overview](../README.md)**
