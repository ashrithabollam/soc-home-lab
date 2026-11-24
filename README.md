# soc-home-lab
# SOC Home Lab — Report 1: Building a Wazuh SIEM from Scratch

**Author:** Ashritha Bollam
**Date:** 23 November 2026
**Project type:** Personal cybersecurity home lab (Security Operations Center simulation)

---

## 1. Project Overview

This project documents the end-to-end build of a home Security Operations Center (SOC) lab using **Wazuh**, a free and open-source SIEM (Security Information and Event Management) platform. The goal was to simulate a small enterprise security monitoring environment entirely on a personal laptop, using free and open-source tools only.

The lab consists of three virtual machines networked together in an isolated environment:

| Role | Machine | Purpose |
|---|---|---|
| SIEM Server | Ubuntu Server 24.04 LTS running Wazuh | Collects logs, generates alerts, hosts the dashboard |
| Monitored Endpoint | Ubuntu Server 24.04 LTS | A "victim" machine running the Wazuh agent, sends logs to the SIEM |
| Attacker Machine | Kali Linux | Used to simulate real attack techniques against the monitored endpoint |

This report covers the full infrastructure build: virtualization setup, networking, SIEM installation, and connecting a monitored endpoint. A second report covers the attack simulation and incident response findings.

---

## 2. Architecture

```
                         Isolated Virtual Network (192.168.50.0/24)
   ┌─────────────────────────────────────────────────────────────────┐
   │                                                                   │
   │   Kali Linux (Attacker)        Ubuntu Server (Monitored Endpoint)│
   │   192.168.50.X                  192.168.50.X  — Wazuh Agent      │
   │          │                              │                        │
   │          │  (attacks)                   │  (logs/telemetry)      │
   │          └───────────────┬──────────────┘                        │
   │                          ▼                                       │
   │              Ubuntu Server — Wazuh SIEM                          │
   │              192.168.50.10                                       │
   │              • Wazuh Indexer (data storage/search)                │
   │              • Wazuh Manager (rule engine, alerting)              │
   │              • Wazuh Dashboard (web UI)                           │
   │              • Filebeat (log shipping)                            │
   └─────────────────────────────────────────────────────────────────┘
                          ▲
                          │ HTTPS (port-forwarded to host)
                   Windows 11 Host (Browser + SSH client)
```

All three machines run as VirtualBox virtual machines on a single Windows 11 host, connected through a private **NAT Network** so they can communicate with each other and the internet, while remaining isolated from the home network.

---

## 3. Tools & Technologies Used

- **Virtualization:** Oracle VirtualBox
- **Operating Systems:** Ubuntu Server 24.04 LTS (x2), Kali Linux
- **SIEM Platform:** Wazuh 4.14 (Indexer, Manager, Dashboard, Filebeat — all-in-one deployment)
- **Networking:** VirtualBox NAT Network, static IP addressing, SSH port forwarding
- **Access tools:** Windows Terminal / Command Prompt (SSH client), Chrome/Edge (dashboard access)
- **Supporting Linux skills:** `netplan`, `systemd`/`systemctl`, LVM disk management, `apt` package management, `dpkg`, log analysis with `journalctl`

---

## 4. Build Process Summary

### 4.1 Environment Preparation
- Verified host hardware (18 GB RAM, virtualization enabled, 820 GB free disk) met lab requirements.
- Installed Oracle VirtualBox as the hypervisor.
- Downloaded Ubuntu Server 24.04 LTS (chosen specifically for Wazuh 4.x compatibility) and the official Kali Linux VirtualBox image.

### 4.2 Networking Design
- Created a dedicated **NAT Network** (`SOCLab`, `192.168.50.0/24`) in VirtualBox so all three VMs could communicate privately, isolated from the home network.
- Assigned static IP addresses via Netplan for predictable addressing:
  - Wazuh Server: `192.168.50.10`
  - Monitored Endpoint: `192.168.50.4`
  - Kali Attacker: `192.168.50.5`
- Configured SSH and HTTPS port forwarding (`2222 → 22`, `8443 → 443`) so the dashboard and SSH could be accessed from the Windows host browser/terminal.

### 4.3 Wazuh SIEM Installation
- Installed Wazuh 4.14 using the official all-in-one installation assistant, deploying the Indexer, Manager, Dashboard, and Filebeat on a single Ubuntu Server VM.
- Verified all core services (`wazuh-indexer`, `wazuh-manager`, `filebeat`, `wazuh-dashboard`) were active using `systemctl`.
- Enabled all services to auto-start on boot.
- Accessed the Wazuh web dashboard securely over HTTPS from the host browser.

### 4.4 Endpoint Onboarding
- Built a second Ubuntu Server VM to act as a monitored endpoint.
- Added the official Wazuh APT repository and GPG key, then installed the Wazuh Agent (version-pinned to match the server).
- Configured the agent to report to the SIEM server's IP address and confirmed registration.
- Verified the endpoint appeared as an **Active** agent in the Wazuh dashboard's Endpoints Summary.

### 4.5 Attacker Machine
- Deployed the official Kali Linux pre-built VM as the attack platform.
- Connected it to the same isolated network and verified connectivity to both the SIEM server and the monitored endpoint.

---

## 5. Troubleshooting & Problem-Solving Highlights

A real SOC build rarely goes smoothly, and documenting the recovery process is often more valuable to employers than a clean install. Key issues diagnosed and resolved during this build:

| Issue | Root Cause | Resolution |
|---|---|---|
| Wazuh indexer failed to start after reboot | `/var/log/wazuh-indexer` directory was being deleted | Recreated the directory and added a persistent `systemd-tmpfiles` rule so it survives reboots |
| Full disk during installation | Initial virtual disk (23 GB) was undersized for Wazuh's indexer | Resized the VirtualBox disk to 60 GB and grew the Linux LVM partition/filesystem live |
| Dashboard installation failing (`node name mismatch`) | Installer was given an incorrect internal node name | Diagnosed the correct name from the installation certificate bundle and re-ran with the matching value |
| Wazuh API showing "Offline" / "Invalid credentials" | API passwords were out of sync after repeated partial installs | Used Wazuh's official `wazuh-passwords-tool.sh` to reset the API user password and updated the dashboard's stored credentials |
| Services not surviving VM reboot | Services were not enabled to auto-start | Used `systemctl enable` on all four core Wazuh services |

These troubleshooting steps mirror real-world SOC/sysadmin work: service failures, disk capacity planning, credential management, and systemd service administration.

---

## 6. Result

At the end of this phase, the lab consisted of:
- A fully functional Wazuh SIEM with all core services running and enabled on boot
- A working, HTTPS-accessible security dashboard
- One actively monitored Linux endpoint reporting real-time logs and events
- An isolated attacker machine ready for Phase 2 (attack simulation)

Wazuh's default ruleset was already detecting real authentication activity (failed logins, non-existent user attempts) generated incidentally during the build, demonstrating the SIEM's out-of-the-box detection capability before any intentional attacks were run.

---

## 7. Screenshots

- `screenshots/01-victim-vm-network-config.png` — VirtualBox configuration for the monitored endpoint: memory, CPU, and NAT Network (SOCLab) settings
- `screenshots/02-endpoint-active.png` — Wazuh Endpoints Summary showing `ubuntu-victim` registered and reporting as **Active**
- `screenshots/03-kali-attacker-deployed.png` — Kali Linux attacker VM running and logged in
- `screenshots/04-agent-install-check.png` — Verifying connectivity to the Wazuh package repository before installing the agent
- `screenshots/05-api-credential-troubleshooting.png` — Using the Wazuh passwords tool to resolve an API authentication issue during setup
---

## 8. Skills Demonstrated

- Linux system administration (Ubuntu Server, `systemctl`, `netplan`, LVM disk resizing)
- SIEM deployment and configuration (Wazuh architecture: Indexer, Manager, Dashboard)
- Virtualization and network segmentation (VirtualBox NAT Networks, static IP design)
- Endpoint detection and response (EDR) agent deployment
- Troubleshooting production-style service failures under real constraints (disk space, credentials, service dependencies)
- Security-first lab design (isolated network, no exposure to the host network)

---

## 9. Next Steps

See **Report 2: Attack Simulation and Incident Detection**, which covers:
- SSH brute-force attack simulation using Hydra
- Nmap reconnaissance scanning
- Analysis of Wazuh-generated alerts and MITRE ATT&CK technique mapping
- A formal incident report
