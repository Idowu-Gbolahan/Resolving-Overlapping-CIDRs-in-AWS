# Resolving Overlapping VPC CIDRs with AWS Transit Gateway & Sophos Firewall NAT

> **A production-grade cloud network architecture that achieves full bidirectional inter-VPC routing between VPCs sharing identical CIDR blocks, without re-IPing a single workload, modifying a single application, or causing a single minute of downtime.**

---

## The Problem

In enterprise cloud environments, overlapping VPC CIDR blocks are one of the most common, and most consequential, networking problems that infrastructure engineers encounter. They arise from mergers and acquisitions, decentralized team cloud adoption, IaC template reuse, and the absence of centralized IP address governance.

When two VPCs share the same address space, AWS's native connectivity mechanisms break down completely:

- **VPC Peering** rejects the connection outright
- **Transit Gateway** cannot route between them when both advertise the same prefix
- **Re-IPing** one side sounds simple but is, in practice, a months-long infrastructure migration project with significant blast radius, hardcoded IPs in application configs, DNS records, TLS certificates, security groups, IaC state files, and third-party allowlists all need to be found, audited, and updated atomically

This engagement presented exactly that challenge. Two AWS VPCs, provisioned independently across different accounts, both used `10.2.0.0/16`. A business requirement mandated secure, routed communication between workloads in both VPCs to support a centralized **Sophos Zero Trust Network Access (ZTNA)** deployment. Deploying a separate firewall instance in every VPC was technically feasible but cost-prohibitive. Something more elegant was needed.

---

## The Solution

A **hub-and-spoke NAT architecture** using a dedicated Security VPC as a central traffic translation boundary.

Rather than eliminating the CIDR conflict at its source, the architecture inserts a controlled translation layer between the two address spaces. Each VPC is assigned a unique, non-overlapping **translated CIDR range**. A centralized **Sophos Next-Generation Firewall**, deployed in the Security VPC, performs bidirectional **Source NAT (SNAT) and Destination NAT (DNAT)** on all inter-VPC flows. An **AWS Transit Gateway** with a carefully configured appliance-mode attachment provides the routing fabric that steers all traffic through the firewall consistently and symmetrically.

The result: full, routable, inspected, bidirectional connectivity between two VPCs with identical CIDRs, with zero changes to any existing workload, application, or IP address.

---

## Architecture Overview
<img width="921" height="812" alt="Image" src="https://github.com/user-attachments/assets/e6681796-00d1-48d3-99e5-db7f2665dd17" />

**Traffic flow (Barcelona → Madrid):**

```
Barcelona Server (10.2.1.67)
  → Barcelona TGW Attachment
    → Transit Gateway
      → Security TGW Attachment (Appliance Mode)
        → Sophos Firewall LAN Port
          [SNAT: 10.2.1.67 → 172.17.1.67]
          [DNAT: 172.16.1.157 → 10.2.1.157]
        → Transit Gateway (via Security Attachment)
          → Madrid TGW Attachment
            → Madrid Server (10.2.1.157) ✓
```

---

## IP Address Plan

| IP Address / Range | Role |
|---|---|
| `10.2.0.0/16` | Original CIDR — both VPCs (overlapping) |
| `10.2.1.67/32` | Barcelona Server — real IP |
| `10.2.1.157/32` | Madrid Server — real IP |
| `172.17.0.0/16` | Translation CIDR assigned to Barcelona VPC |
| `172.17.1.67/32` | Barcelona Server — translated IP |
| `172.16.0.0/16` | Translation CIDR assigned to Madrid VPC |
| `172.16.1.157/32` | Madrid Server — translated IP |
| `192.168.0.0/16` | Security VPC CIDR (non-overlapping hub) |
| `192.168.2.181` | Security TGW Attachment ENI (firewall gateway IP) |
| `192.168.2.219` | Sophos Firewall — LAN Port (Port A) |
| `192.168.4.214` | Sophos Firewall — WAN Port (Port B) |

---

## Five Non-Negotiable Configuration Requirements

This architecture has five critical constraints. Miss any one of them and the deployment will silently fail, traffic will drop, route asymmetrically, or NAT incorrectly with no obvious error message.

### 1. Disable Source/Destination Check on the Firewall EC2 Instance

By default, AWS drops any packet arriving at an EC2 instance where that instance is neither the source nor the destination. Since the Sophos firewall is forwarding transit traffic on behalf of other hosts, not terminating it, this check must be explicitly disabled. Without this, the firewall discards every packet it is supposed to translate.

### 2. Firewall and Security TGW Attachment Must Share the Same Subnet

The firewall's static routing table must point translated-range traffic toward the Transit Gateway attachment IP as its next-hop. This is only possible if the firewall and the Security TGW attachment are in the same subnet, making the attachment IP directly reachable at Layer 3 without an intermediate hop.

### 3. Configure the TGW Attachment IP as the Firewall's Gateway

The Security TGW attachment's ENI IP (`192.168.2.181`) must be registered as a gateway on the firewall's LAN interface. This is required not only for correct return-traffic forwarding but also for the firewall to monitor the health of the attachment. Without it, the firewall has no next-hop awareness for traffic leaving the Security VPC.

### 4. Appliance Mode Must Be Enabled ONLY on the Security VPC Attachment

AWS Transit Gateway uses flow-based routing across Availability Zones. Without appliance mode, a return flow from a spoke may be processed by a different AZ's TGW attachment than the outbound flow, bypassing the stateful firewall entirely and breaking NAT translation. Appliance mode must be enabled exclusively on the Security VPC attachment, enabling it on the spoke attachments will break the routing symmetry this architecture depends on.

### 5. SNAT and DNAT Must Be Combined in a Single NAT Rule on Sophos XG

Sophos XG Firewall evaluates NAT rules as a single match-and-apply operation per flow. If SNAT and DNAT are configured as separate rules, the firewall applies only one translation per packet, resulting in asymmetric addressing, the source is translated but the destination is not, or vice versa. Both translations must be defined in a single combined NAT rule for each direction of traffic.

---

## Transit Gateway Route Table Design

### Spoke-to-Security Route Table (`Madrid-Barcelona-To-Sec-VPC`)
Applied to the Barcelona and Madrid TGW attachments. A single default route sends all traffic to the Security VPC:

| Destination | Target |
|---|---|
| `0.0.0.0/0` | Security TGW Attachment |

### Security-to-Spokes Route Table (`Sec-vpc-To-Spokes`)
Applied to the Security TGW attachment. Routes translated and real IPs back to the correct spoke:

| Destination | Target |
|---|---|
| `10.2.1.157/32` | Madrid TGW Attachment |
| `10.2.1.67/32` | Barcelona TGW Attachment |
| `172.16.0.0/16` | Madrid TGW Attachment |
| `172.17.0.0/16` | Barcelona TGW Attachment |

---

## Sophos Firewall NAT Rules

Both rules combine SNAT and DNAT in a single policy entry, a requirement of the Sophos XG NAT processing model.

| Rule | Pre-NAT Source | Pre-NAT Destination | Post-NAT Source | Post-NAT Destination |
|---|---|---|---|---|
| `Barcelona-SNAT-and-Madrid-DNAT` | `10.2.1.67` | `172.16.1.157` | `172.17.1.67` | `10.2.1.157` |
| `Madrid-SNAT-and-Barcelona-DNAT` | `10.2.1.157` | `172.17.1.67` | `172.16.1.157` | `10.2.1.67` |

---

## Validation

End-to-end connectivity was confirmed with live traffic in both directions:

```bash
# From Barcelona Server — ping Madrid translated IP
ubuntu@ip-10-2-1-67:~$ ping 172.16.1.157
64 bytes from 172.16.1.157: icmp_seq=1 ttl=61 time=1.55 ms
64 bytes from 172.16.1.157: icmp_seq=2 ttl=61 time=1.16 ms
64 bytes from 172.16.1.157: icmp_seq=3 ttl=61 time=1.05 ms
...
```

```bash
# From Madrid Server — ping Barcelona translated IP
ubuntu@ip-10-2-1-157:~$ ping 172.17.1.67
64 bytes from 172.17.1.67: icmp_seq=1 ttl=61 time=1.68 ms
64 bytes from 172.17.1.67: icmp_seq=2 ttl=61 time=1.15 ms
64 bytes from 172.17.1.67: icmp_seq=3 ttl=61 time=1.15 ms
...
```

Firewall logs confirmed correct rule matching, NAT application, and bidirectional traffic inspection across both ICMP and TCP flows. Round-trip latency consistently sub-2ms, no measurable overhead introduced by the NAT layer.

---

## Business Context & Impact

This architecture was designed in response to a real organizational constraint: the client needed a centralized Sophos ZTNA deployment across multiple VPCs, but per-VPC firewall instances would have multiplied licensing and operational costs beyond budget. The overlapping CIDRs made conventional Transit Gateway routing impossible.

**Alternative approaches evaluated and rejected:**

| Approach | Why It Was Ruled Out |
|---|---|
| Re-IP one VPC | Months-long migration, high blast radius, hardcoded IPs across app configs, DNS, certs, security groups, and IaC state |
| Deploy Sophos in every VPC | Viable but cost-prohibitive; multiplies licensing and management overhead linearly with VPC count |
| AWS PrivateLink | Service-specific, not suitable for general IP routing; doesn't solve the underlying routing problem |
| Overlay network (Aviatrix, Cilium) | Additional vendor dependency and cost; overkill for this topology |

The NAT hub architecture delivered the business outcome, a single centralized firewall serving all VPCs, without any of the migration risk or cost escalation of the alternatives.

---

## Skills Demonstrated

- **AWS Advanced Networking** — Multi-VPC Transit Gateway topology design with custom route tables, appliance mode, and static routing
- **Network Security Engineering** — Sophos NGFW deployment, bidirectional NAT policy design, firewall rule authoring, and gateway health configuration
- **Cloud Architecture** — Hub-and-spoke Security VPC design from first principles for a non-standard routing scenario
- **Root Cause Analysis** — Systematic evaluation of re-IPing vs. NAT remediation paths with cost-risk tradeoff analysis
- **Cost Optimization** — Centralized firewall architecture eliminating the need for per-VPC appliance deployments
- **Zero Trust / ZTNA** — Network foundation design enabling a Sophos ZTNA deployment across a conflicting address space
- **Technical Documentation** — End-to-end architecture documentation with design rationale, configuration reference tables, and validation evidence

---

## Full Case Study

A complete, professionally formatted technical case study document, including architecture diagrams, full configuration walkthrough, and implementation guide, is available in this repository. 

[Full Documentation for this project](https://github.com/Idowu-Gbolahan/Resolving-Overlapping-CIDRs-in-AWS/blob/main/DOCUMENTATION%20FOR%20RESOLVING%20OVERLAPPING%20CIDRs%20IN%20AWS.pdf)

---

---

*All architecture, configuration, and validation work in this repository was designed and executed independently as an original engagement.*
