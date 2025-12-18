# Azure Sentinel Honeypot – Live Attack Monitoring & World Map Visualization

This project builds a **Windows honeypot in Microsoft Azure** and intentionally exposes it to the public Internet to observe how quickly real attackers start interacting with an unprotected system.  
The goal is to **collect, centralize, and analyze security logs** in a SIEM (Microsoft Sentinel), then **enrich and visualize attacks on a global map**.

The main objectives are:
- Deploy a Windows VM in Azure and make it discoverable from the Internet (honeypot approach)
- Centralize Windows security logs into a **Log Analytics Workspace (LAW)**
- Connect LAW to **Microsoft Sentinel** (SIEM)
- Query failed logon attempts (**Event ID 4625**) using **KQL**
- Enrich attacker IPs with **GeoIP** data via a Sentinel **Watchlist**
- Build an interactive **Sentinel Workbook** to map attack origins worldwide

---

## Tools used

- Microsoft Azure (Resource Group, VNet, VM)
- Windows 10 VM (honeypot target)
- Network Security Group (NSG) *(intentionally disabled for the lab)*
- Windows Defender Firewall *(intentionally disabled on the VM for the lab)*
- Log Analytics Workspace (LAW)
- Azure Monitor Agent (AMA)
- Microsoft Sentinel
- Kusto Query Language (KQL)
- Sentinel Watchlists (GeoIP dataset)
- Sentinel Workbooks (attack map dashboard)

---

## Requirements

- An Azure subscription (Free tier is enough to reproduce the core lab)
- Permissions to create and manage:
  - Resource Groups, VNets, VMs
  - Log Analytics Workspace
  - Microsoft Sentinel
- A local RDP client to access the VM (for configuration and verification)
- A GeoIP file for Watchlist import (CSV/Excel format)

---

## Lab topology

- **Public Internet** → real scanners / brute-force traffic
- **Azure VNet** hosting a **Windows VM (honeypot)** with a public IP
- **Intentional exposure (lab purpose):**
  - **NSG disabled**
  - **Windows Firewall disabled**
- **Log pipeline:**
  - VM → **Azure Monitor Agent (AMA)** → **Log Analytics Workspace (LAW)** → **Microsoft Sentinel (SIEM)**
- **Analysis layer:**
  - KQL focuses on **failed logons (Event ID 4625)**
  - GeoIP **Watchlist** enriches attacker IPs
  - Sentinel **Workbook** displays a **world attack map**

> ![Lab topology – Azure Sentinel Honeypot](images/sentinel-lab-topology.png)

---

## Project steps

### Step 1 — Azure baseline (honeypot VM + Internet exposure)

- Lab resources were grouped in a dedicated **Resource Group** (France Central).
  > ![Resource Group created](images/step1-resource-group.png)

- A **Windows VM** was deployed as the honeypot, with its core networking components (**VNet**, **NIC**, **Public IP**) in place.
  > ![VM + networking resources](images/step1-resources.png)

- The VM was intentionally made reachable from the Internet by relaxing the **NSG inbound rules** (honeypot design choice).
  > ![NSG inbound rules (intentional exposure)](images/step1-nsg-inbound.png)

---

### Step 2 — Increase exposure at OS level (Windows Firewall)

- RDP access confirmed using the local admin account created during deployment.
- Baseline check: **ping was blocked** initially (default Windows Firewall behavior).
  > ![Ping blocked before disabling firewall](images/step2-ping-blocked.png)

- Windows Defender Firewall was then **disabled for all profiles** (Domain / Private / Public) to maximize honeypot exposure.
  > ![Windows Firewall enabled (default)](images/step2-firewall-enabled.png)  
  > ![Windows Firewall disabled (all profiles)](images/step2-firewall-disabled.png)

- Result: **ping succeeds** after disabling the firewall, confirming the VM is reachable end-to-end from the Internet side.
  > ![Ping succeeds after disabling firewall](images/step2-ping-success.png)

---

### Step 3 — Generate logon telemetry (RDP failures → Event ID 4625)

- To produce realistic security logs, I triggered multiple **failed RDP sign-in attempts** using random credentials.
  > ![Failed RDP logon attempt](images/step3-rdp-failed.png)

- On the VM, these attempts immediately appear in **Event Viewer → Windows Logs → Security** as **Audit Failure** events (**Event ID 4625**).
  > ![Security log showing 4625 events](images/step3-eventviewer-4625-list.png)

- I filtered the Security log specifically on **4625** to focus only on failed logons.
  > ![Filtering Security log on 4625](images/step3-eventviewer-filter-4625.png)

- Each event contains the key fields I’ll later use in Sentinel: **Account Name** (attempted user) and **Source Network Address** (attacker IP).
  > ![4625 event details (username + source IP)](images/step3-4625-details.png)

---

### Step 4 — Centralize Windows security logs (LAW + AMA + Sentinel connector)

- A dedicated **Log Analytics Workspace (LAW)** was created to act as the central log repository for the lab.
  > ![Log Analytics Workspace created](images/step4-law-created.png)

- In **Microsoft Sentinel**, I enabled the **Windows Security Events** content and selected the **Windows Security Events via AMA** connector.
  > ![Sentinel Content hub – Windows Security Events](images/step4-sentinel-content-hub-wse.png)  
  > ![Windows Security Events via AMA selected](images/step4-wse-via-ama-connector.png)

- A **Data Collection Rule (DCR)** was configured to onboard the honeypot VM and collect Windows events (lab scope set to **AllEvents** for maximum telemetry).
  > ![DCR Wizard – Review + create](images/step4-dcr-wizard.png)

- On the VM side, the **Azure Monitor Agent** is visible as an installed extension, confirming the VM is ready to ship logs to LAW/Sentinel.
  > ![Azure Monitor Agent installed (VM extension)](images/step4-ama-extension.png)

---

### Step 5 — GeoIP enrichment + attack map (Watchlist + KQL Workbook)

- Once ingestion was active, **SecurityEvent** started filling up quickly with **4625** failed logons from multiple source IPs.
  > ![SecurityEvent logs in Sentinel](images/step5-securityevent-logs.png)  
  > ![Large volume of results](images/step5-securityevent-volume.png)

- To turn raw IPs into something readable, I imported a **GeoIP Watchlist** into Sentinel (a large spreadsheet mapping **IP ranges → city/country/latitude/longitude**).  
  The Workbook query enriches events using `ipv4_lookup()` and summarizes failures per location.

- Final output: a **Sentinel Workbook map** showing the geographic origin and volume of failed logons.
  > ![Workbook query (advanced editor)](images/step5-workbook-query.png)  
  > ![Sentinel Workbook – world attack map](images/step5-world-attack-map.png)

- The **Workbook item JSON** used for the map is available in the project’s additional files: `additional-files/workbook-attack-map.json`.

---

