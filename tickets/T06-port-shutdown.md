# Ticket #T06: "Printer is offline, nobody can print"

**[← Back to lab overview](../README.md)**

**Affected device:** Printer0 (VLAN 20)
**Severity:** Sev-B (entire office impacted)
**Packet Tracer file:** [`packet-tracer-files/SOHO-Lab-01-t06.pkt`](../packet-tracer-files/SOHO-Lab-01-t06.pkt)

---

## Reported symptom

> *"The whole office is complaining. Print jobs are just sitting in the queue. Even the printer's web page won't load."*

## Diagnosis

### 1. Test L3 reachability to the printer

From PC1, `ping 192.168.20.10` returns 4 timeouts. Printer unreachable at Layer 3.

![ping printer fails](../screenshots/t06-01-symptom-pc1-ping-printer-fail.png)

### 2. Visual check on the physical link

In the Logical workspace, the cable between Switch0 and Printer0 shows a red link indicator. Physical and link layer are down.

![red link light](../screenshots/t06-02-topology-red-link-printer.png)

### 3. Check the switchport status

On Switch0, `show interfaces FastEthernet0/20` reads *"administratively down, line protocol is down."* The phrase "administratively down" is the signature of a deliberate `shutdown` command, not a broken cable or powered-off device.

![switchport admin down](../screenshots/t06-03-switch0-show-int-admin-down.png)

## Root cause

Switch0's Fa0/20 was left administratively shut down after maintenance. The printer itself is healthy; it's just been cut off at the switch.

## Fix

`no shutdown` on Fa0/20:

```
enable
configure terminal
interface FastEthernet0/20
 no shutdown
end
write memory
```

![fix: no shutdown](../screenshots/t06-04-fix-switch0-no-shutdown.png)

## Verification

Link indicator turns green; ping to printer succeeds.

![verify: green link](../screenshots/t06-05-verify-topology-green-link.png)
![verify: ping printer](../screenshots/t06-06-verify-pc1-ping-printer-success.png)

---

**[← Back to lab overview](../README.md)**
