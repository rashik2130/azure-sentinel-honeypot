# Azure Sentinel Cloud SOC Lab

A cloud-native SOC analyst portfolio project built entirely on Microsoft Azure, using a public-facing honeypot and Microsoft Entra ID identity monitoring as two independent data sources feeding into Microsoft Sentinel.

This project is a deliberate companion to my [AD/Splunk home lab](#) — that project covers on-prem Active Directory detection and Splunk SOAR automation; this one covers cloud infrastructure exposure and cloud identity monitoring, using Sentinel as the SIEM and Azure Logic Apps for response automation instead of Shuffle.

## Architecture

*(diagram to be added)*

- **Honeypot VM** (`corpnet-app01`, Windows 11, East US) — deliberately exposed to the public internet: NSG allows all inbound traffic, Windows Firewall disabled. Attracts real, unsolicited internet scanning and brute-force traffic.
- **Microsoft Entra ID** — a test tenant with a dedicated test user, generating real sign-in activity including MFA registration and legacy-auth-style attempts.
- **Log Analytics workspace** (`law-sentinel-lab`) — central log repository for both data sources.
- **Microsoft Sentinel** — connected to the workspace, running analytics rules against both the honeypot's Windows Security Events and Entra ID sign-in logs.
- **Azure Logic App** — automated notification playbook triggered on incident creation.

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
*(to be added)*

### 4. Geo-enrichment and attack map
*(to be added)*

### 5. Incident triage
*(to be added — real incidents worked end-to-end with documented reasoning)*

### 6. Response automation (Logic App)
*(to be added)*

## Known limitations

*(documented honestly as they come up — e.g. anything scoped out or left as a manual step, same approach as the AD/Splunk project)*

## Screenshots

*(added as the project progresses)*
