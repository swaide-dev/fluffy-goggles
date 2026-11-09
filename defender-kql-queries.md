# Defender XDR advanced hunting queries for the application analysis

As of 9 November 2026 · Defender portal only (no Sentinel), so every query is limited to 30 days · Built from `application-analysis-data-requirements.csv`

These queries cover every data-requirements field whose Primary or Secondary source names Defender Advanced Hunting or Defender for Identity. They also include four inferred fields whose source detail uses Defender. Every table, column and ActionType was checked against its Microsoft Learn page. Anything Learn doesn't document is flagged in the query header and listed in [Values to confirm in the portal](#values-to-confirm-in-the-portal).

## How to use this file

1. **Run the preflight first.** KQL-00a shows which tables have data and which licences are live. KQL-00b shows the ActionType values and column fill rates in your data, by OS. KQL-00c shows what the server rule picks up. If a preflight result contradicts something in a query header, fix that first.
2. **Copy each query whole** into Advanced hunting (advanced mode). Every query is self-contained. The parameter block and the SCOPE BLOCK are pasted into each one, identical everywhere, because advanced hunting can't pass one query's results into another. If you change the scope logic, change it in every query. Alternatively, save the scope as a custom function and call that ([custom functions](https://learn.microsoft.com/defender-xdr/advanced-hunting-custom-functions)).
3. **Leave the parameters at their defaults for the first full run.** Each query discovers its own scope. To narrow later, set `HostFilter` (short names or FQDNs), `SubnetFilter` (IPv4 CIDRs), or the window (`Lookback`, `WindowEnd`). Some queries add their own optional parameters at the top, such as `PortFilter` or thresholds.
4. **Time is set in the query.** Leave the portal's time picker on "Set in query". All times are UTC.
5. **If a query times out or truncates**, slice it. Run four 7-day windows (`Lookback = 7d` with `WindowEnd` set to `now()`, `ago(7d)`, `ago(14d)` and `ago(21d)`), or narrow by subnet, then combine the exports. Event-level counts can be summed across slices. Distinct counts can't: recompute those from the combined rows.
6. **Keep provenance.** Every query outputs `QueryId`, `ObservationStartUtc` and `ObservationEndUtc`. Keep them, and the export time, with every file (PRV-01, PRV-02).

### Run order

| Order | Queries | Why this order |
|---|---|---|
| 1 | KQL-00a, KQL-00b, KQL-00c | Confirm tables, licences, ActionTypes and the server rule |
| 2 | KQL-01, KQL-02 | Scope and host identity: the join keys for VMware, Netbox and RunZero |
| 3 | KQL-03, KQL-03b, KQL-04 | Software, end of support and certificates (current snapshot) |
| 4 | KQL-05, KQL-05b | Listening services: the server IP + port key every later query uses |
| 5 | KQL-06, KQL-07, KQL-08 | Who uses each endpoint and what stands in front of them |
| 6 | KQL-09 | FQDNs and probable IP-only access |
| 7 | KQL-10, KQL-10b, KQL-11, KQL-12, KQL-13, KQL-14 | Authentication |
| 8 | KQL-15 | Dependencies |
| 9 | KQL-16, KQL-16b, KQL-17 | Scheduled and temporal |
| 10 | KQL-18 to KQL-23 | Coverage gaps and validation tests |

## Platform limits

All limits are from Microsoft Learn ([quotas and usage parameters](https://learn.microsoft.com/defender-xdr/advanced-hunting-overview#quotas-and-usage-parameters); [API quotas](https://learn.microsoft.com/defender-xdr/api-advanced-hunting)).

| Limit | Value | What it means here |
|---|---|---|
| Data range | 30 days of native Defender XDR data | No Sentinel, so 30 days is the ceiling for every field. Monthly and longer patterns need Splunk. |
| Result rows | 100,000 per query | Grain is one summarised row per key. The largest outputs (KQL-03, KQL-07, KQL-10b, KQL-15) say how to slice them. |
| Result size | 64 MB per query | Wide outputs truncate before 100,000 rows. Arrays are capped with `make_set(..., n)`. |
| Timeout | 10 minutes per query | Filter early, aggregate before joins, smaller table on the left. Slice the window if needed. |
| CPU | Per-tenant allocation in 15-minute cycles | Warning above 10%; queries are blocked at 100% until the next cycle. Space out the heavy queries (KQL-06, KQL-07, KQL-18, KQL-19). |
| API (if scripted) | 100,000 rows; at least 45 calls a minute (varies by tenant); 3-minute timeout | Scripted runs need smaller slices than the portal. |
| Time zone | UTC in queries | Results display in the portal's time-zone setting. Keep exports in UTC. |
| `lookup` right side | Broadcast to every node; fails above "several tens of MBs" ([lookup operator](https://learn.microsoft.com/kusto/query/lookup-operator)) | The estate-wide lookups (`IpOwner`, `DeviceLatest`) are small for most estates. If a query fails with a lookup size error, filter the right side to keys on the left, or narrow by subnet. |

## Evidence conventions

- **Observed** means a value taken directly from a Defender column, or a plain aggregate of one (a set, count, minimum or maximum), with no interpretation.
- **Inferred** means anything derived by logic in the query. That includes classifications, thresholds, port-to-protocol guesses, IP-to-device matches and conclusions drawn from absence (for example "probable IP-only access"). Never copy an Inferred value into the workbook as Observed.
- **Provenance** means `QueryId`, `ObservationStartUtc`, `ObservationEndUtc`, and `ObservationNote` on snapshot tables.
- **Counts are events recorded, not volume.** Learn says Defender for Endpoint "reduces noise in collected data" and "limits data collection" ([aggregated reporting](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)). KQL-23 measures what a count means in your tenant. Treat first seen, last seen and days seen as the reliable measures.
- **IP-to-device matches** use the device that most recently held the IP in DeviceNetworkInfo during the window. Every IP an adapter held in the window is kept, not only its latest. Matches are reliable for servers with static IPs and weaker for DHCP workstations. `IpOwnerCount > 1` marks an IP that more than one device held during the window, and `ServerIpHolders > 1` marks a server IP shared by several servers (Docker bridge, cluster VIP, overlapping ranges). That endpoint row then merges them.
- **IPv4-mapped addresses.** Learn lists `FourToSixMapping` as its own `RemoteIPType`, so an address such as `::ffff:10.1.2.3` is neither "Private" nor "Public". The queries normalise these to IPv4 and decide private, public and loopback themselves (the `IsPrivateIp`, `IsPublicIp` and `IsLoopbackIp` helpers in the SCOPE BLOCK).
- **Aggregated-report rows** (`*AggregatedReport`) summarise events that may also exist as raw rows. They count towards presence, first seen and last seen, but they're kept apart in an `AggregatedReportRows` column so they never double an event count. RemoteUrl fill rates use raw rows only.
- **People are counted by SID** (`InitiatingProcessAccountSid`), so `jdoe@corp.example` and `CORP\jdoe` count once. Labels show the UPN or domain\name. A rule removes system and machine accounts (well-known SIDs, names ending `$`, `dwm-`, `umfd-`), which makes user columns Inferred.
- **Distinct counts use `dcount()`**, which Learn documents as an estimate. It's exact for small numbers and within a few per cent for large ones. Treat large user and client counts as approximate.
- **ConnectionSuccess is a completed TCP handshake**, not proof the application accepted the session ([network protection](https://learn.microsoft.com/defender-endpoint/network-protection#network-protection-and-the-tcp-three-way-handshake)).

## Values to confirm in the portal

Learn's schema pages don't list ActionType values. They point to the in-portal schema reference ([schema](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables)). **To check:** open Advanced hunting, go to the **Schema** tab, select **View reference** next to the table (or open **Schema reference**), then open **Action types**. KQL-00b lists the values your data actually contains.

| Value | Used in | Learn status |
|---|---|---|
| DeviceNetworkEvents `InboundConnectionAccepted` | KQL-05, 05b, 06, 07, 08, 17, 19, 20, 21, 22 | **Not found on Learn.** Confirm in the portal. If it's named differently, change it everywhere. |
| DeviceNetworkEvents `ListeningConnectionCreated` | KQL-05, 05b, 11, 20, 22 | Linux release notes only |
| DeviceNetworkEvents `ConnectionSuccess` | Most queries | Documented ([behind proxies](https://learn.microsoft.com/defender-endpoint/investigate-behind-proxy)) |
| DeviceNetworkEvents `ConnectionAttempt`, `ConnectionAcknowledged` | KQL-12 | Documented ([assess devices](https://learn.microsoft.com/defender-endpoint/assess-devices)) |
| DeviceNetworkEvents `ConnectionFailed` | KQL-12, KQL-15 | **Not found** (only `ConnectionFailedAggregatedReport` is documented) |
| DeviceNetworkEvents `InboundInternetScanInspected` | KQL-05 | Documented ([investigate devices](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)) |
| `*AggregatedReport` ActionTypes | KQL-00a, 06, 07, 08, 12, 15, 17, 18, 19, 20 | Documented ([aggregated reporting](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)) |
| DeviceLogonEvents `LogonSuccess` | KQL-10, KQL-10b | **Not found** (`LogonFailed` appears in a Learn sample). The queries accept any ActionType starting with "Logon". |
| DeviceEvents `ScheduledTask*` and the `TaskName` key in AdditionalFields | KQL-16b | **Not found** |
| IdentityLogonEvents LogonType strings | KQL-11 | Meanings documented, exact strings not, so the queries match with `contains` |
| DeviceLogonEvents LogonType and Protocol strings | KQL-10, KQL-10b, KQL-16b | Not given, so the queries match with `contains` |
| DeviceInfo OSPlatform strings (for example `WindowsServer2019`, `Linux`) | Every query (SCOPE BLOCK) | Not given; KQL-00c shows them |
| DeviceNetworkInfo.IPAddresses JSON keys (`IPAddress`, `SubnetPrefix`) | Every query (SCOPE BLOCK) | Not given; KQL-00a returns a sample |
| DeviceInfo.DeviceRoles JSON shape | KQL-20 | Not given, so the query uses a text search; KQL-00a returns a sample |
| DeviceTvmSoftwareInventory EndOfSupportStatus values | KQL-03 | Not given |
| DeviceInfo.AdditionalFields internet-facing keys | KQL-02, KQL-05 | Documented ([investigate devices](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)); `InternetFacingLastSeen` appears only in the sample query |
| DeviceNetworkInfo `TunnelType` values | KQL-08 | Examples only (6to4, Teredo, ISATAP, PPTP, SSTP, SSH). The query treats None, Teredo, ISATAP and 6to4 as "not a VPN"; check with `DeviceNetworkInfo \| summarize count() by TunnelType` |
| DeviceInfo `ClientVersion` format on Linux | KQL-02 | The release notes give a build number (101.x) and a release version (30.x); the query accepts either. KQL-00c shows which form you have |
| DeviceNetworkEvents `RemoteIPType` value `FourToSixMapping` | Most queries | Documented on the DeviceNetworkEvents page; handled by the classification helpers |

## Windows and Linux differences

| Area | Windows | Linux | Source |
|---|---|---|---|
| Listening ports (`ListeningConnectionCreated`) | Recorded | Only from agent 101.26091.0004 (October 2026), rolling out gradually; KQL-02 flags older agents | [Linux release notes](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases) |
| Internet-facing flag | Windows devices only | Not yet supported | [Investigate devices](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices) |
| Aggregated reporting | Windows 10/11 and Server 2019 or later only | Not listed | [Aggregated reporting](https://learn.microsoft.com/defender-endpoint/aggregated-reporting) |
| Device discovery (finds non-onboarded devices) | Runs from Windows 10 or later and Server 2019 or later | Not listed as a discovery source | [Assess devices](https://learn.microsoft.com/defender-endpoint/assess-devices) |
| DeviceLogonEvents | Not collected on Windows 7 or Server 2008 R2 | Few rows expected | [DeviceLogonEvents](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table) |
| User attribution | UPN and Entra columns | Original login user is in `AdditionalFields` (`InitiatingProcessPosixEffectiveUser`, `LogonID`); UPN usually empty (KQL-00b shows the fill rate) | [DeviceNetworkEvents](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table); [Linux release notes](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases) |
| Unique process ID | `ProcessUniqueId` equals the Process Start Key | Use PID plus process creation time | [Best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices) |
| Agent version numbering | 10.x client version | Release notes show a 101.x build and a 30.x release version for the same release | [Linux release notes](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases) |
| Server rule | Windows Server platforms and DeviceType Server | Linux counts as a server only if it's an Endpoint-category device or onboarded | [DeviceInfo](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table) (DeviceCategory values) |
| RDP session columns (`IsInitiatingProcessRemoteSession` and so on) | Windows RDP sessions | Not applicable | [DeviceNetworkEvents](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table) |
| Process details on network events | TCP only | TCP only | [Investigate devices](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices) |

## Caveats tested, not assumed

| Caveat | What Learn says | Tested by | How |
|---|---|---|---|
| Listener events may only be recorded when a listener opens | Learn doesn't describe when the event fires. Linux only gained it in October 2026. | KQL-22 (and `ListenerEvidence` in KQL-05) | Known-port host test, plus a service restart (see KQL-22) |
| DeviceNetworkEvents may not record every connection | "Defender for Endpoint reduces noise in collected data ... It limits data collection." Aggregated reports add hourly roll-ups on supported Windows versions. | KQL-23 | 100-connection controlled test (see KQL-23). Until it's run, treat counts as presence. |
| RemoteUrl is patchy, so "accessed by IP" is inferred | RemoteUrl is described as the "URL or fully qualified domain name (FQDN) that was being connected to", with no completeness statement | KQL-09 (per-port baseline), KQL-23 (fill rate) | By-name versus by-IP test (see KQL-09) |
| Many users behind one shared IP | IsInitiatingProcessRemoteSession and InitiatingProcessRemoteSessionIP expose the real client behind RDP sessions | KQL-08; `IntermediaryHint` in KQL-06 | Users are counted per client device, so DHCP reuse of an IP isn't mistaken for a shared host. NAT, SNAT and connectors are inferred only from IPs servers see that no onboarded device holds. Check your known Citrix, RDS, ZPA and scanner IPs appear (see KQL-08). |
| Devices that aren't onboarded only appear as the remote end | ConnectionAttempt and ConnectionAcknowledged record non-onboarded devices talking to onboarded ones | KQL-07 (server-side-only rows), KQL-15 (`DestScope`), KQL-18 (gap type 3), KQL-19 | Look up a sample in RunZero and VMware (see KQL-18) |
| Entra sign-in tables need licensing | `EntraIdSignInEvents` needs Entra ID P2. `AadSignInEventsBeta` is retired (Learn gives two removal dates, 9 Dec 2025 and 19 Oct 2026; both have passed). | KQL-00a, KQL-13 | If KQL-00a shows 0 rows: ATH-01 loses SAML/OIDC SSO evidence, ATH-06 loses SaaS and SSO users, and ACC-07 loses the Global Secure Access signal. |
| Defender for Identity sensor health | Shown on the Sensors page and in the Graph sensor resource. No advanced hunting table carries it. | KQL-20 (inference) | Compare with the portal sensor list (see KQL-20) |

## Field coverage

Ratings: **Full** means Defender gives the field as defined for onboarded or sensored hosts. **Partial** means only part of the field, or some onboarded hosts systematically miss it. **None** means Defender can't give it. Devices that aren't onboarded are a separate gap, measured by KQL-18 and KQL-19. The same table is in `defender-kql-index.csv`.

| Field | Queries | Coverage | Notes |
|---|---|---|---|
| APP-05 | KQL-03; KQL-03b | Full | Onboarded servers only. Current snapshot (no timestamp). Names are Defender-normalised, so keep the alias list. |
| APP-06 | KQL-03; KQL-03b | Full | EndOfSupportStatus values aren't documented on Learn; KQL-00a shows them. |
| APP-07 | - | None | Outside the column rule (Primary = Inferred). Application grain needs App IDs; KQL-03 and KQL-03b give vendor inputs only. |
| HST-01 | KQL-01; KQL-02 | Full | FQDN only where the device reports one. Prefer HardwareUuid or MAC for joins. |
| HST-03 | KQL-01; KQL-02 | Full | Test byte order of the first three UUID groups against VMware on a sample. |
| HST-04 | KQL-01; KQL-02 | Full | IPAddresses JSON keys aren't documented; KQL-00a returns a sample. Record the observation date. |
| HST-05 | KQL-01; KQL-02 | Full | KQL-02 Nics keeps each MAC with its own IPs. |
| HST-08 | KQL-02 | Partial | Vendor and Model only when discovery has enough information. HostingPlatformHint is inferred. VMware and RunZero are primary. |
| HST-10 | KQL-01; KQL-02; KQL-03 | Full | OS end of support appears in KQL-03. |
| HST-13 | KQL-01; KQL-02; KQL-18 | Partial | Last seen within 30 days only; no power state (VMware). |
| HST-14 | KQL-01; KQL-18; KQL-19; KQL-21 | Partial | Blind to devices nothing has discovered. Discovered Linux appliances are left out of the server rule. RunZero and VMware supply the full device count. |
| HST-15 | KQL-20 | Partial | Inferred from Defender for Identity event presence. Sensor list and health are only in the portal or Graph. |
| CMP-01 | KQL-05 | Partial | svchost hides service names; UDP has no process; Linux listener events need agent 101.26091.0004 or later. |
| CMP-02 | KQL-05 | Full | For processes seen in the window. |
| CMP-03 | KQL-05; KQL-05b; KQL-22 | Partial | Listener events are recorded when a listener opens; KQL-22 measures the gap. |
| CMP-04 | KQL-05; KQL-05b | Partial | Outside the column rule (Primary = Inferred); included as inferred columns. |
| CMP-05 | KQL-05; KQL-15 | Partial | Outside the column rule; inferred tier hint only. Confirm with the technical owner. |
| CMP-06 | KQL-05; KQL-10; KQL-10b | Full | Account the listening process runs as; Service logons cross-check. |
| CMP-07 | KQL-06; KQL-07 | Partial | Clients and users of port 445 only; no share names (Splunk 5140/5145). |
| CMP-08 | KQL-04 | Partial | Preview table: installed certificates only, no SAN column, no port binding. RunZero is primary. |
| ACC-01 | KQL-06; KQL-09 | Partial | RemoteUrl isn't filled for every connection; KQL-23 measures the fill rate per port. |
| ACC-02 | KQL-09; KQL-23 | Partial | Inferred against a per-port RemoteUrl baseline. Corroborate with Splunk DNS logs before listing. |
| ACC-03 | KQL-06; KQL-07 | Full | Onboarded clients only. |
| ACC-04 | KQL-05; KQL-06; KQL-07 | Partial | Inferred from port and process. RunZero fingerprints are primary. |
| ACC-05 | KQL-05; KQL-10; KQL-11 | Partial | Cleartext LDAP binds, NTLM, cleartext ports. No SMBv1 or NTLMv1 signal. |
| ACC-06 | KQL-07; KQL-06 | Partial | Client IP and /24 for the Netbox join. IdentityInfo city and country is the user's home location, not the connection site. |
| ACC-07 | KQL-08; KQL-13 | Partial | Inferred hints only (tunnel adapter, RDS, connector fan-out, Global Secure Access). ZPA logs are primary. |
| ACC-08 | KQL-08; KQL-06 | Partial | Inferred from thresholds. Owned by the network team. |
| ACC-09 | KQL-02; KQL-05 | Partial | Windows devices only; the tag drops after 48 hours without events; not a full exposure assessment. |
| ACC-10 | KQL-06 | Partial | Outside the column rule; inferred latency-risk hint from the protocol. |
| ATH-01 | KQL-10; KQL-11; KQL-13 | Partial | Kerberos, NTLM, LDAP bind, local accounts, Entra SSO. Forms, SQL and API-key auth leave no trace. |
| ATH-02 | KQL-02; KQL-10; KQL-11 | Partial | Domains from logons. DCs without a sensor are missing from KQL-11. |
| ATH-03 | KQL-12; KQL-20 | Partial | Server-side network evidence works without a sensor. Which DC users use needs DC logs. |
| ATH-04 | KQL-11 | Partial | Delegation in use only. SPNs and delegation settings need a directory query. |
| ATH-05 | KQL-14; KQL-16 | Partial | Masked command-line hints only. |
| ATH-06 | KQL-06; KQL-07; KQL-10b; KQL-13 | Partial | No UPN on Linux; intermediaries hide users; back-end tiers show service accounts. |
| ATH-07 | KQL-06; KQL-07 | Partial | Count covers the window only. Department and country need a UPN match in IdentityInfo. |
| DEP-01 | KQL-15 | Full | TCP. Process visible only at the onboarded end. |
| DEP-02 | KQL-15; KQL-12; KQL-02 | Partial | UDP services (DNS, NTP, DHCP) may be missing. Configured DNS and DHCP servers come from KQL-02. |
| DEP-03 | KQL-15 | Partial | Proxied egress shows the proxy IP. Partner allow-lists of a post's public IP are invisible. |
| TMP-01 | KQL-16; KQL-16b; KQL-10 | Partial | 30 days misses monthly and longer jobs. Application schedulers aren't recognised. |
| TMP-02 | KQL-17; KQL-16 | Partial | One month-end at most (a run today covers 31 Oct). Rerun the week of 7 Dec 2026. |
| TMP-03 | KQL-06; KQL-17 | Full | Within 30 days only. No use in 30 days doesn't prove unused. |

## The SCOPE BLOCK

Every query that needs servers starts with the same block, so each one discovers its own scope from the data. In order, it:

1. defines helpers: `Norm()` turns `::ffff:a.b.c.d` into `a.b.c.d`; `IsPrivateIp()`, `IsPublicIp()` and `IsLoopbackIp()` classify addresses correctly for IPv4-mapped forms; `ServerRule()` is the server rule
2. takes the latest `DeviceInfo` row per device in the window, dropping merged device IDs (`DeviceLatest`)
3. expands every IP each adapter held in the window from `DeviceNetworkInfo` (`DeviceIPs`) and builds an IP-to-device map with a holder count (`IpOwner`)
4. classes a device as a server when `DeviceType` is Server, `OSPlatform` starts with WindowsServer, or `OSPlatform` is Linux and the device is in the Endpoint category or onboarded (this keeps out discovered Linux appliances), then applies `HostFilter` and `SubnetFilter` (`Servers`)
5. builds the list of server IPs, including the IPv4-mapped form DeviceNetworkEvents can record (`ServerIPs`, `ServerIPMatch`), and one name per server IP with a count of servers sharing it (`ServerIpName`)

Domain controllers are found from the data too: devices that accept Kerberos on port 88 (the method Microsoft's own sample uses), IPs that several onboarded devices send Kerberos to, and devices whose role mentions domain controller (KQL-11, KQL-12, KQL-20).

```kql
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
```


## Preflight (run first)

### KQL-00a: Preflight: table availability, licensing and undocumented JSON shapes

**CSV fields:** HST-14, HST-15, ATH-01, APP-05, APP-06, CMP-08, DEP-01  
**Purpose:** Shows which tables hold data in your tenant, so you know which licences and features are live before trusting any later result. A table with Rows = 0 means the service or licence behind it is missing, and every field that depends on it loses coverage. It also returns one sample value from each JSON column whose keys aren't documented on Learn, so you can confirm the keys the later queries parse.  
**Tables:** `DeviceInfo`, `DeviceNetworkInfo`, `DeviceNetworkEvents`, `DeviceProcessEvents`, `DeviceLogonEvents`, `DeviceEvents`, `DeviceTvmSoftwareInventory`, `DeviceTvmCertificateInfo`, `IdentityLogonEvents`, `IdentityDirectoryEvents`, `IdentityQueryEvents`, `IdentityInfo`, `EntraIdSignInEvents`  
**Verified against (Microsoft Learn):**

- [Advanced hunting overview: quotas and usage parameters](https://learn.microsoft.com/defender-xdr/advanced-hunting-overview#quotas-and-usage-parameters)
- [Understand the advanced hunting schema (in-portal schema reference)](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables)
- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [DeviceTvmSoftwareInventory table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmsoftwareinventory-table)
- [DeviceTvmCertificateInfo table (preview)](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmcertificateinfo-table)
- [IdentityLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitylogonevents-table)
- [IdentityDirectoryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitydirectoryevents-table)
- [IdentityQueryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityqueryevents-table)
- [IdentityInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityinfo-table)
- [EntraIdSignInEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-entraidsigninevents-table)
- [AADSignInEventsBeta (deprecated)](https://learn.microsoft.com/defender-xdr/advanced-hunting-aadsignineventsbeta-table)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [Investigate devices: internet-facing devices, InboundInternetScanInspected, TCP-only process data](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)

```kql
// KQL-00a | Preflight: table availability, licensing and undocumented JSON shapes
// Grain: one row per check. Run this before anything else.
// HostFilter and SubnetFilter aren't used: preflight checks are tenant-wide.
// Observed: Rows, Devices, LatestUtc, Values, Sample.  Inferred: none.  Provenance: Check (label written by this query).
// Rows = 0 for EntraIdSignInEvents: no Entra ID P2 (ATH-01 SSO, ATH-06 SaaS users, ACC-07 GSA lose coverage).
// Rows = 0 for Identity* tables: no Defender for Identity data (ATH-01..04, ACC-05 MDI parts, HST-15 lose coverage).
// Rows = 0 for the aggregated-report check: aggregated reporting is off (counts are presence only; see KQL-23).
// ---- Parameters (defaults = everything) ----
let Lookback    = 30d;             // native Defender data stops at 30 days
let WindowEnd   = now();           // set to ago(7d)... to run the window in slices
let WindowStart = WindowEnd - Lookback;
union isfuzzy=true
    (DeviceInfo | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceInfo"),
    (DeviceNetworkInfo | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceNetworkInfo"),
    (DeviceNetworkEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceNetworkEvents"),
    (DeviceProcessEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceProcessEvents"),
    (DeviceLogonEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceLogonEvents"),
    (DeviceEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Table: DeviceEvents"),
    (DeviceTvmSoftwareInventory
        | summarize Rows = count(), Devices = dcount(DeviceId)
        | extend Check = "Table: DeviceTvmSoftwareInventory (snapshot, no Timestamp)"),
    (DeviceTvmCertificateInfo
        | summarize Rows = count(), Devices = dcount(DeviceId)
        | extend Check = "Table: DeviceTvmCertificateInfo (preview, snapshot)"),
    (IdentityLogonEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), LatestUtc = max(Timestamp), Values = make_set(Application, 10)
        | extend Check = "Table: IdentityLogonEvents (Values = Application)"),
    (IdentityDirectoryEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), LatestUtc = max(Timestamp), Values = make_set(Application, 10)
        | extend Check = "Table: IdentityDirectoryEvents (Values = Application)"),
    (IdentityQueryEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), LatestUtc = max(Timestamp), Values = make_set(Protocol, 10)
        | extend Check = "Table: IdentityQueryEvents (Values = Protocol)"),
    (IdentityInfo | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), LatestUtc = max(Timestamp), Values = make_set(SourceProviders, 10)
        | extend Check = "Table: IdentityInfo (Values = SourceProviders)"),
    (EntraIdSignInEvents | where Timestamp between (WindowStart .. WindowEnd)
        | summarize Rows = count(), LatestUtc = max(Timestamp)
        | extend Check = "Table: EntraIdSignInEvents (needs Entra ID P2)"),
    (DeviceNetworkEvents | where Timestamp between (WindowStart .. WindowEnd)
        | where ActionType endswith "AggregatedReport"
        | summarize Rows = count(), Devices = dcount(DeviceId), LatestUtc = max(Timestamp)
        | extend Check = "Feature: aggregated reporting (*AggregatedReport rows in DeviceNetworkEvents)"),
    (DeviceNetworkInfo | where Timestamp > WindowEnd - 1d
        | where isnotempty(IPAddresses) and IPAddresses != "[]"
        | take 1
        | project Check = "Sample: DeviceNetworkInfo.IPAddresses (confirm keys IPAddress and SubnetPrefix)", Sample = IPAddresses),
    (DeviceInfo | where Timestamp > WindowEnd - 1d
        | where isnotempty(DeviceRoles) and DeviceRoles != "[]"
        | take 1
        | project Check = "Sample: DeviceInfo.DeviceRoles (confirm how a domain controller role is written)", Sample = DeviceRoles),
    (DeviceInfo | where Timestamp > WindowEnd - 1d
        | where IsInternetFacing == true
        | take 1
        | project Check = "Sample: DeviceInfo.AdditionalFields on an internet-facing device", Sample = AdditionalFields),
    (DeviceTvmCertificateInfo
        | take 1
        | project Check = "Sample: DeviceTvmCertificateInfo.IssuedTo (confirm keys)", Sample = tostring(IssuedTo))
| extend Devices = column_ifexists("Devices", long(null)), LatestUtc = column_ifexists("LatestUtc", datetime(null)),
         Values = column_ifexists("Values", dynamic(null)), Sample = column_ifexists("Sample", "")
| project Check, Rows, Devices, LatestUtc, Values, Sample
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| Check | Provenance | Name of the check (written by the query) |
| Rows | Observed | Rows in the window (TVM tables: current rows) |
| Devices | Observed | Distinct DeviceId values |
| LatestUtc | Observed | Most recent Timestamp: shows whether data is still arriving |
| Values | Observed | Distinct Application, Protocol or SourceProviders values |
| Sample | Observed | One raw JSON value, for confirming key names |

**Known gaps**

- Rows = 0 can mean "no licence", "not deployed" or "nothing happened in the window". LatestUtc tells them apart for event tables.
- The TVM tables have no Timestamp column, so they can't show freshness.

**Validation check**

Compare the device counts with the Defender portal: **Assets > Devices** should show a similar onboarded total to the `DeviceInfo` Devices figure. For Defender for Identity, the **Settings > Identities > Sensors** list should be non-empty if the Identity* rows are above zero.

### KQL-00b: Preflight: ActionType values and column fill rates by OS platform

**CSV fields:** CMP-03, CMP-06, ACC-01, ACC-02, ATH-06, TMP-01  
**Purpose:** Lists the ActionType values your tenant actually records, split by OS platform, and how often the columns later queries depend on are filled (RemoteUrl, UPN, account, process). This is the in-data check for ActionType values that Learn doesn't list, and it shows Windows and Linux differences in your own data rather than in theory.  
**Tables:** `DeviceNetworkEvents`, `DeviceLogonEvents`, `DeviceEvents`, `DeviceProcessEvents`, `DeviceInfo`  
**Verified against (Microsoft Learn):**

- [Understand the advanced hunting schema (in-portal schema reference)](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables)
- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)
- [DeviceEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceevents-table)
- [DeviceProcessEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceprocessevents-table)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)

```kql
// KQL-00b | Preflight: ActionType values and column fill rates by OS platform
// Grain: one row per table + ActionType + OSPlatform.
// HostFilter and SubnetFilter aren't used: preflight checks are tenant-wide.
// Observed: Table, ActionType, OSPlatform, Devices, Events, Protocols, Values.
// Inferred: PctRemoteUrl, PctUpn, PctAccount, PctProcess (ratios computed here from observed counts).
// Use the result to confirm the ActionType names used later: ListeningConnectionCreated, InboundConnectionAccepted,
// ConnectionSuccess, ConnectionFailed, ConnectionAttempt, LogonSuccess, LogonFailed, ScheduledTask*.
// The authority is the in-portal schema reference: Advanced hunting > Schema > (table) > View reference > Action types.
// ---- Parameters (defaults = everything) ----
let Lookback    = 30d;             // native Defender data stops at 30 days
let WindowEnd   = now();           // set to ago(7d)... to run the window in slices
let WindowStart = WindowEnd - Lookback;
let Platform = DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform)
    | summarize arg_max(Timestamp, OSPlatform) by DeviceId
    | project DeviceId, OSPlatform;
let Net = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | summarize Events = count(),
                WithRemoteUrl = countif(isnotempty(RemoteUrl)),
                WithUpn = countif(isnotempty(InitiatingProcessAccountUpn)),
                WithAccount = countif(isnotempty(InitiatingProcessAccountName)),
                WithProcess = countif(isnotempty(InitiatingProcessFileName)),
                Protocols = make_set(Protocol, 5)
        by DeviceId, ActionType
    | extend Table = "DeviceNetworkEvents";
let Logon = DeviceLogonEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | summarize Events = count(), WithAccount = countif(isnotempty(AccountName)),
                Values = make_set(strcat(LogonType, " / ", Protocol), 20)
        by DeviceId, ActionType
    | extend Table = "DeviceLogonEvents (Values = LogonType / Protocol)";
let Misc = DeviceEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType startswith "ScheduledTask" or ActionType startswith "Dns"
    | summarize Events = count() by DeviceId, ActionType
    | extend Table = "DeviceEvents (ScheduledTask* and Dns* only)";
let Proc = DeviceProcessEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | summarize Events = count(), WithAccount = countif(isnotempty(AccountName)) by DeviceId, ActionType
    | extend Table = "DeviceProcessEvents";
union Net, Logon, Misc, Proc
| lookup kind=leftouter Platform on DeviceId
| extend OSPlatform = iff(isempty(OSPlatform), "(no DeviceInfo row)", OSPlatform)
| summarize Devices = dcount(DeviceId), Events = sum(Events),
            WithRemoteUrl = sum(WithRemoteUrl), WithUpn = sum(WithUpn), WithAccount = sum(WithAccount),
            WithProcess = sum(WithProcess), Values = make_set(Values, 30), Protocols = make_set(Protocols, 5)
    by Table, ActionType, OSPlatform
| extend PctRemoteUrl = round(100.0 * WithRemoteUrl / Events, 1),
         PctUpn = round(100.0 * WithUpn / Events, 1),
         PctAccount = round(100.0 * WithAccount / Events, 1),
         PctProcess = round(100.0 * WithProcess / Events, 1)
| project Table, ActionType, OSPlatform, Devices, Events, PctRemoteUrl, PctUpn, PctAccount, PctProcess, Protocols, Values
| order by Table asc, ActionType asc, Devices desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| Table | Provenance | Source table (label written by the query) |
| ActionType | Observed | ActionType value as recorded |
| OSPlatform | Observed | DeviceInfo.OSPlatform of the reporting device |
| Devices | Observed | Distinct devices reporting this ActionType |
| Events | Observed | Events recorded (not activity volume) |
| PctRemoteUrl | Inferred | Share of events with RemoteUrl filled |
| PctUpn | Inferred | Share with InitiatingProcessAccountUpn filled |
| PctAccount | Inferred | Share with an account name filled |
| PctProcess | Inferred | Share with InitiatingProcessFileName filled |
| Protocols | Observed | Protocol values seen |
| Values | Observed | LogonType / Protocol pairs (DeviceLogonEvents only) |

**Known gaps**

- It scans 30 days of four large tables. If it times out, set `WindowEnd = ago(23d)` and `Lookback = 7d` and run it per week.
- It shows which ActionTypes exist, not what each one means. Use the in-portal schema reference for meanings.

**Validation check**

On a Linux row, `ListeningConnectionCreated` should appear only for devices on agent 101.26091.0004 or later (KQL-02 shows each host's agent). If Linux shows no `InboundConnectionAccepted` at all, treat Linux inbound evidence in KQL-05, KQL-06 and KQL-21 as missing, not as zero use.

### KQL-00c: Preflight: DeviceType and OSPlatform values, and what the server rule picks up

**CSV fields:** HST-10, HST-14  
**Purpose:** Answers "what does the server rule catch and miss" before you rely on it. You said there's no reliable naming convention, tag or group, so servers are classed by DeviceType and OSPlatform. This query shows every combination of those values with a device count, and whether the rule includes it.  
**Tables:** `DeviceInfo`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)

```kql
// KQL-00c | Preflight: DeviceType x OSPlatform x OnboardingStatus, with the server rule applied
// Grain: one row per OSPlatform + DeviceType + DeviceCategory + OnboardingStatus + InServerScope.
// HostFilter and SubnetFilter aren't used: preflight checks are tenant-wide.
// Observed: OSPlatform, DeviceType, DeviceCategory, OnboardingStatus, Devices, SampleClientVersions.
// Inferred: InServerScope (the same rule as the SCOPE BLOCK).
// The OSPlatform strings (for example "WindowsServer2019", "Linux") aren't listed on the DeviceInfo schema page;
// this query shows the strings your tenant actually uses.
// ---- Parameters (defaults = everything) ----
let Lookback    = 30d;             // native Defender data stops at 30 days
let WindowEnd   = now();           // set to ago(7d)... to run the window in slices
let WindowStart = WindowEnd - Lookback;
DeviceInfo
| where Timestamp between (WindowStart .. WindowEnd)
| where isnotempty(OSPlatform) or isnotempty(DeviceType)
| summarize arg_max(Timestamp, OSPlatform, DeviceType, DeviceCategory, OnboardingStatus, MergedToDeviceId, ClientVersion) by DeviceId
| where isempty(MergedToDeviceId)
| extend InServerScope = DeviceType =~ "Server" or OSPlatform startswith "WindowsServer"
                        or (OSPlatform =~ "Linux" and (DeviceCategory =~ "Endpoint" or OnboardingStatus == "Onboarded"))
| summarize Devices = dcount(DeviceId), SampleClientVersions = make_set(ClientVersion, 5)
    by OSPlatform, DeviceType, DeviceCategory, OnboardingStatus, InServerScope
| order by InServerScope desc, Devices desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| OSPlatform | Observed | DeviceInfo.OSPlatform |
| DeviceType | Observed | DeviceInfo.DeviceType |
| DeviceCategory | Observed | DeviceInfo.DeviceCategory |
| OnboardingStatus | Observed | DeviceInfo.OnboardingStatus |
| InServerScope | Inferred | True if the scope rule classes it as a server |
| Devices | Observed | Distinct devices |
| SampleClientVersions | Observed | Up to five agent versions |

**Known gaps**

- What the rule misses, by design: Windows 10/11 machines running server workloads (KQL-21 looks for these); Linux workstations (wrongly included; KQL-21 flags Linux "servers" with no inbound traffic); discovered Linux devices outside the Endpoint category (NAS, appliances), which are left out on purpose; hypervisors Defender can't onboard; and devices nothing has discovered.

**Validation check**

Pick five server names you know (for example a DC, a file server and a Linux host) and confirm each appears in a row with `InServerScope = true`. Then filter to `InServerScope = false` and `DeviceType = "Workstation"`, and look for names that are really servers.


## Scope

### KQL-01: Server scope

**CSV fields:** HST-01, HST-03, HST-04, HST-05, HST-10, HST-13, HST-14  
**Purpose:** The starting list: every device Defender classes as a server, onboarded or only discovered, with the reason it was classed as a server and the keys you need to join it to VMware, Netbox and RunZero. Every later query re-derives this same list with the identical SCOPE BLOCK, because advanced hunting can't pass results from one query to the next.  
**Tables:** `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)
- [Advanced hunting custom functions](https://learn.microsoft.com/defender-xdr/advanced-hunting-custom-functions)

```kql
// KQL-01 | Server scope: every device classed as a server, with join keys
// Fields: HST-01, HST-03, HST-04, HST-05, HST-10, HST-13, HST-14
// Grain: one row per DeviceId.
// Observed: DeviceId, DeviceName, HardwareUuid, MacAddresses, IPAddresses, OnboardingStatus, SensorHealthState,
//           OSPlatform, OSDistribution, OSVersion, DeviceType, ClientVersion, LastSeenUtc.
// Inferred: HostShortName, ServerReason, DefenderAgent.
// Unconfirmed on Learn: JSON keys inside DeviceNetworkInfo.IPAddresses (check with KQL-00a).
// SCOPE BLOCK logic: the latest DeviceInfo row per device in the window (merged IDs removed); a device is a server
// if DeviceType is Server, OSPlatform starts with WindowsServer, or OSPlatform is Linux and the device is an Endpoint-category
// device or onboarded; then HostFilter and SubnetFilter. IPAddresses lists every IP the device held in the window.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
Servers
| lookup kind=leftouter (DeviceIPs
    | summarize IPAddresses = make_set(IP, 50), MacAddresses = make_set_if(MacAddress, isnotempty(MacAddress), 20)
        by DeviceId) on DeviceId
| extend DefenderAgent = iff(OnboardingStatus == "Onboarded", "Yes", "No")
| project DeviceId, DeviceName, HostShortName, HardwareUuid, MacAddresses, IPAddresses,
          ServerReason, DefenderAgent, OnboardingStatus, SensorHealthState,
          OSPlatform, OSDistribution, OSVersion, DeviceType, ClientVersion, LastSeenUtc
| extend QueryId = "KQL-01", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DefenderAgent asc, DeviceName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId | Observed | DeviceInfo.DeviceId |
| DeviceName | Observed | DeviceInfo.DeviceName (FQDN where the device reports one) |
| HostShortName | Inferred | First label of DeviceName, lower case |
| HardwareUuid | Observed | DeviceInfo.HardwareUuid (joins to the VMware BIOS UUID) |
| MacAddresses | Observed | DeviceNetworkInfo.MacAddress of adapters that have an IP |
| IPAddresses | Observed | DeviceNetworkInfo.IPAddresses: every IP the device held in the window, flattened |
| ServerReason | Inferred | Which part of the server rule matched |
| DefenderAgent | Inferred | Yes when OnboardingStatus is Onboarded (workbook "Defender agent?" column) |
| OnboardingStatus | Observed | Onboarded, Can be onboarded, Unsupported or Insufficient info |
| SensorHealthState | Observed | Active, Inactive, Misconfigured and so on |
| OSPlatform, OSDistribution, OSVersion, DeviceType, ClientVersion | Observed | DeviceInfo columns |
| LastSeenUtc | Observed | Timestamp of the latest DeviceInfo row in the window |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window, in UTC |

**Known gaps**

- A device that hasn't reported or been discovered in the window doesn't exist here. Powered-off DR and period-end hosts drop out after 30 days, so VMware and RunZero must supply the full inventory.
- Discovered (not onboarded) devices often have no NIC rows, so IPs and MACs can be empty.
- `HardwareUuid` byte order may differ from VMware's BIOS UUID in the first three groups (CSV HST-03 caveat).

**Validation check**

Take ten VMs from the VMware Dataset 1 extract and match them on `HardwareUuid`. Try the raw value first, then with the first three groups byte-swapped. Record which form matches; that decides the join rule for the whole estate. Also check that one known Linux server and one DC appear here.


## Host identity

### KQL-02: Host identity, NICs and network configuration

**CSV fields:** HST-01, HST-03, HST-04, HST-05, HST-08, HST-10, HST-13, HST-14, ACC-09, ATH-02, DEP-02  
**Purpose:** One row per server with everything Defender knows about the host. That covers join keys, operating system detail, hosting hints, tags, internet exposure, and NIC detail with each MAC kept beside its own IPs. It also includes the configured DNS servers, DHCP server and gateways (DEP-02 configuration evidence). It flags Linux hosts whose agent is too old to record listening ports.  
**Tables:** `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Investigate devices: internet-facing devices, InboundInternetScanInspected, TCP-only process data](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)

```kql
// KQL-02 | Host identity, NICs and network configuration
// Fields: HST-01, HST-03, HST-04, HST-05, HST-08, HST-10, HST-13, HST-14, ACC-09, ATH-02 (JoinType), DEP-02 (DNS/DHCP config)
// Grain: one row per DeviceId.
// Observed: DeviceId, DeviceName, HardwareUuid, MacAddresses, IPv4Addresses, IPv6Addresses, Nics, OnboardingStatus,
//   SensorHealthState, ClientVersion, OSPlatform, OSDistribution, OSVersion, OSVersionInfo, OSBuild, OsBuildRevision,
//   OSArchitecture, DeviceType, Vendor, Model, CloudPlatforms, JoinType, IsAzureADJoined, MachineGroup, DeviceManualTags,
//   DeviceDynamicTags, RegistryDeviceTag, DeviceRoles, DiscoverySources, IsTransient, ExposureLevel, AssetValue,
//   IsInternetFacing, InternetFacing* (documented AdditionalFields keys), PublicIP, DnsServerSets, DhcpServers,
//   GatewaySets, AdapterDnsSuffixes, LastSeenUtc, NicReportedUtc.
// Inferred: HostShortName, DomainSuffix, ServerReason, DefenderAgent, LinuxListenerTelemetryExpected, HostingPlatformHint.
// Platform: IsInternetFacing is set for Windows devices only (Learn: investigate-machines). The tag drops after 48 hours without events.
// The release notes give Linux builds two numbers (101.x build, 30.x release); either ClientVersion form is handled.
// IPv4Addresses and IPv6Addresses list every IP held in the window; Nics shows each adapter's current IPs.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let LinuxListenerMinBuild   = "101.26091.0004";    // first Linux build with ListeningConnectionCreated (release notes, Oct 2026)
let LinuxListenerMinRelease = "30.126091.0004.0";  // the same release in Microsoft's "release version" numbering
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let ScopeIds = Servers | project DeviceId;
let Detail = DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where DeviceId in (ScopeIds)
    | summarize arg_max(Timestamp, OSVersionInfo, OSBuild, OsBuildRevision, OSArchitecture, Vendor, Model,
                        JoinType, IsAzureADJoined, MachineGroup, DeviceManualTags, DeviceDynamicTags, RegistryDeviceTag,
                        IsInternetFacing, AdditionalFields, ExposureLevel, AssetValue, DeviceRoles, DiscoverySources,
                        CloudPlatforms, IsTransient, PublicIP) by DeviceId
    | extend InternetFacingReason = extractjson("$.InternetFacingReason", AdditionalFields, typeof(string)),
             InternetFacingLocalIp = extractjson("$.InternetFacingLocalIp", AdditionalFields, typeof(string)),
             InternetFacingLocalPort = extractjson("$.InternetFacingLocalPort", AdditionalFields, typeof(int)),
             InternetFacingPublicScannedIp = extractjson("$.InternetFacingPublicScannedIp", AdditionalFields, typeof(string)),
             InternetFacingPublicScannedPort = extractjson("$.InternetFacingPublicScannedPort", AdditionalFields, typeof(int)),
             InternetFacingTransportProtocol = extractjson("$.InternetFacingTransportProtocol", AdditionalFields, typeof(string)),
             InternetFacingLastSeen = extractjson("$.InternetFacingLastSeen", AdditionalFields, typeof(datetime))
    | project-away Timestamp, AdditionalFields;
let NicBase = DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where DeviceId in (ScopeIds)
    | summarize arg_max(Timestamp, MacAddress, IPAddresses, NetworkAdapterType, NetworkAdapterStatus, TunnelType,
                        DnsAddresses, IPv4Dhcp, DefaultGateways, NetworkAdapterDnsSuffix, NetworkAdapterVendor)
        by DeviceId, NetworkAdapterName;
let NicIps = NicBase
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IpText = strcat(tostring(IpObj.IPAddress), "/", tostring(IpObj.SubnetPrefix))
    | where IpText != "/"
    | summarize NicIpList = make_set(IpText, 20) by DeviceId, NetworkAdapterName;
let Nics = NicBase
    | lookup kind=leftouter NicIps on DeviceId, NetworkAdapterName
    | extend NicIpText = iff(isnull(NicIpList), "", strcat_array(NicIpList, ","))
    | summarize Nics = make_list(strcat(NetworkAdapterName, " | ", MacAddress, " | ", NicIpText, " | ", NetworkAdapterStatus), 20),
                MacAddresses = make_set_if(MacAddress, isnotempty(MacAddress), 20),
                DnsServerSets = make_set_if(DnsAddresses, isnotempty(DnsAddresses) and DnsAddresses != "[]", 10),
                DhcpServers = make_set_if(IPv4Dhcp, isnotempty(IPv4Dhcp), 10),
                GatewaySets = make_set_if(DefaultGateways, isnotempty(DefaultGateways) and DefaultGateways != "[]", 10),
                AdapterDnsSuffixes = make_set_if(NetworkAdapterDnsSuffix, isnotempty(NetworkAdapterDnsSuffix), 10),
                NicReportedUtc = max(Timestamp)
        by DeviceId;
let IpSplit = DeviceIPs
    | where DeviceId in (ScopeIds)
    | summarize IPv4Addresses = make_set_if(IP, not(IP contains ":"), 50),
                IPv6Addresses = make_set_if(IP, IP contains ":", 50) by DeviceId;
Servers
| lookup kind=leftouter Detail on DeviceId
| lookup kind=leftouter Nics on DeviceId
| lookup kind=leftouter IpSplit on DeviceId
| extend DomainSuffix = iff(DeviceName contains ".", substring(DeviceName, indexof(DeviceName, ".") + 1), ""),
         DefenderAgent = iff(OnboardingStatus == "Onboarded", "Yes", "No"),
         LinuxListenerTelemetryExpected = case(
             not(OSPlatform =~ "Linux"), "Not Linux",
             isempty(ClientVersion), "Unknown: no agent version recorded",
             ClientVersion startswith "101." and parse_version(ClientVersion) >= parse_version(LinuxListenerMinBuild), "Yes",
             ClientVersion startswith "30." and parse_version(ClientVersion) >= parse_version(LinuxListenerMinRelease), "Yes",
             ClientVersion startswith "101." or ClientVersion startswith "30.", "No: agent older than 101.26091.0004",
             "Unknown: version format not recognised"),
         HostingPlatformHint = case(
             Vendor has "VMware" or Model has "VMware", "VMware VM",
             Vendor has "Microsoft" and Model has "Virtual", "Hyper-V or Azure VM",
             isnotempty(CloudPlatforms) and CloudPlatforms != "[]", strcat("Cloud: ", CloudPlatforms),
             isnotempty(Vendor), "Physical or other hypervisor: check RunZero",
             "Unknown: no Vendor or Model recorded")
| project DeviceId, DeviceName, HostShortName, DomainSuffix, HardwareUuid, MacAddresses, IPv4Addresses, IPv6Addresses, Nics,
          ServerReason, DefenderAgent, OnboardingStatus, SensorHealthState, ClientVersion, LinuxListenerTelemetryExpected,
          OSPlatform, OSDistribution, OSVersion, OSVersionInfo, OSBuild, OsBuildRevision, OSArchitecture, DeviceType,
          Vendor, Model, HostingPlatformHint, CloudPlatforms, JoinType, IsAzureADJoined, MachineGroup,
          DeviceManualTags, DeviceDynamicTags, RegistryDeviceTag, DeviceRoles, DiscoverySources, IsTransient,
          ExposureLevel, AssetValue, IsInternetFacing, InternetFacingReason, InternetFacingLocalIp, InternetFacingLocalPort,
          InternetFacingPublicScannedIp, InternetFacingPublicScannedPort, InternetFacingTransportProtocol, InternetFacingLastSeen,
          PublicIP, DnsServerSets, DhcpServers, GatewaySets, AdapterDnsSuffixes, LastSeenUtc, NicReportedUtc
| extend QueryId = "KQL-02", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid | Observed | DeviceInfo |
| HostShortName, DomainSuffix | Inferred | Split from DeviceName |
| MacAddresses | Observed | DeviceNetworkInfo.MacAddress, all adapters (including adapters without an IP) |
| IPv4Addresses, IPv6Addresses | Observed | Every IP held in the window (DeviceNetworkInfo.IPAddresses), split by family |
| Nics | Observed | One string per adapter, current values: name, MAC, IP/prefix list, status. Keeps each MAC with its own IPs |
| ServerReason | Inferred | Which part of the server rule matched |
| DefenderAgent | Inferred | Yes when OnboardingStatus is Onboarded |
| OnboardingStatus, SensorHealthState, ClientVersion | Observed | DeviceInfo |
| LinuxListenerTelemetryExpected | Inferred | Whether this Linux agent version can record ListeningConnectionCreated |
| OSPlatform, OSDistribution, OSVersion, OSVersionInfo, OSBuild, OsBuildRevision, OSArchitecture, DeviceType | Observed | DeviceInfo |
| Vendor, Model, CloudPlatforms | Observed | DeviceInfo (Vendor and Model only when discovery has enough information) |
| HostingPlatformHint | Inferred | VM, cloud or physical guess from Vendor, Model and CloudPlatforms |
| JoinType, IsAzureADJoined, MachineGroup, DeviceManualTags, DeviceDynamicTags, RegistryDeviceTag, DeviceRoles, DiscoverySources, IsTransient, ExposureLevel, AssetValue | Observed | DeviceInfo |
| IsInternetFacing, InternetFacingReason … InternetFacingLastSeen | Observed | DeviceInfo and the AdditionalFields keys in Learn's internet-facing sample query |
| PublicIP | Observed | Egress IP used to reach Defender (may be NAT or proxy) |
| DnsServerSets, DhcpServers, GatewaySets, AdapterDnsSuffixes | Observed | DeviceNetworkInfo configuration (raw JSON arrays) |
| LastSeenUtc, NicReportedUtc | Observed | Latest DeviceInfo and DeviceNetworkInfo rows |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- No power state. Last seen only exists within 30 days (HST-13 needs VMware).
- HostingPlatformHint is a guess. VMware and RunZero are the primary sources for HST-08.
- Hard-coded resolvers in application config don't show here, only the OS resolver settings.

**Validation check**

For three servers, compare `Nics` with `ipconfig /all` or `ip addr`. The MAC, IPs and DNS servers should match. For one internet-facing Windows server, compare `InternetFacingLocalPort` with the published port on the firewall (network team).


## Software

### KQL-03: Installed software and end of support (per server)

**CSV fields:** APP-05, APP-06, HST-10  
**Purpose:** Every installed software product and version on each onboarded server, with Defender Vulnerability Management's end-of-support status and date. Unsupported software is a migration exception (APP-06).  
**Tables:** `DeviceTvmSoftwareInventory`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceTvmSoftwareInventory table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmsoftwareinventory-table)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)
- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)

```kql
// KQL-03 | Installed software and end of support, per server
// Fields: APP-05, APP-06, HST-10 (OS end of support appears as software)
// Grain: one row per DeviceId + SoftwareVendor + SoftwareName + SoftwareVersion.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, OSVersion, SoftwareVendor, SoftwareName, SoftwareVersion,
//           EndOfSupportStatus, EndOfSupportDate, ProductCodeCpe.
// Inferred: IsPastEndOfSupport, DaysToEndOfSupport.
// DeviceTvmSoftwareInventory has no Timestamp: it's the current snapshot, so Lookback doesn't apply to it
// (Lookback still decides which servers are in scope). EndOfSupportStatus values aren't listed on Learn; KQL-00a shows them.
// Row limit: 100,000. If you hit it, set OSPlatformFilter, HostFilter or SubnetFilter, or use KQL-03b.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let OSPlatformFilter = dynamic([]);   // optional, e.g. dynamic(["WindowsServer2016"])
let OnlyEndOfSupport = false;         // true = only rows with an end-of-support status or date
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
Servers
| where OnboardingStatus == "Onboarded"
| where array_length(OSPlatformFilter) == 0 or OSPlatform in~ (OSPlatformFilter)
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, OSVersion
| join kind=inner hint.strategy=broadcast (
    DeviceTvmSoftwareInventory
    | project DeviceId, SoftwareVendor, SoftwareName, SoftwareVersion, EndOfSupportStatus, EndOfSupportDate, ProductCodeCpe
  ) on DeviceId
| where not(OnlyEndOfSupport) or isnotempty(EndOfSupportStatus) or isnotnull(EndOfSupportDate)
| extend IsPastEndOfSupport = isnotnull(EndOfSupportDate) and EndOfSupportDate < now(),
         DaysToEndOfSupport = datetime_diff("day", EndOfSupportDate, now())
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, OSVersion, SoftwareVendor, SoftwareName, SoftwareVersion,
          EndOfSupportStatus, EndOfSupportDate, IsPastEndOfSupport, DaysToEndOfSupport, ProductCodeCpe
| extend QueryId = "KQL-03", ObservationEndUtc = now(), ObservationNote = "Current TVM snapshot; Lookback sets server scope only"
| order by DeviceName asc, SoftwareVendor asc, SoftwareName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform, OSVersion | Observed | DeviceInfo (via the scope block) |
| SoftwareVendor, SoftwareName, SoftwareVersion | Observed | DeviceTvmSoftwareInventory |
| EndOfSupportStatus, EndOfSupportDate | Observed | DeviceTvmSoftwareInventory |
| ProductCodeCpe | Observed | CPE name, or "not available" |
| IsPastEndOfSupport | Inferred | EndOfSupportDate is before the extract time |
| DaysToEndOfSupport | Inferred | Days from now to EndOfSupportDate (negative = already past) |
| QueryId, ObservationEndUtc, ObservationNote | Provenance | Snapshot time |

**Known gaps**

- Onboarded devices only.
- Software names are Defender's normalised names. They won't match the Application Inventory names, so keep the alias list (APP-02 caveat).
- Bespoke in-house software may be missing or listed without a vendor.

**Validation check**

On one Windows server, compare a handful of rows with **Programs and Features** or `Get-Package`. On one Linux server, compare with `rpm -qa` or `dpkg -l`. Pick one product whose end-of-support date you know (for example Windows Server 2012 R2, 10 Oct 2023) and check the date.

### KQL-03b: Installed software rolled up by product and version

**CSV fields:** APP-05, APP-06  
**Purpose:** The component-grain view of APP-05 and APP-06: one row per product version, with how many servers run it. This is the fastest way to find version exceptions without paging through KQL-03.  
**Tables:** `DeviceTvmSoftwareInventory`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceTvmSoftwareInventory table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmsoftwareinventory-table)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-03b | Installed software rolled up by product and version
// Fields: APP-05, APP-06
// Grain: one row per SoftwareVendor + SoftwareName + SoftwareVersion.
// Observed: SoftwareVendor, SoftwareName, SoftwareVersion, EndOfSupportStatus, EndOfSupportDate (max), Servers (count),
//           ServerNames (first 100), OSPlatforms.
// Inferred: IsPastEndOfSupport.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
Servers
| where OnboardingStatus == "Onboarded"
| project DeviceId, DeviceName, OSPlatform
| join kind=inner hint.strategy=broadcast (
    DeviceTvmSoftwareInventory
    | project DeviceId, SoftwareVendor, SoftwareName, SoftwareVersion, EndOfSupportStatus, EndOfSupportDate
  ) on DeviceId
| summarize Servers = dcount(DeviceId), ServerNames = make_set(DeviceName, 100), OSPlatforms = make_set(OSPlatform, 10),
            EndOfSupportStatus = make_set_if(EndOfSupportStatus, isnotempty(EndOfSupportStatus), 3),
            EndOfSupportDate = max(EndOfSupportDate)
    by SoftwareVendor, SoftwareName, SoftwareVersion
| extend IsPastEndOfSupport = isnotnull(EndOfSupportDate) and EndOfSupportDate < now()
| extend QueryId = "KQL-03b", ObservationEndUtc = now(), ObservationNote = "Current TVM snapshot"
| order by IsPastEndOfSupport desc, Servers desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| SoftwareVendor, SoftwareName, SoftwareVersion | Observed | DeviceTvmSoftwareInventory |
| Servers | Observed | Distinct servers with this version |
| ServerNames | Observed | Up to 100 server names |
| OSPlatforms | Observed | Platforms the version runs on |
| EndOfSupportStatus, EndOfSupportDate | Observed | DeviceTvmSoftwareInventory (latest date if several) |
| IsPastEndOfSupport | Inferred | Date is before now |
| QueryId, ObservationEndUtc, ObservationNote | Provenance | Snapshot time |

**Known gaps**

- Same as KQL-03.

**Validation check**

Pick any product version and check that `Servers` equals the KQL-03 row count for that version.

### KQL-04: Installed certificates on servers

**CSV fields:** CMP-08  
**Purpose:** Certificates installed on each onboarded server, from Defender Vulnerability Management (preview table). It's a partial cross-check for CMP-08. The table records installed certificates, not the certificate a service presents on a port, and it has no Subject Alternative Name column. RunZero's TLS scan stays the primary source.  
**Tables:** `DeviceTvmCertificateInfo`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceTvmCertificateInfo table (preview)](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmcertificateinfo-table)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-04 | Installed certificates on servers (DeviceTvmCertificateInfo, preview)
// Fields: CMP-08 (partial)
// Grain: one row per DeviceId + Thumbprint (+ Path).
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, Thumbprint, Path, FriendlyName, IssuedToText, IssuedByText,
//           SerialNumber, SubjectType, KeyUsage, ExtendedKeyUsage, SignatureAlgorithm, KeySize, IssuedUtc, ExpiryUtc.
// Inferred: HasServerAuthEku, IsExpired, DaysToExpiry, LooksSelfSigned.
// No SAN column on the schema page. IssuedTo and IssuedBy are dynamic and their keys aren't documented; they're
// returned as text. Snapshot table: Lookback doesn't apply.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let OnlyServerAuthCerts = false;   // true = only certificates with the Server Authentication EKU
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
Servers
| where OnboardingStatus == "Onboarded"
| project DeviceId, DeviceName, HardwareUuid, OSPlatform
| join kind=inner hint.strategy=broadcast (
    DeviceTvmCertificateInfo
    | project DeviceId, Thumbprint, Path, SerialNumber, IssuedTo, IssuedBy, FriendlyName, SignatureAlgorithm, KeySize,
              ExpirationDate, IssueDate, SubjectType, KeyUsage, ExtendedKeyUsage
  ) on DeviceId
| extend ExpiryUtc = todatetime(ExpirationDate), IssuedUtc = todatetime(IssueDate),
         IssuedToText = tostring(IssuedTo), IssuedByText = tostring(IssuedBy)
| extend HasServerAuthEku = ExtendedKeyUsage contains "Server Authentication" or ExtendedKeyUsage contains "1.3.6.1.5.5.7.3.1",
         IsExpired = ExpiryUtc < now(),
         DaysToExpiry = datetime_diff("day", ExpiryUtc, now()),
         LooksSelfSigned = IssuedToText == IssuedByText
| where not(OnlyServerAuthCerts) or HasServerAuthEku
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, Thumbprint, Path, FriendlyName, IssuedToText, IssuedByText,
          SerialNumber, SubjectType, KeyUsage, ExtendedKeyUsage, SignatureAlgorithm, KeySize, IssuedUtc, ExpiryUtc,
          HasServerAuthEku, IsExpired, DaysToExpiry, LooksSelfSigned
| extend QueryId = "KQL-04", ObservationEndUtc = now(), ObservationNote = "Current TVM snapshot (preview table)"
| order by DeviceName asc, ExpiryUtc asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform | Observed | Scope block |
| Thumbprint, Path, FriendlyName, SerialNumber, SubjectType, KeyUsage, ExtendedKeyUsage, SignatureAlgorithm, KeySize | Observed | DeviceTvmCertificateInfo |
| IssuedToText, IssuedByText | Observed | IssuedTo and IssuedBy as text |
| IssuedUtc, ExpiryUtc | Observed | IssueDate and ExpirationDate converted to datetime |
| HasServerAuthEku | Inferred | EKU text contains Server Authentication |
| IsExpired, DaysToExpiry | Inferred | Compared with now |
| LooksSelfSigned | Inferred | Issued-to text equals issued-by text |
| QueryId, ObservationEndUtc, ObservationNote | Provenance | Snapshot time |

**Known gaps**

- No SAN names, so CMP-08's main value (FQDNs missing from DNS evidence) needs RunZero.
- No link between a certificate and the port it's bound to.
- Preview table: columns may change.

**Validation check**

On one IIS server, compare `Thumbprint` with the binding in IIS Manager (or `netsh http show sslcert`). Then compare with RunZero's scan of port 443 on the same IP; the thumbprints should match.


## Listening services

### KQL-05: Listening services per server endpoint

**CSV fields:** CMP-01, CMP-02, CMP-03, CMP-04, CMP-05, CMP-06, ACC-04, ACC-05, ACC-09  
**Purpose:** The core endpoint list: each port a server listens on or accepts connections on, the process behind it, its path and version, and the account it runs as. Inferred hints mark dynamic ports, protocol, tier, legacy-protocol risk and internet exposure. Server IP plus port from this query is the key every user and dependency query joins on.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Investigate devices: internet-facing devices, InboundInternetScanInspected, TCP-only process data](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)

```kql
// KQL-05 | Listening services per server endpoint
// Fields: CMP-01, CMP-02, CMP-03, CMP-04 (inferred), CMP-05 (inferred), CMP-06, ACC-04 (inferred), ACC-05 (port hints), ACC-09
// Grain: one row per DeviceId + LocalPort + Protocol + process.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, BoundLocalIPs, LocalPort, Protocol, ProcessName,
//   InboundLocalIPs, ProcessFolderPaths, ProcessVersions, ProductNames, CompanyNames, FileDescriptions, RunAsAccounts, ListenerEvents,
//   InboundEvents, InternetScanEvents, EarliestProcessStartUtc, InboundClientIPs, InboundPublicClientIPs,
//   SampleClientIPs, FirstSeenUtc, LastSeenUtc, DaysSeen.
// Inferred: EndpointIPs, BindScope, ServiceHint, ListenerEvidence, IsDynamicPort, DynamicPortReason, ProtocolHint,
//   TierHint, LegacyProtocolHint, InternetExposure.
// ActionTypes: ListeningConnectionCreated (Learn: Linux release notes), InboundInternetScanInspected (Learn:
//   investigate-machines), InboundConnectionAccepted (NOT found on Learn: confirm in the in-portal schema reference).
// Caveat under test (KQL-22): a listener event is recorded when the listener opens, so long-running services may only
//   show up through inbound connections. Linux listener events need agent 101.26091.0004 or later.
// Process details are recorded for TCP only (Learn: investigate-machines). UDP listeners usually show no process.
// Counts are events recorded, not connection volume.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let WinDynamicPortMin   = 49152;   // Windows default dynamic port range start
let LinuxDynamicPortMin = 32768;   // Linux default ip_local_port_range
let LinuxDynamicPortMax = 60999;
let RpcHostProcesses = dynamic(["lsass.exe", "svchost.exe", "services.exe", "wininit.exe", "spoolsv.exe",
                                "dfsrs.exe", "dns.exe", "ntfrs.exe", "msdtc.exe", "certsrv.exe"]);
let ListenActionTypes = dynamic(["ListeningConnectionCreated", "InboundConnectionAccepted", "InboundInternetScanInspected"]);
let IncludeLoopbackOnly = true;    // false = drop listeners bound only to 127.0.0.1 or ::1
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
// Well-known port hints. Inferred: a port number suggests a protocol; RunZero fingerprints confirm it.
let PortMap = datatable(Port: int, Service: string, Tier: string, LegacyRisk: string, LatencyRisk: string) [
    21, "FTP", "File transfer", "Cleartext (FTP)", "Medium",
    22, "SSH or SFTP", "Remote admin or file transfer", "", "Low",
    23, "Telnet", "Remote admin", "Cleartext (Telnet)", "Low",
    25, "SMTP", "Mail relay", "Cleartext unless STARTTLS", "Low",
    53, "DNS", "Infrastructure (DNS)", "", "Low",
    67, "DHCP", "Infrastructure (DHCP)", "", "Low",
    69, "TFTP", "File transfer", "Cleartext (TFTP)", "Medium",
    80, "HTTP", "Web", "Cleartext (HTTP)", "Low",
    88, "Kerberos", "Directory (DC)", "", "Low",
    123, "NTP", "Infrastructure (NTP)", "", "Low",
    135, "RPC endpoint mapper", "RPC", "", "High",
    137, "NetBIOS name", "Legacy Windows networking", "NetBIOS", "Medium",
    139, "NetBIOS session (SMB)", "File", "NetBIOS SMB", "High",
    161, "SNMP", "Management", "SNMP v1/v2c possible", "Low",
    389, "LDAP", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    443, "HTTPS", "Web", "", "Low",
    445, "SMB", "File", "SMB version not visible here", "High",
    464, "Kerberos password change", "Directory (DC)", "", "Low",
    465, "SMTPS", "Mail relay", "", "Low",
    514, "Syslog", "Logging", "Cleartext (syslog)", "Low",
    515, "LPD", "Print", "Cleartext (LPD)", "Low",
    587, "SMTP submission", "Mail relay", "", "Low",
    631, "IPP", "Print", "", "Low",
    636, "LDAPS", "Directory (DC)", "", "Medium",
    1433, "SQL Server", "Database", "", "High",
    1434, "SQL Server Browser", "Database", "", "Medium",
    1494, "Citrix ICA", "VDI or app delivery", "", "Low",
    1521, "Oracle", "Database", "", "High",
    2598, "Citrix CGP", "VDI or app delivery", "", "Low",
    3268, "Global Catalog", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    3269, "Global Catalog over SSL", "Directory (DC)", "", "Medium",
    3306, "MySQL", "Database", "", "High",
    3389, "RDP", "Remote desktop", "", "Low",
    5432, "PostgreSQL", "Database", "", "High",
    5985, "WinRM over HTTP", "Remote admin", "", "Low",
    5986, "WinRM over HTTPS", "Remote admin", "", "Low",
    8080, "HTTP (alternate)", "Web", "Cleartext (HTTP)", "Low",
    8443, "HTTPS (alternate)", "Web", "", "Low",
    9100, "Raw printing", "Print", "", "Low",
    27017, "MongoDB", "Database", "", "High"
];
let HostInfo = DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where DeviceId in (OnboardedServerIds)
    | summarize arg_max(Timestamp, IsInternetFacing, AdditionalFields) by DeviceId
    | project DeviceId, HostInternetFacing = IsInternetFacing,
              InternetFacingLocalPort = extractjson("$.InternetFacingLocalPort", AdditionalFields, typeof(int));
let HostIPs = DeviceIPs | where DeviceId in (OnboardedServerIds) | summarize HostIPs = make_set(IP, 50) by DeviceId;
DeviceNetworkEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where ActionType in (ListenActionTypes)
| where DeviceId in (OnboardedServerIds)
| summarize FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp),
            DaysSeen = dcount(startofday(Timestamp)),
            ListenerEvents = countif(ActionType == "ListeningConnectionCreated"),
            InboundEvents = countif(ActionType == "InboundConnectionAccepted"),
            InternetScanEvents = countif(ActionType == "InboundInternetScanInspected"),
            BoundLocalIPs = make_set_if(Norm(LocalIP), ActionType == "ListeningConnectionCreated", 20),
            InboundLocalIPs = make_set_if(Norm(LocalIP), ActionType == "InboundConnectionAccepted", 20),
            InboundClientIPs = dcountif(RemoteIP, ActionType == "InboundConnectionAccepted" and RemoteIP != LocalIP),
            InboundPublicClientIPs = dcountif(RemoteIP, ActionType == "InboundConnectionAccepted" and IsPublicIp(RemoteIPType, RemoteIP)),
            SampleClientIPs = make_set_if(Norm(RemoteIP), ActionType == "InboundConnectionAccepted", 10),
            ProcessFolderPaths = make_set(InitiatingProcessFolderPath, 5),
            ProcessVersions = make_set(InitiatingProcessVersionInfoProductVersion, 5),
            ProductNames = make_set(InitiatingProcessVersionInfoProductName, 3),
            CompanyNames = make_set(InitiatingProcessVersionInfoCompanyName, 3),
            FileDescriptions = make_set(InitiatingProcessVersionInfoFileDescription, 3),
            RunAsAccounts = make_set(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName), 5),
            CommandLines = make_set(InitiatingProcessCommandLine, 5),
            EarliestProcessStartUtc = min(InitiatingProcessCreationTime)
    by DeviceId, DeviceName, LocalPort, Protocol, ProcessName = InitiatingProcessFileName
| lookup kind=leftouter (Servers | project DeviceId, HardwareUuid, OSPlatform) on DeviceId
| lookup kind=leftouter HostInfo on DeviceId
| lookup kind=leftouter HostIPs on DeviceId
| lookup kind=leftouter PortMap on $left.LocalPort == $right.Port
| extend HasBind = array_length(BoundLocalIPs) > 0,
         AllInterfaces = set_has_element(BoundLocalIPs, "0.0.0.0") or set_has_element(BoundLocalIPs, "::")
| extend LoopbackOnly = iff(HasBind,
             array_length(set_difference(BoundLocalIPs, dynamic(["127.0.0.1", "::1"]))) == 0,
             array_length(InboundLocalIPs) > 0 and array_length(set_difference(InboundLocalIPs, dynamic(["127.0.0.1", "::1"]))) == 0)
| where IncludeLoopbackOnly or not(LoopbackOnly)
| extend BindScope = case(
             not(HasBind) and LoopbackOnly, "No listener event; reached only via loopback",
             not(HasBind), "Unknown: no listener event in window (see InboundLocalIPs)",
             LoopbackOnly, "Loopback only", AllInterfaces, "All interfaces", "Specific IPs"),
         EndpointIPs = case(AllInterfaces, HostIPs,
                            HasBind, set_difference(BoundLocalIPs, dynamic(["0.0.0.0", "::"])),
                            InboundLocalIPs),
         IsDynamicPort = iff(OSPlatform =~ "Linux", LocalPort between (LinuxDynamicPortMin .. LinuxDynamicPortMax),
                             LocalPort >= WinDynamicPortMin)
| extend DynamicPortReason = case(IsDynamicPort and ProcessName in~ (RpcHostProcesses), "RPC-assigned port (RPC host process): match on process, not port",
                                  IsDynamicPort, "Port in the OS dynamic range: match on process, not port", ""),
         ServiceHint = iff(ProcessName =~ "svchost.exe",
                           extract_all(@"(?i)\s-s\s+([A-Za-z0-9_\-]+)", tostring(CommandLines)), dynamic([])),
         ListenerEvidence = case(
             ListenerEvents > 0 and InboundEvents > 0, "Listener opened and inbound accepted in window",
             ListenerEvents > 0, "Listener opened in window; no inbound recorded",
             InboundEvents > 0 and EarliestProcessStartUtc < WindowStart, "Inbound only; process started before window",
             InboundEvents > 0, "Inbound only; process started in window but no listener event",
             InternetScanEvents > 0, "External scan only",
             "Other"),
         ProtocolHint = iff(isnotempty(Service), Service, "Unknown: fingerprint with RunZero"),
         TierHint = case(
             ProcessName in~ ("sqlservr.exe", "oracle.exe", "tnslsnr.exe", "mysqld.exe", "mysqld", "postgres", "postgres.exe", "mongod", "mongod.exe"), "Database",
             ProcessName in~ ("w3wp.exe", "httpd", "httpd.exe", "nginx", "nginx.exe", "java", "java.exe", "node", "node.exe", "tomcat.exe", "tomcat9.exe"), "Web or application server",
             ProcessName =~ "dns.exe", "Infrastructure (DNS)",
             ProcessName =~ "spoolsv.exe", "Print",
             ProcessName =~ "lsass.exe" and LocalPort in (88, 389, 636, 3268, 3269, 464), "Directory (DC)",
             isnotempty(Tier), Tier,
             "Application or other"),
         LegacyProtocolHint = LegacyRisk,
         InternetExposure = case(
             InternetScanEvents > 0, "External scan reached this port",
             HostInternetFacing == true and InternetFacingLocalPort == LocalPort, "Device flagged internet-facing on this port",
             InboundPublicClientIPs > 0, "Inbound from public IPs recorded",
             HostInternetFacing == true, "Device flagged internet-facing on another port",
             "")
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, EndpointIPs, BoundLocalIPs, InboundLocalIPs, BindScope, LocalPort, Protocol,
          ProcessName, ProcessFolderPaths, ProcessVersions, ProductNames, CompanyNames, FileDescriptions, ServiceHint,
          RunAsAccounts, ListenerEvents, InboundEvents, InternetScanEvents, ListenerEvidence, EarliestProcessStartUtc,
          InboundClientIPs, InboundPublicClientIPs, SampleClientIPs, FirstSeenUtc, LastSeenUtc, DaysSeen,
          IsDynamicPort, DynamicPortReason, ProtocolHint, TierHint, LegacyProtocolHint, InternetExposure
| extend QueryId = "KQL-05", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, LocalPort asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform | Observed | Scope block |
| EndpointIPs | Inferred | Host IPs if bound to all interfaces, else the bound IPs, else the IPs clients reached. Use with LocalPort as the endpoint key |
| BoundLocalIPs | Observed | LocalIP on ListeningConnectionCreated events: the bind address (0.0.0.0 or :: = all interfaces) |
| InboundLocalIPs | Observed | LocalIP on InboundConnectionAccepted events: the address clients connected to |
| BindScope | Inferred | All interfaces, specific IPs, loopback only, or unknown when no listener event exists |
| LocalPort, Protocol | Observed | DeviceNetworkEvents |
| ProcessName, ProcessFolderPaths, ProcessVersions, ProductNames, CompanyNames, FileDescriptions | Observed | InitiatingProcess* columns (CMP-01, CMP-02) |
| ServiceHint | Inferred | Service name parsed from svchost `-s` arguments |
| RunAsAccounts | Observed | InitiatingProcessAccountDomain\Name of the listening process (CMP-06) |
| ListenerEvents, InboundEvents, InternetScanEvents | Observed | Events recorded per ActionType (not volume) |
| ListenerEvidence | Inferred | How the endpoint was evidenced; "process started before window" is the expected listener gap |
| EarliestProcessStartUtc | Observed | Earliest InitiatingProcessCreationTime |
| InboundClientIPs | Observed | Distinct remote IPs on accepted inbound connections (the server's own address excluded) |
| InboundPublicClientIPs | Inferred | Distinct public remote IPs (RemoteIPType Public, or an IPv4-mapped address outside private ranges) |
| SampleClientIPs | Observed | Up to 10 client IPs (non-onboarded clients included) |
| FirstSeenUtc, LastSeenUtc, DaysSeen | Observed | Min, max and distinct days of events |
| IsDynamicPort, DynamicPortReason | Inferred | CMP-04: OS dynamic range, RPC host process |
| ProtocolHint | Inferred | ACC-04 from the port map |
| TierHint | Inferred | CMP-05 from process and port |
| LegacyProtocolHint | Inferred | ACC-05 from the port map (FTP, Telnet, HTTP, LDAP and so on) |
| InternetExposure | Inferred | ACC-09 at port level, from external scan events, the internet-facing flag and public client IPs |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Long-running listeners with no clients in the window may be missing entirely. KQL-22 measures how often that happens.
- `svchost.exe` hosts many Windows services. ServiceHint only works where the command line has `-s <name>` (split service hosts).
- UDP listeners usually have no process information.
- Linux on agents older than 101.26091.0004 has no listener events (KQL-02 flags these hosts).
- Protocol and tier are guesses from port and process. Confirm non-standard ports with RunZero and the technical owner.

**Validation check**

On two servers whose ports you know (a DC and an IIS or SQL server), run `Get-NetTCPConnection -State Listen | Sort-Object LocalPort` (or `ss -ltnup` on Linux). Then run this query with `HostFilter` set to those two servers. Every non-loopback listener should appear, unless KQL-22 labels it "listener predates window, no clients". Each process name and path should match the `OwningProcess`.

### KQL-05b: Listening services rolled up per server (workbook format)

**CSV fields:** CMP-03, CMP-04  
**Purpose:** One row per server with listening ports in the workbook's `port/proto; port/proto` format, for the Components tab's "Listening services" column. Dynamic ports are counted separately, because they can't be matched by port.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)

```kql
// KQL-05b | Listening services rolled up per server, in the workbook's "port/proto; port/proto" format
// Fields: CMP-03, CMP-04
// Grain: one row per DeviceId.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, IPAddresses, MacAddresses, ListeningProcesses.
// Inferred: ListeningServices (formatted, dynamic ports removed), DynamicPortCount, DynamicPortProcesses.
// Unconfirmed on Learn: ActionType "InboundConnectionAccepted".
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let WinDynamicPortMin   = 49152;
let LinuxDynamicPortMin = 32768;
let LinuxDynamicPortMax = 60999;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceNetworkEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where ActionType in ("ListeningConnectionCreated", "InboundConnectionAccepted")
| where DeviceId in (OnboardedServerIds)
| where not(Norm(LocalIP) in ("127.0.0.1", "::1"))
| summarize by DeviceId, LocalPort, Protocol, ProcessName = InitiatingProcessFileName
| lookup kind=leftouter (Servers | project DeviceId, DeviceName, HardwareUuid, OSPlatform) on DeviceId
| extend IsDynamicPort = iff(OSPlatform =~ "Linux", LocalPort between (LinuxDynamicPortMin .. LinuxDynamicPortMax),
                             LocalPort >= WinDynamicPortMin)
| summarize ListeningServices = strcat_array(array_sort_asc(make_set_if(strcat(LocalPort, "/", tolower(Protocol)), not(IsDynamicPort), 300)), "; "),
            DynamicPortCount = dcountif(LocalPort, IsDynamicPort),
            DynamicPortProcesses = make_set_if(ProcessName, IsDynamicPort, 20),
            ListeningProcesses = make_set(ProcessName, 50)
    by DeviceId, DeviceName, HardwareUuid, OSPlatform
| lookup kind=leftouter (DeviceIPs
    | summarize IPAddresses = make_set(IP, 50), MacAddresses = make_set_if(MacAddress, isnotempty(MacAddress), 20)
        by DeviceId) on DeviceId
| extend QueryId = "KQL-05b", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform, IPAddresses, MacAddresses | Observed | Scope block and DeviceNetworkInfo |
| ListeningServices | Inferred | Formatted list of observed port/protocol pairs, dynamic ports and loopback removed |
| DynamicPortCount, DynamicPortProcesses | Inferred | Ports in the OS dynamic range and the processes using them |
| ListeningProcesses | Observed | Distinct process names |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Servers with no listener or inbound events in the window don't appear. Use KQL-01 for the full list.
- Same listener caveats as KQL-05.

**Validation check**

`ListeningServices` for any server should equal the distinct non-dynamic `LocalPort/Protocol` pairs in KQL-05 for that DeviceId.


## Inbound clients and users

### KQL-06: Usage summary per server endpoint

**CSV fields:** ACC-01, ACC-03, ACC-04, ACC-06, ACC-08, ACC-10, ATH-06, ATH-07, CMP-07, TMP-03  
**Purpose:** One row per server endpoint (server IP + port) summarising who uses it and how. It combines two views. The client side covers onboarded clients connecting out, with user, process and RemoteUrl. The server side covers inbound connections accepted by onboarded servers, which includes clients that aren't onboarded. Port 445 rows answer CMP-07 (who uses the file shares). LastObservedUseUtc answers TMP-03.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`, `IdentityInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [IdentityInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityinfo-table)
- [Investigate connection events behind forward proxies (ConnectionSuccess)](https://learn.microsoft.com/defender-endpoint/investigate-behind-proxy)
- [Network protection and the TCP three-way handshake](https://learn.microsoft.com/defender-endpoint/network-protection#network-protection-and-the-tcp-three-way-handshake)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-06 | Usage summary per server endpoint (server IP + port)
// Fields: ACC-01, ACC-03, ACC-04 (inferred), ACC-06 (inputs), ACC-08 (inferred), ACC-10 (inferred), ATH-06, ATH-07,
//   CMP-07 (port 445), TMP-03
// Grain: one row per ServerIP + ServerPort + Protocol.
// Observed: ServerIP, ServerPort, Protocol, ServerProcesses, ClientIPs, OnboardedClientDevices, ClientProcesses,
//   FQDNsObserved, ClientEvents, EventsWithUrl, AggregatedReportRows, InboundEvents, FirstSeenUtc, LastObservedUseUtc,
//   Departments, UserCountries, UserCities (IdentityInfo, joined on SID: home location, not where users connect from).
// Inferred: ServerDeviceId, ServerDeviceName, ServerOnboarding, ServerIpHolders (IP-to-device match), DistinctUsers and
//   SampleUsers (system and machine accounts removed by a name/SID rule), ClientIPsServerSideOnly,
//   MaxUsersOnOneClientDevice, IntermediaryClientDevices, IntermediaryHint, AccessMethods, ProtocolHint, LatencyRiskHint,
//   UrlFillPct, DaysSinceLastUse.
// ActionTypes: ConnectionSuccess (Learn: investigate-behind-proxy); ConnectionSuccessAggregatedReport (Learn: aggregated
//   reporting; used for presence and first/last seen only, not counts); InboundConnectionAccepted (NOT found on Learn:
//   confirm in the in-portal schema reference).
// ConnectionSuccess is a completed TCP handshake, not proof the application accepted the session (Learn: network protection).
// Users are counted by InitiatingProcessAccountSid (so jdoe@corp and corp\jdoe count once); labels show UPN or domain\name.
// ServerIpHolders > 1: the IP belongs to more than one server (Docker bridge, cluster VIP, overlapping ranges), so the row
//   merges them.
// Counts are events recorded, not connection volume.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let PortFilter = dynamic([]);             // optional server ports, e.g. dynamic([445, 1433])
let IntermediaryUserThreshold = 10;       // distinct users (or RDP session sources) on one client device
let ClientActionTypes = dynamic(["ConnectionSuccess", "ConnectionSuccessAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
// Well-known port hints. Inferred: a port number suggests a protocol; RunZero fingerprints confirm it.
let PortMap = datatable(Port: int, Service: string, Tier: string, LegacyRisk: string, LatencyRisk: string) [
    21, "FTP", "File transfer", "Cleartext (FTP)", "Medium",
    22, "SSH or SFTP", "Remote admin or file transfer", "", "Low",
    23, "Telnet", "Remote admin", "Cleartext (Telnet)", "Low",
    25, "SMTP", "Mail relay", "Cleartext unless STARTTLS", "Low",
    53, "DNS", "Infrastructure (DNS)", "", "Low",
    67, "DHCP", "Infrastructure (DHCP)", "", "Low",
    69, "TFTP", "File transfer", "Cleartext (TFTP)", "Medium",
    80, "HTTP", "Web", "Cleartext (HTTP)", "Low",
    88, "Kerberos", "Directory (DC)", "", "Low",
    123, "NTP", "Infrastructure (NTP)", "", "Low",
    135, "RPC endpoint mapper", "RPC", "", "High",
    137, "NetBIOS name", "Legacy Windows networking", "NetBIOS", "Medium",
    139, "NetBIOS session (SMB)", "File", "NetBIOS SMB", "High",
    161, "SNMP", "Management", "SNMP v1/v2c possible", "Low",
    389, "LDAP", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    443, "HTTPS", "Web", "", "Low",
    445, "SMB", "File", "SMB version not visible here", "High",
    464, "Kerberos password change", "Directory (DC)", "", "Low",
    465, "SMTPS", "Mail relay", "", "Low",
    514, "Syslog", "Logging", "Cleartext (syslog)", "Low",
    515, "LPD", "Print", "Cleartext (LPD)", "Low",
    587, "SMTP submission", "Mail relay", "", "Low",
    631, "IPP", "Print", "", "Low",
    636, "LDAPS", "Directory (DC)", "", "Medium",
    1433, "SQL Server", "Database", "", "High",
    1434, "SQL Server Browser", "Database", "", "Medium",
    1494, "Citrix ICA", "VDI or app delivery", "", "Low",
    1521, "Oracle", "Database", "", "High",
    2598, "Citrix CGP", "VDI or app delivery", "", "Low",
    3268, "Global Catalog", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    3269, "Global Catalog over SSL", "Directory (DC)", "", "Medium",
    3306, "MySQL", "Database", "", "High",
    3389, "RDP", "Remote desktop", "", "Low",
    5432, "PostgreSQL", "Database", "", "High",
    5985, "WinRM over HTTP", "Remote admin", "", "Low",
    5986, "WinRM over HTTPS", "Remote admin", "", "Low",
    8080, "HTTP (alternate)", "Web", "Cleartext (HTTP)", "Low",
    8443, "HTTPS (alternate)", "Web", "", "Low",
    9100, "Raw printing", "Print", "", "Low",
    27017, "MongoDB", "Database", "", "High"
];
let UserDir = union                                // IdentityInfo keyed by SID (on-premises and cloud)
        (IdentityInfo | where Timestamp between (WindowStart .. WindowEnd) | where isnotempty(OnPremSid)
         | summarize arg_max(Timestamp, Department, City, Country) by Sid = OnPremSid),
        (IdentityInfo | where Timestamp between (WindowStart .. WindowEnd) | where isnotempty(CloudSid)
         | summarize arg_max(Timestamp, Department, City, Country) by Sid = CloudSid)
    | summarize arg_max(Timestamp, Department, City, Country) by Sid
    | project Sid, Department, City, Country;
let ClientPairs = materialize(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType in (ClientActionTypes)
    | where RemoteIP in (ServerIPMatch)
    | where array_length(PortFilter) == 0 or RemotePort in (PortFilter)
    | extend IsSystemAccount = InitiatingProcessAccountSid in ("S-1-5-18", "S-1-5-19", "S-1-5-20")
                or InitiatingProcessAccountName in~ ("system", "local service", "network service")
                or InitiatingProcessAccountName endswith "$"
                or InitiatingProcessAccountName startswith "dwm-"
                or InitiatingProcessAccountName startswith "umfd-",
             UserLabel = iff(isnotempty(InitiatingProcessAccountUpn), tolower(InitiatingProcessAccountUpn),
                         iff(isnotempty(InitiatingProcessAccountName),
                             tolower(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName)), ""))
    | extend UserKey = iff(isnotempty(InitiatingProcessAccountSid), InitiatingProcessAccountSid, UserLabel)   // count people by SID
    | summarize Events = countif(ActionType == "ConnectionSuccess"),
                EventsWithUrl = countif(ActionType == "ConnectionSuccess" and isnotempty(RemoteUrl)),
                AggregatedReportRows = countif(ActionType endswith "AggregatedReport"),
                FirstUtc = min(Timestamp), LastUtc = max(Timestamp),
                Users = make_set_if(UserKey, not(IsSystemAccount) and isnotempty(UserKey), 1000),
                UserLabels = make_set_if(UserLabel, not(IsSystemAccount) and isnotempty(UserLabel), 200),
                Processes = make_set(InitiatingProcessFileName, 20),
                Urls = make_set_if(tolower(RemoteUrl), isnotempty(RemoteUrl), 20),
                RemoteSessionIPs = make_set_if(InitiatingProcessRemoteSessionIP,
                    IsInitiatingProcessRemoteSession == true and isnotempty(InitiatingProcessRemoteSessionIP), 200)
        by ServerIP = Norm(RemoteIP), ServerPort = RemotePort, Protocol, ClientDeviceId = DeviceId,
           ClientDeviceName = DeviceName, ClientIP = Norm(LocalIP)
    | join kind=leftanti (ServerIPs | project ServerIP, ClientDeviceId = ServerDeviceId) on ServerIP, ClientDeviceId);
let ServerSide = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "InboundConnectionAccepted"
    | where DeviceId in (OnboardedServerIds)
    | where array_length(PortFilter) == 0 or LocalPort in (PortFilter)
    | summarize InboundEvents = count(), FirstUtc = min(Timestamp), LastUtc = max(Timestamp),
                ServerProcesses = make_set(InitiatingProcessFileName, 5)
        by ServerDeviceId = DeviceId, ServerIP = Norm(LocalIP), ServerPort = LocalPort, Protocol, ClientIP = Norm(RemoteIP)
    | where not(ClientIP in ("127.0.0.1", "::1"))
    | join kind=leftanti (ServerIPs | project ServerDeviceId, ClientIP = ServerIP) on ServerDeviceId, ClientIP   // drop self
    | project-away ServerDeviceId;
let PerClientIp = union
        (ClientPairs | extend Side = "Client"),
        (ServerSide | extend Side = "Server")
    | summarize ClientSideRows = countif(Side == "Client"),
                ClientDeviceIds = make_set_if(ClientDeviceId, isnotempty(ClientDeviceId), 100),
                Users = make_set(Users, 2000), UserLabels = make_set(UserLabels, 200),
                Processes = make_set(Processes, 30), Urls = make_set(Urls, 30),
                ServerProcesses = make_set(ServerProcesses, 10),
                ClientEvents = sum(Events), EventsWithUrl = sum(EventsWithUrl),
                AggregatedReportRows = sum(AggregatedReportRows), InboundEvents = sum(InboundEvents),
                FirstUtc = min(FirstUtc), LastUtc = max(LastUtc)
        by ServerIP, ServerPort, Protocol, ClientIP;
let PerDevice = ClientPairs                          // intermediaries are judged per client device, not per IP (DHCP reuse)
    | summarize Users = make_set(Users, 2000), RemoteSessionIPs = make_set(RemoteSessionIPs, 500)
        by ServerIP, ServerPort, Protocol, ClientDeviceId, ClientDeviceName
    | extend UsersOnDevice = array_length(Users), SessionSourcesOnDevice = array_length(RemoteSessionIPs)
    | summarize MaxUsersOnOneClientDevice = max(UsersOnDevice),
                IntermediaryClientDevices = make_set_if(ClientDeviceName,
                    UsersOnDevice >= IntermediaryUserThreshold or SessionSourcesOnDevice >= IntermediaryUserThreshold, 20),
                DevicesInRdpSessions = dcountif(ClientDeviceId, SessionSourcesOnDevice > 0)
        by ServerIP, ServerPort, Protocol;
let EndpointUserAttrs = ClientPairs
    | mv-expand User = Users to typeof(string)
    | where isnotempty(User)
    | distinct ServerIP, ServerPort, Protocol, User
    | lookup kind=leftouter UserDir on $left.User == $right.Sid
    | summarize Departments = make_set_if(Department, isnotempty(Department), 30),
                UserCountries = make_set_if(Country, isnotempty(Country), 20),
                UserCities = make_set_if(City, isnotempty(City), 30)
        by ServerIP, ServerPort, Protocol;
PerClientIp
| summarize ClientIPs = dcount(ClientIP),
            ClientIPsServerSideOnly = dcountif(ClientIP, ClientSideRows == 0),
            ClientDeviceSet = make_set(ClientDeviceIds, 100000),
            UserSet = make_set(Users, 100000), UserLabelSet = make_set(UserLabels, 100),
            ClientProcesses = make_set(Processes, 30),
            FQDNsObserved = make_set(Urls, 30),
            ServerProcesses = make_set(ServerProcesses, 10),
            ClientEvents = sum(ClientEvents), EventsWithUrl = sum(EventsWithUrl),
            AggregatedReportRows = sum(AggregatedReportRows), InboundEvents = sum(InboundEvents),
            FirstSeenUtc = min(FirstUtc), LastObservedUseUtc = max(LastUtc)
    by ServerIP, ServerPort, Protocol
| extend OnboardedClientDevices = array_length(ClientDeviceSet), DistinctUsers = array_length(UserSet),
         SampleUsers = array_slice(UserLabelSet, 0, 24)
| project-away ClientDeviceSet, UserSet, UserLabelSet
| lookup kind=leftouter PerDevice on ServerIP, ServerPort, Protocol
| lookup kind=leftouter ServerIpName on ServerIP
| lookup kind=leftouter EndpointUserAttrs on ServerIP, ServerPort, Protocol
| lookup kind=leftouter PortMap on $left.ServerPort == $right.Port
| extend UrlFillPct = iff(ClientEvents > 0, round(100.0 * EventsWithUrl / ClientEvents, 1), real(null)),
         AccessMethods = set_union(
             iff(tostring(ClientProcesses) has_any ("msedge.exe", "chrome.exe", "firefox.exe", "iexplore.exe"), dynamic(["Browser"]), dynamic([])),
             iff(tostring(ClientProcesses) has_any ("mstsc.exe", "msrdc.exe"), dynamic(["RDP client"]), dynamic([])),
             iff(tostring(ClientProcesses) has_any ("wfica32.exe", "cdviewer.exe", "selfservice.exe"), dynamic(["Citrix client"]), dynamic([])),
             iff(ServerPort in (139, 445), dynamic(["File share (SMB)"]), dynamic([])),
             iff(ServerPort in (1433, 1521, 3306, 5432, 27017), dynamic(["Database client"]), dynamic([])),
             iff(DevicesInRdpSessions > 0, dynamic(["Client process ran in an RDP session (RDS or VDI host in front)"]), dynamic([]))),
         ProtocolHint = iff(isnotempty(Service), Service, "Unknown: fingerprint with RunZero"),
         IntermediaryHint = case(
             array_length(IntermediaryClientDevices) > 0, "Probable RDS, Citrix or shared host in front of users: see KQL-08",
             ClientIPsServerSideOnly > 0, "Some clients are seen only by the server (not onboarded, NAT, proxy or connector): see KQL-08",
             ""),
         LatencyRiskHint = iff(LatencyRisk == "High", "Chatty protocol: test at hub latency before deciding", ""),
         DaysSinceLastUse = datetime_diff("day", WindowEnd, LastObservedUseUtc)
| project ServerDeviceId, ServerDeviceName, ServerOnboarding, ServerIpHolders, ServerIP, ServerPort, Protocol, ProtocolHint,
          ServerProcesses, ClientIPs, ClientIPsServerSideOnly, OnboardedClientDevices, DistinctUsers, SampleUsers,
          Departments, UserCountries, UserCities, ClientProcesses, AccessMethods, FQDNsObserved, UrlFillPct,
          MaxUsersOnOneClientDevice, IntermediaryClientDevices, IntermediaryHint, LatencyRiskHint, ClientEvents,
          EventsWithUrl, AggregatedReportRows, InboundEvents, FirstSeenUtc, LastObservedUseUtc, DaysSinceLastUse
| extend QueryId = "KQL-06", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by ServerDeviceName asc, ServerPort asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| ServerDeviceId, ServerDeviceName, ServerOnboarding | Inferred | Server holding ServerIP (IP-to-device match) |
| ServerIpHolders | Inferred | Servers that held this IP in the window; above 1 means the row merges several servers |
| ServerIP, ServerPort, Protocol | Observed | RemoteIP/RemotePort (client side) or LocalIP/LocalPort (server side) |
| ProtocolHint | Inferred | ACC-04 port-map hint |
| ServerProcesses | Observed | Process accepting the connection (server side) |
| ClientIPs | Observed | Distinct client IPs from both sides (server's own IPs removed) |
| ClientIPsServerSideOnly | Inferred | Client IPs seen only by the server: not onboarded, NAT, proxy, or client telemetry missing |
| OnboardedClientDevices | Observed | Distinct client DeviceIds (client side) |
| DistinctUsers | Inferred | Distinct account SIDs (or names) after a rule removes system and machine accounts (ATH-06, ATH-07) |
| SampleUsers | Inferred | Up to 25 UPN or domain\name labels, after the same rule |
| Departments, UserCountries, UserCities | Observed | IdentityInfo values for those SIDs: home location, not connection site (ATH-07, ACC-06) |
| ClientProcesses | Observed | Client InitiatingProcessFileName values (ACC-03) |
| AccessMethods | Inferred | Browser, RDP, Citrix, SMB, database client, RDS/VDI, from processes and port |
| FQDNsObserved | Observed | RemoteUrl values clients used (ACC-01) |
| UrlFillPct | Inferred | Share of raw ConnectionSuccess events with RemoteUrl |
| MaxUsersOnOneClientDevice | Inferred | Most users seen on one client device (after the system-account rule) |
| IntermediaryClientDevices, IntermediaryHint | Inferred | ACC-08: client devices over the threshold, and server-side-only clients |
| LatencyRiskHint | Inferred | ACC-10: chatty protocol from the port map |
| ClientEvents, EventsWithUrl | Observed | Raw ConnectionSuccess events recorded (presence, not volume) |
| AggregatedReportRows | Observed | Aggregated-report rows (counted separately so they don't double the event counts) |
| InboundEvents | Observed | InboundConnectionAccepted events recorded on the server |
| FirstSeenUtc, LastObservedUseUtc | Observed | Earliest and latest event (TMP-03) |
| DaysSinceLastUse | Inferred | Days from LastObservedUseUtc to the window end |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Users only come from onboarded clients. Server-side rows carry no user, because the account there is the service.
- Citrix, RDS, proxies and ZPA App Connectors put many users behind one host or IP. Back-end tiers only show service accounts (CSV ATH-06 caveat).
- SMB client connections often show `System` as the process, so ACC-03 for file shares is "File share (SMB)" by port.
- An endpoint with no use in the window isn't here. KQL-05 rows with `InboundEvents = 0` are the "no observed use" list, and no use in 30 days doesn't prove it's unused.
- DistinctUsers comes from `dcount`-style set sizes and counts SIDs. A person with two accounts counts twice.

**Validation check**

Pick one file server and one web application with known users. Ask two users to open the share and the app, wait an hour, then run with `HostFilter` set to that server. Both users should appear in `SampleUsers`, and `ClientProcesses` should show the expected browser or thick client. If a Citrix path exists, the Citrix VDA servers should appear in `IntermediaryClientDevices`.

### KQL-07: Client-to-endpoint pairs with users

**CSV fields:** ATH-06, ATH-07, ACC-03, ACC-04, ACC-06, CMP-07  
**Purpose:** The detail behind KQL-06: one row per client and server endpoint, with that client's users and processes. Client IPs go to Netbox for client site (ACC-06). Clients seen only from the server side are kept, not dropped, and labelled by whether their IP matches anything Defender knows. Those unmatched IPs are your follow-up list for RunZero and VMware.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`, `IdentityInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [IdentityInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)

```kql
// KQL-07 | Client-to-endpoint pairs with users
// Fields: ATH-06, ATH-07, ACC-03, ACC-04 (inferred), ACC-06 (client IP and subnet for Netbox), CMP-07 (port 445)
// Grain: one row per ServerIP + ServerPort + Protocol + client (DeviceId when onboarded, otherwise client IP).
// Observed: ServerIP, ServerPort, Protocol, ClientIP, ClientProcesses, FQDNsObserved, RemoteSessionIPs, ClientEvents,
//   EventsWithUrl, AggregatedReportRows, ServerSideInboundEvents, FirstSeenUtc, LastSeenUtc, ServerDeviceId (server-side
//   rows), ClientDeviceId (client-side rows), Departments, UserCountries, UserCities (IdentityInfo, joined on SID).
// Inferred: ServerDeviceId (client-side rows, IP match), ServerDeviceName, ServerIpHolders, ClientDeviceId (server-side rows,
//   IP match), ClientDeviceName, ClientDeviceType, ClientOS, ClientOnboarded, ClientIdentifiedBy, ClientSubnet24,
//   UserCount and Users (system and machine accounts removed by a rule), ProtocolHint.
// ActionTypes: ConnectionSuccess (Learn); ConnectionSuccessAggregatedReport (Learn; presence only); InboundConnectionAccepted
//   (NOT on Learn: confirm in portal).
// Row limit 100,000: narrow with HostFilter, SubnetFilter (server side) or PortFilter if you hit it.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let PortFilter = dynamic([]);              // optional server ports
let ClientActionTypes = dynamic(["ConnectionSuccess", "ConnectionSuccessAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
// Well-known port hints. Inferred: a port number suggests a protocol; RunZero fingerprints confirm it.
let PortMap = datatable(Port: int, Service: string, Tier: string, LegacyRisk: string, LatencyRisk: string) [
    21, "FTP", "File transfer", "Cleartext (FTP)", "Medium",
    22, "SSH or SFTP", "Remote admin or file transfer", "", "Low",
    23, "Telnet", "Remote admin", "Cleartext (Telnet)", "Low",
    25, "SMTP", "Mail relay", "Cleartext unless STARTTLS", "Low",
    53, "DNS", "Infrastructure (DNS)", "", "Low",
    67, "DHCP", "Infrastructure (DHCP)", "", "Low",
    69, "TFTP", "File transfer", "Cleartext (TFTP)", "Medium",
    80, "HTTP", "Web", "Cleartext (HTTP)", "Low",
    88, "Kerberos", "Directory (DC)", "", "Low",
    123, "NTP", "Infrastructure (NTP)", "", "Low",
    135, "RPC endpoint mapper", "RPC", "", "High",
    137, "NetBIOS name", "Legacy Windows networking", "NetBIOS", "Medium",
    139, "NetBIOS session (SMB)", "File", "NetBIOS SMB", "High",
    161, "SNMP", "Management", "SNMP v1/v2c possible", "Low",
    389, "LDAP", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    443, "HTTPS", "Web", "", "Low",
    445, "SMB", "File", "SMB version not visible here", "High",
    464, "Kerberos password change", "Directory (DC)", "", "Low",
    465, "SMTPS", "Mail relay", "", "Low",
    514, "Syslog", "Logging", "Cleartext (syslog)", "Low",
    515, "LPD", "Print", "Cleartext (LPD)", "Low",
    587, "SMTP submission", "Mail relay", "", "Low",
    631, "IPP", "Print", "", "Low",
    636, "LDAPS", "Directory (DC)", "", "Medium",
    1433, "SQL Server", "Database", "", "High",
    1434, "SQL Server Browser", "Database", "", "Medium",
    1494, "Citrix ICA", "VDI or app delivery", "", "Low",
    1521, "Oracle", "Database", "", "High",
    2598, "Citrix CGP", "VDI or app delivery", "", "Low",
    3268, "Global Catalog", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    3269, "Global Catalog over SSL", "Directory (DC)", "", "Medium",
    3306, "MySQL", "Database", "", "High",
    3389, "RDP", "Remote desktop", "", "Low",
    5432, "PostgreSQL", "Database", "", "High",
    5985, "WinRM over HTTP", "Remote admin", "", "Low",
    5986, "WinRM over HTTPS", "Remote admin", "", "Low",
    8080, "HTTP (alternate)", "Web", "Cleartext (HTTP)", "Low",
    8443, "HTTPS (alternate)", "Web", "", "Low",
    9100, "Raw printing", "Print", "", "Low",
    27017, "MongoDB", "Database", "", "High"
];
let UserDir = union                                // IdentityInfo keyed by SID (on-premises and cloud)
        (IdentityInfo | where Timestamp between (WindowStart .. WindowEnd) | where isnotempty(OnPremSid)
         | summarize arg_max(Timestamp, Department, City, Country) by Sid = OnPremSid),
        (IdentityInfo | where Timestamp between (WindowStart .. WindowEnd) | where isnotempty(CloudSid)
         | summarize arg_max(Timestamp, Department, City, Country) by Sid = CloudSid)
    | summarize arg_max(Timestamp, Department, City, Country) by Sid
    | project Sid, Department, City, Country;
let ClientPairs = materialize(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType in (ClientActionTypes)
    | where RemoteIP in (ServerIPMatch)
    | where array_length(PortFilter) == 0 or RemotePort in (PortFilter)
    | extend IsSystemAccount = InitiatingProcessAccountSid in ("S-1-5-18", "S-1-5-19", "S-1-5-20")
                or InitiatingProcessAccountName in~ ("system", "local service", "network service")
                or InitiatingProcessAccountName endswith "$"
                or InitiatingProcessAccountName startswith "dwm-"
                or InitiatingProcessAccountName startswith "umfd-",
             UserLabel = iff(isnotempty(InitiatingProcessAccountUpn), tolower(InitiatingProcessAccountUpn),
                         iff(isnotempty(InitiatingProcessAccountName),
                             tolower(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName)), ""))
    | extend UserKey = iff(isnotempty(InitiatingProcessAccountSid), InitiatingProcessAccountSid, UserLabel)   // count people by SID
    | summarize ClientEvents = countif(ActionType == "ConnectionSuccess"),
                EventsWithUrl = countif(ActionType == "ConnectionSuccess" and isnotempty(RemoteUrl)),
                AggregatedReportRows = countif(ActionType endswith "AggregatedReport"),
                FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp),
                UserSids = make_set_if(UserKey, not(IsSystemAccount) and isnotempty(UserKey), 500),
                Users = make_set_if(UserLabel, not(IsSystemAccount) and isnotempty(UserLabel), 100),
                ClientProcesses = make_set(InitiatingProcessFileName, 20),
                FQDNsObserved = make_set_if(tolower(RemoteUrl), isnotempty(RemoteUrl), 20),
                RemoteSessionIPs = make_set_if(InitiatingProcessRemoteSessionIP,
                    IsInitiatingProcessRemoteSession == true and isnotempty(InitiatingProcessRemoteSessionIP), 100)
        by ServerIP = Norm(RemoteIP), ServerPort = RemotePort, Protocol, ClientDeviceId = DeviceId, ClientIP = Norm(LocalIP)
    | join kind=leftanti (ServerIPs | project ServerIP, ClientDeviceId = ServerDeviceId) on ServerIP, ClientDeviceId);
let ServerSide = materialize(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "InboundConnectionAccepted"
    | where DeviceId in (OnboardedServerIds)
    | where array_length(PortFilter) == 0 or LocalPort in (PortFilter)
    | summarize ServerSideInboundEvents = count(), SsFirstUtc = min(Timestamp), SsLastUtc = max(Timestamp)
        by ServerDeviceId = DeviceId, ServerIP = Norm(LocalIP), ServerPort = LocalPort, Protocol, ClientIP = Norm(RemoteIP)
    | where not(ClientIP in ("127.0.0.1", "::1"))
    | join kind=leftanti (ServerIPs | project ServerDeviceId, ClientIP = ServerIP) on ServerDeviceId, ClientIP);   // drop self
let PairUserAttrs = ClientPairs
    | mv-expand User = UserSids to typeof(string)
    | where isnotempty(User)
    | lookup kind=leftouter UserDir on $left.User == $right.Sid
    | summarize Departments = make_set_if(Department, isnotempty(Department), 10),
                UserCountries = make_set_if(Country, isnotempty(Country), 10),
                UserCities = make_set_if(City, isnotempty(City), 10)
        by ServerIP, ServerPort, Protocol, ClientDeviceId, ClientIP;
let ClientSideRows = ClientPairs
    | lookup kind=leftouter PairUserAttrs on ServerIP, ServerPort, Protocol, ClientDeviceId, ClientIP
    | lookup kind=leftouter (ServerSide
        | summarize ServerSideInboundEvents = sum(ServerSideInboundEvents) by ServerIP, ServerPort, Protocol, ClientIP)
        on ServerIP, ServerPort, Protocol, ClientIP
    | lookup kind=leftouter ServerIpName on ServerIP
    | extend ClientIdentifiedBy = "Client-side event (DeviceId)", ClientOnboarded = "Yes";
let ServerOnlyRows = ServerSide
    | join kind=leftanti (ClientPairs | distinct ServerIP, ServerPort, Protocol, ClientIP) on ServerIP, ServerPort, Protocol, ClientIP
    | lookup kind=leftouter (IpOwner | project ClientIP = IP, ClientDeviceId = OwnerDeviceId, OwnerOnboarding) on ClientIP
    | lookup kind=leftouter (Servers | project ServerDeviceId = DeviceId, ServerDeviceName = DeviceName) on ServerDeviceId
    | extend ClientIdentifiedBy = iff(isnotempty(ClientDeviceId), "IP matched to Defender inventory", "Unmatched IP: not known to Defender"),
             ClientOnboarded = case(OwnerOnboarding == "Onboarded", "Yes by IP match, but no client-side event (check client telemetry or NAT)",
                                    isnotempty(ClientDeviceId), "No (discovered device)", "Unknown"),
             FirstSeenUtc = SsFirstUtc, LastSeenUtc = SsLastUtc, ServerIpHolders = long(1)
    | project-away OwnerOnboarding, SsFirstUtc, SsLastUtc;
union ClientSideRows, ServerOnlyRows
| lookup kind=leftouter (DeviceLatest | project ClientDeviceId = DeviceId, ClientDeviceName = DeviceName,
                                                ClientDeviceType = DeviceType, ClientOS = OSPlatform) on ClientDeviceId
| lookup kind=leftouter PortMap on $left.ServerPort == $right.Port
| extend UserCount = array_length(UserSids),
         ClientSubnet24 = iff(ClientIP contains ":", "", format_ipv4_mask(ClientIP, 24)),
         ProtocolHint = iff(isnotempty(Service), Service, "Unknown: fingerprint with RunZero")
| project ServerDeviceId, ServerDeviceName, ServerIpHolders, ServerIP, ServerPort, Protocol, ProtocolHint,
          ClientDeviceId, ClientDeviceName, ClientDeviceType, ClientOS, ClientIP, ClientSubnet24,
          ClientOnboarded, ClientIdentifiedBy, UserCount, Users, Departments, UserCountries, UserCities,
          ClientProcesses, FQDNsObserved, RemoteSessionIPs, ClientEvents, EventsWithUrl, AggregatedReportRows,
          ServerSideInboundEvents, FirstSeenUtc, LastSeenUtc
| extend QueryId = "KQL-07", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by ServerDeviceName asc, ServerPort asc, ClientDeviceName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| ServerDeviceId | Observed (server-side rows) / Inferred (client-side rows, IP match) | Server for this endpoint |
| ServerDeviceName, ServerIpHolders | Inferred | Name lookup; servers sharing the IP (above 1 = merged) |
| ServerIP, ServerPort, Protocol | Observed | DeviceNetworkEvents |
| ProtocolHint | Inferred | Port map |
| ClientDeviceId | Observed (client-side rows) / Inferred (server-side rows) | DeviceId of the reporting client, or IP match |
| ClientDeviceName, ClientDeviceType, ClientOS | Inferred | Looked up by ClientDeviceId |
| ClientIP | Observed | LocalIP (client side) or RemoteIP (server side) |
| ClientSubnet24 | Inferred | Helper for Netbox prefix matching (ACC-06) |
| ClientOnboarded, ClientIdentifiedBy | Inferred | How the client was identified |
| UserCount, Users | Inferred | Distinct SIDs, and UPN or domain\name labels, after a rule removes system and machine accounts |
| Departments, UserCountries, UserCities | Observed | IdentityInfo, joined on OnPremSid or CloudSid |
| ClientProcesses, FQDNsObserved | Observed | InitiatingProcessFileName, RemoteUrl |
| RemoteSessionIPs | Observed | InitiatingProcessRemoteSessionIP: the real client behind an RDP session |
| ClientEvents, EventsWithUrl, AggregatedReportRows, ServerSideInboundEvents | Observed | Events recorded (not volume); aggregated rows counted separately |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max event times |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Users only for onboarded clients. Server-side-only rows show an IP and nothing else.
- Departments and countries need a SID that matches IdentityInfo (OnPremSid or CloudSid). Linux accounts get none.
- Client IPs are point-in-time. DHCP clients change IPs, so match each to Netbox for the date seen.

**Validation check**

For one row with `ClientIdentifiedBy = "Unmatched IP"`, look the IP up in RunZero. It should be a device without a Defender agent (an appliance, a non-onboarded server, a ZPA App Connector or a VPN pool address). If it's a normal Windows workstation, record a client onboarding gap.

### KQL-08: Probable intermediaries and remote access paths, by source IP

**CSV fields:** ACC-07, ACC-08, ATH-06  
**Purpose:** Finds source IPs that probably stand in front of real users. That covers RDS or Citrix hosts, proxies, load balancer SNAT, ZPA App Connectors, VPN tunnels and scanners. Behind these the server can't see who the users are. Each flag is a hint for the network team handoff (ACC-08) and the remote access path (ACC-07), not a finding.  
**Tables:** `DeviceNetworkEvents`, `DeviceNetworkInfo`, `DeviceInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)

```kql
// KQL-08 | Probable intermediaries and remote access paths, by source IP
// Fields: ACC-07 (inferred hints), ACC-08 (inferred), ATH-06 (where users are hidden)
// Grain: one row per source IP that connects to a scoped server endpoint.
// Observed: SourceIP, OnboardedDevicesUsingIp30d, MaxDevicesInOneHour, RemoteSessionSourceIPs, ServersReached,
//   EndpointsReached, PortsReached, AdapterNames, AdapterTypes, TunnelTypes, FirstSeenUtc, LastSeenUtc.
// Inferred: OwnerDeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding (IP-to-device match), DistinctUsers,
//   MaxUsersOnOneDevice and SampleUsers (system-account rule applied), Reasons, RemoteAccessPathHint.
// On client-side events LocalIP is the device's own interface address, so several devices on one IP over 30 days is
// DHCP reuse, not NAT. Concurrency is therefore measured per hour (MaxDevicesInOneHour). NAT, SNAT, proxies and
// connectors are inferred only from IPs the servers see that no onboarded device holds.
// TunnelType values aren't listed on Learn; None, Teredo, ISATAP and 6to4 are treated as "not a VPN tunnel".
// NetworkAdapterType follows the .NET NetworkInterfaceType enumeration (Learn: DeviceNetworkInfo), e.g. Ppp.
// OnlyFlagged = true returns only IPs that meet a threshold. It filters on the flags, not on scope.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let IntermediaryUserThreshold = 10;   // distinct users, or RDP session sources, on one device
let SharedIpDeviceThreshold   = 2;    // onboarded devices using one IP in the same hour
let FanOutServerThreshold     = 5;    // distinct servers reached by an IP no onboarded device holds
let ScannerPortThreshold      = 50;   // distinct server ports reached: probable scanner
let OnlyFlagged = true;               // false = every source IP
let NonVpnTunnelTypes = dynamic(["", "None", "Teredo", "ISATAP", "6to4"]);
let ClientActionTypes = dynamic(["ConnectionSuccess", "ConnectionSuccessAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let ClientRaw = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType in (ClientActionTypes)
    | where RemoteIP in (ServerIPMatch);
let PerDevice = ClientRaw
    | extend IsSystemAccount = InitiatingProcessAccountSid in ("S-1-5-18", "S-1-5-19", "S-1-5-20")
                or InitiatingProcessAccountName in~ ("system", "local service", "network service")
                or InitiatingProcessAccountName endswith "$"
                or InitiatingProcessAccountName startswith "dwm-"
                or InitiatingProcessAccountName startswith "umfd-",
             UserLabel = iff(isnotempty(InitiatingProcessAccountUpn), tolower(InitiatingProcessAccountUpn),
                         iff(isnotempty(InitiatingProcessAccountName),
                             tolower(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName)), ""))
    | extend UserKey = iff(isnotempty(InitiatingProcessAccountSid), InitiatingProcessAccountSid, UserLabel)   // count people by SID
    | summarize Users = make_set_if(UserKey, not(IsSystemAccount) and isnotempty(UserKey), 2000),
                UserLabels = make_set_if(UserLabel, not(IsSystemAccount) and isnotempty(UserLabel), 25),
                RemoteSessionIPs = make_set_if(InitiatingProcessRemoteSessionIP,
                    IsInitiatingProcessRemoteSession == true and isnotempty(InitiatingProcessRemoteSessionIP), 1000),
                Endpoints = make_set(strcat(Norm(RemoteIP), ":", RemotePort), 1000),
                FirstUtc = min(Timestamp), LastUtc = max(Timestamp)
        by SourceIP = Norm(LocalIP), DeviceId
    | extend UsersOnDevice = array_length(Users);
let ClientSide = PerDevice
    | summarize CsUsers = make_set(Users, 5000), CsUserLabels = make_set(UserLabels, 25),
                MaxUsersOnOneDevice = max(UsersOnDevice), OnboardedDevicesUsingIp30d = dcount(DeviceId),
                CsRemoteSessionIPs = make_set(RemoteSessionIPs, 1000), CsEndpoints = make_set(Endpoints, 1000),
                CsFirstUtc = min(FirstUtc), CsLastUtc = max(LastUtc)
        by SourceIP;
let Concurrency = ClientRaw
    | summarize by SourceIP = Norm(LocalIP), Hour = bin(Timestamp, 1h), DeviceId
    | summarize DevicesThisHour = count() by SourceIP, Hour
    | summarize MaxDevicesInOneHour = max(DevicesThisHour) by SourceIP;
let ServerSide = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "InboundConnectionAccepted"
    | where DeviceId in (OnboardedServerIds)
    | summarize SsServers = make_set(DeviceId, 1000), SsEndpoints = make_set(strcat(Norm(LocalIP), ":", LocalPort), 1000),
                SsPorts = make_set(LocalPort, 500), SsFirstUtc = min(Timestamp), SsLastUtc = max(Timestamp)
        by SourceIP = Norm(RemoteIP)
    | where not(SourceIP in ("127.0.0.1", "::1"))
    | join kind=leftanti (ServerIPs | project SourceIP = ServerIP) on SourceIP;     // a server's own IPs aren't clients
let NicAttrs = DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize by DeviceId, NetworkAdapterName, NetworkAdapterType, TunnelType, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend SourceIP = tostring(IpObj.IPAddress)
    | where isnotempty(SourceIP)
    | summarize AdapterNames = make_set(NetworkAdapterName, 5), AdapterTypes = make_set(NetworkAdapterType, 5),
                TunnelTypes = make_set_if(TunnelType, not(TunnelType in~ (NonVpnTunnelTypes)), 5) by SourceIP;
ClientSide
| join kind=fullouter ServerSide on SourceIP
| extend SourceIP = iff(isempty(SourceIP), SourceIP1, SourceIP)
| project-away SourceIP1
| lookup kind=leftouter Concurrency on SourceIP
| lookup kind=leftouter NicAttrs on SourceIP
| lookup kind=leftouter (IpOwner | project SourceIP = IP, OwnerDeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding) on SourceIP
| extend CsUsers = iff(isnull(CsUsers), dynamic([]), CsUsers),
         CsUserLabels = iff(isnull(CsUserLabels), dynamic([]), CsUserLabels),
         CsRemoteSessionIPs = iff(isnull(CsRemoteSessionIPs), dynamic([]), CsRemoteSessionIPs),
         CsEndpoints = iff(isnull(CsEndpoints), dynamic([]), CsEndpoints),
         SsServers = iff(isnull(SsServers), dynamic([]), SsServers),
         SsEndpoints = iff(isnull(SsEndpoints), dynamic([]), SsEndpoints),
         SsPorts = iff(isnull(SsPorts), dynamic([]), SsPorts),
         TunnelTypes = iff(isnull(TunnelTypes), dynamic([]), TunnelTypes)
| extend DistinctUsers = array_length(CsUsers), SampleUsers = CsUserLabels,
         RemoteSessionSourceIPs = array_length(CsRemoteSessionIPs),
         ServersReached = array_length(SsServers),
         EndpointsReached = array_length(set_union(CsEndpoints, SsEndpoints)),
         PortsReached = array_length(SsPorts),
         FirstSeenUtc = min_of(coalesce(CsFirstUtc, SsFirstUtc), coalesce(SsFirstUtc, CsFirstUtc)),
         LastSeenUtc = max_of(coalesce(CsLastUtc, SsLastUtc), coalesce(SsLastUtc, CsLastUtc)),
         HeldByOnboardedDevice = OwnerOnboarding == "Onboarded",
         IsTunnelAdapter = array_length(TunnelTypes) > 0 or tostring(AdapterTypes) has "Ppp"
| extend Reasons = set_union(
             iff(MaxUsersOnOneDevice >= IntermediaryUserThreshold, dynamic(["Many users on one device using this IP: RDS, Citrix or shared host"]), dynamic([])),
             iff(RemoteSessionSourceIPs >= IntermediaryUserThreshold, dynamic(["Many RDP session sources: RDS or VDI host"]), dynamic([])),
             iff(MaxDevicesInOneHour >= SharedIpDeviceThreshold, dynamic(["Several onboarded devices used this IP in the same hour: overlapping private ranges (home or remote networks) or a shared address"]), dynamic([])),
             iff(not(HeldByOnboardedDevice) and ServersReached >= FanOutServerThreshold, dynamic(["No onboarded device holds this IP, and it reaches many servers: NAT, load balancer SNAT, proxy or ZPA App Connector"]), dynamic([])),
             iff(PortsReached >= ScannerPortThreshold, dynamic(["Reaches many ports: probable scanner"]), dynamic([])),
             iff(IsTunnelAdapter, dynamic(["IP is on a VPN tunnel or PPP adapter"]), dynamic([])))
| where not(OnlyFlagged) or array_length(Reasons) > 0
| extend RemoteAccessPathHint = case(
             IsTunnelAdapter, "VPN tunnel",
             RemoteSessionSourceIPs > 0 or MaxUsersOnOneDevice >= IntermediaryUserThreshold, "RDS, Citrix or VDI session host",
             not(HeldByOnboardedDevice) and ServersReached >= FanOutServerThreshold, "NAT, connector or proxy (ZPA App Connector, load balancer SNAT, reverse proxy)",
             HeldByOnboardedDevice, "Direct from an onboarded device",
             "Unknown: not onboarded")
| project SourceIP, OwnerDeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, Reasons, RemoteAccessPathHint,
          DistinctUsers, MaxUsersOnOneDevice, SampleUsers, OnboardedDevicesUsingIp30d, MaxDevicesInOneHour,
          RemoteSessionSourceIPs, ServersReached, EndpointsReached, PortsReached, AdapterNames, AdapterTypes, TunnelTypes,
          FirstSeenUtc, LastSeenUtc
| extend QueryId = "KQL-08", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by MaxUsersOnOneDevice desc, ServersReached desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| SourceIP | Observed | Client LocalIP (client side) or RemoteIP (server side) |
| OwnerDeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding | Inferred | Device that held this IP in DeviceNetworkInfo |
| Reasons | Inferred | Which thresholds the IP crossed (ACC-08) |
| RemoteAccessPathHint | Inferred | ACC-07 hint: VPN tunnel, RDS/Citrix/VDI, NAT/connector/proxy, direct, unknown |
| DistinctUsers, MaxUsersOnOneDevice, SampleUsers | Inferred | User SIDs and labels after the system-account rule |
| OnboardedDevicesUsingIp30d | Observed | Distinct client DeviceIds using this IP over the window (mostly DHCP reuse) |
| MaxDevicesInOneHour | Observed | Most onboarded devices using this IP within one hour (aggregate) |
| RemoteSessionSourceIPs | Observed | Distinct InitiatingProcessRemoteSessionIP values |
| ServersReached, EndpointsReached, PortsReached | Observed | Distinct servers, endpoints and ports reached |
| AdapterNames, AdapterTypes, TunnelTypes | Observed | DeviceNetworkInfo for adapters that held this IP (TunnelTypes without None, Teredo, ISATAP, 6to4) |
| FirstSeenUtc, LastSeenUtc | Observed | Earliest and latest event |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Thresholds are judgement calls. Change them, and record what you used in the decision log.
- A ZPA App Connector that isn't onboarded looks the same as a load balancer or reverse proxy here. ZPA logs (ACC-07 primary) tell them apart.
- Citrix ICA sessions may not set `IsInitiatingProcessRemoteSession` (Learn describes RDP). Citrix VDAs are caught by "many users on one device".
- Zscaler Client Connector and some VPN clients present as ordinary Ethernet adapters, so they don't trip the tunnel flag.
- It scans client-side connection events twice (per device and per hour). Slice the window if it times out.

**Validation check**

List your known intermediaries (Citrix VDAs, RDS hosts, ZPA App Connectors, the RunZero scanner, any load balancer SNAT addresses). Each should appear here with a matching reason. Any intermediary that doesn't appear is either onboarded and unused in the window, or proof that the thresholds are too high. Also run `DeviceNetworkInfo | summarize count() by TunnelType` once: if a value other than None, Teredo, ISATAP or 6to4 appears on ordinary desktops, add it to `NonVpnTunnelTypes`.


## FQDNs and IP-only access

### KQL-09: FQDNs used and probable IP-only access

**CSV fields:** ACC-01, ACC-02  
**Purpose:** For each server endpoint and client process, records the names clients used (ACC-01). It then infers whether some clients connect by IP only (ACC-02), which breaks when the server moves. Because RemoteUrl isn't filled for every connection, an endpoint is only called "probable IP-only" where names are usually recorded on that port. Otherwise it's "unknown".  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Investigate connection events behind forward proxies (ConnectionSuccess)](https://learn.microsoft.com/defender-endpoint/investigate-behind-proxy)

```kql
// KQL-09 | FQDNs used and probable IP-only access
// Fields: ACC-01 (observed names), ACC-02 (inferred)
// Grain: one row per ServerIP + ServerPort + Protocol + ClientProcess.
// Observed: ServerIP, ServerPort, Protocol, ClientProcess, ClientDevices, ClientsWithName, ClientsWithoutName,
//   SampleClientsWithoutName, NamesObserved, Events, EventsWithUrl, FirstSeenUtc, LastSeenUtc.
// Inferred: ServerDeviceId, ServerDeviceName, ServerIpHolders (IP match), PortUrlFillPct, UrlFillPct, NamesBeyondHostName,
//   Addressing.
// An empty RemoteUrl can be missing data, not IP access (CSV ACC-02 caveat). Corroborate with Splunk DNS logs before
// adding an endpoint to the exception list.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let PortFilter = dynamic([]);
let UrlBaselineMinPct = 50.0;   // names must be recorded at least this often on a port before "no name" means anything
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let Base = materialize(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "ConnectionSuccess"
    | where RemoteIP in (ServerIPMatch)
    | where array_length(PortFilter) == 0 or RemotePort in (PortFilter)
    | summarize Events = count(), EventsWithUrl = countif(isnotempty(RemoteUrl)),
                Urls = make_set_if(tolower(RemoteUrl), isnotempty(RemoteUrl), 20),
                FirstUtc = min(Timestamp), LastUtc = max(Timestamp)
        by ServerIP = Norm(RemoteIP), ServerPort = RemotePort, Protocol, ClientProcess = InitiatingProcessFileName,
           ClientDeviceId = DeviceId, ClientDeviceName = DeviceName
    | join kind=leftanti (ServerIPs | project ServerIP, ClientDeviceId = ServerDeviceId) on ServerIP, ClientDeviceId);
let PortBaseline = Base
    | summarize BaselineEvents = sum(Events), BaselineWithUrl = sum(EventsWithUrl) by ServerPort
    | extend PortUrlFillPct = round(100.0 * BaselineWithUrl / BaselineEvents, 1)
    | project ServerPort, PortUrlFillPct;
Base
| summarize ClientDevices = dcount(ClientDeviceId),
            ClientsWithName = dcountif(ClientDeviceId, EventsWithUrl > 0),
            ClientsWithoutName = dcountif(ClientDeviceId, EventsWithUrl == 0),
            SampleClientsWithoutName = make_set_if(ClientDeviceName, EventsWithUrl == 0, 10),
            NamesObserved = make_set(Urls, 30),
            Events = sum(Events), EventsWithUrl = sum(EventsWithUrl),
            FirstSeenUtc = min(FirstUtc), LastSeenUtc = max(LastUtc)
    by ServerIP, ServerPort, Protocol, ClientProcess
| lookup kind=leftouter PortBaseline on ServerPort
| lookup kind=leftouter ServerIpName on ServerIP
| extend ServerShortName = tolower(tostring(split(ServerDeviceName, ".")[0]))
| extend NamesBeyondHostName = set_difference(NamesObserved, pack_array(tolower(ServerDeviceName), ServerShortName)),
         UrlFillPct = round(100.0 * EventsWithUrl / Events, 1),
         Addressing = case(
             ClientsWithName > 0 and ClientsWithoutName == 0, "FQDN: every client recorded a name",
             ClientsWithName > 0, "Mixed: some clients recorded a name, some did not",
             PortUrlFillPct >= UrlBaselineMinPct, "Probable IP-only access: no client recorded a name, though names are usually recorded on this port",
             "Unknown: names are rarely recorded on this port, so absence proves nothing")
| project ServerDeviceId, ServerDeviceName, ServerIpHolders, ServerIP, ServerPort, Protocol, ClientProcess, Addressing,
          NamesObserved, NamesBeyondHostName, ClientDevices, ClientsWithName, ClientsWithoutName, SampleClientsWithoutName,
          UrlFillPct, PortUrlFillPct, Events, EventsWithUrl, FirstSeenUtc, LastSeenUtc
| extend QueryId = "KQL-09", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by Addressing asc, ServerDeviceName asc, ServerPort asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| ServerDeviceId, ServerDeviceName, ServerIpHolders | Inferred | IP-to-device match (holders above 1 = merged servers) |
| ServerIP, ServerPort, Protocol, ClientProcess | Observed | DeviceNetworkEvents |
| Addressing | Inferred | ACC-02 classification against the port baseline |
| NamesObserved | Observed | RemoteUrl values (ACC-01) |
| NamesBeyondHostName | Inferred | Names other than the server's own host name: aliases, CNAMEs, published names |
| ClientDevices, ClientsWithName, ClientsWithoutName | Observed | Distinct client devices, split by whether any name was recorded |
| SampleClientsWithoutName | Observed | Up to 10 client device names |
| UrlFillPct | Inferred | Share of events with RemoteUrl for this endpoint and process |
| PortUrlFillPct | Inferred | Same share across all scoped endpoints on this port (the baseline) |
| Events, EventsWithUrl | Observed | Raw ConnectionSuccess events recorded |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max event times |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Non-onboarded clients aren't here, because only the client records RemoteUrl.
- A client resolving a name through a hosts file looks the same as one using DNS. Neither breaks when the IP changes unless the hosts file holds the old IP; that's an exception KQL can't see.
- The baseline is per port, not per process. A process that never resolves names on a well-filled port is the strongest IP-only signal.

**Validation check**

On a test client, connect to one server by name (`\\server\share`) and to another by IP (`\\10.x.x.x\share`). After about an hour, run with `HostFilter` set to both servers. The by-name connection should show the FQDN; the by-IP connection should have an empty name and classify as "Probable IP-only" or "Mixed". If the by-name connection also shows no name, RemoteUrl isn't reliable for SMB in your tenant, and ACC-02 for SMB must come from Splunk DNS logs.


## Authentication

### KQL-10: Inbound logons on servers: authentication profile

**CSV fields:** ATH-01, ATH-02, ACC-05, CMP-06, TMP-01  
**Purpose:** What authentication each server actually accepts: logon type × protocol × account domain, with account and remote-IP counts. It's recorded on the server itself, so it works even where a domain controller has no Defender for Identity sensor. NTLM, local-account and cross-domain logons are the exceptions to look for. Service and batch logons cross-check CMP-06 and TMP-01.  
**Tables:** `DeviceLogonEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-10 | Inbound logons on servers: authentication profile
// Fields: ATH-01 (Kerberos/NTLM/local), ATH-02 (domains), ACC-05 (NTLM), CMP-06 (Service logons), TMP-01 (Batch logons)
// Grain: one row per DeviceId + LogonType + Protocol + AccountDomain.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, LogonType, Protocol, AccountDomain, Events, Failures,
//   DistinctAccounts, SampleAccounts, DistinctRemoteIPs, SampleRemoteIPs, RemoteDevices, LocalAdminLogons,
//   FirstSeenUtc, LastSeenUtc.
// Inferred: LogonClass, AuthMethodHint, IsLocalAccountDomain.
// ActionTypes: LogonFailed (Learn: best-practices sample) and the *AggregatedReport variants (Learn: aggregated reporting);
//   LogonSuccess is NOT on a Learn page: confirm in the in-portal schema reference. The query takes any ActionType starting "Logon".
// LogonType and Protocol literal values aren't listed on Learn; classification uses contains() so either spelling works.
// Platform: not collected on Windows 7 or Server 2008 R2 (Learn: DeviceLogonEvents). Such servers show no rows.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceLogonEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where ActionType startswith "Logon"
| extend AccountKey = tolower(strcat(AccountDomain, "\\", AccountName))
| summarize Events = countif(not(ActionType endswith "AggregatedReport")),
            Failures = countif(ActionType contains "Failed" and not(ActionType endswith "AggregatedReport")),
            AggregatedReportRows = countif(ActionType endswith "AggregatedReport"),
            DistinctAccounts = dcount(AccountKey), SampleAccounts = make_set(AccountKey, 20),
            DistinctRemoteIPs = dcountif(RemoteIP, isnotempty(RemoteIP)), SampleRemoteIPs = make_set_if(RemoteIP, isnotempty(RemoteIP), 10),
            RemoteDevices = make_set_if(RemoteDeviceName, isnotempty(RemoteDeviceName), 10),
            LocalAdminLogons = countif(IsLocalAdmin == true),
            FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp)
    by DeviceId, DeviceName, LogonType, Protocol, AccountDomain
| lookup kind=leftouter (Servers | project DeviceId, HardwareUuid, OSPlatform, HostShortName) on DeviceId
| extend LogonClass = case(
             LogonType contains "Remote", "Remote interactive (RDP)",
             LogonType contains "Network", "Network (SMB, IIS, SQL Windows auth, WinRM and so on)",
             LogonType contains "Batch", "Batch (scheduled task)",
             LogonType contains "Service", "Service start",
             LogonType contains "Cached", "Cached interactive",
             LogonType contains "Interactive", "Interactive (console)",
             LogonType),
         AuthMethodHint = case(
             Protocol contains "Kerberos", "Kerberos",
             Protocol contains "NTLM", "NTLM",
             Protocol contains "Negotiate", "Negotiate (Kerberos or NTLM)",
             isempty(Protocol), "Not recorded",
             Protocol),
         IsLocalAccountDomain = tolower(AccountDomain) == HostShortName
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, LogonType, LogonClass, Protocol, AuthMethodHint, AccountDomain,
          IsLocalAccountDomain, Events, Failures, AggregatedReportRows, DistinctAccounts, SampleAccounts, DistinctRemoteIPs, SampleRemoteIPs,
          RemoteDevices, LocalAdminLogons, FirstSeenUtc, LastSeenUtc
| extend QueryId = "KQL-10", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, LogonClass asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform | Observed | Scope block |
| LogonType, Protocol, AccountDomain | Observed | DeviceLogonEvents |
| LogonClass | Inferred | Normalised logon type |
| AuthMethodHint | Inferred | Kerberos, NTLM, Negotiate or not recorded (ATH-01) |
| IsLocalAccountDomain | Inferred | AccountDomain equals the host's own name: a local account (ATH-01 "local auth") |
| Events, Failures | Observed | Raw events recorded; Failures where ActionType contains "Failed" |
| AggregatedReportRows | Observed | Logon*AggregatedReport rows, kept apart so they don't double the counts |
| DistinctAccounts, SampleAccounts | Observed | domain\account keys |
| DistinctRemoteIPs, SampleRemoteIPs, RemoteDevices | Observed | RemoteIP and RemoteDeviceName |
| LocalAdminLogons | Observed | Events where IsLocalAdmin is true |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max event times |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Shows Windows authentication only. Application forms logins, SQL logins, API keys and SAML to the app never create a Windows logon. "No AD evidence" suggests local or forms auth; confirm with the owner (CSV ATH-01 caveat).
- NTLM version (v1 or v2) isn't shown.
- Linux servers have few or no rows.

**Validation check**

On one IIS server using Windows authentication, open the site from a domain client. A `Network` row with `Kerberos` (or `Negotiate`) for that user's domain should appear within an hour. Then open it by IP address (`http://10.x.x.x`). That usually forces NTLM, and an NTLM row should appear. This proves the query can see an NTLM fallback.

### KQL-10b: Inbound logons on servers: accounts

**CSV fields:** ATH-06, CMP-06, ATH-01  
**Purpose:** The account-level view behind KQL-10: which accounts log on to each server, by which logon type and protocol. Network logons from non-onboarded clients show users that client-side network events can't. Service and batch accounts are the application's service accounts (CMP-06 cross-check).  
**Tables:** `DeviceLogonEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)

```kql
// KQL-10b | Inbound logons on servers: accounts
// Fields: ATH-06 (server-side users, including from non-onboarded clients), CMP-06 (Service/Batch accounts), ATH-01
// Grain: one row per DeviceId + account + LogonType + Protocol.
// Observed: DeviceId, DeviceName, AccountDomain, AccountName, AccountSid, LogonType, Protocol, Events, Failures,
//   DistinctRemoteIPs, SampleRemoteIPs, IsLocalAdminSeen, FirstSeenUtc, LastSeenUtc.
// Inferred: AccountClass, LogonClass.
// ActionTypes: as KQL-10 (LogonSuccess not on Learn; confirm in portal).
// Row limit 100,000: busy file servers have many users. Use LogonTypeFilter, HostFilter or SubnetFilter to slice.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let LogonTypeFilter = dynamic([]);    // optional, e.g. dynamic(["Network"]); matched with has_any
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceLogonEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where ActionType startswith "Logon"
| where array_length(LogonTypeFilter) == 0 or LogonType has_any (LogonTypeFilter)
| summarize Events = countif(not(ActionType endswith "AggregatedReport")),
            Failures = countif(ActionType contains "Failed" and not(ActionType endswith "AggregatedReport")),
            AggregatedReportRows = countif(ActionType endswith "AggregatedReport"),
            DistinctRemoteIPs = dcountif(RemoteIP, isnotempty(RemoteIP)), SampleRemoteIPs = make_set_if(RemoteIP, isnotempty(RemoteIP), 10),
            IsLocalAdminSeen = max(toint(IsLocalAdmin)),
            FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp)
    by DeviceId, DeviceName, AccountDomain, AccountName, AccountSid, LogonType, Protocol
| lookup kind=leftouter (Servers | project DeviceId, HostShortName) on DeviceId
| extend AccountClass = case(
             AccountName endswith "$", "Machine account",
             AccountName in~ ("system", "local service", "network service", "anonymous logon"), "Built-in system account",
             tolower(AccountDomain) == HostShortName, "Local account",
             LogonType contains "Service" or LogonType contains "Batch", "Probable service account",
             "Domain account"),
         LogonClass = case(
             LogonType contains "Remote", "Remote interactive (RDP)",
             LogonType contains "Network", "Network",
             LogonType contains "Batch", "Batch (scheduled task)",
             LogonType contains "Service", "Service start",
             LogonType contains "Cached", "Cached interactive",
             LogonType contains "Interactive", "Interactive (console)",
             LogonType)
| project-away HostShortName
| extend QueryId = "KQL-10b", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, LogonClass asc, Events desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName | Observed | DeviceLogonEvents |
| AccountDomain, AccountName, AccountSid, LogonType, Protocol | Observed | DeviceLogonEvents |
| Events, Failures, AggregatedReportRows | Observed | Raw events recorded; aggregated rows kept apart |
| DistinctRemoteIPs, SampleRemoteIPs | Observed | RemoteIP |
| IsLocalAdminSeen | Observed | 1 if any event had IsLocalAdmin = true |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max event times |
| AccountClass | Inferred | Machine, built-in, local, probable service, domain |
| LogonClass | Inferred | Normalised logon type |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- "Probable service account" is inferred from logon type only. Service accounts used interactively are classed as domain accounts.
- No UPN column in DeviceLogonEvents, so joining to IdentityInfo needs AccountSid (IdentityInfo.OnPremSid) or name matching.

**Validation check**

For one application server, compare the `Service` rows with `Get-CimInstance Win32_Service | Select Name, StartName`. Every non-built-in StartName should appear once its service has started at least once in the window.

### KQL-11: Active Directory authentication seen by Defender for Identity

**CSV fields:** ATH-01, ATH-02, ATH-04, ACC-05  
**Purpose:** What Defender for Identity saw at the domain controllers: protocol and logon type by destination and account domain. It's the main source for cleartext LDAP binds (ACC-05), Kerberos delegation in use (ATH-04) and the identity domains in play (ATH-02). It only covers DCs with a working sensor, so read it with KQL-20.  
**Tables:** `IdentityLogonEvents`, `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [IdentityLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitylogonevents-table)
- [Automatic attack disruption results (DC sample: LocalPort 88; Application == "Active Directory")](https://learn.microsoft.com/defender-xdr/autoad-results#track-the-actions-in-advanced-hunting)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-11 | Active Directory authentication seen by Defender for Identity
// Fields: ATH-01, ATH-02, ATH-04 (delegation in use), ACC-05 (cleartext LDAP, NTLM)
// Grain: one row per DestinationDeviceName + DestinationIPAddress + DestinationPort + Protocol + LogonType + AccountDomain.
// Observed: DestinationDeviceName, DestinationIP, DestinationPort, Protocol, LogonType, AccountDomain, Events, Failures,
//   DistinctAccounts, SampleAccounts, DistinctSourceDevices, SampleSourceDevices, SampleSourceIPs, FirstSeenUtc, LastSeenUtc.
// Inferred: ServerDeviceId, ServerDeviceName (IP match), DestinationRole, AuthMethod, LegacyFlag.
// LogonType values documented on Learn: Credentials Validation, Interactive Logon, Resource Access, Delegated Resource
//   Access, LDAP Cleartext, Remote Desktop, Failed Logon (exact strings: confirm with KQL-00b or the portal schema reference).
// Application == "Active Directory" follows the Learn sample (autoad-results). ActionType LogonFailed: Learn best-practices sample.
// Coverage: only DCs with a Defender for Identity sensor (HST-15). With HostFilter or SubnetFilter set, only rows whose
//   destination is a scoped server are kept.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let DcMinClients = 3;    // onboarded devices that must reach an IP on 88 before it's treated as a DC
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let FiltersSet = array_length(HostFilter) > 0 or array_length(SubnetFilter) > 0;
let DcIPSet = toscalar(
    union
        (DeviceNetworkEvents
         | where Timestamp between (WindowStart .. WindowEnd)
         | where LocalPort == 88 and ActionType in ("ListeningConnectionCreated", "InboundConnectionAccepted")
         | distinct DeviceId
         | join kind=inner (DeviceIPs | project DeviceId, IP) on DeviceId
         | project IP),
        (DeviceNetworkEvents
         | where Timestamp between (WindowStart .. WindowEnd)
         | where RemotePort == 88 and ActionType == "ConnectionSuccess"
         | summarize Clients = dcount(DeviceId) by IP = Norm(RemoteIP)
         | where Clients >= DcMinClients
         | project IP)
    | summarize make_set(IP));
IdentityLogonEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where Application == "Active Directory"
| extend AccountKey = iff(isnotempty(AccountUpn), tolower(AccountUpn), tolower(strcat(AccountDomain, "\\", AccountName)))
| summarize Events = count(), Failures = countif(ActionType contains "Failed"),
            DistinctAccounts = dcount(AccountKey), SampleAccounts = make_set(AccountKey, 20),
            DistinctSourceDevices = dcountif(DeviceName, isnotempty(DeviceName)), SampleSourceDevices = make_set_if(DeviceName, isnotempty(DeviceName), 20),
            SampleSourceIPs = make_set_if(IPAddress, isnotempty(IPAddress), 20),
            FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp)
    by DestinationDeviceName, DestinationIP = Norm(DestinationIPAddress), DestinationPort, Protocol, LogonType, AccountDomain
| lookup kind=leftouter ServerIpName on $left.DestinationIP == $right.ServerIP
| where not(FiltersSet) or isnotempty(ServerDeviceId)
| extend DestinationRole = case(
             set_has_element(DcIPSet, DestinationIP), "Domain controller (inferred from Kerberos traffic)",
             isnotempty(ServerDeviceId), "Scoped server",
             isempty(DestinationIP) and isempty(DestinationDeviceName), "Not recorded",
             "Other device"),
         AuthMethod = case(
             LogonType contains "Cleartext", "LDAP simple bind (cleartext)",
             LogonType contains "Delegat", "Kerberos delegation",
             LogonType contains "Certificate", "Certificate",
             LogonType contains "VPN", "RADIUS (VPN)",
             Protocol contains "Kerberos", "Kerberos",
             Protocol contains "Ntlm", "NTLM",
             Protocol contains "Ldap", "LDAP",
             isnotempty(Protocol), Protocol,
             "Not recorded"),
         LegacyFlag = case(
             LogonType contains "Cleartext", "Cleartext LDAP bind",
             Protocol contains "Ntlm", "NTLM (version not shown)",
             "")
| project DestinationRole, ServerDeviceId, ServerDeviceName, DestinationDeviceName, DestinationIP, DestinationPort,
          Protocol, LogonType, AuthMethod, LegacyFlag, AccountDomain, Events, Failures, DistinctAccounts, SampleAccounts,
          DistinctSourceDevices, SampleSourceDevices, SampleSourceIPs, FirstSeenUtc, LastSeenUtc
| extend QueryId = "KQL-11", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DestinationRole asc, DestinationDeviceName asc, Events desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DestinationRole | Inferred | DC (from Kerberos traffic), scoped server, other, not recorded |
| ServerDeviceId, ServerDeviceName | Inferred | Scoped server holding DestinationIP |
| DestinationDeviceName, DestinationIP, DestinationPort | Observed | IdentityLogonEvents (the server application that processed the logon) |
| Protocol, LogonType, AccountDomain | Observed | IdentityLogonEvents (ATH-02 domains) |
| AuthMethod | Inferred | Normalised method (ATH-01, ATH-04) |
| LegacyFlag | Inferred | Cleartext LDAP or NTLM (ACC-05) |
| Events, Failures | Observed | Events recorded |
| DistinctAccounts, SampleAccounts | Observed | UPN, or domain\name |
| DistinctSourceDevices, SampleSourceDevices, SampleSourceIPs | Observed | Source DeviceName and IPAddress |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max event times |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Blind to every DC without a working sensor (HST-15). A post DC with no sensor shows no logons at all.
- Learn doesn't say whether `DestinationDeviceName` is the DC or the target resource for each logon type. DestinationRole shows which it is in your data.
- No SPNs and no delegation configuration (ATH-04 needs a directory query).
- NTLM version isn't shown, so NTLMv1 (ACC-05) needs DC security logs (Splunk 4624 LmPackageName).

**Validation check**

From a test client, run an LDAP simple bind against a DC you know has a healthy sensor (for example `ldp.exe` > Bind > Simple bind, on port 389). Within an hour a row with `AuthMethod = "LDAP simple bind (cleartext)"` and the test account should appear. If it doesn't, the cleartext-LDAP evidence for ACC-05 can't be trusted for that DC.

### KQL-12: Domain controller dependency per server

**CSV fields:** ATH-03, DEP-02  
**Purpose:** Which DCs each server talks to on the Kerberos, LDAP, LDAPS, Global Catalog and password-change ports. It uses network evidence from the server itself, so it works whether or not the DC has a sensor. A server that depends on a single DC at a post is an exception: it fails silently when that DC is removed. Defender for Identity evidence for the same server and DC pair is added where it exists.  
**Tables:** `DeviceNetworkEvents`, `IdentityLogonEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [IdentityLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitylogonevents-table)
- [Automatic attack disruption results (DC sample: LocalPort 88; Application == "Active Directory")](https://learn.microsoft.com/defender-xdr/autoad-results#track-the-actions-in-advanced-hunting)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)

```kql
// KQL-12 | Domain controller dependency per server
// Fields: ATH-03, DEP-02 (AD part)
// Grain: one row per server DeviceId + DC IP.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, DcIP, Ports, Processes, Successes, SuccessReportRows, Failures,
//   FailureReportRows, Attempts, FirstSeenUtc, LastSeenUtc, DaysSeen.
// Inferred: DcDeviceId, DcName, DcOnboarding, DcIpOwnerCount (IP match), IsKnownDc, DcSubnet24, DcCountForServer (known
//   DCs reached successfully, self excluded), SingleDcDependency, DependencyNote, MdiLogonEvents and MdiProtocols
//   (tied to the server by IP match).
// ActionTypes: ConnectionSuccess, ConnectionAttempt (Learn); ConnectionSuccess/FailedAggregatedReport (Learn: aggregated
//   reporting); ConnectionFailed (NOT on Learn: confirm in portal).
// UDP Kerberos and CLDAP (389/UDP) may be missing; process details are TCP-only (Learn: investigate-machines).
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let DcPorts = dynamic([88, 389, 636, 3268, 3269, 464]);
let DcMinClients = 3;
let ConnActionTypes = dynamic(["ConnectionSuccess", "ConnectionFailed", "ConnectionAttempt",
                               "ConnectionSuccessAggregatedReport", "ConnectionFailedAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let KnownDcIPs = toscalar(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where RemotePort == 88 and ActionType == "ConnectionSuccess"
    | summarize Clients = dcount(DeviceId) by IP = Norm(RemoteIP)
    | where Clients >= DcMinClients
    | summarize make_set(IP));
let MdiPairs = ServerIPs
    | project SrcIP = ServerIP, DeviceId = ServerDeviceId
    | join kind=inner (
        IdentityLogonEvents
        | where Timestamp between (WindowStart .. WindowEnd)
        | where Application == "Active Directory"
        | where isnotempty(IPAddress) and isnotempty(DestinationIPAddress)
        | summarize MdiLogonEvents = count(), MdiProtocols = make_set(Protocol, 5)
            by SrcIP = Norm(IPAddress), DcIP = Norm(DestinationIPAddress)
      ) on SrcIP
    | summarize MdiLogonEvents = sum(MdiLogonEvents), MdiProtocols = make_set(MdiProtocols, 5) by DeviceId, DcIP;
let Pairs = materialize(
    DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where DeviceId in (OnboardedServerIds)
    | where RemotePort in (DcPorts)
    | where ActionType in (ConnActionTypes)
    | summarize Successes = countif(ActionType == "ConnectionSuccess"),
                SuccessReportRows = countif(ActionType == "ConnectionSuccessAggregatedReport"),
                Failures = countif(ActionType == "ConnectionFailed"),
                FailureReportRows = countif(ActionType == "ConnectionFailedAggregatedReport"),
                Attempts = countif(ActionType == "ConnectionAttempt"),
                Ports = make_set(RemotePort, 10), Processes = make_set(InitiatingProcessFileName, 10),
                FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp), DaysSeen = dcount(startofday(Timestamp))
        by DeviceId, DeviceName, DcIP = Norm(RemoteIP)
    | where not(IsLoopbackIp("", DcIP))
    | lookup kind=leftouter (IpOwner | project DcIP = IP, DcDeviceId = OwnerDeviceId, DcName = OwnerName,
                                                DcOnboarding = OwnerOnboarding, DcIpOwnerCount = IpOwnerCount) on DcIP
    | where isempty(DcDeviceId) or DcDeviceId != DeviceId                       // a DC talking to itself isn't a dependency
    | extend IsKnownDc = set_has_element(KnownDcIPs, DcIP), SuccessSeen = Successes > 0 or SuccessReportRows > 0);
Pairs
| lookup kind=leftouter (Pairs | summarize DcCountForServer = dcountif(DcIP, SuccessSeen and IsKnownDc) by DeviceId) on DeviceId
| lookup kind=leftouter (Servers | project DeviceId, HardwareUuid, OSPlatform) on DeviceId
| lookup kind=leftouter MdiPairs on DeviceId, DcIP
| extend DcSubnet24 = iff(DcIP contains ":", "", format_ipv4_mask(DcIP, 24)),
         SingleDcDependency = DcCountForServer == 1 and IsKnownDc and SuccessSeen,
         DependencyNote = case(
             not(IsKnownDc), "Few clients reach this IP on DC ports: confirm it's a DC (LDAP proxy or application LDAP target?)",
             DcCountForServer == 1 and SuccessSeen, "Only one DC used in the window: check for a pinned DC or a site with one DC",
             not(SuccessSeen) and (Failures > 0 or FailureReportRows > 0 or Attempts > 0), "Attempts or failures only: DC unreachable or decommissioned?",
             "")
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, DcIP, DcDeviceId, DcName, DcOnboarding, DcIpOwnerCount, IsKnownDc,
          DcSubnet24, Ports, Processes, Successes, SuccessReportRows, Failures, FailureReportRows, Attempts, DcCountForServer,
          SingleDcDependency, DependencyNote, MdiLogonEvents, MdiProtocols, FirstSeenUtc, LastSeenUtc, DaysSeen
| extend QueryId = "KQL-12", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, Successes desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform | Observed | Server (scope block) |
| DcIP | Observed | RemoteIP on DC ports |
| DcDeviceId, DcName, DcOnboarding, DcIpOwnerCount | Inferred | IP-to-device match |
| IsKnownDc | Inferred | At least DcMinClients onboarded devices reach this IP on 88 |
| DcSubnet24 | Inferred | Helper for Netbox (which post or data centre the DC is in) |
| Ports, Processes | Observed | RemotePort and InitiatingProcessFileName |
| Successes, Failures, Attempts | Observed | Raw events per outcome |
| SuccessReportRows, FailureReportRows | Observed | Aggregated-report rows, kept apart so they don't double the counts |
| DcCountForServer | Inferred | Known DCs (IsKnownDc) this server reached successfully, not counting itself |
| SingleDcDependency, DependencyNote | Inferred | ATH-03 exception flags |
| MdiLogonEvents, MdiProtocols | Inferred | IdentityLogonEvents whose source IP belongs to this server and destination is this IP |
| FirstSeenUtc, LastSeenUtc, DaysSeen | Observed | Min, max, distinct days |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Server side only. Which DC each user authenticates against needs client-side evidence or DC logs (Splunk 4624/4769).
- An application that queries LDAP itself (an app LDAP bind) and Windows authentication both show as the server talking to a DC. The process column separates them: `lsass.exe` is Windows, anything else is the application.

**Validation check**

On one server, run `nltest /dsgetdc:<domain>` and `nltest /sc_query:<domain>`. The DC it reports should appear here with the highest `Successes`. Then compare `DcSubnet24` with Netbox: if the DC sits in the post's own prefix and `SingleDcDependency` is true, the server is on the ATH-03 exception list.

### KQL-13: Entra ID sign-ins by application

**CSV fields:** ATH-01, ATH-06, ACC-07  
**Purpose:** Which applications users sign into through Entra ID (SAML, OIDC, federated through AD FS), how many users each has, and whether sign-ins came through Global Secure Access. This is ATH-01 evidence for SSO applications. It needs Entra ID P2; if KQL-00a shows zero rows, these fields lose that coverage.  
**Tables:** `EntraIdSignInEvents`  
**Verified against (Microsoft Learn):**

- [EntraIdSignInEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-entraidsigninevents-table)
- [AADSignInEventsBeta (deprecated)](https://learn.microsoft.com/defender-xdr/advanced-hunting-aadsignineventsbeta-table)
- [Advanced hunting schema naming changes](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-changes)

```kql
// KQL-13 | Entra ID sign-ins by application
// Fields: ATH-01 (SAML/OIDC/federated SSO apps), ATH-06 (SaaS and SSO users), ACC-07 (Global Secure Access sign-ins)
// Grain: one row per Application + ApplicationId + ResourceDisplayName + ResourceId.
// Observed: Application, ApplicationId, ResourceDisplayName, ResourceId, SignIns, Failures, DistinctUsers, SampleUsers,
//   ClientApps, TokenIssuers, LogonTypes, ViaGlobalSecureAccess, Countries, DistinctIPs, SampleIPs, DeviceTrustTypes,
//   FirstSeenUtc, LastSeenUtc.
// Inferred: FederatedViaAdfsHint.
// Licensing: Entra ID P2 is required (Learn). AADSignInEventsBeta is retired (Learn: schema changes); use this table.
// HostFilter and SubnetFilter don't apply: Entra sign-ins aren't tied to a scoped server.
// ---- Parameters (defaults = everything) ----
let Lookback    = 30d;             // native Defender data stops at 30 days
let WindowEnd   = now();           // set to ago(7d)... to run the window in slices
let WindowStart = WindowEnd - Lookback;
let AppFilter = dynamic([]);    // optional application names, e.g. dynamic(["Contoso HR"])
EntraIdSignInEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where array_length(AppFilter) == 0 or Application in~ (AppFilter)
| summarize SignIns = count(), Failures = countif(ErrorCode != 0), DistinctUsers = dcount(AccountObjectId),
            SampleUsers = make_set(AccountUpn, 20), ClientApps = make_set(ClientAppUsed, 10),
            TokenIssuers = make_set(TokenIssuerType, 5), LogonTypes = make_set(LogonType, 5),
            ViaGlobalSecureAccess = countif(IsSignInThroughGlobalSecureAccess == true),
            Countries = make_set(Country, 20), DistinctIPs = dcount(IPAddress), SampleIPs = make_set(IPAddress, 10),
            DeviceTrustTypes = make_set(DeviceTrustType, 5),
            FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp)
    by Application, ApplicationId, ResourceDisplayName, ResourceId
| extend FederatedViaAdfsHint = tostring(TokenIssuers) contains "ADFS" or tostring(TokenIssuers) contains "Federation"
| extend QueryId = "KQL-13", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DistinctUsers desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| Application, ApplicationId, ResourceDisplayName, ResourceId | Observed | EntraIdSignInEvents |
| SignIns, Failures | Observed | Rows; ErrorCode not 0 |
| DistinctUsers, SampleUsers | Observed | AccountObjectId and AccountUpn |
| ClientApps, TokenIssuers, LogonTypes, DeviceTrustTypes | Observed | EntraIdSignInEvents |
| ViaGlobalSecureAccess | Observed | Rows with IsSignInThroughGlobalSecureAccess = true |
| Countries, DistinctIPs, SampleIPs | Observed | Country and IPAddress (public egress, not the internal client IP) |
| FirstSeenUtc, LastSeenUtc | Observed | Min and max |
| FederatedViaAdfsHint | Inferred | Token issuer text mentions AD FS or federation |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- No link to the on-premises server hosting the app. You match Application names to App IDs by hand, or through App Proxy or ZPA app segments.
- IP addresses are public egress addresses, so ACC-06 client sites can't come from here.

**Validation check**

Pick one SAML application you know and one user who uses it. That user should appear in `SampleUsers` or be counted in `DistinctUsers`, and the count should be close to the Entra admin centre's sign-in report for the same app and window.

### KQL-14: Credential hints in server command lines (masked)

**CSV fields:** ATH-05  
**Purpose:** Hints of hard-coded credentials, such as passwords, keys or tokens passed on server command lines (scripts, scheduled tasks, `net use`, `sqlcmd`). Values are masked in the query, and the raw command line is never output. Every hit is a risk to test with the owner, not a finding (CSV ATH-05: low reliability).  
**Tables:** `DeviceProcessEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceProcessEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceprocessevents-table)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-14 | Credential hints in server command lines (masked)
// Fields: ATH-05 (hints only)
// Grain: one row per DeviceId + FileName + masked command line + account.
// Observed: DeviceId, DeviceName, FileName, RunAs, Runs, FirstSeenUtc, LastSeenUtc, ParentProcesses.
// Inferred: HintType, MaskedCommandLine (ProcessCommandLine with likely secrets replaced by ****).
// Masking is best effort. Review the output before sharing it, and never export ProcessCommandLine unmasked.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceProcessEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where ProcessCommandLine contains "pass" or ProcessCommandLine contains "pwd" or ProcessCommandLine contains "secret"
     or ProcessCommandLine contains "key" or ProcessCommandLine contains "token" or ProcessCommandLine contains "connection"
     or ProcessCommandLine contains "/p:" or ProcessCommandLine contains " -p" or ProcessCommandLine contains "/user:"
     or ProcessCommandLine contains " -u " 
| where ProcessCommandLine matches regex @"(?i)((password|passwd|pwd|secret|api[_-]?key|token|connectionstring)\s*[=:])|(\s/p(ass(word)?)?:\S)|(\s-p\s+\S)|(\s-pw\s+\S)|(/user:\S)|(\s-u\s+\S+:\S)"
| extend HintType = case(
             ProcessCommandLine matches regex @"(?i)(password|passwd|pwd)\s*[=:]", "password= style argument",
             ProcessCommandLine matches regex @"(?i)(secret|api[_-]?key|token)\s*[=:]", "secret, key or token argument",
             ProcessCommandLine matches regex @"(?i)connectionstring", "connection string on the command line",
             ProcessCommandLine matches regex @"(?i)/user:\S", "net use or similar with /user:",
             ProcessCommandLine matches regex @"(?i)\s-u\s+\S+:\S", "user:password pair (-u)",
             "-p, -pw or /p: switch")
| extend MaskedCommandLine = replace_regex(ProcessCommandLine,
             @"(?i)((?:password|passwd|pwd|secret|api[_-]?key|token)\s*[=:]\s*)(""[^""]*""|'[^']*'|[^\s;""']+)", @"\1****")
| extend MaskedCommandLine = replace_regex(MaskedCommandLine, @"(?i)(\s/p(?:ass(?:word)?)?:)(\S+)", @"\1****")
| extend MaskedCommandLine = replace_regex(MaskedCommandLine, @"(\s-(?:p|pw|P)\s+)(\S+)", @"\1****")
| extend MaskedCommandLine = replace_regex(MaskedCommandLine, @"(?i)(net1?(?:\.exe)?\s+use\s+\S+\s+)(\S+)(\s+/user:)", @"\1****\3")
| extend MaskedCommandLine = replace_regex(MaskedCommandLine, @"(\s-u\s+[^\s:]+:)(\S+)", @"\1****")
| extend MaskedCommandLine = replace_regex(MaskedCommandLine, @"(?i)(password\s*=\s*)([^;""']+)", @"\1****")
| summarize Runs = count(), FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp),
            ParentProcesses = make_set(InitiatingProcessFileName, 5)
    by DeviceId, DeviceName, FileName, HintType, MaskedCommandLine, RunAs = strcat(AccountDomain, "\\", AccountName)
| extend QueryId = "KQL-14", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, Runs desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, FileName | Observed | DeviceProcessEvents |
| HintType | Inferred | Which pattern matched |
| MaskedCommandLine | Inferred | ProcessCommandLine with likely secrets replaced by **** |
| RunAs | Observed | AccountDomain\AccountName of the new process |
| Runs, FirstSeenUtc, LastSeenUtc | Observed | Events recorded and time span |
| ParentProcesses | Observed | InitiatingProcessFileName values |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Credentials in configuration files, the registry or connection-string files never appear on a command line. Those need an owner review (ATH-05 primary).
- Over-matching is likely (for example `-p 8080` as a port). Under-matching too (for example `-pSecret` with no space).

**Validation check**

On a test server, run `cmd /c echo test --password=NotARealSecret`. Within an hour it should appear with `HintType = "password= style argument"` and `MaskedCommandLine` showing `--password=****`. The string `NotARealSecret` must not appear anywhere in the output.


## Dependencies

### KQL-15: Server-to-server and outbound dependencies

**CSV fields:** DEP-01, DEP-02, DEP-03, CMP-05  
**Purpose:** Everything each onboarded server connects out to: other servers (DEP-01 move groups), local infrastructure such as DNS, DHCP, NTP, AD, SMTP relays, print and file shares (DEP-02), and external or third-party destinations (DEP-03). Destinations unknown to Defender are kept and labelled for RunZero follow-up.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)
- [Investigate devices: internet-facing devices, InboundInternetScanInspected, TCP-only process data](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)

```kql
// KQL-15 | Server-to-server and outbound dependencies
// Fields: DEP-01, DEP-02, DEP-03, CMP-05 (inferred from destination tier)
// Grain: one row per source server + source process + destination IP + destination port + protocol.
// Observed: DeviceId, DeviceName, HardwareUuid, OSPlatform, SourceProcess, ProcessFolderPaths, Accounts, DestIP, DestPort,
//   Protocol, RemoteIPType, RemoteUrls, Successes, Failures, FirstSeenUtc, LastSeenUtc, DaysSeen.
// Inferred: DestDeviceId, DestDeviceName, DestDeviceType, DestOnboarding, DestIpOwnerCount, DestIsScopedServer (IP match),
//   DestIsPublic, DestScope, DependencyClass, NameScope, Addressing, DestSubnet24, IsAgentTraffic.
// RemoteIPType gives IPv4-mapped addresses their own type (FourToSixMapping), so public/loopback are decided after Norm().
// Aggregated-report rows are counted apart (AggregatedReportRows) so they don't double Successes and Failures.
// ActionTypes: ConnectionSuccess (Learn); *AggregatedReport (Learn); ConnectionFailed (NOT on Learn: confirm in portal).
// UDP services (DNS, NTP, DHCP) may be missing or lack a process: configured resolvers and DHCP come from KQL-02 instead.
// Traffic through a proxy shows the proxy as DestIP; RemoteUrl may still carry the real destination.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let PortFilter = dynamic([]);          // optional destination ports
let IncludeExternal = true;            // false = internal destinations only
let AgentProcesses = dynamic(["MsSense.exe", "SenseNdr.exe", "SenseCncProxy.exe", "SenseIR.exe", "MsMpEng.exe",
                              "mdatp", "wdavdaemon"]);   // tagged, not dropped
let ConnActionTypes = dynamic(["ConnectionSuccess", "ConnectionFailed",
                               "ConnectionSuccessAggregatedReport", "ConnectionFailedAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
// Well-known port hints. Inferred: a port number suggests a protocol; RunZero fingerprints confirm it.
let PortMap = datatable(Port: int, Service: string, Tier: string, LegacyRisk: string, LatencyRisk: string) [
    21, "FTP", "File transfer", "Cleartext (FTP)", "Medium",
    22, "SSH or SFTP", "Remote admin or file transfer", "", "Low",
    23, "Telnet", "Remote admin", "Cleartext (Telnet)", "Low",
    25, "SMTP", "Mail relay", "Cleartext unless STARTTLS", "Low",
    53, "DNS", "Infrastructure (DNS)", "", "Low",
    67, "DHCP", "Infrastructure (DHCP)", "", "Low",
    69, "TFTP", "File transfer", "Cleartext (TFTP)", "Medium",
    80, "HTTP", "Web", "Cleartext (HTTP)", "Low",
    88, "Kerberos", "Directory (DC)", "", "Low",
    123, "NTP", "Infrastructure (NTP)", "", "Low",
    135, "RPC endpoint mapper", "RPC", "", "High",
    137, "NetBIOS name", "Legacy Windows networking", "NetBIOS", "Medium",
    139, "NetBIOS session (SMB)", "File", "NetBIOS SMB", "High",
    161, "SNMP", "Management", "SNMP v1/v2c possible", "Low",
    389, "LDAP", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    443, "HTTPS", "Web", "", "Low",
    445, "SMB", "File", "SMB version not visible here", "High",
    464, "Kerberos password change", "Directory (DC)", "", "Low",
    465, "SMTPS", "Mail relay", "", "Low",
    514, "Syslog", "Logging", "Cleartext (syslog)", "Low",
    515, "LPD", "Print", "Cleartext (LPD)", "Low",
    587, "SMTP submission", "Mail relay", "", "Low",
    631, "IPP", "Print", "", "Low",
    636, "LDAPS", "Directory (DC)", "", "Medium",
    1433, "SQL Server", "Database", "", "High",
    1434, "SQL Server Browser", "Database", "", "Medium",
    1494, "Citrix ICA", "VDI or app delivery", "", "Low",
    1521, "Oracle", "Database", "", "High",
    2598, "Citrix CGP", "VDI or app delivery", "", "Low",
    3268, "Global Catalog", "Directory (DC)", "Cleartext LDAP possible", "Medium",
    3269, "Global Catalog over SSL", "Directory (DC)", "", "Medium",
    3306, "MySQL", "Database", "", "High",
    3389, "RDP", "Remote desktop", "", "Low",
    5432, "PostgreSQL", "Database", "", "High",
    5985, "WinRM over HTTP", "Remote admin", "", "Low",
    5986, "WinRM over HTTPS", "Remote admin", "", "Low",
    8080, "HTTP (alternate)", "Web", "Cleartext (HTTP)", "Low",
    8443, "HTTPS (alternate)", "Web", "", "Low",
    9100, "Raw printing", "Print", "", "Low",
    27017, "MongoDB", "Database", "", "High"
];
let InternalSuffixes = toscalar(
    DeviceLatest
    | where DeviceName contains "."
    | summarize make_set(tolower(substring(DeviceName, indexof(DeviceName, ".") + 1)), 200));
DeviceNetworkEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where ActionType in (ConnActionTypes)
| where RemoteIPType != "Loopback"
| where array_length(PortFilter) == 0 or RemotePort in (PortFilter)
| summarize Successes = countif(ActionType == "ConnectionSuccess"),
            Failures = countif(ActionType == "ConnectionFailed"),
            AggregatedReportRows = countif(ActionType endswith "AggregatedReport"),
            RemoteUrls = make_set_if(tolower(RemoteUrl), isnotempty(RemoteUrl), 10),
            ProcessFolderPaths = make_set(InitiatingProcessFolderPath, 3),
            Accounts = make_set(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName), 5),
            FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp), DaysSeen = dcount(startofday(Timestamp))
    by DeviceId, DeviceName, SourceProcess = InitiatingProcessFileName, DestIP = Norm(RemoteIP), DestPort = RemotePort,
       Protocol, RemoteIPType
| where not(IsLoopbackIp(RemoteIPType, DestIP))
| extend DestIsPublic = IsPublicIp(RemoteIPType, DestIP)          // RemoteIPType alone misses IPv4-mapped addresses
| where IncludeExternal or not(DestIsPublic)
| lookup kind=leftouter (IpOwner | project DestIP = IP, DestDeviceId = OwnerDeviceId, DestDeviceName = OwnerName,
                                            DestDeviceType = OwnerType, DestOnboarding = OwnerOnboarding,
                                            DestIpOwnerCount = IpOwnerCount) on DestIP
| lookup kind=leftouter (ServerIPs | summarize ScopedServerId = take_any(ServerDeviceId) by DestIP = ServerIP) on DestIP
| lookup kind=leftouter (Servers | project DeviceId, HardwareUuid, OSPlatform) on DeviceId
| lookup kind=leftouter PortMap on $left.DestPort == $right.Port
| where isempty(DestDeviceId) or DestDeviceId != DeviceId
| extend DestIsScopedServer = isnotempty(ScopedServerId),
         IsAgentTraffic = SourceProcess in~ (AgentProcesses),
         DestSubnet24 = iff(DestIP contains ":", "", format_ipv4_mask(DestIP, 24))
| extend DestScope = case(
             DestIsPublic, "External (public IP)",
             DestIsScopedServer and DestOnboarding == "Onboarded", "Server to server (onboarded destination)",
             DestIsScopedServer, "Server to server (destination discovered, not onboarded)",
             isnotempty(DestDeviceId) and DestOnboarding == "Onboarded", "To an onboarded non-server device",
             isnotempty(DestDeviceId), "To a discovered device, not onboarded",
             "To an internal IP unknown to Defender: follow up in RunZero"),
         DependencyClass = case(
             IsAgentTraffic, "Defender agent traffic",
             DestPort in (88, 389, 636, 3268, 3269, 464), "Directory (AD)",
             DestPort == 53, "DNS",
             DestPort in (67, 68), "DHCP",
             DestPort == 123, "NTP",
             DestPort in (25, 465, 587), "SMTP relay",
             DestPort in (139, 445), "File share (SMB)",
             DestPort in (9100, 515, 631), "Print",
             DestPort == 80 and tostring(RemoteUrls) has_any ("crl", "ocsp", "pki", "aia"), "PKI (CRL or OCSP)",
             isnotempty(Tier), Tier,
             "Application or other"),
         NameScope = case(
             array_length(RemoteUrls) == 0, "No name recorded",
             tostring(RemoteUrls) has_any (InternalSuffixes), "Internal DNS suffix",
             "External or unrecognised suffix"),
         Addressing = iff(array_length(RemoteUrls) > 0, "FQDN recorded", "IP only (no RemoteUrl recorded)")
| project DeviceId, DeviceName, HardwareUuid, OSPlatform, SourceProcess, ProcessFolderPaths, Accounts,
          DestIP, DestPort, Protocol, RemoteIPType, DestIsPublic, DestScope, DependencyClass, DestDeviceId, DestDeviceName,
          DestDeviceType, DestOnboarding, DestIpOwnerCount, DestIsScopedServer, DestSubnet24, RemoteUrls, NameScope, Addressing,
          IsAgentTraffic, Successes, Failures, AggregatedReportRows, FirstSeenUtc, LastSeenUtc, DaysSeen
| extend QueryId = "KQL-15", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, DependencyClass asc, DestIP asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, HardwareUuid, OSPlatform | Observed | Source server |
| SourceProcess, ProcessFolderPaths, Accounts | Observed | InitiatingProcess* columns |
| DestIP, DestPort, Protocol, RemoteIPType | Observed | DeviceNetworkEvents |
| DestIsPublic | Inferred | Public by RemoteIPType, or an IPv4-mapped address outside private ranges |
| DestScope | Inferred | Server to server, internal unknown, external and so on (DEP-01, DEP-03) |
| DependencyClass | Inferred | DNS, DHCP, NTP, AD, SMTP, SMB, print, PKI, tier from the port map (DEP-02, CMP-05) |
| DestDeviceId, DestDeviceName, DestDeviceType, DestOnboarding, DestIpOwnerCount, DestIsScopedServer | Inferred | IP-to-device match |
| DestSubnet24 | Inferred | Helper for Netbox (is the service at the same post?) |
| RemoteUrls | Observed | RemoteUrl values |
| NameScope | Inferred | Name has an internal suffix, an external suffix, or none recorded |
| Addressing | Inferred | FQDN recorded, or IP only (workbook Dependencies "Addressing") |
| IsAgentTraffic | Inferred | Defender's own processes |
| Successes, Failures, AggregatedReportRows | Observed | Raw events per outcome; aggregated rows kept apart |
| FirstSeenUtc, LastSeenUtc, DaysSeen | Observed | Min, max, distinct days |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Monthly and annual integrations outside the 30 days are invisible (re-run after month-end; see KQL-17).
- The process is visible only at the onboarded end (CSV DEP-01).
- Partner allow-lists of a post's public egress IP can't be seen here. Ask DEP-03 partners directly; KQL-02 `PublicIP` shows the egress IP each server uses today.
- Row limit 100,000: slice with HostFilter, SubnetFilter or PortFilter, or set IncludeExternal = false.

**Validation check**

For one application with a known database back end, the app server's row to the DB server's IP on 1433 should appear with `DestScope = "Server to server (onboarded destination)"` and the app's process name. Then compare one server's `DNS` and `NTP` rows with its configured servers (KQL-02 `DnsServerSets`; `w32tm /query /source`). If NTP never appears, UDP is not recorded for that host and DEP-02 NTP needs RunZero or the server team.


## Scheduled and temporal

### KQL-16: Scheduled task and cron job runs

**CSV fields:** TMP-01, TMP-02, ATH-05  
**Purpose:** Processes launched by the Windows Task Scheduler or Linux cron/at on each server, with run counts, active days, hours and days of month, and a cadence hint. Jobs that run near month-end are period-end candidates (TMP-02). Command lines are masked for secrets.  
**Tables:** `DeviceProcessEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceProcessEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceprocessevents-table)
- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-16 | Scheduled task and cron job runs
// Fields: TMP-01, TMP-02 (inputs), ATH-05 (masked)
// Grain: one row per DeviceId + scheduler + FileName + normalised command line + RunAs account.
// Observed: DeviceId, DeviceName, FileName, FolderPaths, RunAs, Runs, DistinctDays, FirstRunUtc, LastRunUtc,
//   HoursUtc, DaysOfWeek, DaysOfMonth.
// Inferred: Scheduler, CommandLineNorm (secrets masked, long digit runs collapsed), CadenceHint.
// How a scheduled run is recognised: the parent is svchost.exe running the "Schedule" service, taskeng.exe or
//   taskhostw.exe (Windows), or the parent or grandparent is cron, crond, anacron or atd (Linux). This is logic, not a
//   documented Defender field. On older Windows, Task Scheduler shares an svchost with other services and its runs are missed.
// systemd timers aren't recognised: everything on Linux descends from systemd.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let LinuxSchedulers = dynamic(["cron", "crond", "anacron", "atd"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceProcessEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where (InitiatingProcessFileName =~ "svchost.exe" and InitiatingProcessCommandLine has "Schedule")
     or InitiatingProcessFileName in~ ("taskeng.exe", "taskhostw.exe")
     or InitiatingProcessFileName in~ (LinuxSchedulers)
     or InitiatingProcessParentFileName in~ (LinuxSchedulers)
| extend Scheduler = case(
             InitiatingProcessFileName =~ "svchost.exe", "Windows Task Scheduler",
             InitiatingProcessFileName in~ ("taskeng.exe", "taskhostw.exe"), "Windows Task Scheduler host",
             "Linux cron or at")
| extend CommandLineNorm = replace_regex(ProcessCommandLine,
             @"(?i)((?:password|passwd|pwd|secret|token)\s*[=:]\s*)(\S+)", @"\1****")
| extend CommandLineNorm = replace_regex(CommandLineNorm, @"\d{4,}", "#")
| summarize Runs = count(), DistinctDays = dcount(startofday(Timestamp)),
            FirstRunUtc = min(Timestamp), LastRunUtc = max(Timestamp),
            HoursUtc = make_set(hourofday(Timestamp), 24),
            DaysOfWeek = make_set(toint(dayofweek(Timestamp) / 1d), 7),
            DaysOfMonth = make_set(dayofmonth(Timestamp), 31),
            FolderPaths = make_set(FolderPath, 3)
    by DeviceId, DeviceName, Scheduler, FileName, CommandLineNorm, RunAs = strcat(AccountDomain, "\\", AccountName)
| extend FirstDayOfMonth = toint(array_sort_asc(DaysOfMonth)[0])
| extend CadenceHint = case(
             DistinctDays >= 20, "Daily or more often",
             array_length(DaysOfWeek) == 1 and DistinctDays >= 3, "Weekly",
             DistinctDays <= 2 and (FirstDayOfMonth >= 28 or FirstDayOfMonth <= 2), "Once or twice, near month-end: possible monthly or period-end job",
             DistinctDays <= 2, "Once or twice in the window: monthly, ad hoc or rare",
             "Irregular")
| project-away FirstDayOfMonth
| extend QueryId = "KQL-16", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, Runs desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, FileName, FolderPaths | Observed | DeviceProcessEvents (the launched process) |
| Scheduler | Inferred | Which scheduler launched it, from the parent process |
| CommandLineNorm | Inferred | ProcessCommandLine with secrets masked and long digit runs replaced by # |
| RunAs | Observed | AccountDomain\AccountName |
| Runs, DistinctDays, FirstRunUtc, LastRunUtc | Observed | Events recorded and time span |
| HoursUtc, DaysOfWeek, DaysOfMonth | Observed | When it ran (UTC; DaysOfWeek 0 = Sunday) |
| CadenceHint | Inferred | Daily, weekly, near month-end, rare, irregular |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- 30 days misses quarterly, half-yearly and EOFY jobs (CSV TMP-01). Splunk 4698 and owner confirmation are the primary sources.
- Jobs run by an application's own scheduler (SQL Agent, Control-M, a Windows service) aren't recognised.
- Times are UTC. Convert to the post's local time before calling something "overnight" or "business hours".

**Validation check**

On a test server, create a task that runs `cmd /c echo kql16test` every 15 minutes. After two hours it should appear with `Scheduler = "Windows Task Scheduler"` and about eight runs. If it doesn't, the svchost "Schedule" detection doesn't fit that OS build; run KQL-16b, and check the parent of `cmd.exe` on that host in DeviceProcessEvents.

### KQL-16b: Scheduled task changes and batch logon accounts

**CSV fields:** TMP-01, CMP-06  
**Purpose:** Two supporting signals for TMP-01. The first is task create, update and delete events, which name the task even when it didn't run in the window. The second is the accounts that started batch sessions; Learn documents the Batch logon type as "Session initiated by scheduled tasks".  
**Tables:** `DeviceEvents`, `DeviceLogonEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceevents-table)
- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)

```kql
// KQL-16b | Scheduled task changes and batch logon accounts
// Fields: TMP-01, CMP-06 (service accounts used by tasks)
// Grain: one row per DeviceId + task name (DeviceEvents) or + account (DeviceLogonEvents Batch).
// Observed: DeviceId, DeviceName, TaskOrAccount, Events, ActionTypes, FirstUtc, LastUtc, Actors, SampleAdditionalFields, DistinctDays.
// Inferred: EvidenceSource (label).
// ActionTypes: ScheduledTask* in DeviceEvents is NOT on Learn: confirm the names in the in-portal schema reference.
//   The TaskName key inside AdditionalFields isn't documented; SampleAdditionalFields shows the real keys.
// Batch logon type: Learn DeviceLogonEvents ("Session initiated by scheduled tasks").
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
union
    (DeviceEvents
     | where Timestamp between (WindowStart .. WindowEnd)
     | where DeviceId in (OnboardedServerIds)
     | where ActionType startswith "ScheduledTask"
     | extend TaskName = tostring(parse_json(AdditionalFields).TaskName)
     | summarize Events = count(), ActionTypes = make_set(ActionType, 5), FirstUtc = min(Timestamp), LastUtc = max(Timestamp),
                 Actors = make_set(strcat(InitiatingProcessAccountDomain, "\\", InitiatingProcessAccountName), 5),
                 SampleAdditionalFields = take_any(AdditionalFields)
         by DeviceId, DeviceName, TaskOrAccount = TaskName
     | extend EvidenceSource = "DeviceEvents ScheduledTask* (task definition changed)"),
    (DeviceLogonEvents
     | where Timestamp between (WindowStart .. WindowEnd)
     | where DeviceId in (OnboardedServerIds)
     | where LogonType contains "Batch"
     | summarize Events = count(), ActionTypes = make_set(ActionType, 5), FirstUtc = min(Timestamp), LastUtc = max(Timestamp),
                 DistinctDays = dcount(startofday(Timestamp))
         by DeviceId, DeviceName, TaskOrAccount = tolower(strcat(AccountDomain, "\\", AccountName))
     | extend EvidenceSource = "DeviceLogonEvents Batch logon (account that ran scheduled tasks)")
| project DeviceId, DeviceName, EvidenceSource, TaskOrAccount, Events, ActionTypes, DistinctDays, FirstUtc, LastUtc,
          Actors, SampleAdditionalFields
| extend QueryId = "KQL-16b", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by DeviceName asc, EvidenceSource asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName | Observed | DeviceEvents or DeviceLogonEvents |
| EvidenceSource | Inferred | Which signal the row came from (label) |
| TaskOrAccount | Observed | Task name from AdditionalFields, or the batch logon account |
| Events, ActionTypes, DistinctDays, FirstUtc, LastUtc | Observed | Events recorded and time span |
| Actors | Observed | Account that created or changed the task |
| SampleAdditionalFields | Observed | One raw AdditionalFields value, for confirming keys |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Task change events only exist for tasks created or changed in the window. A task set up years ago only shows through its runs (KQL-16) or Splunk 4698.

**Validation check**

The test task from KQL-16 should produce a `ScheduledTask*` row (task created) and, if it runs as a domain account, a batch logon row for that account.

### KQL-17: Period-end and last-use profile per server endpoint

**CSV fields:** TMP-02, TMP-03  
**Purpose:** The daily rhythm of each server endpoint: active days, last day of use, peak days, and how much of its client activity falls in the days around month-end compared with an even spread. An endpoint used mostly near month-end is a period-end candidate (TMP-02). A run today covers the 31 October month-end; repeat it in the week of 7 December, after November month-end.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Advanced hunting overview: quotas and usage parameters](https://learn.microsoft.com/defender-xdr/advanced-hunting-overview#quotas-and-usage-parameters)

```kql
// KQL-17 | Period-end and last-use profile per server endpoint
// Fields: TMP-02 (one month-end at most), TMP-03
// Grain: one row per ServerIP + ServerPort.
// Observed: ServerIP, ServerPort, ActiveDays, FirstUseDayUtc, LastUseDayUtc, MaxClientsInADay, MedianClientsPerActiveDay,
//   ClientDaysTotal, ClientDaysMonthEdge, MonthEdgeDaysActive.
// Inferred: ServerDeviceId, ServerDeviceName, ServerIpHolders (IP match), PeakDays, MonthEdgeShareOfUse, MonthEdgeShareOfWindow,
//   MonthEdgeLift, PeriodEndHint, DaysSinceLastUse.
// Clients are counted by IP on both sides (client LocalIP and server RemoteIP). A client seen by both sides counts once,
// but behind NAT or ZPA the server sees a different IP, so that client counts twice. Self-connections are removed.
// Days are UTC days. Posts many hours from UTC see month-end shifted by up to a day.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let PortFilter = dynamic([]);
let MonthEdgeDays = 2;    // days before the last day of the month, and from the 1st, that count as "month-end"
let ClientActionTypes = dynamic(["ConnectionSuccess", "ConnectionSuccessAggregatedReport"]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let IsEdge = (d: datetime) { d >= startofday(endofmonth(d)) - MonthEdgeDays * 1d or dayofmonth(d) <= MonthEdgeDays };
let WindowShape = range Day from startofday(WindowStart) to startofday(WindowEnd) step 1d
    | extend Edge = IsEdge(Day)
    | summarize WindowDays = count(), WindowEdgeDays = countif(Edge);
let WindowEdgeShare = toscalar(WindowShape | project Share = 1.0 * WindowEdgeDays / WindowDays);
let Daily = materialize(
    union
        (DeviceNetworkEvents
         | where Timestamp between (WindowStart .. WindowEnd)
         | where ActionType in (ClientActionTypes)
         | where RemoteIP in (ServerIPMatch)
         | where array_length(PortFilter) == 0 or RemotePort in (PortFilter)
         | summarize by ServerIP = Norm(RemoteIP), ServerPort = RemotePort, Day = startofday(Timestamp), ClientIP = Norm(LocalIP),
                        ClientDeviceId = DeviceId
         | join kind=leftanti (ServerIPs | project ServerIP, ClientDeviceId = ServerDeviceId) on ServerIP, ClientDeviceId
         | project-away ClientDeviceId),
        (DeviceNetworkEvents
         | where Timestamp between (WindowStart .. WindowEnd)
         | where ActionType == "InboundConnectionAccepted"
         | where DeviceId in (OnboardedServerIds)
         | where array_length(PortFilter) == 0 or LocalPort in (PortFilter)
         | summarize by ServerDeviceId = DeviceId, ServerIP = Norm(LocalIP), ServerPort = LocalPort, Day = startofday(Timestamp),
                        ClientIP = Norm(RemoteIP)
         | join kind=leftanti (ServerIPs | project ServerDeviceId, ClientIP = ServerIP) on ServerDeviceId, ClientIP
         | project-away ServerDeviceId)
    | where not(ClientIP in ("127.0.0.1", "::1"))
    | summarize Clients = dcount(ClientIP) by ServerIP, ServerPort, Day
    | extend Edge = IsEdge(Day));
let Stats = Daily
    | summarize ActiveDays = count(), FirstUseDayUtc = min(Day), LastUseDayUtc = max(Day),
                MaxClientsInADay = max(Clients), MedianClientsPerActiveDay = percentile(Clients, 50),
                ClientDaysTotal = sum(Clients), ClientDaysMonthEdge = sumif(Clients, Edge),
                MonthEdgeDaysActive = countif(Edge)
        by ServerIP, ServerPort;
let Peaks = Daily
    | lookup kind=inner (Stats | project ServerIP, ServerPort, MedianClientsPerActiveDay) on ServerIP, ServerPort
    | where Clients >= 2 * MedianClientsPerActiveDay and Clients >= 3
    | summarize PeakDays = make_set(format_datetime(Day, "yyyy-MM-dd"), 10) by ServerIP, ServerPort;
Stats
| lookup kind=leftouter Peaks on ServerIP, ServerPort
| lookup kind=leftouter ServerIpName on ServerIP
| extend MonthEdgeShareOfUse = round(1.0 * ClientDaysMonthEdge / ClientDaysTotal, 3),
         MonthEdgeShareOfWindow = round(WindowEdgeShare, 3)
| extend MonthEdgeLift = iff(MonthEdgeShareOfWindow > 0, round(MonthEdgeShareOfUse / MonthEdgeShareOfWindow, 2), real(null)),
         DaysSinceLastUse = datetime_diff("day", WindowEnd, LastUseDayUtc)
| extend PeriodEndHint = case(
             MonthEdgeShareOfWindow == 0, "Window has no month-end days: can't tell",
             ActiveDays == MonthEdgeDaysActive, "Used only around month-end in this window",
             MonthEdgeLift >= 2.0, "Use concentrated around month-end",
             "No period-end pattern in this window (one month-end at most)")
| project ServerDeviceId, ServerDeviceName, ServerIpHolders, ServerIP, ServerPort, ActiveDays, FirstUseDayUtc, LastUseDayUtc,
          DaysSinceLastUse, MaxClientsInADay, MedianClientsPerActiveDay, PeakDays, ClientDaysTotal, ClientDaysMonthEdge,
          MonthEdgeDaysActive, MonthEdgeShareOfUse, MonthEdgeShareOfWindow, MonthEdgeLift, PeriodEndHint
| extend QueryId = "KQL-17", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by MonthEdgeLift desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| ServerDeviceId, ServerDeviceName, ServerIpHolders | Inferred | IP-to-device match (holders above 1 = merged servers) |
| ServerIP, ServerPort | Observed | DeviceNetworkEvents |
| ActiveDays, FirstUseDayUtc, LastUseDayUtc | Observed | UTC days with at least one client connection (TMP-03) |
| DaysSinceLastUse | Inferred | Window end minus last use day |
| MaxClientsInADay, MedianClientsPerActiveDay | Observed | Distinct client IPs per day (aggregate) |
| PeakDays | Inferred | Days with at least twice the median and at least 3 clients |
| ClientDaysTotal, ClientDaysMonthEdge, MonthEdgeDaysActive | Observed | Sum of daily client counts, all days and month-end days |
| MonthEdgeShareOfUse, MonthEdgeShareOfWindow, MonthEdgeLift | Inferred | Share of use near month-end against the share of the window that's near month-end |
| PeriodEndHint | Inferred | TMP-02 hint |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- One window holds at most one month-end. Quarter-end and EOFY (30 June) need Splunk history or owner confirmation (CSV TMP-02).
- Endpoints with no use in the window aren't here. KQL-05 rows with no inbound are the "no observed use" list.

**Validation check**

For one finance application you know is busier at month-end, `MonthEdgeLift` should be well above 1. If it isn't, check whether its month-end work is server-to-server batch (that appears in KQL-15 and KQL-16, not here).


## Coverage and validation

### KQL-18: Onboarding and sensor gaps (device and IP level)

**CSV fields:** HST-14, HST-13, DEP-01  
**Purpose:** What's missing, not just what's present. There are three gap types: servers Defender has discovered but not onboarded; onboarded servers whose sensor isn't Active; and internal IPs that accept connections from onboarded devices but match nothing in Defender's inventory. Those last ones exist only as the remote end of a connection. This is your follow-up list for RunZero and VMware.  
**Tables:** `DeviceInfo`, `DeviceNetworkInfo`, `DeviceNetworkEvents`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)
- [SeenBy() function](https://learn.microsoft.com/defender-xdr/advanced-hunting-seenby-function)
- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)

```kql
// KQL-18 | Onboarding and sensor gaps (device and IP level)
// Fields: HST-14, HST-13 (last seen), DEP-01 (endpoints Defender can't see the process for)
// Grain: one row per DeviceId (gap types 1 and 2) or per IP (gap type 3).
// Observed: DeviceId, DeviceName, HardwareUuid, OnboardingStatus, SensorHealthState, DeviceType, OSPlatform, Vendor, Model,
//   DiscoverySources, LastSeenUtc, IPs, MacAddresses, ServedPorts, DistinctClientDevices, SampleClients, NamesSeen,
//   FirstSeenUtc.
// Inferred: GapType, MatchedDeviceId (IP match for gap type 3).
// OnboardingStatus values (Learn: assess-devices): Onboarded, Can be onboarded, Unsupported, Insufficient info.
// To see which onboarded devices observed a discovered device, run SeenBy() on the DeviceIds (Learn: SeenBy, max 1,000 IDs).
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let ServingPortMax = 49151;     // remote ports above this are treated as ephemeral (client side), not a served port
let ServingMinClients = 1;      // onboarded devices that must have connected before an unknown IP is listed
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let DiscoInfo = DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where DeviceId in ((Servers | project DeviceId))
    | summarize arg_max(Timestamp, Vendor, Model, DiscoverySources) by DeviceId
    | project DeviceId, Vendor, Model, DiscoverySources;
let DeviceNics = DeviceIPs
    | summarize IPs = make_set(IP, 20), MacAddresses = make_set_if(MacAddress, isnotempty(MacAddress), 10) by DeviceId;
let DeviceGaps = Servers
    | where OnboardingStatus != "Onboarded" or SensorHealthState != "Active"
    | extend GapType = iff(OnboardingStatus != "Onboarded",
                           strcat("Server discovered but not onboarded (", OnboardingStatus, ")"),
                           strcat("Onboarded but sensor not Active (", SensorHealthState, "): partial or no telemetry"))
    | lookup kind=leftouter DiscoInfo on DeviceId
    | lookup kind=leftouter DeviceNics on DeviceId
    | project GapType, DeviceId, DeviceName, HardwareUuid, IPs, MacAddresses, OnboardingStatus, SensorHealthState,
              DeviceType, OSPlatform, Vendor, Model, DiscoverySources, LastSeenUtc;
let IpGaps = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType in ("ConnectionSuccess", "ConnectionSuccessAggregatedReport")
    | where RemoteIPType in~ ("Private", "FourToSixMapping") and RemotePort <= ServingPortMax
    | summarize ServedPorts = make_set(RemotePort, 30), DistinctClientDevices = dcount(DeviceId),
                SampleClients = make_set(DeviceName, 10), NamesSeen = make_set_if(tolower(RemoteUrl), isnotempty(RemoteUrl), 10),
                FirstSeenUtc = min(Timestamp), LastSeenUtc = max(Timestamp)
        by IP = Norm(RemoteIP)
    | where DistinctClientDevices >= ServingMinClients
    | where IP contains ":" or ipv4_is_private(IP)                  // mapped public IPs drop out here; IPv6 left are private (ULA)
    | where ipv4_is_in_any_range(IP, SubnetFilterSafe) or (IP contains ":" and array_length(SubnetFilter) == 0)
    | lookup kind=leftouter (IpOwner | project IP, MatchedDeviceId = OwnerDeviceId, OwnerName, OwnerOnboarding) on IP
    | where OwnerOnboarding != "Onboarded"
    | extend GapType = iff(isnotempty(MatchedDeviceId),
                           strcat("Discovered device serving connections, not onboarded (", OwnerOnboarding, ")"),
                           "IP serving connections, unknown to Defender: follow up in RunZero and VMware"),
             DeviceName = OwnerName, IPs = pack_array(IP)
    | project GapType, DeviceId = MatchedDeviceId, DeviceName, IPs, ServedPorts, DistinctClientDevices, SampleClients,
              NamesSeen, FirstSeenUtc, LastSeenUtc;
union DeviceGaps, IpGaps
| extend QueryId = "KQL-18", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by GapType asc, DistinctClientDevices desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| GapType | Inferred | One of three gap types (and the observed status inside it) |
| DeviceId | Observed (gap types 1, 2) / Inferred (gap type 3, IP match) | DeviceInfo, or the device that held the IP |
| DeviceName, HardwareUuid, OnboardingStatus, SensorHealthState, DeviceType, OSPlatform, LastSeenUtc | Observed | DeviceInfo (scope block) |
| Vendor, Model, DiscoverySources | Observed | DeviceInfo |
| IPs, MacAddresses | Observed | DeviceNetworkInfo, or the remote IP itself (gap type 3) |
| ServedPorts | Observed | RemotePort values connected to on this IP |
| DistinctClientDevices, SampleClients | Observed | Onboarded devices that connected to it |
| NamesSeen | Observed | RemoteUrl values used for this IP (helps identify it) |
| FirstSeenUtc, LastSeenUtc | Observed | Connection time span (gap type 3) |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Gap type 3 includes non-server things that accept connections: printers, appliances, load balancer VIPs and network devices. That's correct for a RunZero follow-up list, but they aren't all "servers to onboard".
- A device that never talks to an onboarded device, and that discovery never saw, is invisible here. Only VMware and RunZero can close that gap (CSV HST-14: "Unmatched VMs are a finding in themselves").
- HostFilter doesn't apply to gap type 3, because those IPs have no host name.

**Validation check**

Take ten IPs from gap type 3 and look them up in RunZero and the VMware Dataset 1 extract. Each should be a real device with no Defender agent. Any that turn out to be onboarded means the IP-to-device map is stale (DHCP or recent IP changes): check DeviceNetworkInfo for that IP.

### KQL-19: Coverage by subnet (feeds the workbook Coverage tab)

**CSV fields:** HST-14  
**Purpose:** Defender coverage per subnet: onboarded servers, servers discovered but not onboarded, and unknown IPs serving connections. From those it computes an estimated server coverage percentage. Joining the subnets to Netbox prefixes and sites turns this into the workbook's "Defender server coverage (%)" per post. Other onboarded and discovered devices are counted too, for workstation coverage.  
**Tables:** `DeviceInfo`, `DeviceNetworkInfo`, `DeviceNetworkEvents`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)
- [Kusto format_ipv4_mask()](https://learn.microsoft.com/kusto/query/format-ipv4-mask-function)

```kql
// KQL-19 | Coverage by subnet (feeds the workbook Coverage tab after a Netbox join)
// Fields: HST-14
// Grain: one row per IPv4 subnet of CoveragePrefixLength bits.
// Observed: none directly (every column is a count derived by joins and classification).
// Inferred: Subnet, OnboardedServers, OnboardedServersSensorNotActive, DiscoveredServersNotOnboarded, UnknownServingIPs,
//   OnboardedOtherDevices, DiscoveredOtherNotOnboarded, UnknownInitiatingOnlyIPs, EstimatedServerCoveragePct.
// The denominator is everything Defender knows of. The true denominator is RunZero plus VMware, so the percentage is an
// upper bound.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let CoveragePrefixLength = 24;   // set to your typical Netbox prefix length
let ServingPortMax = 49151;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let Known = DeviceLatest
    | project DeviceId, DeviceType, OSPlatform, DeviceCategory, OnboardingStatus, SensorHealthState
    | join kind=inner (DeviceIPs | where not(IP contains ":") | project DeviceId, IP) on DeviceId
    | where ipv4_is_in_any_range(IP, SubnetFilterSafe)
    | extend IsServer = isnotempty(ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)),
             Subnet = format_ipv4_mask(IP, CoveragePrefixLength)
    | summarize by Subnet, DeviceId, IsServer, OnboardingStatus, SensorHealthState;
let KnownIPs = DeviceIPs | distinct IP;
let Unknown = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where (ActionType in ("ConnectionSuccess", "ConnectionSuccessAggregatedReport") and RemotePort <= ServingPortMax)
         or ActionType == "InboundConnectionAccepted"
    | where RemoteIPType in~ ("Private", "FourToSixMapping")
    | summarize Roles = make_set(iff(ActionType == "InboundConnectionAccepted", "Initiating", "Serving"), 2) by IP = Norm(RemoteIP)
    | where not(IP contains ":") and ipv4_is_private(IP) and ipv4_is_in_any_range(IP, SubnetFilterSafe)
    | join kind=leftanti KnownIPs on IP
    | extend Subnet = format_ipv4_mask(IP, CoveragePrefixLength);
let PerSubnetKnown = Known
    | summarize OnboardedServers = dcountif(DeviceId, IsServer and OnboardingStatus == "Onboarded"),
                OnboardedServersSensorNotActive = dcountif(DeviceId, IsServer and OnboardingStatus == "Onboarded" and SensorHealthState != "Active"),
                DiscoveredServersNotOnboarded = dcountif(DeviceId, IsServer and OnboardingStatus != "Onboarded"),
                OnboardedOtherDevices = dcountif(DeviceId, not(IsServer) and OnboardingStatus == "Onboarded"),
                DiscoveredOtherNotOnboarded = dcountif(DeviceId, not(IsServer) and OnboardingStatus != "Onboarded")
        by Subnet;
let PerSubnetUnknown = Unknown
    | summarize UnknownServingIPs = dcountif(IP, set_has_element(Roles, "Serving")),
                UnknownInitiatingOnlyIPs = dcountif(IP, not(set_has_element(Roles, "Serving")))
        by Subnet;
PerSubnetKnown
| join kind=fullouter PerSubnetUnknown on Subnet
| extend Subnet = iff(isempty(Subnet), Subnet1, Subnet)
| project-away Subnet1
| extend OnboardedServers = coalesce(OnboardedServers, 0), OnboardedServersSensorNotActive = coalesce(OnboardedServersSensorNotActive, 0),
         DiscoveredServersNotOnboarded = coalesce(DiscoveredServersNotOnboarded, 0), OnboardedOtherDevices = coalesce(OnboardedOtherDevices, 0),
         DiscoveredOtherNotOnboarded = coalesce(DiscoveredOtherNotOnboarded, 0), UnknownServingIPs = coalesce(UnknownServingIPs, 0),
         UnknownInitiatingOnlyIPs = coalesce(UnknownInitiatingOnlyIPs, 0)
| extend ServerDenominator = OnboardedServers + DiscoveredServersNotOnboarded + UnknownServingIPs
| extend EstimatedServerCoveragePct = iff(ServerDenominator > 0, round(100.0 * OnboardedServers / ServerDenominator, 1), real(null))
| project Subnet, OnboardedServers, OnboardedServersSensorNotActive, DiscoveredServersNotOnboarded, UnknownServingIPs,
          EstimatedServerCoveragePct, OnboardedOtherDevices, DiscoveredOtherNotOnboarded, UnknownInitiatingOnlyIPs
| extend QueryId = "KQL-19", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by EstimatedServerCoveragePct asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| Subnet | Inferred | IPv4 subnet of CoveragePrefixLength bits (join to Netbox prefixes) |
| OnboardedServers, OnboardedServersSensorNotActive, DiscoveredServersNotOnboarded | Inferred | Distinct devices by class and status (classification is the scope rule) |
| UnknownServingIPs | Inferred | Private IPs onboarded devices connected to on a non-ephemeral port, with no Defender record |
| EstimatedServerCoveragePct | Inferred | OnboardedServers / (onboarded + discovered + unknown serving) |
| OnboardedOtherDevices, DiscoveredOtherNotOnboarded | Inferred | Non-server devices (mostly workstations) |
| UnknownInitiatingOnlyIPs | Inferred | Unknown IPs that only connected in to onboarded devices |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Fixed-length subnets won't match Netbox exactly. Re-aggregate by Netbox prefix and site in the workbook before quoting a per-post figure.
- UnknownServingIPs includes printers, appliances and VIPs, which pulls coverage down. RunZero's device type separates them.
- It scans 30 days of DeviceNetworkEvents across all devices. Slice the window if it times out.

**Validation check**

For one post where you know the server count (from VMware), add up `OnboardedServers + DiscoveredServersNotOnboarded` for that post's prefixes and compare. A large shortfall means Defender's discovery doesn't reach that post: record it in the workbook Coverage tab, and treat that post's Defender-derived fields as low confidence.

### KQL-20: Domain controllers and Defender for Identity coverage

**CSV fields:** HST-15, ATH-03  
**Purpose:** Finds domain controllers from the data, without a DC list, then checks whether Defender for Identity has events processed by each one. There are three DC signals: the device itself accepts Kerberos on port 88 (Microsoft's own sample method), several onboarded devices send Kerberos to the IP, or the device role says domain controller. A DC with no Defender for Identity events in the window probably has no working sensor. That's a silent gap in every authentication field for its site.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`, `IdentityLogonEvents`, `IdentityQueryEvents`, `IdentityDirectoryEvents`  
**Verified against (Microsoft Learn):**

- [Automatic attack disruption results (DC sample: LocalPort 88; Application == "Active Directory")](https://learn.microsoft.com/defender-xdr/autoad-results#track-the-actions-in-advanced-hunting)
- [Validate Defender for Identity sensor deployment (advanced hunting checks)](https://learn.microsoft.com/defender-for-identity/deploy/test-sensor#verify-data-in-advanced-hunting-tables)
- [Manage Defender for Identity sensors: status and health columns](https://learn.microsoft.com/defender-for-identity/sensor-settings#sensor-status-and-property-details)
- [Microsoft Graph sensor resource (healthStatus, serviceStatus)](https://learn.microsoft.com/graph/api/resources/security-sensor?view=graph-rest-1.0)
- [IdentityLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitylogonevents-table)
- [IdentityQueryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityqueryevents-table)
- [IdentityDirectoryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitydirectoryevents-table)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)

```kql
// KQL-20 | Domain controllers and Defender for Identity coverage
// Fields: HST-15 (inferred), ATH-03
// Grain: one row per DC (DeviceId when known, otherwise IP).
// Observed: IPs, KerberosClientDevices, KdcInboundClients, MdiProcessedEvents, MdiProcessedLastUtc, MdiProcessedSources,
//   MdiMentionedEvents, MdiMentionedLastUtc, MdiMentionedSources.
// Inferred: DeviceId, DeviceName, Onboarding, OSPlatform, OSVersion, SensorHealthState (the MDE sensor, not MDI),
//   HardwareUuid (all attached by IP-to-device match), Evidence (DC signals), MdiSensorInference.
// Sensor list and health are NOT in advanced hunting. They're in Settings > Identities > Sensors, and in Graph
//   GET /security/identities/sensors (healthStatus, serviceStatus). Confirm every "probable no sensor" row there.
// "Processed" = the DC appears in a Destination* column ("device running the server application that processed the
//   action"). "Mentioned" = it appears in a column Microsoft's sensor validation page checks (IdentityDirectoryEvents.
//   TargetDeviceName, IdentityQueryEvents.DeviceName); weaker, because another DC's sensor can see it.
// DeviceRoles JSON shape isn't documented: the role match is a text search for "domaincontroller" (check KQL-00a sample).
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let DcMinClients = 3;          // onboarded devices that must reach an IP on 88/TCP before it counts
let MdiStaleHours = 24;        // no processed MDI event for this long = stale
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let KdcListeners = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where LocalPort == 88 and ActionType in ("ListeningConnectionCreated", "InboundConnectionAccepted")
    | summarize KdcInboundClients = dcountif(RemoteIP, ActionType == "InboundConnectionAccepted") by DeviceId
    | join kind=inner (DeviceIPs | project DeviceId, IP) on DeviceId
    | project IP, KdcInboundClients;
let KerberosTargets = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where RemotePort == 88 and ActionType in ("ConnectionSuccess", "ConnectionSuccessAggregatedReport")
    | summarize KerberosClientDevices = dcount(DeviceId) by IP = Norm(RemoteIP)
    | where KerberosClientDevices >= DcMinClients;
let RoleDCs = DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(DeviceRoles)
    | summarize arg_max(Timestamp, DeviceRoles) by DeviceId
    | where replace_string(tolower(DeviceRoles), " ", "") contains "domaincontroller"
    | join kind=inner (DeviceIPs | project DeviceId, IP) on DeviceId
    | project IP;
let Candidates = union
        (KdcListeners | project IP, Evidence = "Accepts Kerberos (88) itself"),
        (KerberosTargets | project IP, Evidence = "Receives Kerberos (88) from onboarded devices"),
        (RoleDCs | project IP, Evidence = "Device role mentions domain controller")
    | where ipv4_is_in_any_range(IP, SubnetFilterSafe) or (IP contains ":" and array_length(SubnetFilter) == 0)
    | summarize Evidence = make_set(Evidence) by IP;
let MdiProcessed = union
        (IdentityLogonEvents | where Timestamp between (WindowStart .. WindowEnd)
         | where isnotempty(DestinationIPAddress)
         | summarize Events = count(), LastUtc = max(Timestamp) by IP = Norm(DestinationIPAddress)
         | extend Source = "IdentityLogonEvents.DestinationIPAddress"),
        (IdentityQueryEvents | where Timestamp between (WindowStart .. WindowEnd)
         | where isnotempty(DestinationIPAddress)
         | summarize Events = count(), LastUtc = max(Timestamp) by IP = Norm(DestinationIPAddress)
         | extend Source = "IdentityQueryEvents.DestinationIPAddress"),
        (IdentityDirectoryEvents | where Timestamp between (WindowStart .. WindowEnd)
         | where isnotempty(DestinationIPAddress)
         | summarize Events = count(), LastUtc = max(Timestamp) by IP = Norm(DestinationIPAddress)
         | extend Source = "IdentityDirectoryEvents.DestinationIPAddress")
    | summarize MdiProcessedEvents = sum(Events), MdiProcessedLastUtc = max(LastUtc), MdiProcessedSources = make_set(Source) by IP;
let MdiMentioned = union
        (IdentityDirectoryEvents | where Timestamp between (WindowStart .. WindowEnd)
         | where isnotempty(TargetDeviceName)
         | summarize Events = count(), LastUtc = max(Timestamp) by Name = tolower(tostring(split(TargetDeviceName, ".")[0]))
         | extend Source = "IdentityDirectoryEvents.TargetDeviceName"),
        (IdentityQueryEvents | where Timestamp between (WindowStart .. WindowEnd)
         | where isnotempty(DeviceName)
         | summarize Events = count(), LastUtc = max(Timestamp) by Name = tolower(tostring(split(DeviceName, ".")[0]))
         | extend Source = "IdentityQueryEvents.DeviceName")
    | summarize MdiMentionedEvents = sum(Events), MdiMentionedLastUtc = max(LastUtc), MdiMentionedSources = make_set(Source) by Name;
Candidates
| lookup kind=leftouter (IpOwner | project IP, DeviceId = OwnerDeviceId, DeviceName = OwnerName, Onboarding = OwnerOnboarding) on IP
| lookup kind=leftouter (DeviceLatest | project DeviceId, OSPlatform, OSVersion, SensorHealthState, HardwareUuid, HostShortName) on DeviceId
| lookup kind=leftouter KerberosTargets on IP
| lookup kind=leftouter (KdcListeners | summarize KdcInboundClients = max(KdcInboundClients) by IP) on IP
| lookup kind=leftouter MdiProcessed on IP
| summarize IPs = make_set(IP, 10), Evidence = make_set(Evidence, 5),
            KerberosClientDevices = max(KerberosClientDevices), KdcInboundClients = max(KdcInboundClients),
            MdiProcessedEvents = sum(MdiProcessedEvents), MdiProcessedLastUtc = max(MdiProcessedLastUtc),
            MdiProcessedSources = make_set(MdiProcessedSources, 5)
    by DcKey = iff(isnotempty(DeviceId), DeviceId, IP), DeviceId, DeviceName, Onboarding, OSPlatform, OSVersion,
       SensorHealthState, HardwareUuid, HostShortName
| lookup kind=leftouter MdiMentioned on $left.HostShortName == $right.Name
| extend MdiSensorInference = case(
             MdiProcessedEvents > 0 and MdiProcessedLastUtc >= WindowEnd - MdiStaleHours * 1h, "MDI events processed at this DC recently: sensor probably reporting",
             MdiProcessedEvents > 0, strcat("MDI events processed, but none in the last ", MdiStaleHours, " h: sensor stale or DC idle"),
             MdiMentionedEvents > 0, "Only mentioned in MDI data (seen by other DCs): probably no sensor on this DC",
             "No MDI events: probably no sensor or an unhealthy sensor (confirm in the portal or Graph)")
| project DeviceId, DeviceName, IPs, Evidence, Onboarding, OSPlatform, OSVersion, SensorHealthState, HardwareUuid,
          KerberosClientDevices, KdcInboundClients, MdiSensorInference, MdiProcessedEvents, MdiProcessedLastUtc,
          MdiProcessedSources, MdiMentionedEvents, MdiMentionedLastUtc, MdiMentionedSources
| extend QueryId = "KQL-20", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by MdiSensorInference asc, DeviceName asc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, Onboarding | Inferred | Device that held the DC IP (IP match) |
| IPs | Observed | DC IPs from the evidence |
| Evidence | Inferred | Which DC signals matched |
| OSPlatform, OSVersion, SensorHealthState, HardwareUuid | Inferred | DeviceInfo values attached by the IP match (SensorHealthState is the Defender for Endpoint sensor, not Defender for Identity) |
| KerberosClientDevices, KdcInboundClients | Observed | Distinct devices sending or delivering Kerberos (aggregate) |
| MdiSensorInference | Inferred | HST-15 inference from the presence and freshness of MDI events |
| MdiProcessedEvents, MdiProcessedLastUtc, MdiProcessedSources | Observed | Events whose Destination* column is this DC |
| MdiMentionedEvents, MdiMentionedLastUtc, MdiMentionedSources | Observed | Events that name this DC in other columns |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- Inference only. A DC can have a sensor that's installed but unhealthy and still produce some events, so health must come from the portal or Graph (`healthStatus`, `serviceStatus`).
- A read-only DC or a DC no onboarded device talks to in 30 days may be missed. Cross-check with the identity team's DC list (CSV HST-15 secondary).

**Validation check**

Export the sensor list (**Settings > Identities > Sensors**, or `Get-MgSecurityIdentitySensor` in Microsoft Graph PowerShell). Every DC listed there as healthy should show "sensor probably reporting" here. Every DC here that isn't in the sensor list is a confirmed HST-15 gap. A DC that is in the list but shows "No MDI events" means the inference columns are wrong for your tenant: tell me and I'll change the matching.

### KQL-21: Scope check: what the DeviceType and OS rule misses

**CSV fields:** HST-14, HST-10  
**Purpose:** Tests the server rule against behaviour. It flags non-server devices that accept inbound connections from many clients on a non-workstation port; those are probably servers the rule misses. It also flags devices classed as servers that accepted no inbound connections; those may be Linux workstations, idle servers, or hosts with missing telemetry.  
**Tables:** `DeviceInfo`, `DeviceNetworkEvents`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)

```kql
// KQL-21 | Scope check: what the DeviceType and OS rule misses
// Fields: HST-14, HST-10
// Grain: one row per DeviceId.
// Observed: DeviceId, DeviceName, DeviceType, OSPlatform, OSDistribution, OnboardingStatus, ClientVersion, ServingPorts,
//   MaxInboundClients.
// Inferred: MissReason.
// WorkstationNoisePorts are ports workstations commonly accept connections on. Edit the list for your estate.
// Unconfirmed on Learn: ActionType "InboundConnectionAccepted". If KQL-00b shows none for Linux, every Linux row here
//   is "no telemetry", not "idle".
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
let MinInboundClients = 5;   // distinct remote IPs on one port before a non-server looks like it's serving
let WorkstationNoisePorts = dynamic([135, 137, 138, 139, 445, 1900, 3389, 5040, 5353, 5355, 5357, 5985, 7680]);
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
let NonServers = DeviceLatest
    | where OnboardingStatus == "Onboarded"
    | where isempty(ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus))
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | project DeviceId;
let ServingNonServers = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "InboundConnectionAccepted"
    | where DeviceId in (NonServers)
    | where LocalPort < 49152 and not(LocalPort in (WorkstationNoisePorts))
    | summarize InboundClients = dcount(RemoteIP) by DeviceId, LocalPort, ProcessName = InitiatingProcessFileName
    | where InboundClients >= MinInboundClients
    | summarize ServingPorts = make_set(strcat(LocalPort, "/", ProcessName, " (", InboundClients, " clients)"), 20),
                MaxInboundClients = max(InboundClients) by DeviceId
    | extend MissReason = "Not classed as a server, but accepts inbound connections from many clients: probable server";
let InboundServers = DeviceNetworkEvents
    | where Timestamp between (WindowStart .. WindowEnd)
    | where ActionType == "InboundConnectionAccepted"
    | where DeviceId in (OnboardedServerIds)
    | distinct DeviceId;
let QuietServers = Servers
    | where OnboardingStatus == "Onboarded"
    | join kind=leftanti InboundServers on DeviceId
    | project DeviceId
    | extend MissReason = "Classed as a server, but no inbound connections recorded: Linux workstation, idle host, or missing telemetry";
union ServingNonServers, QuietServers
| lookup kind=leftouter (DeviceLatest | project DeviceId, DeviceName, DeviceType, OSPlatform, OSDistribution, OnboardingStatus, ClientVersion) on DeviceId
| project DeviceId, DeviceName, DeviceType, OSPlatform, OSDistribution, OnboardingStatus, ClientVersion, MissReason,
          ServingPorts, MaxInboundClients
| extend QueryId = "KQL-21", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by MissReason asc, MaxInboundClients desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, DeviceType, OSPlatform, OSDistribution, OnboardingStatus, ClientVersion | Observed | DeviceInfo |
| MissReason | Inferred | Why the device might be mis-scoped |
| ServingPorts | Inferred | Formatted port/process/client-count strings built from observed values |
| MaxInboundClients | Observed | Most distinct remote IPs on any one port (aggregate) |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- A workstation serving one or two clients (a dongle licence server, a lab instrument PC) falls below the threshold. Lower `MinInboundClients` to 1 for a one-off sweep.
- Discovered (not onboarded) workstations can't be tested, because they record no events.

**Validation check**

Every "probable server" row should be checked with the device owner. If any is confirmed, add it to the scope by name (`HostFilter` can't add devices, so record the exception and include it in VMware and RunZero follow-up). If more than a handful are confirmed, ask me to add a "behaviour" branch to the SCOPE BLOCK.

### KQL-22: Listener completeness test

**CSV fields:** CMP-03  
**Purpose:** Tests the caveat that listening-port events are only recorded when a listener opens, so long-running services fall outside the window. For each endpoint it classifies the evidence: both listener and inbound, listener only, or inbound only. Inbound-only endpoints are split by whether the process started before the window (the expected gap) or inside it (a telemetry gap).  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)

```kql
// KQL-22 | Listener completeness test
// Fields: CMP-03 (caveat test)
// Grain: one row per DeviceId + LocalPort + Protocol + process.
// Observed: DeviceId, DeviceName, OSPlatform, ClientVersion, LocalPort, Protocol, ProcessName, ListenerEvents, InboundEvents,
//   FirstListenerUtc, FirstInboundUtc, ProcessStartUtc.
// Inferred: Category.
// For a per-platform summary, uncomment the last line.
// Unconfirmed on Learn: ActionType "InboundConnectionAccepted".
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceNetworkEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where DeviceId in (OnboardedServerIds)
| where ActionType in ("ListeningConnectionCreated", "InboundConnectionAccepted")
| where not(Norm(LocalIP) in ("127.0.0.1", "::1"))
| summarize ListenerEvents = countif(ActionType == "ListeningConnectionCreated"),
            InboundEvents = countif(ActionType == "InboundConnectionAccepted"),
            FirstListenerUtc = minif(Timestamp, ActionType == "ListeningConnectionCreated"),
            FirstInboundUtc = minif(Timestamp, ActionType == "InboundConnectionAccepted"),
            ProcessStartUtc = min(InitiatingProcessCreationTime)
    by DeviceId, DeviceName, LocalPort, Protocol, ProcessName = InitiatingProcessFileName
| lookup kind=leftouter (Servers | project DeviceId, OSPlatform, ClientVersion) on DeviceId
| extend Category = case(
             ListenerEvents > 0 and InboundEvents > 0, "A: listener and inbound both recorded",
             ListenerEvents > 0, "B: listener only (no inbound in window)",
             InboundEvents > 0 and ProcessStartUtc < WindowStart, "C: inbound only; process started before window (expected listener gap)",
             InboundEvents > 0 and isnull(ProcessStartUtc), "D: inbound only; process start time not recorded",
             InboundEvents > 0, "E: inbound only; process started inside window but no listener event (telemetry gap)",
             "Other")
| project DeviceId, DeviceName, OSPlatform, ClientVersion, LocalPort, Protocol, ProcessName, Category,
          ListenerEvents, InboundEvents, FirstListenerUtc, FirstInboundUtc, ProcessStartUtc
| extend QueryId = "KQL-22", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by Category asc, DeviceName asc
// | summarize Endpoints = count(), Servers = dcount(DeviceId) by OSPlatform, Category
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| DeviceId, DeviceName, OSPlatform, ClientVersion | Observed | Scope block |
| LocalPort, Protocol, ProcessName | Observed | DeviceNetworkEvents |
| Category | Inferred | A to E evidence classes |
| ListenerEvents, InboundEvents | Observed | Events recorded |
| FirstListenerUtc, FirstInboundUtc, ProcessStartUtc | Observed | Earliest listener event, earliest inbound event, earliest InitiatingProcessCreationTime |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- It can't see the one case that matters most: a long-running listener with no clients in the window produces no events at all. Only the host test below (or RunZero's port scan) finds those.

**Validation check**

**How to test against a server whose ports you know.**

1. Pick a Windows server (for example a DC) and a Linux server on agent 101.26091.0004 or later. Record their listening ports with `Get-NetTCPConnection -State Listen` and `ss -ltnup`.
2. Run this query with `HostFilter` set to both. Each known port should be in category A, B or C. A known port that's absent is a long-running listener with no clients (the blind spot), so record it.
3. Restart one service that listens on a fixed port (for example the print spooler, or a test web server) and wait 30 minutes. Re-run: that port should move to category A or B with a `FirstListenerUtc` after the restart. If it doesn't, listener events aren't being recorded for that platform at all.
4. Summarise: the share of known ports found in category C plus the absent ones is your measured listener gap. Record it in the decision log as the CMP-03 confidence figure.

### KQL-23: Connection completeness and RemoteUrl fill rate

**CSV fields:** ACC-01, ACC-02, TMP-03, DEP-01  
**Purpose:** Tests what an event count means. Learn says Defender for Endpoint limits data collection to reduce noise, so this query measures how many events one client, process and endpoint pair produces per day, by port and client OS. A median near 1 means counts show presence, not volume. It also measures how often RemoteUrl is filled per port, which is the baseline for ACC-02.  
**Tables:** `DeviceNetworkEvents`, `DeviceInfo`, `DeviceNetworkInfo`  
**Verified against (Microsoft Learn):**

- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [Investigate connection events behind forward proxies (ConnectionSuccess)](https://learn.microsoft.com/defender-endpoint/investigate-behind-proxy)

```kql
// KQL-23 | Connection completeness and RemoteUrl fill rate
// Fields: ACC-01, ACC-02 (baseline), TMP-03, DEP-01 (what counts mean)
// Grain: one row per RemotePort + client OSPlatform.
// Observed: RemotePort, ClientOS, PairDays, Pairs, EventsTotal, EventsWithUrl, MedianEventsPerPairDay, P90EventsPerPairDay,
//   MaxEventsPerPairDay, PairDaysWithOneEvent.
// Inferred: UrlFillPct, PctPairDaysWithOneEvent, CountInterpretation.
// Learn (aggregated reporting): "Defender for Endpoint reduces noise in collected data ... It limits data collection".
// Aggregated reports (hourly roll-ups) are excluded here so they don't distort per-day counts.
// ---- Parameters (defaults = everything; narrow later without rewriting) ----
let Lookback     = 30d;            // native Defender data stops at 30 days
let WindowEnd    = now();          // set to ago(7d), ago(14d)... to run the window in slices
let HostFilter   = dynamic([]);    // optional host names, short or FQDN: dynamic(["srv01", "srv02.corp.example"])
let SubnetFilter = dynamic([]);    // optional IPv4 CIDRs: dynamic(["10.20.0.0/16"])
let WindowStart  = WindowEnd - Lookback;
// ==== SCOPE BLOCK v2: identical in every query that uses it; KQL-01 explains it ====
let Norm = (ip: string) { iff(ip startswith "::ffff:", substring(ip, 7), ip) };   // IPv4-mapped IPv6 -> IPv4
// RemoteIPType gives ::ffff:a.b.c.d its own type (FourToSixMapping), so private/public/loopback are decided here:
let IsPrivateIp = (ipType: string, ip: string) {
    ipType =~ "Private" or (ipType =~ "FourToSixMapping" and ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip))) };
let IsPublicIp = (ipType: string, ip: string) {
    ipType =~ "Public" or (ipType =~ "FourToSixMapping"
        and not(ipv4_is_private(iff(ip startswith "::ffff:", substring(ip, 7), ip)))
        and not(iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127.")) };
let IsLoopbackIp = (ipType: string, ip: string) {
    ipType =~ "Loopback" or ip == "::1" or iff(ip startswith "::ffff:", substring(ip, 7), ip) startswith "127." };
// The server rule. Linux counts only when it's an endpoint-category device or onboarded (keeps out Linux appliances).
let ServerRule = (dt: string, os: string, cat: string, onb: string) {
    case(dt =~ "Server", "DeviceType is Server",
         os startswith "WindowsServer", "OSPlatform is Windows Server",
         os =~ "Linux" and (cat =~ "Endpoint" or onb == "Onboarded"), "OSPlatform is Linux endpoint (assumed server)",
         "") };
let SubnetFilterSafe = iff(array_length(SubnetFilter) == 0, dynamic(["0.0.0.0/0"]), SubnetFilter);
let DeviceLatest = materialize(
    DeviceInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(OSPlatform) or isnotempty(DeviceType)
    | summarize arg_max(Timestamp, DeviceName, OSPlatform, OSDistribution, OSVersion, DeviceType, DeviceCategory,
                        OnboardingStatus, SensorHealthState, HardwareUuid, MergedToDeviceId, ClientVersion)
        by DeviceId
    | where isempty(MergedToDeviceId)
    | extend LastSeenUtc = Timestamp, HostShortName = tolower(tostring(split(DeviceName, ".")[0]))
    | project-away Timestamp, MergedToDeviceId);
let DeviceIPs = materialize(                         // every IP each adapter held in the window, not only the latest
    DeviceNetworkInfo
    | where Timestamp between (WindowStart .. WindowEnd)
    | where isnotempty(IPAddresses) and IPAddresses != "[]"
    | summarize NicSeenUtc = max(Timestamp) by DeviceId, NetworkAdapterName, MacAddress, IPAddresses
    | mv-expand IpObj = parse_json(IPAddresses)
    | extend IP = tostring(IpObj.IPAddress)            // key name not on the schema page: confirm with KQL-00a
    | where isnotempty(IP) and not(IP startswith "127.") and IP != "::1"
        and not(IP startswith "169.254.") and not(tolower(IP) startswith "fe80")
    | summarize NicSeenUtc = max(NicSeenUtc) by DeviceId, NetworkAdapterName, MacAddress, IP);
let IpOwner = materialize(                           // latest holder of each IP; IpOwnerCount = devices that held it in the window
    DeviceIPs
    | summarize arg_max(NicSeenUtc, DeviceId) by IP
    | lookup kind=leftouter (DeviceIPs | summarize IpOwnerCount = dcount(DeviceId) by IP) on IP
    | lookup kind=leftouter (DeviceLatest
        | project DeviceId, OwnerName = DeviceName, OwnerType = DeviceType, OwnerOS = OSPlatform,
                  OwnerOnboarding = OnboardingStatus) on DeviceId
    | project IP, OwnerDeviceId = DeviceId, OwnerName, OwnerType, OwnerOS, OwnerOnboarding, IpOwnerCount);
let Servers = materialize(
    DeviceLatest
    | extend ServerReason = ServerRule(DeviceType, OSPlatform, DeviceCategory, OnboardingStatus)
    | where isnotempty(ServerReason)
    | where array_length(HostFilter) == 0 or DeviceName in~ (HostFilter) or HostShortName in~ (HostFilter)
    | lookup kind=leftouter (DeviceIPs
        | summarize InSubnet = countif(ipv4_is_in_any_range(IP, SubnetFilterSafe)) by DeviceId) on DeviceId
    | where array_length(SubnetFilter) == 0 or InSubnet > 0
    | project-away InSubnet);
let OnboardedServerIds = Servers | where OnboardingStatus == "Onboarded" | project DeviceId;
let ServerIPs = materialize(
    Servers
    | project DeviceId, ServerDeviceName = DeviceName, ServerOnboarding = OnboardingStatus
    | join kind=inner (DeviceIPs | summarize ServerMac = take_any(MacAddress) by DeviceId, IP) on DeviceId
    | project ServerDeviceId = DeviceId, ServerDeviceName, ServerOnboarding, ServerIP = IP, ServerMac);
let ServerIpName = ServerIPs                         // one name per server IP; ServerIpHolders > 1 = several servers share the IP
    | summarize arg_max(ServerDeviceId, ServerDeviceName, ServerOnboarding) by ServerIP
    | lookup kind=leftouter (ServerIPs | summarize ServerIpHolders = dcount(ServerDeviceId) by ServerIP) on ServerIP;
let ServerIPMatch = toscalar(
    ServerIPs
    | project Ip = ServerIP
    | union (ServerIPs | where not(ServerIP contains ":") | project Ip = strcat("::ffff:", ServerIP))
    | summarize make_set(Ip));
// ==== END SCOPE BLOCK ====
DeviceNetworkEvents
| where Timestamp between (WindowStart .. WindowEnd)
| where ActionType == "ConnectionSuccess"
| where RemoteIP in (ServerIPMatch)
| summarize Events = count(), WithUrl = countif(isnotempty(RemoteUrl))
    by DeviceId, InitiatingProcessFileName, RemoteIP, RemotePort, Day = startofday(Timestamp)
| lookup kind=leftouter (DeviceLatest | project DeviceId, ClientOS = OSPlatform) on DeviceId
| summarize PairDays = count(), Pairs = dcount(strcat(DeviceId, "|", InitiatingProcessFileName, "|", RemoteIP)),
            EventsTotal = sum(Events), EventsWithUrl = sum(WithUrl),
            MedianEventsPerPairDay = percentile(Events, 50), P90EventsPerPairDay = percentile(Events, 90),
            MaxEventsPerPairDay = max(Events), PairDaysWithOneEvent = countif(Events == 1)
    by RemotePort, ClientOS
| extend UrlFillPct = round(100.0 * EventsWithUrl / EventsTotal, 1),
         PctPairDaysWithOneEvent = round(100.0 * PairDaysWithOneEvent / PairDays, 1),
         CountInterpretation = case(
             P90EventsPerPairDay <= 2, "Looks de-duplicated: counts mean presence, not volume",
             MedianEventsPerPairDay >= 50, "High per-day counts: closer to volume, but still not guaranteed complete",
             "Partial: treat counts as a lower bound, not volume")
| extend QueryId = "KQL-23", ObservationStartUtc = WindowStart, ObservationEndUtc = WindowEnd
| order by PairDays desc
```

**Output columns**

| Column | Evidence | Meaning |
|---|---|---|
| RemotePort, ClientOS | Observed | DeviceNetworkEvents RemotePort; DeviceInfo OSPlatform of the client |
| PairDays, Pairs | Observed | Client-process-endpoint pairs, and pair-days, with at least one event (aggregate) |
| EventsTotal, EventsWithUrl | Observed | Events recorded |
| MedianEventsPerPairDay, P90EventsPerPairDay, MaxEventsPerPairDay, PairDaysWithOneEvent | Observed | Distribution of events per pair per day (aggregate) |
| UrlFillPct | Inferred | Share of events with RemoteUrl: the ACC-02 baseline for this port |
| PctPairDaysWithOneEvent | Inferred | Share of pair-days with a single event |
| CountInterpretation | Inferred | What a count means on this port |
| QueryId, ObservationStartUtc, ObservationEndUtc | Provenance | Query ID and window |

**Known gaps**

- It shows the shape of the data, not the true connection rate. Only the controlled test below gives a ground truth.
- No byte counts exist in Defender (CSV ACC-11), so traffic volume needs flow data from the network team.

**Validation check**

**Controlled test (gives the ground truth).** On one onboarded test client, open 100 separate TCP connections to one server port over 10 minutes. In PowerShell: `1..100 | % { $c = New-Object Net.Sockets.TcpClient('server', 443); $c.Close(); Start-Sleep 6 }`. After an hour, count `ConnectionSuccess` events for that client, process (`powershell.exe`) and endpoint. Fewer than 100 confirms de-duplication or throttling; record the ratio in the decision log. If aggregated reporting is on (KQL-00a), also read `uniqueEventsAggregated` from `AdditionalFields` on the `ConnectionSuccessAggregatedReport` rows for the same hour.

## Fields that need a non-Defender source

| Field | Why Defender can't supply it | Use instead |
|---|---|---|
| HST-15 | Sensor list and health aren't in advanced hunting; KQL-20 can only infer them. | Defender portal **Settings > Identities > Sensors**, or Graph `GET /security/identities/sensors` (`healthStatus`, `serviceStatus`). Same vendor, different interface. |
| HST-13 | Defender knows last seen within 30 days, not power state. | VMware power state and last power-on. |
| HST-08 | Vendor and Model are only filled when discovery has enough information. | VMware inventory; RunZero asset type. |
| CMP-07 | No share-level events; only who connects to 445. | Splunk 5140/5145 (if auditing is on), or `Get-SmbShare` per server. |
| CMP-08 | DeviceTvmCertificateInfo has no SAN column and no port binding. | RunZero TLS certificate details per IP and port. |
| ACC-04 | Protocol is inferred from the port number. | RunZero service fingerprints, especially for non-standard ports. |
| ACC-05 | No protocol-version field, so SMBv1 and NTLMv1 can't be seen. | RunZero fingerprints; Splunk DC 4624 (`LmPackageName`). |
| ACC-06 | Defender gives the client IP, not the site. | Netbox prefix-to-site mapping. |
| ACC-07 | Only patterns that suggest a path (tunnel adapters, RDS, connector fan-out). | ZPA logs (application segment, App Connector, user); VPN logs. |
| ACC-11 | No byte counts; event counts aren't volume (KQL-23). | Flow data from the network team; ZPA bytes for ZPA users. |
| ATH-01 | Forms, SQL, API-key and local application auth leave no Windows or Entra trace. | App owner; Splunk IIS and application logs. |
| ATH-03 | User-to-DC evidence needs the client side or DC logs. | Splunk DC events 4624 and 4769. |
| ATH-04 | Only delegation use is visible, not SPNs or delegation settings. | Directory query by the identity team; Splunk 4769 ServiceName. |
| ATH-05 | Command-line hints only; config files and the registry aren't covered. | Owner configuration review. |
| TMP-01 / TMP-02 | 30-day window: misses quarterly, half-yearly and EOFY (30 June) work. | Splunk (longest retention) and owner confirmation; rerun KQL-16 and KQL-17 the week of 7 Dec 2026. |
| DEP-03 | Partner allow-lists of a post's public egress IP aren't visible. | Partner or vendor confirmation; KQL-02 `PublicIP` shows today's egress IP. |
| APP-07 | Application grain needs App IDs. | Application Inventory and owner. |

## Sources

Every page below was opened and checked on 9 November 2026.

- [Advanced hunting overview: quotas and usage parameters](https://learn.microsoft.com/defender-xdr/advanced-hunting-overview#quotas-and-usage-parameters)
- [Advanced hunting API: quotas](https://learn.microsoft.com/defender-xdr/api-advanced-hunting)
- [Advanced hunting query best practices](https://learn.microsoft.com/defender-xdr/advanced-hunting-best-practices)
- [Understand the advanced hunting schema (in-portal schema reference)](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-tables)
- [Advanced hunting schema naming changes](https://learn.microsoft.com/defender-xdr/advanced-hunting-schema-changes)
- [Advanced hunting custom functions](https://learn.microsoft.com/defender-xdr/advanced-hunting-custom-functions)
- [DeviceInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceinfo-table)
- [DeviceNetworkInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkinfo-table)
- [DeviceNetworkEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicenetworkevents-table)
- [DeviceProcessEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceprocessevents-table)
- [DeviceLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicelogonevents-table)
- [DeviceEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-deviceevents-table)
- [DeviceTvmSoftwareInventory table](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmsoftwareinventory-table)
- [DeviceTvmCertificateInfo table (preview)](https://learn.microsoft.com/defender-xdr/advanced-hunting-devicetvmcertificateinfo-table)
- [IdentityLogonEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitylogonevents-table)
- [IdentityDirectoryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identitydirectoryevents-table)
- [IdentityQueryEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityqueryevents-table)
- [IdentityInfo table](https://learn.microsoft.com/defender-xdr/advanced-hunting-identityinfo-table)
- [EntraIdSignInEvents table](https://learn.microsoft.com/defender-xdr/advanced-hunting-entraidsigninevents-table)
- [AADSignInEventsBeta (deprecated)](https://learn.microsoft.com/defender-xdr/advanced-hunting-aadsignineventsbeta-table)
- [SeenBy() function](https://learn.microsoft.com/defender-xdr/advanced-hunting-seenby-function)
- [Automatic attack disruption results (DC sample: LocalPort 88; Application == "Active Directory")](https://learn.microsoft.com/defender-xdr/autoad-results#track-the-actions-in-advanced-hunting)
- [Aggregated reporting in Defender for Endpoint](https://learn.microsoft.com/defender-endpoint/aggregated-reporting)
- [Review and assess discovered devices (OnboardingStatus values; ConnectionAttempt and ConnectionAcknowledged)](https://learn.microsoft.com/defender-endpoint/assess-devices)
- [Device inventory field reference](https://learn.microsoft.com/defender-endpoint/device-inventory-field-reference)
- [Investigate devices: internet-facing devices, InboundInternetScanInspected, TCP-only process data](https://learn.microsoft.com/defender-endpoint/investigate-machines#investigate-internet-facing-devices)
- [Investigate connection events behind forward proxies (ConnectionSuccess)](https://learn.microsoft.com/defender-endpoint/investigate-behind-proxy)
- [Network protection and the TCP three-way handshake](https://learn.microsoft.com/defender-endpoint/network-protection#network-protection-and-the-tcp-three-way-handshake)
- [Defender for Endpoint release notes: Linux (ListeningConnectionCreated from 101.26091.0004)](https://learn.microsoft.com/defender-endpoint/microsoft-defender-endpoint-releases#linux-releases)
- [Validate Defender for Identity sensor deployment (advanced hunting checks)](https://learn.microsoft.com/defender-for-identity/deploy/test-sensor#verify-data-in-advanced-hunting-tables)
- [Manage Defender for Identity sensors: status and health columns](https://learn.microsoft.com/defender-for-identity/sensor-settings#sensor-status-and-property-details)
- [Microsoft Graph sensor resource (healthStatus, serviceStatus)](https://learn.microsoft.com/graph/api/resources/security-sensor?view=graph-rest-1.0)
- [Kusto ipv4_is_in_any_range()](https://learn.microsoft.com/kusto/query/ipv4-is-in-any-range-function)
- [Kusto format_ipv4_mask()](https://learn.microsoft.com/kusto/query/format-ipv4-mask-function)
- [Kusto make_set() (flattens array inputs)](https://learn.microsoft.com/kusto/query/make-set-aggregation-function)
- [Kusto mv-expand (empty arrays produce no row)](https://learn.microsoft.com/kusto/query/mv-expand-operator)
- [Kusto lookup operator (right side is broadcast; size limit)](https://learn.microsoft.com/kusto/query/lookup-operator)
