# 🏢 Enterprise IT Infrastructure & SOC HomeLab

![Arch Linux|180](https://img.shields.io/badge/Arch%20Linux-1793D1?logo=arch-linux&logoColor=fff)
![Platform|180](https://img.shields.io/badge/Hypervisor-KVM%2FQEMU-white?)
![Windows|180](https://custom-icon-badges.demolab.com/badge/Windows-0078D6?logo=windows11&logoColor=white)
![Platform|180](https://img.shields.io/badge/Hypervisor-VirtualBox-blue?logo=windows)

![AD|180](https://img.shields.io/badge/Identity-Active%20Directory-0078D4?logo=windows)
![Firewall|180](https://img.shields.io/badge/Firewall-OPNsense-F26721)
![SIEM|180](https://img.shields.io/badge/SIEM-Wazuh-blue)

![Status|100](https://img.shields.io/badge/Status-Active-brightgreen)

> A production-grade simulation of a corporate IT environment and Security Operations Center (SOC), built from scratch to demonstrate end-to-end capabilities in **Systems Administration, Network Defense, Threat Detection, and Automated Incident Response**.

---

## 📋 Table of Contents
- [Project Overview](#-project-overview)
- [Architecture & Network Topology](#️-architecture--network-topology)
- [Technologies Used](#️-technologies-used)
- [Implementation Phases](#-implementation-phases)
- [MITRE ATT&CK Coverage](#-mitre-attck-coverage)
- [Key Skills Demonstrated](#-key-skills-demonstrated)
- [Roadmap & Future Enhancements](#️-roadmap--future-enhancements)

---

## 🎯 Project Overview

This HomeLab replicates the infrastructure of a small-to-medium enterprise, encompassing:

- **Centralized Identity Management** via Active Directory with GPO-enforced policies
- **Role-Based Access Control** with department-level file share segmentation
- **Real-time Threat Detection** using a self-hosted Wazuh SIEM with custom detection rules
- **Automated Incident Response** that autonomously isolates attackers without human intervention
- **Network Perimeter Defense** with IDS/IPS and Layer 3/7 web filtering

The lab is intentionally *adversarial* — every defensive control was validated by actively simulating the attack it was designed to stop.

---

## 🏗️ Architecture & Network Topology

```
                        [ Internet / WAN ]
                                │
                    ┌───────────▼───────────┐
                    │   OPNsense Firewall   │  ← Suricata IPS, Web Filter
                    │   WAN + LAN Gateway   │
                    └───────────┬───────────┘
                                │
              ══════════════════▼══════════════════
                       LAN: 10.10.10.0/24
              ══╦══════════╦═══════════╦═══════╦══
                │          │           │       │
          ┌───────▼────┐ ┌───▼───┐ ┌────▼───┐ ┌─▼──────┐
          │  DC/DNS    │ │  FS   │ │ Wazuh  │ │Win 10  │
          │10.10.10.10 │ │ .11   │ │  .9    │ │Clients │
          │hlab.lan    │ │FS-CORE│ │ SIEM   │ │(Domain)│
          └────────────┘ └───────┘ └────────┘ └────────┘
```

| Host | IP Address | Role |
|------|-----------|------|
| OPNsense | `10.10.10.1` | Firewall, Gateway, DNS Forwarder |
| Domain Controller | `10.10.10.10` | AD DS, DNS, DHCP (`dc.hlab.lan`) |
| File Server Core | `10.10.10.11` | SMB Shares, NTFS Permissions (`FS`) |
| Wazuh SIEM | `10.10.10.9` | Log Aggregation, Alerting, Active Response |
| Client Endpoints | `10.10.10.x` | Windows 10 Pro (Domain-Joined) |

> ⚠️ **Note on Network Evolution:**
> The lab was originally provisioned on the `192.168.10.0/24` subnet. During the hardening phase, the entire infrastructure was migrated to `10.10.10.0/24`. This change was implemented to better simulate enterprise-grade IP addressing schemes and to facilitate cleaner routing during the upcoming VLAN implementation phase.

---

## 🛠️ Technologies Used

| Category | Technology                                                                                        |
| ------------------ | ------------------------------------------------------------------------------------------------- |
| Hypervisor | KVM/QEMU on Arch Linux (bridged via `br0`) · VirtualBox on Windows 11 (bridged via Ethernet)|
| Identity & Access | Windows Server 2022 — AD DS, DNS, DHCP, GPO                                                       |
| File Services | Windows Server 2022 **Core** (headless, remote-managed)                                           |
| Perimeter Security | OPNsense — Suricata IDS/IPS, Aliases, Firewall Rules                                              |
| SIEM / SOC | Wazuh — Agent Enrollment, Custom Rules, Active Response                                           |
| Scripting | PowerShell — Bulk User Provisioning, Agent Deployment                                             |
| Endpoint OS | Windows 10 Pro                                                                                    |

---

## 🚀 Implementation Phases

### Phase 1 — Core Infrastructure & Identity Management

**Objective:** Establish a functional corporate domain with automated user management. 

- Deployed Windows Server 2022 as the primary Domain Controller for `hlab.lan`
- Disabled OPNsense DHCP to consolidate IP management under the DC
- Wrote a **PowerShell provisioning script** to bulk-import users from CSV into appropriate Organizational Units (OUs)
- Enrolled Windows 10 workstations to the domain for centralized policy management

> 💡 *Why it matters:* Manual user creation doesn't scale. Automated provisioning reduces human error and ensures consistent OU/group membership from day one.

![AD User Provisioning](images/ad-ous.png)![](images/ad-user-script.png)

---

### Phase 2 — Role-Based Access Control (RBAC) & File Services

**Objective:** Enforce least-privilege access to sensitive departmental data.

- Deployed **Windows Server Core** (`FS-CORE`) to minimize the attack surface — no GUI, no unnecessary services
- Remotely managed via the DC's Server Manager over the network
- Created `IT_Data` and `HR_Data` SMB shares with NTFS permissions tied to AD Security Groups (`SG_IT_Dept`, `SG_HR_Dept`)
- Configured **GPO with Item-Level Targeting** to auto-mount the correct drive letter (`I:` or `H:`) based on the authenticated user's group membership

> 💡 *Why it matters:* RBAC prevents lateral movement — a compromised HR account cannot access IT files, limiting blast radius.

![Mapped Drives](images/gpo-drive-maps.png)![](images/gpo-item-level-targeting.png)

---

### Phase 3 — SIEM Deployment & Data Loss Prevention (DLP)

**Objective:** Gain visibility into endpoint activity and block data exfiltration vectors.

- Deployed **Wazuh All-in-One** on Ubuntu Server; enrolled Windows endpoints via PowerShell agent scripts
- Blocked USB mass storage devices via GPO (`Access Denied` on removable media)
- Modified Wazuh agent config to ingest `DriverFrameworks-UserMode` Windows event logs
- Authored **Custom Rule ID 110001** in `local_rules.xml` to trigger a **High-Severity SOC alert** on every USB insertion event — even when blocked by policy

> 💡 *Why it matters:* Blocking without logging is blind enforcement. This combination ensures policy violations are both *prevented* and *recorded* for audit trails.

![USB Block & Alert](images/wazuh-usb-log-2.png)![](images/wazuh-usb-log-3.png)![](images/wazuh-usb-log-4.png)

---

### Phase 4 — File Integrity Monitoring (FIM) & Insider Threat Detection

**Objective:** Detect unauthorized file operations on sensitive network shares in real-time.

- Configured Wazuh FIM agent on `FS-CORE` to actively monitor `IT_Data` and `HR_Data` directories
- Simulated insider threat scenarios: unauthorized file creation, modification, and deletion
- SIEM displayed precise **timestamps, affected file paths, and the responsible user account** for each event

> 💡 *Why it matters:* Insider threats are notoriously hard to detect. FIM creates a forensic audit trail that can prove *who* changed *what* and *when*.

![File Integrity Monitoring](images/fs-fim-ossec-conf.png)![](images/wazuh-fim-alert.png)

---

### Phase 5 — Active Directory Security & Privilege Escalation Detection

**Objective:** Harden AD against brute force and unauthorized privilege escalation.

- Enforced **Account Lockout Policy** via GPO: account locks after 5 failed attempts
- Simulated an RDP brute force attack — account locked and **Wazuh fired alert on Event ID 4740**
- Simulated an insider threat: standard user added to `Domain Admins` group — immediately flagged as a **Critical alert** by Wazuh

> 💡 *Why it matters:* Privilege escalation is Stage 4 of most kill chains. Detecting it in seconds versus days is the difference between an incident and a breach.

![Account Lockout Alert](images/wazuh-priv-escalation-1.png)![](images/wazuh-priv-escalation-2.png)


---

### Phase 6 — Automated Incident Response (Active Response)

**Objective:** Move from passive detection to autonomous threat containment.

- Configured Wazuh **Active Response** triggered by repeated authentication failures (Rules 60122 / 60204)
- Upon threshold breach, SIEM executed a `firewall-drop` script on the target endpoint **without human intervention**
- The script dynamically created a **Windows Defender Firewall block rule**, isolating the attacker's source IP

> 💡 *Why it matters:* Mean time to respond (MTTR) drops from minutes to milliseconds. Attackers executing automated brute force are stopped before credentials are cracked.

![Active Response Firewall Drop](images/wazuh-ar-rule.png)![](images/wazuh-ar-rule-level-1.png)![](images/wazuh-ar-rdp-fail.png)![](images/wazuh-ar-rule-level-2.png)

---

### Phase 7 — Defense Evasion Detection (MITRE ATT&CK: T1070.001)

**Objective:** Detect adversaries attempting to destroy forensic evidence.

- Elevated specific event rules to **Critical (Level 12)** by overriding defaults in `local_rules.xml`
- Simulated a post-compromise attacker clearing the Windows Security Audit Log to cover tracks
- Wazuh detected Windows Event ID 1102 via Custom Rule 110002 and triggered the Critical alert

> 💡 *Why it matters:* Clearing logs is a standard attacker technique to hinder forensics. Detecting the clearing event itself turns the attacker's evasion attempt into additional evidence.

![Defense Evasion Detection](images/wazuh-evasion-rule.png)![](images/wazuh-evasion-alert.png)

---

### Phase 8 — Network Perimeter Security (OPNsense)

**Objective:** Enforce corporate web policy and block network-level threats.

- Created **Firewall Aliases** grouping social media domains; implemented L3/L4 drop rules on outbound LAN traffic

![OPNsense Webfilter Alias/Rule/Result](images/opnsense-webfilter-alias.png)![](images/opnsense-webfilter-rule.png)![](images/opnsense-webfilter-done.png)

- Enabled **Suricata IPS** in promiscuous mode on LAN/WAN interfaces
- Downloaded **Emerging Threats (ET) Open** rulesets
- Validated IPS effectiveness by simulating a malicious payload request (`curl testmyids.com`) — Suricata converted the alert to a **drop action at Layer 7**

> 💡 *Why it matters:* A firewall without IPS only enforces policy on known-bad IPs/ports. Suricata inspects payload content, catching threats that bypass traditional port-based rules.

![Suricata IPS Drop](images/opnsense-ips-enable.png)![](images/opnsense-ips-alert.png)![](images/opnsense-ips-drop-1.png)![](images/opnsense-ips-drop-2.png)

---

## 🛡️ MITRE ATT&CK Coverage

| Technique | ID | Phase | Detection Method |
|-----------|----|-------|-----------------|
| Brute Force: Password Guessing | T1110.001 | Phase 5 & 6 | Event ID 4740 → Active Response |
| Valid Accounts: Domain Accounts | T1078.002 | Phase 5 | Privilege Escalation Alert |
| Indicator Removal: Clear Windows Event Logs | T1070.001 | Phase 7 | Event ID 1102 → Custom Rule |
| Exfiltration over Removable Media | T1052 | Phase 3 | Custom Rule 110001 |
| Data from Network Shared Drive | T1039 | Phase 4 | FIM Real-time Monitoring |

---

## 🎓 Key Skills Demonstrated

**Systems Administration**
- Windows Server 2022 deployment, AD DS, DNS, DHCP, GPO management
- Server Core remote administration, SMB share management, NTFS ACLs
- PowerShell scripting for bulk automation

**Security Operations (SOC)**
- SIEM deployment, agent enrollment, and custom detection rule authoring
- Log analysis across Windows Event Logs, Sysmon, and application logs
- Incident simulation, triage, and documentation

**Network Security**
- Firewall rule design (L3/L4), alias management, traffic segmentation
- IDS/IPS tuning and ruleset management (Suricata / ET Open)
- Network topology design with defense-in-depth principles

**Threat Hunting & DFIR**
- MITRE ATT&CK framework mapping to real detection scenarios
- File Integrity Monitoring and forensic evidence preservation
- Adversary simulation and defense validation

---

*Built on KVM/QEMU — Arch Linux host | Domain: `hlab.lan` | Subnet: `10.10.10.0/24`*

---

## 🗺️ Roadmap & Future Enhancements

The project is in **Active Development**. Planned phases are scoped to available hardware *(Arch: 16GB — hosts OPNsense, DC, Wazuh | Win11 PC: 16GB — hosts FS-Core, Clients)*.

---

### 🟡 Phase 9 — Active Directory Hardening

- [ ] **Sysmon Deployment** — Install on DC, FS-Core, and Client endpoints; configure Wazuh to ingest `Microsoft-Windows-Sysmon/Operational` channel for deep process, network, and registry telemetry
- [ ] **Tiered Administration Model** — Implement Tier 0/1/2 separation; restrict Tier 0 credentials to Domain Controller access only
- [ ] **Fine-Grained Password Policy (FGPP)** — Create separate PSOs for `Domain Admins` vs standard users
- [ ] **Privileged Access Workstation (PAW)** — Deploy a dedicated admin-only VM with no internet access; RDP to DC exclusively from this machine
- [ ] **AD Recycle Bin + Audit Trail** — Enable AD Recycle Bin; configure granular object-level auditing

---

### 🟣 Phase 10 — Offensive Security & Red/Blue Team Scenarios

- [ ] **Kali Linux VM** — Deploy on Arch host (separate VLAN `10.10.20.0/24` via OPNsense); use as dedicated attacker machine
- [ ] **BloodHound + SharpHound** — Map AD attack paths graphically; identify and remediate shortest path to Domain Admin
- [ ] **Mimikatz Scenarios** — Simulate Pass-the-Hash and Pass-the-Ticket attacks (MITRE T1550); validate Sysmon + Wazuh detection
- [ ] **Atomic Red Team** — Execute automated MITRE ATT&CK technique simulations; expand detection coverage across kill chain stages
- [ ] **DVWA / VulnHub Target** — Add intentionally vulnerable Linux VM on Win11 PC; practice exploitation and post-compromise log analysis

---

### 🔵 Phase 11 — Network & Infrastructure Maturity

- [ ] **VLAN Segmentation** — Create Management, Server, Client, and Attacker VLANs in OPNsense; enforce inter-VLAN firewall policies
- [ ] **Advanced Suricata Tuning** — Write custom rules targeting lateral movement techniques (e.g., SMB enumeration, PsExec patterns); convert alerts to inline drops
- [ ] **Linux Server Hardening** — Add Rocky Linux web/database VM; apply CIS Benchmark Level 1 and monitor with Wazuh agent
- [ ] **Centralized Patch Management (WSUS)** — Deploy WSUS on FS-Core or a dedicated VM; manage update compliance across domain

---

### 🟢 Phase 12 — SOC Maturity & Visibility

- [ ] **ELK Stack Integration** — Forward Wazuh alerts to Elasticsearch + Kibana; build custom SOC dashboards
- [ ] **Vulnerability Management (OpenVAS/Greenbone)** — Schedule scans, document findings and remediation workflow
- [ ] **Honey Pot** — Deploy a low-interaction decoy VM (e.g., `FINANCE-SERVER`, ~512MB); alert on any connection attempt via Wazuh custom rule
- [ ] **SOC Automation (TheHive/Cortex)** — Automate alert-to-ticket pipeline from Wazuh; document incident response playbooks

---

### ⚪ Phase 13 — Cloud Integration

- [ ] **Azure AD Hybrid Join** — Connect on-premises `hlab.lan` AD to Azure AD via Azure AD Connect; simulate hybrid identity scenarios
- [ ] **Cloud Endpoint Monitoring** — Spin up AWS EC2 instance (free tier); enroll as Wazuh agent to extend SIEM visibility beyond on-prem
