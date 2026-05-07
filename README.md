# Azure Sentinel Honeypot - Catching Real Attackers with a Cloud SIEM

> *I deployed a deliberately vulnerable Windows server on Microsoft Azure, left it exposed to the entire internet, and watched 80,000+ real cyberattacks roll in from across the world, all visualised live on a Microsoft Sentinel attack map.*

![Attack Map](screenshots/Microsoft%20Sentinel%20attack%20map%20after%2012%20hours%20%E2%80%94%2079%2C000%2B%20failed%20RDP%20brute%20force%20attempts%20detected%20from%2010%2B%20countries.png)

---

## Table of Contents

- [Overview](#overview)
- [Key Results](#key-results)
- [Architecture](#architecture)
- [Technologies Used](#technologies-used)
- [Project Walkthrough](#project-walkthrough)
  - [Phase 1 - Setting Up Azure](#phase-1---setting-up-azure)
  - [Phase 2 - Deploying the Honeypot VM](#phase-2---deploying-the-honeypot-vm)
  - [Phase 3 - Exposing the VM to the Internet](#phase-3---exposing-the-vm-to-the-internet)
  - [Phase 4 - Setting Up the SIEM Pipeline](#phase-4---setting-up-the-siem-pipeline)
  - [Phase 5 - Building the Attack Map](#phase-5---building-the-attack-map)
  - [Phase 6 - Automated Threat Detection](#phase-6---automated-threat-detection)
  - [Phase 7 - Results and Analysis](#phase-7---results-and-analysis)
- [KQL Queries Used](#kql-queries-used)
- [Key Learnings](#key-learnings)
- [Cost and Cleanup](#cost-and-cleanup)

---

## Overview

This project involved building a cloud-based honeypot on Microsoft Azure from scratch. A **honeypot** is a deliberately exposed and vulnerable system designed to attract attackers, giving us the opportunity to study real-world attack behaviour in a controlled environment.

I configured a Windows Server VM with all firewall rules removed and left it exposed to the public internet via RDP (Remote Desktop Protocol, port 3389). Within minutes, automated bots and threat actors from around the world began attempting to break in. I used **Microsoft Sentinel** - Microsoft's cloud-native SIEM (Security Information and Event Management) platform - to ingest, analyse, and visualise every single failed login attempt in real time.

The result? A live global attack map, 80,000+ detected intrusion attempts, and 78 automated security incidents, all within 24 hours.

---

## Key Results

| Metric | Value |
|--------|-------|
| Duration | ~15 hours |
| Total failed RDP attempts | 80,700+ |
| Countries of origin | 10+ |
| Top attacking location | Jordanow, Poland - 50,000+ attempts |
| Sentinel incidents generated | 78 |
| Total Azure cost | £0.23 |
| Security events ingested | 81,200+ |

---

## Architecture

```
+----------------------------------------------------------+
|                    PUBLIC INTERNET                        |
|         (Real attackers, bots, automated scanners)        |
+---------------------+------------------------------------+
                      | RDP brute force attempts
                      v
+----------------------------------------------------------+
|              AZURE NETWORK SECURITY GROUP                 |
|         Rule: WARNING_AllInBound - Allow Any/Any          |
|         (Intentionally misconfigured - all ports open)    |
+---------------------+------------------------------------+
                      |
                      v
+----------------------------------------------------------+
|           HONEYPOT VM - Corp-Win-Prod-01                  |
|    Windows Server 2019 | Sweden Central | Azure           |
|    - Windows Defender Firewall: DISABLED                  |
|    - RDP port 3389: OPEN to entire internet               |
|    - Azure Monitor Agent: collecting all security events  |
+---------------------+------------------------------------+
                      | Windows Security Event ID 4625
                      | (Failed login events)
                      v
+----------------------------------------------------------+
|         LOG ANALYTICS WORKSPACE - Honey-Pot-01            |
|    - Centralised log storage and query engine             |
|    - GeoIP watchlist: 55,000 IP range mappings            |
|    - KQL queries join failed logins with geolocation      |
+---------------------+------------------------------------+
                      |
                      v
+----------------------------------------------------------+
|           MICROSOFT SENTINEL (SIEM)                       |
|    - Real-time attack map workbook                        |
|    - Custom brute force detection rule (KQL)              |
|    - Automated incident generation every 5 minutes        |
|    - 78 incidents triggered in 15 hours                   |
+----------------------------------------------------------+
```

---

## Technologies Used

| Tool | Purpose |
|------|---------|
| Microsoft Azure | Cloud infrastructure hosting |
| Azure Virtual Machine (Windows Server 2019) | Honeypot target system |
| Azure Network Security Group (NSG) | Cloud-level firewall (intentionally opened) |
| Azure Monitor Agent (AMA) | Log collection from VM to workspace |
| Log Analytics Workspace | Centralised log storage and KQL query engine |
| Microsoft Sentinel | SIEM - detection, visualisation, incident management |
| KQL (Kusto Query Language) | Log querying and threat detection rules |
| GeoIP Watchlist (MaxMind) | IP-to-location mapping for attack visualisation |

---

## Project Walkthrough

### Phase 1 - Setting Up Azure

I used an **Azure for Students** subscription through my university, free with no credit card required. The first step was creating a **Resource Group** called `Honeypot-lab`, which acts as a logical container for all project resources. This is important for organisation and especially useful at cleanup time - deleting the resource group removes everything inside it at once.

![Resource Group Created](screenshots/Resource%20Group%20created%20%E2%80%94%20all%20project%20resources%20will%20be%20organised%20here.png)

---

### Phase 2 - Deploying the Honeypot VM

I created a Windows Server 2019 Datacenter VM named `Corp-Win-Prod-01`, a deliberately realistic-sounding corporate name to make it a more believable target.

**VM Configuration:**
- **Image:** Windows Server 2019 Datacenter x64 Gen2
- **Size:** Standard B2ats v2 (2 vCPUs, 1 GiB RAM)
- **Region:** Sweden Central (Zone 3)
- **OS Disk:** Standard SSD

The VM name `Corp-Win-Prod-01` was chosen intentionally to mimic a real corporate production machine. In practice, attackers don't see this name - they only see the public IP address - but naming resources realistically is good professional habit.

![VM Deployment Complete](screenshots/Honeypot%20VM%20successfully%20deployed%20on%20Microsoft%20Azure%20-%20Sweden%20Central%20region.png)

![VM Overview](screenshots/Honeypot%20VM%20overview.png)

Once deployed, I connected via **RDP (Remote Desktop Protocol)** from my local machine to verify the VM was accessible.

![Connected via RDP](screenshots/Connected%20to%20honeypot%20VM%20via%20RDP.png)

---

### Phase 3 - Exposing the VM to the Internet

This phase involved two deliberate misconfigurations that no production environment should ever have, both done intentionally to maximise the VM's attractiveness to attackers.

**Step 1: Disable the Windows Defender Firewall**

Inside the VM, I opened Windows Defender Firewall with Advanced Security and disabled all three firewall profiles: Domain, Private, and Public. This means the VM no longer filters any incoming traffic at the OS level.

To verify this worked, I ran a `ping` from my local machine to the VM's public IP (`20.240.212.140`). A successful ping confirms the firewall is down - normally Windows blocks ICMP echo requests by default.

```
ping 20.240.212.140
Reply from 20.240.212.140: bytes=32 time=37ms TTL=113
Reply from 20.240.212.140: bytes=32 time=38ms TTL=113
```

![Firewall Disabled](screenshots/turned%20off%20firewall%20stuff.png)

![Ping Success](screenshots/cmd%20ping%20success.png)

**Step 2: Open All Ports in the Network Security Group**

The NSG is Azure's cloud-level firewall that sits in front of the VM. I added a custom inbound rule named `WARNING_AllInBound` with the following configuration, allowing all traffic from any source to any destination on any port:

| Setting | Value |
|---------|-------|
| Priority | 100 |
| Name | WARNING_AllInBound |
| Source | Any |
| Source port | * |
| Destination | Any |
| Destination port | * |
| Protocol | Any |
| Action | Allow |

The warning name is intentional - in a real environment, an allow-all rule like this would be a critical security finding.

![NSG Rule](screenshots/WARNING_ALLOWALL_FIREWALL%20CONFIG.png)

**Verifying Logs in Event Viewer**

Before moving to the SIEM setup, I confirmed that failed login attempts were being captured in Windows Event Viewer. I made a few intentional failed login attempts from my local machine and verified they appeared as **Event ID 4625** (An account failed to log on) in the Security log.

![Event Viewer 4625](screenshots/Windows%20Event%20Viewer%20showing%20Event%20ID%204625%20HIDE%20IP%20.png)

This is the exact event ID our SIEM would be monitoring and alerting on.

---

### Phase 4 - Setting Up the SIEM Pipeline

With the honeypot exposed and logging confirmed, it was time to build the detection pipeline.

**Step 1: Create a Log Analytics Workspace**

The Log Analytics Workspace (`Honey-Pot-01`) serves as the centralised log repository. Think of it as the database that stores all security events - both from the VM itself and from any enrichment we add.

![Log Analytics Created](screenshots/created%20log%20analytics%20.png)

**Step 2: Add Microsoft Sentinel**

Microsoft Sentinel was deployed on top of the Log Analytics Workspace. Sentinel is the actual SIEM layer - it provides the analytics engine, workbooks, incident management, and threat intelligence on top of the raw logs.

![Sentinel Created](screenshots/microsoft%20sentinel%20created%20with%20log%20analytics.png)

**Step 3: Connect the VM via Azure Monitor Agent (AMA)**

To pipe logs from the VM into the workspace, I configured a **Windows Security Events via AMA** data connector in Sentinel. This installed the Azure Monitor Agent on the VM and created a data collection rule (`Rule-Windows`) set to collect All Events.

![AMA Rule Created](screenshots/AMA%20RULE%20CREATED.png)

**Step 4: Upload the GeoIP Watchlist**

To enrich the raw attack data with geolocation, I uploaded a GeoIP CSV database to Sentinel as a **Watchlist** named `geoip`. This file contains 55,000+ IP range-to-location mappings covering the entire IPv4 space. The KQL query joins attacker IP addresses against this watchlist to extract city, country, latitude, and longitude.

![GeoIP Watchlist](screenshots/GEOIP%2055%20THOUSAND.png)

---

### Phase 5 - Building the Attack Map

With logs flowing and geolocation data loaded, I built a **Sentinel Workbook** - a customisable dashboard - to visualise attacks on a world map.

The workbook uses the following KQL query to join failed login events with the GeoIP watchlist and plot each attacking IP as a bubble on the map, sized by attack frequency:

```kql
let GeoIPDB_FULL = _GetWatchlist("geoip");
let WindowsEvents = SecurityEvent;
WindowsEvents
| where EventID == 4625
| order by TimeGenerated desc
| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network)
| summarize FailureCount = count() by IpAddress, latitude, longitude, cityname, countryname
| project FailureCount, AttackerIp = IpAddress, latitude, longitude,
  city = cityname, country = countryname,
  friendly_location = strcat(cityname, " (", countryname, ")")
```

The map uses a **green-to-red heatmap** colour scheme - green for fewer attacks, red for the highest volume. Within the first 30 minutes of the VM being exposed, the first attacks were already appearing.

**First attacks appearing (within 30 minutes):**

![First Attacks](screenshots/FIRST%20SS%20HONEYPOT%20ATTACK%20SUCCESSFUL%20GRAPH%20CAPTURE.png)

**Attack map after 12+ hours:**

![Final Attack Map](screenshots/Microsoft%20Sentinel%20attack%20map%20after%2012%20hours%20%E2%80%94%2079%2C000%2B%20failed%20RDP%20brute%20force%20attempts%20detected%20from%2010%2B%20countries.png)

---

### Phase 6 - Automated Threat Detection

A map is impressive but what makes this a real SIEM setup is **automated detection**. I created a custom **Scheduled Analytics Rule** in Sentinel that automatically generates an incident whenever a single IP address exceeds 5 failed login attempts within a 1-hour window.

**Detection Rule Configuration:**

| Setting | Value |
|---------|-------|
| Rule name | Brute Force Attack Detected - RDP |
| Severity | Medium |
| Run frequency | Every 5 minutes |
| Lookback period | Last 1 hour |
| Alert threshold | Greater than 0 results |

**KQL Detection Query:**

```kql
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(1h)
| summarize AttemptCount = count() by IpAddress, Account
| where AttemptCount > 5
| extend AlertDetails = strcat("IP: ", IpAddress,
  " | Username tried: ", Account,
  " | Attempts: ", AttemptCount)
```

![Detection Rule](screenshots/BRUTE%20FORCE%20RULE%20ADDED.png)

The rule began firing almost immediately. Within the first hour, 5 incidents had been created. By the time the lab concluded, **78 incidents** had been automatically generated and queued for investigation.

![Incidents - Early](screenshots/Microsoft%20Sentinel%20incidents%20%E2%80%94%205%20brute%20force%20incidents%20automatically%20generated%20by%20custom%20detection%20rule.png)

![Incidents - Final](screenshots/detected%2077%20incidents%20in%20brute%20force%20rule%20after%2012%20hours.png)

Each incident can be opened for investigation, showing the timeline, associated alerts, entity details, and similar incidents for correlation.

![Incident Detail](screenshots/incident%2075%20view%20full%20details.png)

---

### Phase 7 - Results and Analysis

**Sentinel Overview - Final State:**

![Sentinel Overview](screenshots/MICROSFOT%20SENTINEL%20old%20overlay%20for%20overview.png)

![Sentinel New Overview](screenshots/MICROSOFT%20SENTINEL%20new%20overlay%20for%20overview.png)

**Attack Breakdown by Location:**

| Location | Country | Attack Count |
|----------|---------|-------------|
| Jordanow | Poland | ~50,000 |
| Tilburg | Netherlands | ~20,000 |
| Maam | Netherlands | ~5,710 |
| Southend-on-Sea | United Kingdom | ~1,440 |
| Chessel | Switzerland | 619 |
| Ponso | Italy | 428 |
| (Unknown) | Georgia | 420 |
| Valencia | Spain | 299 |
| Doncaster | United Kingdom | 98 |
| Other | Various | 89 |
| **Total** | **10+ countries** | **~79,100+** |

**Key Observations:**

The vast majority of attacks were fully automated - bots running 24/7 scanning the internet for open RDP ports. This is evident from the attack rate: approximately **6,500 attacks per hour**, or **108 per minute**, sustained continuously throughout the lab duration.

The dominant source - Jordanow, Poland - is almost certainly a compromised machine or VPN exit node being used as an attack relay, not necessarily a Polish threat actor. This is a common pattern: attackers route traffic through compromised systems in other countries to obscure their true origin.

The most commonly attempted usernames observed in the logs included predictable patterns like `Administrator`, `admin`, `user`, `test`, and `guest` - reflecting standard credential stuffing dictionaries used by automated attack tools.

The Netherlands appearing twice (Tilburg and Maam) with a combined ~25,000 attacks likely indicates a single automated campaign operating from Dutch infrastructure, possibly a botnet or cloud-hosted attack tool.

---

## KQL Queries Used

**1. View all failed login events:**
```kql
SecurityEvent
| where EventID == 4625
| order by TimeGenerated desc
```

**2. Join failed logins with GeoIP data and summarise by attacker:**
```kql
let GeoIPDB_FULL = _GetWatchlist("geoip");
let WindowsEvents = SecurityEvent;
WindowsEvents
| where EventID == 4625
| order by TimeGenerated desc
| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network)
| summarize FailureCount = count() by IpAddress, latitude, longitude, cityname, countryname
| project FailureCount, AttackerIp = IpAddress, latitude, longitude,
  city = cityname, country = countryname,
  friendly_location = strcat(cityname, " (", countryname, ")")
```

**3. Brute force detection - IPs with more than 5 attempts in 1 hour:**
```kql
SecurityEvent
| where EventID == 4625
| where TimeGenerated > ago(1h)
| summarize AttemptCount = count() by IpAddress, Account
| where AttemptCount > 5
| extend AlertDetails = strcat("IP: ", IpAddress,
  " | Username tried: ", Account,
  " | Attempts: ", AttemptCount)
```

---

## Key Learnings

**Cloud infrastructure is under constant attack.** Within 15 minutes of the VM going live, automated scanners had already found and begun attacking it. Any internet-facing service without proper hardening will be discovered and probed almost immediately.

**Defence in depth matters.** This lab demonstrates why layered security is essential - a single open port with a weak password is all an attacker needs. The NSG (cloud firewall), OS firewall, and strong credentials are all separate layers that should never all be disabled simultaneously.

**SIEMs are powerful but require proper data pipelines.** Getting logs from the VM into Sentinel required configuring the Azure Monitor Agent, a data collection rule, and a Log Analytics Workspace before any detection could happen. Understanding this pipeline is fundamental to SOC work.

**KQL is an essential skill for cloud security.** Every meaningful action in Sentinel - from building the attack map to writing detection rules - required KQL queries. It is similar in concept to SQL but optimised for log data and time-series analysis.

**Real attackers behave predictably.** The attack patterns observed - automated scanners, common username dictionaries, sustained high-volume attempts - match documented threat intelligence. This hands-on exposure makes threat actor TTPs (Tactics, Techniques, and Procedures) tangible in a way that theory alone cannot.

**Responsible cloud resource management.** All resources were deleted immediately after the lab concluded. Total cost: £0.23 for approximately 15 hours of operation. Proper resource lifecycle management is a professional expectation in any cloud environment.

---

## Cost and Cleanup

![Cost Analysis](screenshots/cost_analysis.png)

The entire lab - VM, storage, Log Analytics, Sentinel, public IP - cost **£0.23 GBP** for approximately 15 hours of operation. This is well within the Azure for Students free credit allocation.

All resources were deleted by removing the `Honeypot-lab` resource group, which cascaded deletion to all contained resources simultaneously - demonstrating clean infrastructure teardown practice.

---

## Repository Structure

```
azure-sentinel-honeypot-siem/
|
+-- README.md                     <- You are here
+-- queries/
|   +-- failed_logins.kql         <- Basic Event ID 4625 query
|   +-- geoip_enrichment.kql      <- GeoIP join query for attack map
|   +-- brute_force_detection.kql <- Detection rule query
+-- workbook/
|   +-- map.json                  <- Sentinel workbook JSON (attack map)
+-- screenshots/
    +-- (all project screenshots)
```

---

*Tools: Microsoft Azure - Microsoft Sentinel - KQL - Windows Server 2019 - Azure Monitor Agent*
