# Azure Sentinel Cloud SOC Lab

A cloud-native SOC analyst portfolio project built entirely on Microsoft Azure, using a public-facing honeypot and Microsoft Entra ID identity monitoring as two separate data sources feeding into Microsoft Sentinel.

This is meant to be a companion to my [AD/Splunk home lab](https://github.com/rashik2130/Active-Directory-Home-Lab-with-Splunk-SIEM-and-SOAR-Automation). That one covers on-prem Active Directory detection and Splunk SOAR automation. This one covers cloud infrastructure exposure and cloud identity monitoring instead, using Sentinel as the SIEM and Azure Logic Apps for response automation instead of Shuffle.

## Results at a glance

| What | Status |
|---|---|
| Honeypot brute-force detection | ✅ Fired on real, unsolicited internet attack traffic |
| Entra ID failed sign-in detection | ✅ Validated by deliberately triggering failed sign-ins |
| Geo-enrichment attack map | ✅ Built and populated with real attacker IPs |
| Incident triage | ✅ All incidents investigated and formally closed |
| Logic App auto-block playbook | ⚠️ Built and wired to fire on new incidents, but I never verified it end to end against a real incident, see [Known limitations](#known-limitations) |
| Identity-layer remediation (Conditional Access) | ❌ Not implemented, my tenant didn't have the licensing for it |

## Architecture

- **Honeypot VM** (`corpnet-app01`, Windows 11, East US): deliberately exposed to the public internet. NSG allows all inbound traffic, Windows Firewall disabled. Pulls in real, unsolicited internet scanning and brute-force traffic.
- **Microsoft Entra ID**: a test tenant with a dedicated test user, generating real sign-in activity including MFA registration and legacy-auth-style attempts.
- **Log Analytics workspace** (`law-sentinel-lab`): the central log repository for both data sources.
- **Microsoft Sentinel**: connected to the workspace, running analytics rules against both the honeypot's Windows Security Events and Entra ID sign-in logs.
- **Azure Logic App**: an automated response playbook meant to trigger on incident creation and resolve attacker IP entities into NSG block rules (see [Known limitations](#known-limitations) for what I actually confirmed here).

## Why two data sources

Most honeypot writeups stop at "here's a map of who attacked me." I wanted this project to treat the honeypot as one signal among several, so I added Microsoft Entra ID sign-in monitoring on top of it. That's closer to what most real SOC analyst roles actually deal with day to day (cloud identity, not just server-side brute force), and it's something my on-prem AD project doesn't touch at all.

## Build log

### 1. Honeypot infrastructure
- Resource group, VNet, and a Windows 11 VM deployed in Azure (`Standard_D2nls_v6`)
- NSG set to allow all inbound traffic (deliberate exposure, documented on purpose)
- Windows Firewall disabled across all profiles
- Confirmed it was publicly reachable with a ping from an external network
- Azure Monitor Agent plus a Data Collection Rule forwarding Windows Security Events to Log Analytics

### 2. Microsoft Entra ID identity monitoring
- Created a dedicated test user in the tenant
- Set up diagnostic settings to stream sign-in logs (interactive, non-interactive, service principal) to the same Log Analytics workspace
- Generated real sign-in activity and confirmed it was showing up in Sign-in logs

### 3. Analytics rules and incidents
I built two scheduled analytics rules in Sentinel:
- **Honeypot RDP Brute Force Detection** (High severity): queries `SecurityEvent` for `EventID == 4625`, catching repeated failed RDP logons from one source IP against the honeypot.
- **Entra ID Repeated Failed Sign-In Detection** (Medium severity): queries `SigninLogs` for `ResultType != "0"`, grouped by user, triggers at 3+ failures within a 15-minute window (runs every 15 minutes, 30-minute suppression).

Both rules were validated against real data, not just tested on paper:
- The honeypot rule fired on live, unsolicited internet traffic. Two separate attacker IPs, geolocated to Indonesia and Russia, hit the `administrator` account with over 1,900 combined failed RDP attempts in a 24-hour window. I didn't find any successful authentications in the logs for that window.
- I validated the Entra ID rule by deliberately triggering failed sign-ins (including an Entra ID Smart Lockout) against the test account, and confirmed it fires once the threshold is hit.

### 4. Geo-enrichment and attack map
I built a Sentinel Workbook ("Honeypot Attack Map") using Log Analytics' built-in `geo_info_from_ip_address()` KQL function to resolve attacker IPs to city, country, and coordinates, instead of uploading and maintaining a manual GeoIP watchlist CSV. The map plots attacker IPs as markers sized by failed-attempt count. The IPs I saw geolocated to Indonesia, Russia, and Egypt.

### 5. Incident triage
I investigated and formally closed every incident with a documented classification:
- Honeypot incidents classified **True Positive, Malicious user activity**. I looked into both source IPs and didn't find any successful authentication in the logs, despite the volume of attempts.
- The Entra ID test incident was classified **Informational, Security testing**, since I triggered it myself to validate the rule rather than it being a real attack.

### 6. Response automation (Logic App)
I built a Logic App (`la-honeypot-ip-block`) meant to automate the response to Honeypot RDP Brute Force Detection incidents:
- A Sentinel Automation Rule triggers on new incident creation (filtered to this specific analytics rule) and runs the Logic App.
- The workflow uses the Sentinel connector's "Entities - Get IPs" action to pull the incident's IP entities directly instead of looping through every entity type and filtering manually.
- A "For each" loop goes through the resolved IPs, and an HTTP action (authenticated through the Logic App's system-assigned managed identity) calls the Azure Resource Manager REST API to create a Deny inbound rule on `corpnet-app01-nsg` for each attacker IP, generating the rule name and priority per IP dynamically.
- The managed identity only has Network Contributor access scoped to the NSG resource itself, not the whole resource group, to keep it least-privilege.

**Where it actually stands:** the workflow is fully built and wired into Sentinel's automation rules, but I never confirmed a real incident actually produced a working NSG deny rule all the way through. My earliest honeypot incidents were also missing entity mapping (more below), which the Logic App's IP-resolution step depends on, and I didn't go back and re-test after addressing that. So I'd call this a configured playbook, not a verified one.

I also looked into adding Microsoft Entra ID Identity Protection / risk-based Conditional Access as a second, identity-layer remediation path alongside the NSG block, see Known limitations below for why that didn't happen.

## Known limitations

- **Logic App not verified end to end.** The automated NSG-block playbook is built and wired to fire on new incidents, but I never confirmed that a real incident actually produced a working NSG deny rule. My honeypot analytics rule wasn't configured with entity mapping (Account/IP/Host) when I first created it, so its earliest incidents don't show linked entities in the incident graph, and the Logic App's IP-resolution step needs those entities to work. I never went back and re-verified after catching that. My Azure free trial has since run out, so I can't re-test it right now. The next step would be rebuilding this in a new subscription and actually confirming the full incident → Logic App → NSG rule chain against a real alert.
- **NSG rule priority isn't collision-proof.** The Logic App assigns each IP a priority number dynamically, but nothing guarantees it's unique. Two IPs could theoretically land on the same priority, and in Azure NSGs the deny rule also needs a lower priority number than any matching allow rule to actually take effect. I didn't build in any collision handling or priority ordering, so this is a genuine gap in the playbook, not just wording I need to fix.
- Microsoft Entra ID Identity Protection and Conditional Access need P1/P2 licensing that my tenant doesn't have. I confirmed this directly in the Azure portal, both the legacy risk policy blade and Conditional Access itself returned licensing errors when I tried to configure risk-based policies. I designed a second, identity-layer remediation path around these, but never got to implement it because of this.

## Screenshots

**Honeypot VM deployed and running**
![VM Overview](vm-overview.png)

**Entra ID Repeated Failed Sign-In Detection, validation query**
![Entra ID Sign-In Failures](entra-signin-failures.png)

**Honeypot Attack Map workbook, attacker IPs geo-resolved and plotted**
![Attack Map](attack-map.png)

**Honeypot RDP Brute Force Detection, incident view**
![Incident Triage](incident-triage.png)

**Logic App workflow: Sentinel incident trigger → Entities-Get IPs → For each → HTTP (NSG block)**
![Logic App Workflow](logic-app-workflow.png)
