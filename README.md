# Azure Sentinel Cloud SOC Lab

A cloud-native SOC analyst portfolio project built entirely on Microsoft Azure, using a public-facing honeypot and Microsoft Entra ID identity monitoring as two independent data sources feeding into Microsoft Sentinel.

This project is a deliberate companion to my [AD/Splunk home lab](https://github.com/rashik2130/Active-Directory-Home-Lab-with-Splunk-SIEM-and-SOAR-Automation) — that project covers on-prem Active Directory detection and Splunk SOAR automation; this one covers cloud infrastructure exposure and cloud identity monitoring, using Sentinel as the SIEM and Azure Logic Apps for response automation instead of Shuffle.

## Results at a glance

| What | Status |
|---|---|
| Honeypot brute-force detection | ✅ Fired on real, unsolicited internet attack traffic |
| Entra ID failed sign-in detection | ✅ Validated by deliberately triggering failed sign-ins |
| Geo-enrichment attack map | ✅ Built and populated with real attacker IPs |
| Incident triage | ✅ All incidents investigated and formally closed |
| Logic App auto-block playbook | ⚠️ Built and wired to fire on new incidents, but never verified end to end against a real incident — see [Known limitations](#known-limitations) |
| Identity-layer remediation (Conditional Access) | ❌ Not implemented — tenant lacked the required licensing |

## Architecture

- **Honeypot VM** (`corpnet-app01`, Windows 11, East US) — deliberately exposed to the public internet: NSG allows all inbound traffic, Windows Firewall disabled. Attracts real, unsolicited internet scanning and brute-force traffic.
- **Microsoft Entra ID** — a test tenant with a dedicated test user, generating real sign-in activity including MFA registration and legacy-auth-style attempts.
- **Log Analytics workspace** (`law-sentinel-lab`) — central log repository for both data sources.
- **Microsoft Sentinel** — connected to the workspace, running analytics rules against both the honeypot's Windows Security Events and Entra ID sign-in logs.
- **Azure Logic App** — automated response playbook triggered on incident creation, intended to resolve attacker IP entities and create NSG block rules (see [Known limitations](#known-limitations) for its actual verified status).

## Why two data sources

Most honeypot writeups stop at "here's a map of who attacked me." This project treats the honeypot as one signal among several, and deliberately adds Microsoft Entra ID sign-in monitoring — closer to what most real-world SOC analyst roles actually deal with day to day (cloud identity, not just server-side brute force), and something my on-prem AD project doesn't cover at all.

## Build log

### 1. Honeypot infrastructure
- Resource group, VNet, and a Windows 11 VM deployed in Azure (`Standard_D2nls_v6`)
- NSG configured to allow all inbound traffic (documented, deliberate exposure)
- Windows Firewall disabled across all profiles
- Public reachability confirmed via ping from an external network
- Azure Monitor Agent + Data Collection Rule forwarding Windows Security Events to Log Analytics

### 2. Microsoft Entra ID identity monitoring
- Dedicated test user created in the tenant
- Diagnostic settings configured to stream sign-in logs (interactive, non-interactive, service principal) to the same Log Analytics workspace
- Real sign-in activity generated and confirmed in Sign-in logs

### 3. Analytics rules and incidents
Two scheduled analytics rules built in Sentinel:
- **Honeypot RDP Brute Force Detection** (High severity) — queries `SecurityEvent` for `EventID == 4625`, detecting repeated failed RDP logons from a single source IP against the honeypot.
- **Entra ID Repeated Failed Sign-In Detection** (Medium severity) — queries `SigninLogs` for `ResultType != "0"`, grouped by user, triggering at 3+ failures within a 15-minute window (15-minute schedule, 30-minute suppression).

Both rules validated against real data, not just theoretical logic:
- The honeypot rule fired against live, unsolicited internet traffic — two independent attacker IPs, geolocated to Indonesia and Russia, generated over 1,900 combined failed RDP logon attempts against the `administrator` account within a 24-hour window. No successful authentications were found in the reviewed logs for that window.
- The Entra ID rule was validated by deliberately triggering failed sign-ins (including an Entra ID Smart Lockout event) against a dedicated test account, confirming the rule correctly fires once the threshold is met.

### 4. Geo-enrichment and attack map
Built a Sentinel Workbook ("Honeypot Attack Map") using Log Analytics' built-in `geo_info_from_ip_address()` KQL function to resolve attacker IPs to city/country/latitude/longitude — rather than uploading and maintaining a manual GeoIP watchlist CSV. The map plots attacker IPs as markers sized by failed-attempt count; the IPs geolocated to Indonesia, Russia, and Egypt.

### 5. Incident triage
All incidents investigated and formally closed with documented classifications:
- Honeypot incidents classified **True Positive – Malicious user activity**: investigated both source IPs; no successful authentication was found in the reviewed logs despite the volume of attempts.
- Entra ID test incident classified **Informational – Security testing**, since it was a deliberate validation of the detection rule rather than a genuine attack.

### 6. Response automation (Logic App)
Built a Logic App (`la-honeypot-ip-block`) intended to automate the response to Honeypot RDP Brute Force Detection incidents:
- A Sentinel Automation Rule triggers on new incident creation, filtered to this specific analytics rule, and runs the Logic App.
- The workflow uses the Microsoft Sentinel connector's "Entities - Get IPs" action to resolve the incident's IP entities directly (rather than looping over all entity types and filtering manually).
- A "For each" loop iterates over the resolved IPs, and an HTTP action authenticated via the Logic App's system-assigned managed identity calls the Azure Resource Manager REST API to create a Deny inbound security rule on `corpnet-app01-nsg` for each attacker IP, generating the rule name and priority per IP dynamically.
- The managed identity is scoped with Network Contributor access to the NSG resource specifically, not the resource group, following least-privilege.

**Verified status:** the workflow is fully built and wired to Sentinel's automation rules, but I never confirmed a real incident actually produced a working NSG deny rule end to end. The honeypot rule's earliest incidents were missing entity mapping (see below), which the Logic App's IP-resolution step depends on, and I didn't go back and re-test after that was addressed. This should be read as a configured playbook, not a verified one.

I also explored adding Microsoft Entra ID Identity Protection / risk-based Conditional Access as a second, identity-layer remediation path alongside the NSG block — documented under Known limitations below.

## Known limitations

- **Logic App not verified end to end.** The automated NSG-block playbook is built and wired to fire on new incidents, but I never confirmed that a real incident produced an actual NSG deny rule. The honeypot analytics rule wasn't configured with entity mapping (Account/IP/Host) at creation, so its earliest incidents don't show linked entities in the incident graph — and the Logic App's IP-resolution step depends on those entities being present. I didn't go back and re-verify after discovering this. My Azure free trial has since expired, so I can't re-test it now; the next step would be rebuilding this in a new subscription and confirming the full incident → Logic App → NSG rule chain with a real alert.
- **NSG rule priority isn't collision-proof.** The Logic App generates a priority number per IP dynamically, but it isn't guaranteed to be unique — two IPs could theoretically be assigned the same priority, and in Azure NSGs the deny rule also needs a lower priority number than any matching allow rule to actually take effect. I didn't implement explicit collision handling or priority-ordering logic; this is a real gap in the playbook, not just a wording issue.
- Microsoft Entra ID Identity Protection and Conditional Access require P1/P2 licensing that this tenant doesn't have, confirmed directly in the Azure portal (both the legacy risk policy blade and Conditional Access itself returned licensing errors when attempting to configure risk-based policies). A second, identity-layer remediation path using these was designed but not implemented as a result.

## Screenshots

**Honeypot VM deployed and running**
![VM Overview](vm-overview.png)

**Entra ID Repeated Failed Sign-In Detection — validation query**
![Entra ID Sign-In Failures](entra-signin-failures.png)

**Honeypot Attack Map workbook — attacker IPs geo-resolved and plotted**
![Attack Map](attack-map.png)

**Honeypot RDP Brute Force Detection — incident view**
![Incident Triage](incident-triage.png)

**Logic App workflow — Sentinel incident trigger → Entities-Get IPs → For each → HTTP (NSG block)**
![Logic App Workflow](logic-app-workflow.png)
