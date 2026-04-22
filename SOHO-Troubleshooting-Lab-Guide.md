# SOHO Network Troubleshooting Lab (Cisco Packet Tracer)

A complete step-by-step build guide for a small-office network and eight realistic Tier 1 help-desk troubleshooting scenarios. Built and tested end-to-end in Cisco Packet Tracer 8.x.

**What you end up with:** a fully working 3-VLAN network with inter-VLAN routing, DHCP, DNS, and WPA2 Wi-Fi, plus 8 pre-broken save files that each reproduce a specific help-desk scenario for practice or portfolio demonstration.

**Prerequisites:**
- Cisco Packet Tracer 8.x installed (free via Cisco NetAcad)
- Basic familiarity with the Packet Tracer interface (drag devices, click to configure, CLI tab access)
- About 4–5 hours total for a first pass

**Tested on:** Packet Tracer 8.2 on Windows 11. All commands and device model names reflect what PT 8.x actually shows.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Equipment and Topology](#2-equipment-and-topology)
3. [IP Addressing Plan](#3-ip-addressing-plan)
4. [Phase 1: Build the Baseline Network](#4-phase-1-build-the-baseline-network)
   - 4.1 [Place the devices](#41-place-the-devices)
   - 4.2 [Install the wireless NIC on Laptop0](#42-install-the-wireless-nic-on-laptop0)
   - 4.3 [Cable the network](#43-cable-the-network)
   - 4.4 [Configure Router0](#44-configure-router0)
   - 4.5 [Configure Switch0](#45-configure-switch0)
   - 4.6 [Configure Access Point0](#46-configure-access-point0)
   - 4.7 [Configure Server0 (internet simulator)](#47-configure-server0-internet-simulator)
   - 4.8 [Configure Server1 (internal server)](#48-configure-server1-internal-server)
   - 4.9 [Configure Printer0](#49-configure-printer0)
   - 4.10 [Configure the PCs](#410-configure-the-pcs)
   - 4.11 [Configure Laptop0](#411-configure-laptop0)
5. [Phase 2: Verify the Baseline Works](#5-phase-2-verify-the-baseline-works)
6. [Phase 3: Troubleshooting Tickets](#6-phase-3-troubleshooting-tickets)
   - [Ticket workflow](#ticket-workflow)
   - [T01: Wrong default gateway](#t01-wrong-default-gateway)
   - [T02: DHCP pool missing](#t02-dhcp-pool-missing)
   - [T03: Bad DNS](#t03-bad-dns)
   - [T04: Wrong Wi-Fi passphrase](#t04-wrong-wi-fi-passphrase)
   - [T05: Switchport in wrong VLAN](#t05-switchport-in-wrong-vlan)
   - [T06: Switchport administratively down](#t06-switchport-administratively-down)
   - [T07: Wrong SSID](#t07-wrong-ssid)
   - [T08: WAN interface down](#t08-wan-interface-down)
7. [Documenting for a Portfolio](#7-documenting-for-a-portfolio)
8. [Packet Tracer Gotchas](#8-packet-tracer-gotchas)
9. [Appendix: Quick Reference](#9-appendix-quick-reference)

---

## 1. Overview

### What this lab demonstrates

This lab proves three things a help desk hiring manager cares about:

1. **OSI-model reasoning.** You can move from physical (is it plugged in?) through link, network, and application layers in a logical order.
2. **A repeatable process.** Observe symptoms, ask clarifying questions, gather evidence, identify root cause, apply a fix, verify.
3. **Documentation habits.** Help desks live on written tickets. Every scenario here gets a full writeup.

### Target audience for replicating this lab

- IT students or career-changers building a portfolio
- Anyone studying for CompTIA Network+, CCNA, or IT Help Desk roles
- Trainers building practice scenarios for new hires

### Skills you will practice

Cisco IOS CLI, VLANs and 802.1Q trunking, router-on-a-stick inter-VLAN routing, DHCP pool configuration, DNS troubleshooting, WPA2-PSK wireless, static routing, ARP behavior, Windows diagnostic commands (`ipconfig`, `ping`, `tracert`, `nslookup`), Cisco `show` commands, and structured ticket documentation.

---

## 2. Equipment and Topology

### Device inventory

| Qty | Device       | Packet Tracer model | Default name on drop | Role                                  |
|-----|--------------|---------------------|----------------------|---------------------------------------|
| 1   | Router       | `2911`              | Router0              | Inter-VLAN routing, DHCP, default route |
| 1   | Switch       | `2960`              | Switch0              | Access layer, 802.1Q trunk to router  |
| 1   | Access Point | `AP-PT`             | Access Point0        | WPA2-PSK guest Wi-Fi                  |
| 2   | Server       | `Server-PT`         | Server0, Server1     | Server0 = internet/DNS simulator; Server1 = internal |
| 3   | PC           | `PC-PT`             | PC0, PC1, PC2        | Wired clients                         |
| 1   | Laptop       | `Laptop-PT`         | Laptop0              | Wireless client (guest VLAN)          |
| 1   | Printer      | `Printer-PT`        | Printer0             | Networked printer                     |

### Logical topology

```
                    [Server0 — internet simulator]
                              |
                     (Copper Straight-Through)
                              |
                         [Router0 — 2911]
                         Gi0/1 = WAN to Server0 (203.0.113.2/30)
                         Gi0/0 = Trunk to Switch0 (dot1Q subinterfaces per VLAN)
                              |
                     (Copper Straight-Through, trunk)
                              |
                         [Switch0 — 2960]
              ----+--------------+----------------+----+
               Fa0/1          Fa0/5             Fa0/10  Fa0/15  Fa0/20
              VLAN 10        VLAN 20            VLAN 10 VLAN 30 VLAN 20
               PC0 (IT)      PC1 (Staff)        Server1 Access  Printer0
                             PC2 (Fa0/6, Staff)         Point0
                                                         |
                                                    Laptop0 (Wi-Fi)
```

Laptop0 connects wirelessly to Access Point0. No cable.

---

## 3. IP Addressing Plan

### VLANs

| VLAN | Name   | Subnet            | Gateway        | DHCP pool range     | Purpose                   |
|------|--------|-------------------|----------------|---------------------|---------------------------|
| 10   | IT     | 192.168.10.0/24   | 192.168.10.1   | .11 – .100          | IT staff and servers      |
| 20   | Staff  | 192.168.20.0/24   | 192.168.20.1   | .11 – .100          | General employees, printer|
| 30   | Guest  | 192.168.30.0/24   | 192.168.30.1   | .11 – .50           | Guest Wi-Fi (Laptop0)     |
| —    | WAN    | 203.0.113.0/30    | Server0 side   | Static only         | Uplink to "internet"      |

### Static device IPs

| Device   | IP address      | Notes                                   |
|----------|-----------------|-----------------------------------------|
| Router0 WAN | 203.0.113.2/30 | Gi0/1 side                              |
| Server0  | 203.0.113.1/30  | Acts as "internet" and DNS responder    |
| Server1  | 192.168.10.10/24 | Internal server (VLAN 10)              |
| Printer0 | 192.168.20.10/24 | Networked printer (VLAN 20)            |

### DHCP-assigned devices

PC0 (VLAN 10), PC1 (VLAN 20), PC2 (VLAN 20), and Laptop0 (VLAN 30) all receive their IPs from Router0's DHCP pools.

---

## 4. Phase 1: Build the Baseline Network

**Goal:** an all-green topology with full connectivity across VLANs and simulated internet access.

Before you start: create a working folder on your computer (for example `C:\Users\YourName\packet-tracer-soho-lab\`). You'll save all `.pkt` files and screenshots into subfolders there. See §7 for the recommended structure.

### 4.1 Place the devices

1. Open Packet Tracer. **File → Save As** `SOHO-Lab-01-Build.pkt` in your working folder.
2. Drag the following devices onto the workspace in this order. Keep all default names (PT auto-numbers each type starting at 0, which matches this guide exactly).

| Drag from toolbox                          | You get       |
|--------------------------------------------|---------------|
| Routers → `2911`                            | Router0       |
| Switches → `2960`                           | Switch0       |
| Wireless Devices → `AP-PT`                  | Access Point0 |
| End Devices → `Server-PT` (drop twice)      | Server0, Server1 |
| End Devices → `PC-PT` (drop three times)    | PC0, PC1, PC2 |
| End Devices → `Laptop-PT`                   | Laptop0       |
| End Devices → `Printer-PT`                  | Printer0      |

3. Arrange them so the layout roughly matches the topology diagram (Server0 top, then Router0, then Switch0, with end devices fanning out below).

**Screenshot to take:** all devices placed, no cables yet. Save as `01-build-topology-blank.png`.

### 4.2 Install the wireless NIC on Laptop0

Packet Tracer ships the `Laptop-PT` with an Ethernet module installed by default. You need to swap it for the `WPC300N` wireless card so Laptop0 can associate with Access Point0.

1. Click **Laptop0 → Physical** tab.
2. Click the yellow power button on the laptop image to **power off**. You cannot swap modules while powered on.
3. In the **Physical Device View** on the right, find the Ethernet module installed on the laptop's side. Click and drag it out of its slot and drop it back into the **Modules** panel on the left. This uninstalls it.
4. From the **Modules** panel on the left, drag `WPC300N` into the now-empty slot.
5. Click the yellow power button again to **power back on**.
6. Confirm a small wireless card is visible sticking out of the laptop's left side.

### 4.3 Cable the network

Use **Copper Straight-Through** cable for every connection. You can pick it manually from the Connections toolbox, or use the lightning-bolt auto-connector.

| From     | Port           | To             | Port            |
|----------|----------------|----------------|-----------------|
| Router0  | Gi0/1          | Server0        | FastEthernet0   |
| Router0  | Gi0/0          | Switch0        | Gi0/1           |
| Switch0  | Fa0/1          | PC0            | FastEthernet0   |
| Switch0  | Fa0/5          | PC1            | FastEthernet0   |
| Switch0  | Fa0/6          | PC2            | FastEthernet0   |
| Switch0  | Fa0/10         | Server1        | FastEthernet0   |
| Switch0  | Fa0/15         | Access Point0  | Port 0          |
| Switch0  | Fa0/20         | Printer0       | FastEthernet0   |

Laptop0 does not get a cable. It connects wirelessly in step 4.11.

**Expected immediately after cabling:**
- Switch0-to-end-device cables all show green link lights right away.
- Router0-to-Server0 and Router0-to-Switch0 links show **red** link lights. This is normal. Cisco router interfaces default to `shutdown`. Step 4.4 brings them up with `no shutdown`.

You'll capture the final cabled topology screenshot after step 4.4 when all lights are green. Skip `02-build-topology-cabled.png` for now.

### 4.4 Configure Router0

Click **Router0 → CLI** tab. Press Enter to get a prompt. Paste the following block in one piece.

```
enable
configure terminal
hostname Router0
no ip domain-lookup
!
! --- Trunk to switch (router-on-a-stick) ---
interface GigabitEthernet0/0
 description Trunk to Switch0
 no shutdown
!
interface GigabitEthernet0/0.10
 description VLAN 10 - IT
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
!
interface GigabitEthernet0/0.20
 description VLAN 20 - Staff
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
!
interface GigabitEthernet0/0.30
 description VLAN 30 - Guest
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
!
! --- WAN uplink ---
interface GigabitEthernet0/1
 description WAN to ISP
 ip address 203.0.113.2 255.255.255.252
 no shutdown
!
! --- Default route to "internet" (Server0) ---
ip route 0.0.0.0 0.0.0.0 203.0.113.1
!
end
```

Then paste this second block to add the DHCP pools. Note `dns-server 203.0.113.1` points at Server0 (your lab's DNS, configured later in step 4.7).

```
configure terminal
!
ip dhcp excluded-address 192.168.10.1 192.168.10.10
ip dhcp excluded-address 192.168.20.1 192.168.20.10
ip dhcp excluded-address 192.168.30.1 192.168.30.10
!
ip dhcp pool VLAN10-IT
 network 192.168.10.0 255.255.255.0
 default-router 192.168.10.1
 dns-server 203.0.113.1
 domain-name soho.local
!
ip dhcp pool VLAN20-STAFF
 network 192.168.20.0 255.255.255.0
 default-router 192.168.20.1
 dns-server 203.0.113.1
 domain-name soho.local
!
ip dhcp pool VLAN30-GUEST
 network 192.168.30.0 255.255.255.0
 default-router 192.168.30.1
 dns-server 203.0.113.1
 domain-name soho.local
!
end
write memory
```

**Expected:** you'll see link-up messages for Gi0/0 and Gi0/1. The sub-interfaces may print a "LINK-3-UPDOWN: changed state to down" then "LINEPROTO: changed state to up" sequence. The "LINEPROTO up" is what matters; ignore the intermediate "down" message. The trunk will fully stabilize once Switch0 is configured in step 4.5.

After the second block you should see `Building configuration... [OK]`.

**Now switch to the Logical workspace view and confirm all three Router0 cables show green link lights.** Take both of these screenshots:

- `02-build-topology-cabled.png`: Logical view with every link green.
- `03-build-router0-running-config-a.png` and `03-build-router0-running-config-b.png`: run `show running-config` in the CLI and capture the output in two screenshots (scroll between them). The output is too long for a single capture. See §7 for the multi-part naming convention.

### 4.5 Configure Switch0

Click **Switch0 → CLI** tab. Paste this single block.

```
enable
configure terminal
hostname Switch0
!
vlan 10
 name IT
vlan 20
 name STAFF
vlan 30
 name GUEST
!
! --- Trunk to Router0 ---
interface GigabitEthernet0/1
 description Trunk to Router0
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30
!
! --- Access ports ---
interface FastEthernet0/1
 description PC0 IT
 switchport mode access
 switchport access vlan 10
!
interface FastEthernet0/5
 description PC1 Staff
 switchport mode access
 switchport access vlan 20
!
interface FastEthernet0/6
 description PC2 Staff
 switchport mode access
 switchport access vlan 20
!
interface FastEthernet0/10
 description Server1 IT
 switchport mode access
 switchport access vlan 10
!
interface FastEthernet0/15
 description Access Point0 Guest
 switchport mode access
 switchport access vlan 30
!
interface FastEthernet0/20
 description Printer0 Staff
 switchport mode access
 switchport access vlan 20
!
end
write memory
```

**Expected:** the Gi0/1 trunk port may briefly flap (LINEPROTO down, then up) as it renegotiates from access to trunk mode. It ends in the "up" state. You'll see `Building configuration... [OK]` after `write memory`.

Take these two screenshots:

- `04-build-switch0-vlan-brief.png`: run `show vlan brief` and capture. You should see VLANs 10, 20, 30 with the assigned ports listed under each.
- `05-build-switch0-interfaces-status.png`: run `show interfaces status` and capture. Gi0/1 should show `trunk`, and Fa0/1, Fa0/5, Fa0/6, Fa0/10, Fa0/15, Fa0/20 should all show `connected`.

### 4.6 Configure Access Point0

1. Click **Access Point0 → Config** tab → **Port 1** (the wireless interface, not Port 0 which is the Ethernet uplink).
2. Set:
   - **SSID:** `SOHO-Guest`
   - **Authentication:** **WPA2-PSK**
   - **PSK Pass Phrase:** `GuestPass123`
3. Click anywhere else in the panel to save.

**Screenshot:** `06-build-accesspoint0-wireless-config.png` showing the AP's config panel with SSID and authentication settings.

### 4.7 Configure Server0 (internet simulator)

Server0 plays the role of "the internet" in this lab. It also hosts the DNS service that answers queries for `google.com`.

1. Click **Server0 → Config** tab → **FastEthernet0**. Set:
   - **IP Address:** `203.0.113.1`
   - **Subnet Mask:** `255.255.255.252`
2. Click **Config** tab → **Settings** (the top-level settings). Under **Gateway/DNS IPv4**, set:
   - **Gateway:** `203.0.113.2` (Router0's WAN interface)
3. Click **Services** tab → **DNS**. Set:
   - **DNS Service:** **On**
4. Add a DNS A record:
   - **Name:** `google.com`
   - **Type:** A Record
   - **Address:** `203.0.113.1`
   - Click **Add**.

**Why the DNS record points at Server0 itself:** in a real network DNS would resolve `google.com` to a public Google IP. This lab has no public internet. Pointing `google.com` at Server0's own IP (`203.0.113.1`) means `ping google.com` from any client will succeed and hit Server0, which is exactly the behavior you want for testing "can this client resolve and reach the internet?"

### 4.8 Configure Server1 (internal server)

1. Click **Server1 → Desktop** tab → **IP Configuration** → select **Static**.
2. Enter:
   - **IPv4 Address:** `192.168.10.10`
   - **Subnet Mask:** `255.255.255.0`
   - **Default Gateway:** `192.168.10.1`
   - **DNS Server:** `203.0.113.1`

### 4.9 Configure Printer0

Printer0 uses a slightly different config path from the PCs.

1. Click **Printer0 → Config** tab → **FastEthernet0**. Set:
   - **IP Address:** `192.168.20.10`
   - **Subnet Mask:** `255.255.255.0`
2. Click **Config** tab → **Settings**. Under **Gateway/DNS IPv4**, set:
   - **Gateway:** `192.168.20.1`

### 4.10 Configure the PCs

For each of **PC0**, **PC1**, **PC2**:

1. Click the PC → **Desktop** tab → **IP Configuration**.
2. Select **DHCP**.
3. Wait for the status line to show `DHCP request successful`. The IP, mask, gateway, and DNS fields auto-populate.

**Expected IPs** (approximate, depends on lease order):
- PC0 (VLAN 10): `192.168.10.11`
- PC1 (VLAN 20): `192.168.20.11`
- PC2 (VLAN 20): `192.168.20.12`

**Screenshot:** `08-build-pc0-ipconfig.png` showing PC0's `ipconfig /all` output with DHCP-assigned address and DNS server `203.0.113.1`. (Run `ipconfig /all` from Desktop → Command Prompt.)

### 4.11 Configure Laptop0

1. Click **Laptop0 → Desktop** tab → **PC Wireless**.
2. Click the **Connect** tab.
3. Click **Refresh**. `SOHO-Guest` appears in the list.
4. Select `SOHO-Guest` and click **Connect**. Enter passphrase `GuestPass123`.
5. Wait for the status to show "Connected."
6. Go back to Desktop → **IP Configuration** → select **DHCP**. Laptop0 picks up an address in `192.168.30.0/24`.

**Screenshot:** `07-build-laptop0-wireless-associated.png` showing the PC Wireless panel with connected status.

---

## 5. Phase 2: Verify the Baseline Works

Before breaking anything, prove end-to-end connectivity. These tests become your "baseline evidence" screenshots for a portfolio.

Click **PC0 → Desktop** tab → **Command Prompt**, then run each test.

| # | Command                                  | Expected result              |
|---|------------------------------------------|------------------------------|
| 1 | `ping 192.168.10.10` (Server1, same VLAN)| 4/4 replies, 0% loss         |
| 2 | `ping 192.168.20.10` (Printer0, VLAN 20) | 4/4 replies, 0% loss         |
| 3 | `ping 192.168.10.1` (gateway)            | 4/4 replies, 0% loss         |
| 4 | `ping 203.0.113.1` (Server0, external)   | 4/4 replies, 0% loss         |
| 5 | `nslookup google.com`                    | Returns `Address: 203.0.113.1`|
| 6 | `tracert 203.0.113.1`                    | Completes in 2 hops (Router0 → Server0) |

> **Expected quirk on tests 2, 4, and 6:** the first packet may time out due to ARP resolution delay. Run the command a second time for clean 4/4 results before taking the portfolio screenshot.

**Also verify Laptop0 has wireless connectivity:** click **Laptop0 → Desktop** tab → **Command Prompt**, run `ping 203.0.113.1`, confirm 4/4 replies.

**Screenshots to take:**

- `09-baseline-pc0-ping-tests.png`: PC0 Command Prompt showing the four ping tests (tests 1–4).
- `10-baseline-pc0-nslookup.png`: nslookup result.
- `11-baseline-pc0-tracert.png`: tracert result.
- `12-baseline-laptop0-wireless-test.png`: Laptop0 pinging `203.0.113.1` over Wi-Fi.

### Save the baseline

**File → Save As** `SOHO-Lab-01-Build.pkt`. This is your "known-good" file. You'll open it and save as a new file name for each ticket in Phase 3, so that you can always revert to a working network state.

---

## 6. Phase 3: Troubleshooting Tickets

Eight scenarios. Each one is a realistic help-desk ticket that maps to a common fault type a Tier 1 technician resolves regularly.

### Ticket workflow

Follow the same pattern for every ticket:

1. **Open `SOHO-Lab-01-Build.pkt` → File → Save As** `SOHO-Lab-01-t0X.pkt` (where X is the ticket number). This creates a new file that starts from the working baseline.
2. **Apply the break** exactly as documented.
3. **Save the file now with Ctrl+S.** This freezes the broken state on disk. If you're building this for a portfolio, that saved file is what visitors download and open to see the scenario.
4. **Play the user.** Read the symptom, write down clarifying questions you'd ask at ticket intake.
5. **Diagnose.** Run commands in OSI-model order, take screenshots at each step.
6. **Identify the root cause.** Phrase it in one sentence a non-technical manager would understand.
7. **Apply the fix.** Take a screenshot of the fix command or setting.
8. **Verify.** Confirm the original symptom is gone with a specific test.
9. **Close the file without saving again.** The `.pkt` on disk keeps the broken state; your writeup and screenshots document the fix.

### Ticket writeup template

Use this structure for each ticket in your eventual README:

```
### Ticket #T0X: "[Short title]"

**Affected user/device:** [who]
**Severity:** Sev-A / Sev-B / Sev-C
**Packet Tracer file:** SOHO-Lab-01-t0X.pkt

**Reported symptom:** [what the user said, in quotes]

**Clarifying questions:**
- [question and the answer]

**Diagnosis:**
1. [command] → [result] → [what this tells me]
2. ...

**Root cause:** [one sentence]

**Fix:** [what you did]

**Verification:** [how you confirmed the fix worked]

**Screenshots:** [list]
```

---

### T01: Wrong default gateway

**Affected user:** PC0 (IT, VLAN 10)
**Severity:** Sev-C (single user)

**Break:** On PC0, Desktop → IP Configuration → switch from **DHCP** to **Static**. Enter:
- IP Address: `192.168.10.50`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `192.168.10.99`  *(intentionally wrong; the real gateway is `.1`)*
- DNS Server: `203.0.113.1`

Save the file with Ctrl+S.

**Symptom (user's words):** *"Everything was working yesterday. This morning nothing loads: not Outlook, not the shared drive, not websites."*

**Diagnostic steps:**

1. PC0 → Command Prompt → `ipconfig /all`. Default Gateway shows `192.168.10.99` instead of the expected `192.168.10.1`.
2. `ping 192.168.10.99` returns 4 timeouts. The configured gateway does not exist on the network.
3. `ping 192.168.10.1` (the real gateway) returns **4/4 replies**. This is the key diagnostic moment. Same-subnet IPs are reachable via ARP at Layer 2 regardless of what the default-gateway field says; only off-subnet traffic goes through the default gateway.
4. `ping 203.0.113.1` (off-subnet) returns 4 timeouts. This is the test that exposes the real impact: the PC tries to forward this packet via its non-existent default gateway and fails.
5. Compare against a working neighbor PC. Neighbor's gateway is a real reachable IP; PC0's is not.

**Root cause:** PC0's default gateway was manually set to `192.168.10.99`, a non-existent address. Same-subnet traffic works because ARP bypasses the default gateway, but anything off-subnet has no route out.

**Fix:** Desktop → IP Configuration → switch back to **DHCP**. Wait for the lease.

**Verification:** `ipconfig /all` shows Default Gateway `192.168.10.1`. `ping 203.0.113.1` returns 4/4 replies.

**Screenshots to take:**
- `t01-01-symptom-ipconfig.png` (bad ipconfig)
- `t01-02-ping-gateway-fail.png` (ping `192.168.10.99` → timeouts)
- `t01-03-ping-gateway-real-success.png` (ping `192.168.10.1` → 4/4)
- `t01-04-ping-offsubnet-fail.png` (ping `203.0.113.1` → timeouts)
- `t01-05-neighbor-compare.png` (neighbor's working ipconfig)
- `t01-06-fix-dhcp-toggle.png` (IP Configuration panel switched to DHCP)
- `t01-07-verify-ping-internet.png` (ping `203.0.113.1` now succeeds)

---

### T02: DHCP pool missing

**Affected users:** PC1 and PC2 (both VLAN 20)
**Severity:** Sev-B (multiple users in one VLAN)

**Break:** On Router0 CLI:

```
enable
configure terminal
no ip dhcp pool VLAN20-STAFF
end
```

Then on PC1, Desktop → Command Prompt:

```
ipconfig /release
ipconfig /renew
```

The renew fails. All IP fields drop to `0.0.0.0` and the prompt prints repeated `DHCP request failed.` messages. Save the file.

> In real Windows, a PC in this state would fall back to an APIPA address (`169.254.x.x`). Packet Tracer shows `0.0.0.0` instead; both indicate the same thing: no DHCP response received.

**Symptom:** *"My computer says 'limited connectivity' and my IP starts with 169. Nothing works."*

**Diagnostic steps:**

1. PC1 → `ipconfig /all`. All address fields show `0.0.0.0`. No DHCP lease.
2. `ipconfig /release` and `ipconfig /renew` both fail with `DHCP request failed.`
3. Test a second PC in the same VLAN. On PC2, run `ipconfig /release` and `ipconfig /renew`. PC2 also fails. Two PCs in the same VLAN failing DHCP points upstream, not to the PCs.
4. On Router0 CLI → `show ip dhcp pool`. Only `VLAN10-IT` and `VLAN30-GUEST` are listed. `VLAN20-STAFF` is gone.

**Root cause:** The DHCP pool for VLAN 20 was removed from Router0. Without a pool, the router ignores DHCPDISCOVER broadcasts from VLAN 20 clients.

**Fix:** On Router0 CLI, recreate the pool:

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

**Verification:** `show ip dhcp pool` lists all three pools. On PC1, `ipconfig /renew` succeeds with an address in `192.168.20.0/24`. `ping 192.168.20.1` and `ping 203.0.113.1` both return 4/4 replies.

**Screenshots to take:**
- `t02-01-symptom-pc1-no-ip-ipconfig.png`
- `t02-02-pc1-renew-fail.png`
- `t02-03-pc2-also-no-ip.png`
- `t02-04-router0-show-ip-dhcp-pool-missing.png`
- `t02-05-fix-router0-readd-pool.png`
- `t02-06-verify-pc1-renew-success.png`
- `t02-07-verify-pc1-pings.png`

---

### T03: Bad DNS

**Affected user:** PC2 (VLAN 20)
**Severity:** Sev-C

**Break:** On PC2, Desktop → IP Configuration → switch to Static. Enter:
- IP Address: `192.168.20.50`
- Subnet Mask: `255.255.255.0`
- Default Gateway: `192.168.20.1`
- DNS Server: `10.10.10.10`  *(intentionally wrong; no DNS server exists at this IP)*

Save the file.

**Symptom:** *"Ping works when I use numbers but I can't open any websites. Every URL gives me 'could not find host.'"*

**Diagnostic steps:**

1. PC2 → `ping 203.0.113.1`. 4/4 replies. Network path works.
2. `ping google.com`. Returns `Ping request could not find host google.com.` The PC never even sent the ping; name resolution failed at step zero.
3. The pair of results above is the fingerprint of a DNS problem: Layer 3 through 4 work, Layer 7 (DNS) is broken.
4. `nslookup google.com`. Timeout error. Confirms the configured DNS server is unreachable.
5. `ipconfig /all`. DNS Server field shows `10.10.10.10`, an IP that doesn't exist on this network.

**Root cause:** PC2's DNS server is statically set to `10.10.10.10`, which is not reachable. DNS queries are sent and never answered.

**Fix:** Desktop → IP Configuration → switch back to DHCP. The PC receives DNS Server `203.0.113.1` from Router0's DHCP pool.

**Verification:** `ipconfig /all` shows DNS Server `203.0.113.1`. `nslookup google.com` returns `Address: 203.0.113.1`. `ping google.com` returns 4/4 replies.

**Screenshots to take:**
- `t03-01-symptom-ping-ip-success.png`
- `t03-02-ping-name-fails.png`
- `t03-03-nslookup-timeout.png`
- `t03-04-ipconfig-bad-dns.png`
- `t03-05-fix-dhcp-restored.png`
- `t03-06-verify-nslookup-success.png`

---

### T04: Wrong Wi-Fi passphrase

**Affected user:** Laptop0 (guest, VLAN 30)
**Severity:** Sev-C

**Break:** On **Access Point0 → Config** tab → **Port 1**. Keep the SSID as `SOHO-Guest` (unchanged) and keep Authentication as WPA2-PSK. Change the **PSK Pass Phrase** from `GuestPass123` to `WrongPass456`. Click elsewhere to save.

Save the file.

**Symptom (from a visiting contractor):** *"I can see 'SOHO-Guest' on my laptop and I'm typing in the password from the guest card. It just never connects. Did you change the password?"*

**Diagnostic steps:**

1. On Laptop0, PC Wireless → Connect tab → Refresh. `SOHO-Guest` appears in the list. The AP is broadcasting correctly; this rules out an SSID problem (see T07 for the SSID-missing variant).
2. Select `SOHO-Guest` → Connect → enter `GuestPass123`. **Packet Tracer fails this authentication silently.** No error dialog, no popup. The panel just doesn't show a "Connected" indicator.
3. Confirm the silent failure via `ipconfig /all` on Laptop0. All wireless interface fields show `0.0.0.0`. No DHCP lease means no successful association.
4. On Access Point0 → Config → Port 1. PSK Pass Phrase shows `WrongPass456` instead of the documented `GuestPass123`.

**Root cause:** The WPA2 pre-shared key on Access Point0 was changed from the documented value. Users entering the old correct key can no longer authenticate.

**Fix:** On Access Point0 → Config → Port 1. Change PSK Pass Phrase back to `GuestPass123`.

**Verification:** On Laptop0, PC Wireless → Connect → Refresh → select `SOHO-Guest` → enter `GuestPass123` → association succeeds. Desktop → IP Configuration → DHCP → lease assigned in `192.168.30.0/24`. `ping 203.0.113.1` returns 4/4 replies.

**Screenshots to take:**
- `t04-01-symptom-ssid-visible.png`
- `t04-02-symptom-no-connected-indicator.png`
- `t04-03-symptom-laptop-ipconfig-no-ip.png`
- `t04-04-ap-config-wrong-psk.png`
- `t04-05-fix-ap-psk-corrected.png`
- `t04-06-verify-laptop-connected.png`
- `t04-07-verify-laptop-ping.png`

> **Why this ticket and T07 are distinct:** T07 breaks the SSID (the network isn't visible at all). T04 breaks the PSK (the network is visible but authentication fails). Two different Wi-Fi complaints with two different fix paths. Asking "is the SSID showing up on your device?" distinguishes them in one question.

---

### T05: Switchport in wrong VLAN

**Affected user:** PC2 (VLAN 20, new hire setup)
**Severity:** Sev-C

**Break:** On Switch0 CLI:

```
enable
configure terminal
interface FastEthernet0/6
 switchport access vlan 99
end
```

Then on PC2, Desktop → Command Prompt → `ipconfig /release` then `ipconfig /renew`. PC2 drops to `0.0.0.0` because VLAN 99 has no DHCP pool and no router sub-interface.

Save the file.

**Symptom (from IT):** *"I just set up the new intern at the desk by the window. Their PC has link lights but can't get online. Everyone else in the cubicle row is fine."*

**Diagnostic steps:**

1. PC2 → `ipconfig /all`. All fields `0.0.0.0`. No DHCP lease.
2. Open the Logical workspace. PC2's cable shows **green** link lights on both ends. Physical and data-link layer to the switch are fine.
3. Green link plus no DHCP on a single PC (while neighbors work) is the fingerprint of a switchport VLAN problem. Contrast with T02, where multiple PCs in the same VLAN all failed DHCP; that pointed at the DHCP pool. Here, only one PC is affected, so check the switchport first.
4. On Switch0 → `show interfaces FastEthernet0/6 switchport`. Access Mode VLAN is `99`, not the expected `20`.
5. On Switch0 → `show vlan brief`. VLAN 99 either doesn't appear in the database or exists with Fa0/6 as its only member. Either way, it is not part of this network's design.

**Root cause:** Switchport Fa0/6 was misassigned to VLAN 99, a VLAN that has no DHCP pool, no router sub-interface, and no trunk membership. The PC is alive on the wire but stranded.

**Fix:**

```
enable
configure terminal
interface FastEthernet0/6
 switchport access vlan 20
end
write memory
```

**Verification:** `show interfaces Fa0/6 switchport` now reports Access Mode VLAN 20. On PC2, `ipconfig /renew` assigns an address in `192.168.20.0/24`. `ping 192.168.20.1` and `ping 203.0.113.1` both return 4/4 replies.

**Screenshots to take:**
- `t05-01-symptom-pc2-no-ip.png`
- `t05-02-topology-green-link.png`
- `t05-03-switch0-show-switchport-vlan99.png`
- `t05-04-switch0-show-vlan-brief-wrong.png`
- `t05-05-fix-switch0-reassign-vlan20.png`
- `t05-06-verify-show-switchport-vlan20.png`
- `t05-07-verify-pc2-renew-and-ping.png`

---

### T06: Switchport administratively down

**Affected device:** Printer0 (VLAN 20)
**Severity:** Sev-B (entire office affected)

**Break:** On Switch0 CLI:

```
enable
configure terminal
interface FastEthernet0/20
 shutdown
end
```

Save the file.

**Symptom:** *"The whole office is complaining. Print jobs are just sitting in the queue. Even the printer's web page won't load."*

**Diagnostic steps:**

1. From PC1, `ping 192.168.20.10` (Printer0's static IP) returns 4 timeouts. Printer is unreachable at Layer 3.
2. In the Logical workspace, the cable between Switch0 and Printer0 shows a **red** link indicator. Physical or link layer is down.
3. On Switch0 → `show interfaces FastEthernet0/20`. The first line reads: `FastEthernet0/20 is administratively down, line protocol is down`. The phrase "administratively down" is the signature of a deliberate `shutdown` command. The printer itself is healthy; the switchport has been disabled at the CLI.

**Root cause:** Switch0's Fa0/20 was left administratively shut down after unrelated maintenance. The printer is cut off at the switch.

**Fix:**

```
enable
configure terminal
interface FastEthernet0/20
 no shutdown
end
write memory
```

**Verification:** `show interfaces Fa0/20` now reads `FastEthernet0/20 is up, line protocol is up`. The cable's link indicator turns green in the Logical view. From PC1, `ping 192.168.20.10` returns 4/4 replies.

**Screenshots to take:**
- `t06-01-symptom-pc1-ping-printer-fail.png`
- `t06-02-topology-red-link-printer.png`
- `t06-03-switch0-show-int-admin-down.png`
- `t06-04-fix-switch0-no-shutdown.png`
- `t06-05-verify-topology-green-link.png`
- `t06-06-verify-pc1-ping-printer-success.png`

---

### T07: Wrong SSID

**Affected user:** Visiting contractor (Laptop0)
**Severity:** Sev-C

**Break:** On **Access Point0 → Config** tab → **Port 1**. Change the **SSID** from `SOHO-Guest` to `SOHO-Guest-TEMP`. Leave authentication and passphrase unchanged. Click elsewhere to save.

Save the file.

**Symptom:** *"Your Wi-Fi name isn't showing up on my laptop. I see 'SOHO-Guest-TEMP' but nothing called 'SOHO-Guest' like the guest card said."*

**Diagnostic steps:**

1. On Laptop0, PC Wireless → Connect tab → Refresh. Only `SOHO-Guest-TEMP` appears in the list. The documented `SOHO-Guest` is absent.
2. On Access Point0 → Config → Port 1. The SSID field reads `SOHO-Guest-TEMP`.

**Root cause:** Access Point0's broadcast SSID was changed without updating the documented guest-card information.

**Fix:** On Access Point0 → Config → Port 1. Change the SSID back to `SOHO-Guest`.

**Verification:** On Laptop0, PC Wireless → Refresh. `SOHO-Guest` appears. Select it, enter passphrase `GuestPass123`, connect. DHCP assigns an address in `192.168.30.0/24`. `ping 203.0.113.1` returns 4/4 replies.

**Screenshots to take:**
- `t07-01-symptom-laptop-wrong-ssid-list.png`
- `t07-02-ap-config-wrong-ssid.png`
- `t07-03-fix-ap-ssid-corrected.png`
- `t07-04-verify-laptop-ssid-visible.png`
- `t07-05-verify-laptop-connected.png`
- `t07-06-verify-laptop-ping.png`

---

### T08: WAN interface down

**Affected users:** All clients (full network)
**Severity:** Sev-A (total external outage)

**Break:** On Router0 CLI:

```
enable
configure terminal
interface GigabitEthernet0/1
 shutdown
end
```

Save the file. This mirrors a real-world scenario where an admin disables the WAN interface during maintenance and forgets to bring it back up.

**Symptom:** *"Shared drive works, internal sites work, but nobody can reach anything external. Our ping tests to 203.0.113.1 time out."*

**Diagnostic steps:**

1. From PC0, `ping 192.168.20.10` (cross-VLAN internal) returns 4/4 replies. Internal routing is intact.
2. `ping 203.0.113.1` returns 4 timeouts. External is specifically broken.
3. `tracert 203.0.113.1` reaches `192.168.10.1` (Router0's LAN gateway) on hop 1 and then dies. Tracert reaching the edge router and stopping there means the problem is on the router: either the egress interface is down, there's no route for the destination, or an ACL is dropping the traffic.
4. On Router0 → `show ip interface brief`. `GigabitEthernet0/1` is marked `administratively down / down`. All internal sub-interfaces are up.
5. On Router0 → `show ip route`. Neither the connected route `C 203.0.113.0/30` nor the default route `S* 0.0.0.0/0` appears. When Gi0/1 went down, the connected route disappeared; the default route depends on that connected route's next-hop reachability, so it was removed from the routing table as well.

**Root cause:** Router0's WAN interface (`GigabitEthernet0/1`) is administratively shut down. The connected route to `203.0.113.0/30` is gone, and the static default route becomes unreachable and drops from the routing table.

**Fix:**

```
enable
configure terminal
interface GigabitEthernet0/1
 no shutdown
end
write memory
```

**Verification:** `show ip interface brief` shows Gi0/1 as `up / up`. `show ip route` lists both `C 203.0.113.0/30 is directly connected, GigabitEthernet0/1` and `S* 0.0.0.0/0 [1/0] via 203.0.113.1`. On PC0, `ping 203.0.113.1` returns 4/4 replies; `tracert 203.0.113.1` completes in 2 hops.

**Screenshots to take:**
- `t08-01-symptom-pc0-internal-ping-success.png`
- `t08-02-symptom-pc0-external-ping-fail.png`
- `t08-03-pc0-tracert-dies-at-router0.png`
- `t08-04-router0-show-ip-int-brief-gi01-down.png`
- `t08-05-router0-show-ip-route-missing-default.png`
- `t08-06-fix-router0-no-shutdown-gi01.png`
- `t08-07-verify-router0-show-ip-route-restored.png`

---

## 7. Documenting for a Portfolio

If you plan to publish this lab as a GitHub portfolio, use the folder layout and naming conventions below. They give a clean repository that a recruiter can skim in 30 seconds.

### Recommended folder structure

```
packet-tracer-soho-lab/
├── README.md                                   (overview + links to each ticket)
├── tickets/                                    (one markdown file per ticket)
│   ├── T01-wrong-gateway.md
│   ├── T02-dhcp-pool-missing.md
│   ├── T03-bad-dns.md
│   ├── T04-wrong-wifi-psk.md
│   ├── T05-wrong-vlan.md
│   ├── T06-port-shutdown.md
│   ├── T07-wrong-ssid.md
│   └── T08-wan-interface-down.md
├── packet-tracer-files/                        (Packet Tracer save files)
│   ├── SOHO-Lab-01-Build.pkt                   known-good working state
│   ├── SOHO-Lab-01-t01.pkt                     broken state for T01
│   ├── SOHO-Lab-01-t02.pkt                     broken state for T02
│   ├── SOHO-Lab-01-t03.pkt
│   ├── SOHO-Lab-01-t04.pkt
│   ├── SOHO-Lab-01-t05.pkt
│   ├── SOHO-Lab-01-t06.pkt
│   ├── SOHO-Lab-01-t07.pkt
│   └── SOHO-Lab-01-t08.pkt
└── screenshots/                                (all evidence captures)
    ├── 01-build-*.png through 12-baseline-*.png
    └── t01-*.png through t08-*.png
```

### Screenshot naming scheme

**Format:** `[phase/prefix]-[order]-[short-description].png`

| Prefix                   | Meaning                                   | Example                                      |
|--------------------------|-------------------------------------------|----------------------------------------------|
| `01-` through `08-`      | Build phase, zero-padded for sort order   | `03-build-router0-running-config-a.png`      |
| `09-` through `12-`      | Baseline verification                     | `10-baseline-pc0-nslookup.png`               |
| `t0X-0Y-...`             | Ticket X, step Y within that ticket       | `t03-04-ipconfig-bad-dns.png`                |
| `...-a.png`, `...-b.png` | Multi-part capture (output split across images) | `03-build-router0-running-config-a.png` and `-b.png` |

**Rules:**

1. All lowercase, hyphens only, no spaces or underscores.
2. Zero-pad ticket numbers (`t01`, not `t1`) so file explorers sort correctly.
3. Step order within a ticket: `01-symptom`, `02-diagnose-*`, `03-rootcause`, `04-fix`, `05-verify`.
4. Keep descriptions short (3–5 words) but unique. `t03-02-ping-name-fails.png` beats `t03-02-ping.png`.
5. Save as PNG, not JPG. Text stays crisp.
6. For captures that can't fit in one screenshot (long `show running-config`, all six baseline ping tests, etc.), keep the base filename identical and append `-a`, `-b`, `-c`, ordered top-to-bottom.

### How to save the broken state in each ticket's `.pkt` file

For each ticket:

1. Open `SOHO-Lab-01-Build.pkt`.
2. File → Save As `SOHO-Lab-01-t0X.pkt`.
3. Apply the break.
4. **Ctrl+S.** This is the critical step: it freezes the broken state on disk.
5. Take all symptom and diagnostic screenshots.
6. Apply the fix and take verification screenshots.
7. **Close the file without saving again.** The `.pkt` on disk stays broken. Your markdown writeup documents the fix.

**Why save the broken state:** a recruiter who opens `SOHO-Lab-01-t03.pkt` should immediately see the misconfigured DNS server. That is self-evident proof that you built, understood, and preserved the scenario. A fixed `.pkt` looks identical to the baseline and carries no signal.

### Master screenshot checklist

Total: about 65 screenshots.

**Build phase (8):**
- `01-build-topology-blank.png`
- `02-build-topology-cabled.png`
- `03-build-router0-running-config-a.png` and `03-build-router0-running-config-b.png`
- `04-build-switch0-vlan-brief.png`
- `05-build-switch0-interfaces-status.png`
- `06-build-accesspoint0-wireless-config.png`
- `07-build-laptop0-wireless-associated.png`
- `08-build-pc0-ipconfig.png`

**Baseline verification (4):**
- `09-baseline-pc0-ping-tests.png`
- `10-baseline-pc0-nslookup.png`
- `11-baseline-pc0-tracert.png`
- `12-baseline-laptop0-wireless-test.png`

**Tickets (approx. 53):**
- T01: 7 screenshots
- T02: 7 screenshots
- T03: 6 screenshots
- T04: 7 screenshots
- T05: 7 screenshots
- T06: 6 screenshots
- T07: 6 screenshots
- T08: 7 screenshots

---

## 8. Packet Tracer Gotchas

These are real differences between Packet Tracer's simulation and real-world Windows or Cisco IOS behavior. Worth calling out in a portfolio writeup to show that you understand the distinction.

### 1. `0.0.0.0` instead of APIPA when DHCP fails

In real Windows, a PC that doesn't receive a DHCP response falls back to an APIPA address in the `169.254.0.0/16` link-local range. Packet Tracer skips that behavior and just leaves every IP field at `0.0.0.0`, with the Command Prompt printing repeated `DHCP request failed.` messages. Both states mean the same underlying thing: the DHCP exchange didn't complete.

### 2. Proxy ARP is enabled by default on Cisco routers

When a host sends an ARP request for an IP on a different subnet, a Cisco router with Proxy ARP enabled will respond with its own MAC on behalf of the remote host. That means some client-side misconfigurations (wrong subnet mask, for example) that should theoretically break cross-subnet pings work anyway because the router transparently bridges the ARP gap.

This is real Cisco default behavior, not a Packet Tracer quirk. A help-desk technician should know it exists. If you want to disable it for testing, use `no ip proxy-arp` under the relevant sub-interface.

### 3. Duplicate-IP static configurations are blocked at input

If you try to assign a static IP that another active device on the network is already using, Packet Tracer rejects the value and displays *"This address is already used in the network."* This mimics how some modern OS network stacks do ARP probing before committing a static IP, but it means the classic "two devices with the same IP" help-desk ticket can't be reproduced in PT the traditional way. If you see that error while following this guide, check whether you've accidentally picked a pool-assigned address or another static IP.

### 4. First-packet loss on ping

The first ICMP packet to a new destination often times out while ARP resolution completes at one or more hops. This is not a configuration problem. Run the ping a second time for clean 4/4 results before taking a portfolio screenshot.

### 5. Sub-interface link messages during trunk setup

When you bring up a router's trunk sub-interfaces, you'll see messages like `%LINK-3-UPDOWN: Interface GigabitEthernet0/0.10, changed state to down` followed by `%LINEPROTO-5-UPDOWN: Line protocol on Interface GigabitEthernet0/0.10, changed state to up`. The LINEPROTO "up" message is what matters. The earlier "down" is an artifact of PT's event timing while the parent interface's trunk negotiates with the switch side.

---

## 9. Appendix: Quick Reference

### CLI commands used in this lab

**Router0:**

| Command                                   | Purpose                                   |
|-------------------------------------------|-------------------------------------------|
| `enable`                                  | Enter privileged EXEC mode                |
| `configure terminal`                      | Enter global config mode                  |
| `hostname [name]`                         | Set device name                           |
| `interface [interface-id]`                | Enter interface config                    |
| `encapsulation dot1Q [vlan-id]`           | Set 802.1Q tag on sub-interface           |
| `ip address [ip] [mask]`                  | Assign IP to an interface                 |
| `no shutdown` / `shutdown`                | Bring interface up / down                 |
| `ip dhcp pool [name]`                     | Start a DHCP pool definition              |
| `ip dhcp excluded-address [start] [end]`  | Reserve IPs from DHCP assignment          |
| `ip route [network] [mask] [next-hop]`    | Add a static route                        |
| `show ip route`                           | Show routing table                        |
| `show ip interface brief`                 | One-line summary of every interface       |
| `show ip dhcp pool`                       | Show DHCP pool configuration and bindings |
| `write memory`                            | Save running config to startup config     |

**Switch0:**

| Command                                    | Purpose                                  |
|--------------------------------------------|------------------------------------------|
| `vlan [id]` → `name [name]`                | Create a VLAN                            |
| `switchport mode access`                   | Set port to access mode                  |
| `switchport access vlan [id]`              | Assign port to a VLAN                    |
| `switchport mode trunk`                    | Set port to trunk mode                   |
| `switchport trunk allowed vlan [list]`     | Restrict which VLANs cross the trunk     |
| `show vlan brief`                          | List VLANs and member ports              |
| `show interfaces [id] switchport`          | Detailed switchport config for a port    |
| `show interfaces status`                   | One-line status of every port            |

**Client (Windows Command Prompt):**

| Command                 | Purpose                                               |
|-------------------------|-------------------------------------------------------|
| `ipconfig`              | Show basic IP configuration                           |
| `ipconfig /all`         | Show full IP configuration, including DNS and MAC     |
| `ipconfig /release`     | Release current DHCP lease                            |
| `ipconfig /renew`       | Request a new DHCP lease                              |
| `ping [ip-or-name]`     | Send ICMP echo requests                               |
| `tracert [ip-or-name]`  | Show the routed path to a destination                 |
| `nslookup [name]`       | Query DNS for a name's IP                             |
| `arp -a`                | Show the local ARP cache                              |

### Port assignments on Switch0

| Port   | Device        | VLAN | Mode     |
|--------|---------------|------|----------|
| Gi0/1  | Router0 Gi0/0 | All  | Trunk    |
| Fa0/1  | PC0           | 10   | Access   |
| Fa0/5  | PC1           | 20   | Access   |
| Fa0/6  | PC2           | 20   | Access   |
| Fa0/10 | Server1       | 10   | Access   |
| Fa0/15 | Access Point0 | 30   | Access   |
| Fa0/20 | Printer0      | 20   | Access   |

### IP addresses used in the lab

| IP                | Device / role                          |
|-------------------|----------------------------------------|
| 192.168.10.1      | Router0 Gi0/0.10 (VLAN 10 gateway)     |
| 192.168.10.10     | Server1 (static)                       |
| 192.168.10.11+    | PC0 (DHCP), other VLAN 10 devices      |
| 192.168.20.1      | Router0 Gi0/0.20 (VLAN 20 gateway)     |
| 192.168.20.10     | Printer0 (static)                      |
| 192.168.20.11+    | PC1, PC2 (DHCP)                        |
| 192.168.30.1      | Router0 Gi0/0.30 (VLAN 30 gateway)     |
| 192.168.30.11+    | Laptop0 (DHCP)                         |
| 203.0.113.1       | Server0 (internet simulator / DNS)     |
| 203.0.113.2       | Router0 Gi0/1 (WAN side)               |

### Wireless credentials

| SSID         | Auth       | Passphrase      | VLAN |
|--------------|------------|-----------------|------|
| `SOHO-Guest` | WPA2-PSK   | `GuestPass123`  | 30   |

---

## License and credit

Free to fork, remix, and republish. If this guide helped you, a link back is appreciated.
