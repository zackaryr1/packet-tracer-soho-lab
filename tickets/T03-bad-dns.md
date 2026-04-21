# Ticket #T03: "I can ping the server but websites won't load"

**[← Back to lab overview](../README.md)**

**Affected user:** PC2 (VLAN 20 Staff)
**Severity:** Sev-C
**Packet Tracer file:** [`packet-tracer-files/SOHO-Lab-01-t03.pkt`](../packet-tracer-files/SOHO-Lab-01-t03.pkt)

---

## Reported symptom

> *"Ping works when I use numbers but I can't open any websites. Every URL gives me 'could not find host.'"*

## Diagnosis

### 1. `ping 203.0.113.1`: test IP-layer reachability

4/4 replies. Network path and routing are fine.

![ping by IP succeeds](../screenshots/t03-01-symptom-ping-ip-success.png)

### 2. `ping google.com`: test name resolution

*"Ping request could not find host google.com."* **Name resolution specifically** is broken. This is the fingerprint of a DNS problem: Layer 1 through 4 work, Layer 7 (DNS) fails.

![ping by name fails](../screenshots/t03-02-ping-name-fails.png)

### 3. `nslookup google.com`: confirm DNS server is unreachable

DNS request times out. Confirms the configured DNS server is unreachable, not just slow.

![nslookup timeout](../screenshots/t03-03-nslookup-timeout.png)

### 4. `ipconfig /all`: check the configured DNS server

DNS Server shows `10.10.10.10`, an IP that doesn't exist anywhere in this network.

![ipconfig showing bad DNS](../screenshots/t03-04-ipconfig-bad-dns.png)

## Root cause

PC2's DNS server is statically set to `10.10.10.10`, which doesn't exist on our network. DNS queries are sent but never answered.

## Fix

Reset IP Configuration to DHCP. Router0's DHCP pool pushes the correct DNS server (`203.0.113.1`).

![fix: DHCP restores DNS](../screenshots/t03-05-fix-dhcp-restored.png)

## Verification

`nslookup google.com` returns `203.0.113.1`; `ping google.com` returns 4/4 replies from that IP.

![verify: nslookup succeeds](../screenshots/t03-06-verify-nslookup-success.png)

---

**[← Back to lab overview](../README.md)**
