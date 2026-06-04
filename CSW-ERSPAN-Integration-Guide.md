# Cisco Secure Workload — ERSPAN Integration Guide

> **Disclaimer:** Community reference guide by Cisco Solutions Engineering. Always consult [official Cisco Secure Workload documentation](https://www.cisco.com/c/en/us/products/security/tetration/index.html) for authoritative guidance.

## Table of Contents
1. [Overview](#1-overview)
2. [Architecture](#2-architecture)
3. [ERSPAN Types Supported](#3-erspan-types-supported)
4. [Use Cases](#4-use-cases)
5. [Prerequisites](#5-prerequisites)
6. [Step A — Configure ERSPAN Source on Switch](#6-step-a--configure-erspan-source-on-switch)
7. [Step B — Deploy the CSW Ingest Appliance for ERSPAN](#7-step-b--deploy-the-csw-ingest-appliance-for-erspan)
8. [Performance Considerations](#8-performance-considerations)
9. [Security Considerations](#9-security-considerations)
10. [Verification](#10-verification)
11. [Limits](#11-limits)
12. [Troubleshooting](#12-troubleshooting)
13. [Related Resources](#13-related-resources)

---

## 1. Overview

**ERSPAN (Encapsulated Remote Switch Port Analyzer)** allows Cisco Secure Workload to ingest full **packet-level flow visibility** from network switches by mirroring traffic to a remote analyzer — without any software agent on the monitored hosts.

Unlike NetFlow (which provides aggregated flow summaries), ERSPAN delivers **individual mirrored frames**, providing the richest possible agentless visibility: full L2–L7 context, actual payload sizes, exact timing, and precise TCP flag analysis.

### When to use ERSPAN vs NetFlow vs Agent

| Method | Visibility | Deployment | Best for |
|--------|-----------|-----------|---------|
| **CSW Agent** | Deepest (process, user, socket) | Agent on each host | Modern Linux/Windows workloads |
| **ERSPAN** | Rich (full frames, L7 context) | Switch config + Ingest VM | Legacy servers, mainframes, OT/IoT |
| **NetFlow** | Moderate (flow summaries) | Switch config + Ingest VM | High-rate environments, fabric-wide coverage |

> The ERSPAN Ingest appliance is the **same OVA/QCOW2** as the standard Ingest appliance — it runs **3 ERSPAN connectors internally**, each in a dedicated Docker container with exclusive vCPU and vNIC assignment.

---

## 2. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                    Data Center / Campus Network                        │
│                                                                       │
│  ┌────────────────────────────────────────────┐                      │
│  │  Cisco Nexus 9000 (or Catalyst, ASR, etc.) │                      │
│  │                                             │                      │
│  │  ERSPAN Source Session:                     │                      │
│  │    Source: interfaces eth1/23, VLANs 315,512│                      │
│  │    Destination IP: 172.28.126.194 (Ingest)  │                      │
│  │    GRE encapsulation (ERSPAN Type II)       │                      │
│  └────────────────┬───────────────────────────┘                      │
│                   │  GRE-encapsulated mirrored frames                 │
│                   ▼                                                   │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │    CSW Ingest Appliance (ERSPAN mode)                          │  │
│  │    (Same OVA as standard Ingest)                               │  │
│  │                                                                │  │
│  │    Container 1: ERSPAN connector (vNIC 1) — VLAN A            │  │
│  │    Container 2: ERSPAN connector (vNIC 2) — VLAN B            │  │
│  │    Container 3: ERSPAN connector (vNIC 3) — VLAN C            │  │
│  │                                                                │  │
│  │    Each container: dedicated SPAN agent, 2 vCPUs              │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                   │ Processed flows forwarded to CSW                  │
│  ┌────────────────▼───────────────────────────────────────────────┐  │
│  │    Cisco Secure Workload Cluster                               │  │
│  │    ADM, policy, forensics                                      │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 3. ERSPAN Types Supported

| ERSPAN Type | Description | Default on Cisco |
|-------------|-------------|-----------------|
| **Type I** | Lowest overhead — no ERSPAN header | No |
| **Type II** | Standard — includes session ID, VLAN, CoS; QoS-capable | **Yes (most switches)** |
| **Type III** | Extended — additional metadata (timestamp, etc.) | No |

CSW supports all three types. **Type II is the typical default** on Cisco switches and is recommended. Use the smallest version that meets your needs for lower header overhead.

> CSW SPAN agents process ERSPAN packets but do not extract additional metadata from Type II/III headers beyond what is needed for flow identification.

---

## 4. Use Cases

### Use Case 1 — Legacy Server Visibility (no agent possible)
Mirror traffic from the server's access port on the ToR switch. CSW gets full L2–L7 flow visibility for the server with zero software installation on the server itself.

### Use Case 2 — Mainframe / OT Device Monitoring
For IBM mainframes, SCADA/ICS devices, or embedded systems that cannot run any agent, ERSPAN from the connected switch port provides complete flow data.

### Use Case 3 — East-West Visibility on Critical VLANs
Mirror specific VLANs (e.g., PCI DMZ VLAN, Database VLAN) to an ERSPAN connector for high-fidelity flow analysis of the most sensitive traffic segments.

### Use Case 4 — VMware vSphere / VDS Traffic
ERSPAN supports VMware vSphere Distributed Switch (VDS) non-RFC format — enabling visibility into virtualized workloads without per-VM agents.

---

## 5. Prerequisites

### Network device requirements
- [ ] Cisco switch/router supporting ERSPAN (Nexus 9000, Catalyst 9000, ASR, etc.) or VMware VDS
- [ ] L3 reachability from switch to CSW Ingest appliance IP (ERSPAN is IP-encapsulated GRE)
- [ ] Sufficient bandwidth on the path to the Ingest appliance for the mirrored traffic volume
- [ ] ERSPAN source session available (switches have a limited number of SPAN/ERSPAN sessions)

### CSW Ingest appliance requirements
- [ ] Deployed as an ERSPAN-mode Ingest appliance (same OVA, ERSPAN mode selected at deploy)
- [ ] Each of the 3 internal containers has a dedicated vNIC for ERSPAN reception
- [ ] Ingest appliance registered with CSW cluster
- [ ] 2 vCPUs per container (6 total for ERSPAN VM)

---

## 6. Step A — Configure ERSPAN Source on Switch

### Cisco Nexus 9000 — ERSPAN source session

```nxos
! Create ERSPAN source session
monitor session 10 type erspan-source

! Specify source — interfaces or VLANs (or both)
  source interface Ethernet1/23 both        ! Mirror ingress + egress
  source vlan 315,512                       ! Mirror entire VLAN

! Configure ERSPAN destination (CSW Ingest appliance IP)
  destination ip 172.28.126.194             ! Ingest appliance vNIC IP
  erspan-id 10                              ! Must match on source and destination
  vrf default                               ! VRF for ERSPAN transport

! Set source IP (must be a routable L3 interface on this switch)
  ip ttl 64
  origin ip 172.28.126.1                    ! Switch L3 interface IP
  no shut
```

### Cisco IOS/IOS-XE — ERSPAN source session

```ios
monitor session 1 type erspan-source
 source interface GigabitEthernet1/0/1 both
 source vlan 100
 destination
  erspan-id 101
  ip address 10.x.x.x           ! CSW Ingest appliance IP
  origin ip address 10.y.y.y    ! Switch source IP
  mtu 1464                      ! Account for ERSPAN overhead (~160 bytes)
 no shut
```

### VMware vSphere Distributed Switch (VDS)

1. vCenter → **Networking > [Distributed Switch] > Monitor Sessions**
2. Click **New Monitor Session**
3. Type: **Remote Mirroring Source**
4. Source: select port groups or individual vNICs to mirror
5. Destination: ERSPAN destination IP = CSW Ingest appliance IP
6. ERSPAN ID: set a session ID (1–1023)
7. Click **Finish**

---

## 7. Step B — Deploy the CSW Ingest Appliance for ERSPAN

### B1 — Download and deploy the OVA/QCOW2

The ERSPAN Ingest appliance uses the **same OVA or QCOW2** as a standard Ingest appliance. During deployment, select **ERSPAN mode** in the appliance configuration.

The VM internally runs **3 ERSPAN connector containers**, each with:
- 1 dedicated vNIC
- 2 dedicated vCPUs
- No quota limiting on the cores

### B2 — vNIC assignment

Each container gets exclusive access to one vNIC. The ERSPAN source session destination IP must match the IP assigned to that vNIC.

```
Container 1 → vNIC 1 IP: 172.28.126.194   ← ERSPAN session 10 destination
Container 2 → vNIC 2 IP: 172.28.126.195   ← ERSPAN session 11 destination
Container 3 → vNIC 3 IP: 172.28.126.196   ← ERSPAN session 12 destination
```

### B3 — Register with CSW

After deployment, the ERSPAN Ingest appliance appears in **Manage > Virtual Appliances**. Each container registers a **SPAN Agent** with the cluster — visible in the Agent List.

### B4 — No additional connector configuration needed

Unlike NetFlow, the ERSPAN connector **does not require explicit CSW UI configuration** beyond the appliance deployment. The SPAN agents self-register when the appliance boots and containers start.

---

## 8. Performance Considerations

| Factor | Guidance |
|--------|---------|
| **Source interface selection** | Be selective — each mirrored interface adds traffic load. Avoid mirroring uplinks unless necessary. |
| **VLAN-based mirroring** | Mirror entire VLANs only for low-traffic or critical VLANs |
| **ACL filtering** | Use ERSPAN filter ACLs to limit what is mirrored (reduces load) |
| **MTU** | Set MTU to account for ERSPAN overhead (160+ bytes); typical: `mtu 1464` |
| **ERSPAN version** | Use Type I or II for lower overhead vs Type III |
| **Bandwidth** | Ensure the path to Ingest has sufficient capacity for mirrored traffic |
| **Agent Packet Misses** | Monitor in CSW UI: **Agents > [SPAN Agent] > Packet Misses graph** |

If the SPAN agent receives more packets than 2 vCPUs can process, packet misses will increase. Reduce the ERSPAN source scope or add a second ERSPAN Ingest appliance.

---

## 9. Security Considerations

The ERSPAN Ingest appliance is hardened:
- Guest OS: **AlmaLinux 9.2** (CSW 3.8.1.36+) or CentOS 7.9 (earlier versions)
- OpenSSL server/client packages **removed** from the guest OS
- After container deployment, **no network interfaces exist in the guest OS** (moved into containers)
- Containers: no TCP/UDP ports exposed
- Only console access available to the guest OS VM (no SSH)
- Each container runs `almalinux/9-base:9.2` Docker image

---

## 10. Verification

### Check SPAN agent status in CSW
**Manage > Agents** → filter by type **SPAN Agent**
SPAN agents should appear with status **Active**

### Check packet receipt on switch
```nxos
show monitor session 10
! Confirm: State = up, Destination = 172.28.126.194 (reachable)
```

### Check flow data in CSW
**Observe > Traffic** → filter by the VLAN or subnet you are mirroring
Flows should appear with the full src/dst IP:port detail.

### Monitor packet misses
**CSW UI > Agents > [SPAN Agent] > Deep Visibility Agent page**
Check **Agent Packet Misses** graph — should be near 0.

---

## 11. Limits

| Metric | Limit |
|--------|-------|
| ERSPAN connectors per Ingest appliance | 3 (one per container/vNIC) |
| vCPUs per ERSPAN container | 2 (dedicated, no quota) |
| ERSPAN types supported | Type I, II, III + VMware VDS format |
| SPAN sessions per switch | Platform-dependent (typically 4–8 per switch) |
| Console-only access to VM | SSH/TCP not available on guest OS |

---

## 12. Troubleshooting

| Symptom | Check |
|---------|-------|
| No flows from ERSPAN source | Verify `show monitor session` on switch — state should be Up; ping from switch to Ingest vNIC IP; check GRE is not blocked by any firewall on the path |
| SPAN agent not registering | Check Ingest appliance console — containers should start within 2 minutes of first boot; check CSW cluster connectivity from Ingest |
| High packet miss rate | Reduce ERSPAN source scope; add ERSPAN ACL filter on switch; consider additional Ingest appliance |
| VMware VDS traffic not captured | Confirm VDS version supports ERSPAN; use VDS "Remote Mirroring Source" type (not SPAN) |
| Container not appearing | Reboot Ingest VM; check VM has 6+ vCPUs and 3 vNICs assigned |

---

## 13. Related Resources

| Repository | Description | Best for |
|------------|-------------|---------|
| [csw-netflow-integration](https://github.com/chandrapati/csw-netflow-integration) | NetFlow v9/IPFIX agentless flow ingestion | High-rate environments, fabric-wide |
| [CSW-Secure-Firewall-Integration-Guide](https://github.com/chandrapati/CSW-Secure-Firewall-Integration-Guide) | NSEL from Cisco Secure Firewall | Firewall-inserted flows |
| [CSW-Agent-Installation-Guide](https://github.com/chandrapati/CSW-Agent-Installation-Guide) | Deep-visibility agent for modern workloads | Process-level visibility |
| [csw-servicenow-integration](https://github.com/chandrapati/csw-servicenow-integration) | ServiceNow CMDB labels for agentless workloads | Enriching legacy server inventory |
| [CSW-Policy-Lifecycle](https://github.com/chandrapati/CSW-Policy-Lifecycle) | Build policy from ADM flow data | Policy discovery workflow |

---
*Community reference — Cisco Solutions Engineering. Not an official Cisco product document.*
