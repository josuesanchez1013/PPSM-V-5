# PPSM Builder — v3.7

A PPSM register has to say, for every service an enclave uses: what protocol and port, on which system, crossing which boundary, permitted by what, and why it needs to exist. That data is scattered across hosts, firewalls, switches, scanners and DNS, in a dozen formats that do not agree with each other.

This package collects it, cross-references it, and turns it into the **eMASS PPSM Boundary Information Import Template** (`allow.xlsx`), ready to submit.

Everything runs locally. The builder is one HTML file opened in a browser; the collectors are plain scripts. Nothing is installed, nothing is uploaded, there is no network call of any kind. It is built for standalone and air-gapped systems.

---

## Contents

1. [What is in the package](#1-what-is-in-the-package)
2. [Quick start](#2-quick-start)
3. [Supported platforms](#3-supported-platforms)
4. [The whole job — scripted](#4-the-whole-job--scripted)
5. [The whole job — manual](#5-the-whole-job--manual)
6. [What the collectors produce](#6-what-the-collectors-produce)
7. [Every file the builder reads](#7-every-file-the-builder-reads)
8. [How the sources fill each other's gaps](#8-how-the-sources-fill-each-others-gaps)
9. [Hostnames and addresses](#9-hostnames-and-addresses)
10. [Nessus / ACAS scans and the software inventory](#10-nessus--acas-scans-and-the-software-inventory)
11. [Services and application names](#11-services-and-application-names)
12. [Boundaries](#12-boundaries)
13. [What is left for you](#13-what-is-left-for-you)
14. [Dedupe and consolidation](#14-dedupe-and-consolidation)
15. [Review checks and rule coverage](#15-review-checks-and-rule-coverage)
16. [Compare with the previous register](#16-compare-with-the-previous-register)
17. [The eMASS export](#17-the-emass-export)
18. [Other exports](#18-other-exports)
19. [Settings](#19-settings)
20. [Troubleshooting](#20-troubleshooting)
21. [A few things worth remembering](#21-a-few-things-worth-remembering)

---

## 1. What is in the package

| File | What it is |
|---|---|
| `ppsm-builder.html` | The tool. Copy it anywhere local and open it in a browser. 329 built-in self-tests under the **Self-Test** button. |
| `ppsm-collect.ps1` | Windows collector. Reaches hosts over WinRM in parallel, exports AD and DNS where the modules exist, and can pull firewall/switch output over SSH. Windows PowerShell 3.0–5.1 and PowerShell 7. |
| `ppsm-collect.sh` | Linux collector (bash, RHEL 8/9 as shipped). |
| `ppsm.yml` | Ansible playbook for a whole fleet: Linux and Windows hosts, Cisco firewalls and switches. |
| `CHANGELOG.md` | What changed in each version. |
| `README.md` | This guide. |

**Sample inputs** — invented data, safe to drop on the builder to see it work:

| File | What it shows |
|---|---|
| `dns-manager-export.csv` | a DNS Manager right-click export |
| `dnscmd-enum.txt` | `dnscmd /enumrecords` output |
| `dns-export.xlsx` | a DNS export saved as Excel |
| `dns-showconn-names.txt` | ASA `show conn` with object names instead of addresses |
| `ppsm-hostnames.csv` | a hostname-to-address list |
| `register-mixed.csv` | a register kept by hand, hostnames in the address columns |

Available separately (not in this package): `ppsm-discover.ps1`, which sweeps your address ranges and writes the `hosts.txt` that `ppsm-collect.ps1 -ComputerListFile` reads.

---

## 2. Quick start

**1. Collect**

```powershell
# Windows — one domain-joined machine reaches the rest over WinRM
.\ppsm-collect.ps1 -ComputerListFile hosts.txt

# ...and once on a domain controller: that run exports every computer
# account and every DNS zone
.\ppsm-collect.ps1
```
```bash
# Linux
sudo ./ppsm-collect.sh -H hosts.txt
```
```bash
# Whole fleet
ansible-playbook -i inventory.ini ppsm.yml
```
Firewall: `show conn all`, `show running-config access-list`, `show running-config object-group` into `ppsm-results\network\` — or let the collector fetch them with `-NetworkDevices fw01:asa -NetworkUser <user>`. Export the ACAS scan as `.nessus` into `ppsm-results\`.

**2. Build** — open `ppsm-builder.html`. Under Settings set the enclave location, the DNS domain, and Internal ranges if the enclave is not RFC1918. Drag the whole `ppsm-results` folder onto the Ingest box. Press **Finish Register**.

**3. Review** — resolve any hostnames the Endpoints panel says have no address. Run the review checks and clear the *Must fix* items.

**4. Submit** — **Export → eMASS Import (allow.xlsx)**. **Save Session** before closing the tab.

---

## 3. Supported platforms

### The builder
Any current browser opened from a local file (`file://`). Chrome, Edge and Firefox on Windows 10/11; Firefox or Chromium on RHEL 8/9. No plugins, no server, no internet.

### Windows collection — `ppsm-collect.ps1`
Windows PowerShell 3.0–5.1 and PowerShell 7, on the collecting machine and on the targets; they may differ. Each target checks its own `$PSVersionTable` and decides:

| Choice | How it is made |
|---|---|
| Ports | `Get-NetTCPConnection` / `Get-NetUDPEndpoint` where the NetTCPIP module exists (Windows 8 / Server 2012+); otherwise `netstat -ano` is parsed. Older builds, locked-down images and PowerShell without NetTCPIP all take the netstat path. |
| OS name | `Get-CimInstance` on 3.0+, `Get-WmiObject` below. |
| Encoding | 5.1 writes UTF-8 with a BOM; 7 is told to (`utf8BOM`) so Excel opens both the same way. |

The shell version and method used are written per host: on the console, in `READ-ME.txt` and in a `Collector` column of each connections CSV. The builder shows it in the ingest log.

No modules are required. RSAT (ActiveDirectory) adds the AD export and DnsServer adds the DNS zones; both are skipped cleanly when absent. The Windows OpenSSH client (built in since Windows 10 1809 / Server 2019) is used for `-NetworkDevices`.

### Linux collection — `ppsm-collect.sh`
bash 4.4+ (RHEL 8) and 5.x (RHEL 9). Uses `ss` and `ip` from iproute2, present by default. Nothing depends on net-tools.

### Fleet collection — `ppsm.yml`
ansible-core 2.12+ plus the `ansible.windows` and `ansible.netcommon` collections and the `cisco.asa` / `cisco.ios` platforms you use. All are bundled in the full `ansible` package; on an air-gapped control node install them from downloaded tarballs with `ansible-galaxy collection install <file>.tar.gz`.

### Encodings
UTF-8 with or without a BOM and UTF-16 (a PowerShell `>` redirect) are all read. Nothing needs converting.

---

## 4. The whole job — scripted

Both the scripted and the manual path produce the same `ppsm-results` folder and the builder reads it the same way. Use scripted unless scripts are not permitted on the system.

Every step is tagged with the box you run it on:

| Tag | Where |
|---|---|
| `[WINDOWS HOST]` | PowerShell, Run as Administrator |
| `[LINUX HOST]` | a shell, with sudo |
| `[DOMAIN CONTROLLER]` | any machine with the RSAT / DnsServer modules |
| `[FIREWALL]` | SSH to the management address, or the FTD CLI |
| `[SWITCH]` | SSH to the switch management address |
| `[FMC CONSOLE]` | the FMC web console |
| `[ANSIBLE NODE]` | wherever `ansible-playbook` runs |
| `[YOUR WORKSTATION]` | your own desktop |

Run host commands elevated. Without Administrator or sudo the process behind each port is missing, and that is the field that proves a port is a real listening service rather than a passing connection.

### A. Windows hosts *(skip the group if the enclave has none)*

**1.** `[WINDOWS HOST]` one domain-joined machine
```powershell
cd C:\ppsm
.\ppsm-collect.ps1 -ComputerListFile hosts.txt
```
`hosts.txt` is one name per line; `#` lines are ignored. Hosts are probed on the WinRM port all at once; anything that does not answer within 5 seconds is skipped and listed as unreachable, and the reachable ones are collected 32 at a time. A host that answers but stalls is dropped after `-ConnectTimeout` (20 s) / `-HostTimeout` (180 s). The run never waits on one host.

| Option | Use |
|---|---|
| `-ThrottleLimit 64` | more hosts at once on a big fleet |
| `-ProbeTimeout 3` | faster skip on a quiet network |
| `-UseSSL` | WinRM over 5986 |
| `-SkipProbe` | go straight to WinRM |
| `-IncludeEstablished:$false` | listeners only |
| `-Credential (Get-Credential)` | run as another account |

**2.** `[DOMAIN CONTROLLER]` run the collector once on a DC
```powershell
.\ppsm-collect.ps1
```
A DC normally has both the ActiveDirectory and DnsServer modules, so this one run exports every computer account **and** every DNS zone, forward and reverse. Check that `inventory\ad-computers.csv` and `inventory\dns-records.csv` both came out.

Before trusting a fleet run on PowerShell 5.1 hosts, run this once on one of them:
```powershell
.\ppsm-collect.ps1 -SelfTest
```
It collects that machine the normal way and forced through the netstat fallback, compares the two listener lists, and writes `ppsm-selftest\SELFTEST.txt` with a PASS/CHECK verdict plus both CSVs to drop on the builder.

### B. Linux hosts *(skip the group if the enclave has none)*

**3.** `[LINUX HOST]` one host, or the others over SSH in the same run
```bash
cd /opt/ppsm
sudo ./ppsm-collect.sh
sudo ./ppsm-collect.sh -H hosts.txt -u svc-ppsm
```

**4.** `[ANSIBLE NODE]` or the whole fleet at once
```bash
ansible-playbook -i inventory.ini ppsm.yml
```
Replaces steps 1, 3, 5 and 6 if your inventory covers everything. Groups absent from the inventory are skipped with a warning.

### C. Network devices *(the boundary evidence)*

**5.** `[FIREWALL]` the connection table and the ruleset
```
terminal pager 0
show conn all                        -> network\fw01-showconn.txt
show running-config access-list      -> network\fw01-acl.txt
show running-config object-group     -> (same file)
```
Take the object-groups as well, or a rule written against a group cannot be expanded into the hosts and ports it covers.

Or let the Windows collector fetch them over SSH:
```powershell
.\ppsm-collect.ps1 -NetworkDevices fw01:asa,core01:ios -NetworkUser admin
```
Type `asa` pulls the connection table and ruleset; type `ios` pulls the NetFlow cache (`-FlowMonitor` names the monitor). You are prompted for the password unless a key is set up. `READ-ME.txt` lists what came back.

**5b.** `[FMC CONSOLE]` the FMC reports, if the firewall is managed by FMC

| Report | Where |
|---|---|
| Connection Events | Analysis → Connections → Events |
| Servers | Analysis → Hosts → Servers |
| Hosts | Analysis → Hosts → Network Map |
| Access control policy | Policies → Access Control → report icon |
| Site-to-site VPN | Devices → VPN → Site To Site |

Export to CSV where you can; the policy report is also read as PDF.

**6.** `[SWITCH]` the NetFlow cache, per switch
```
show flow monitor ?
show flow monitor <name> cache format csv     -> network\<switch>-netflow.csv
```
Nothing listed after the `?` means NetFlow is not configured there. A NetFlow analyser's Top Conversations report for the last 30 days (`.xlsx`) covers every switch at once instead.

### D. Finish in the tool

**7.** `[YOUR WORKSTATION]` export the ACAS scan as `.nessus` into `ppsm-results\` if the enclave is scanned, then drag the whole folder onto the Ingest box.

**8.** `[YOUR WORKSTATION]` Finish Register, clear the review checks, **Export → eMASS Import (allow.xlsx)**, then **Save Session** before closing the tab.

---

## 5. The whole job — manual

Where scripts are not permitted, every command by hand. **Name files after the host**: that name is how a service listening on `0.0.0.0` is attributed to the right machine.

**Setup.** `[YOUR WORKSTATION]` create the folders `ppsm-results\hosts\`, `inventory\`, `network\`.

### A. Windows hosts *(repeat per host)*

**2.** `[WINDOWS HOST]` ports and connections → `ppsm-results\hosts\`
```powershell
Get-NetTCPConnection |
  Select-Object *,@{n='Process';e={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}} |
  Export-Csv -NoTypeInformation "$env:COMPUTERNAME-tcp.csv"
Get-NetUDPEndpoint |
  Select-Object *,@{n='Process';e={(Get-Process -Id $_.OwningProcess -ErrorAction SilentlyContinue).ProcessName}} |
  Export-Csv -NoTypeInformation "$env:COMPUTERNAME-udp.csv"
```
If `Get-NetTCPConnection` is "not recognized" on that host, or if process names are not needed:
```powershell
netstat -ano > "$env:COMPUTERNAME-netstat.txt"
```
covers TCP and UDP in one command and is read just as well. Elevated, `netstat -anob` adds the executable name.

**3.** `[DOMAIN CONTROLLER]` computer names (once, not per host) → `ppsm-results\inventory\`
```powershell
Get-ADComputer -Filter * -Properties DNSHostName,IPv4Address,OperatingSystem |
  Where-Object IPv4Address |
  Select-Object Name,DNSHostName,IPv4Address,OperatingSystem |
  Export-Csv -NoTypeInformation ad-computers.csv
```

**3b.** `[DOMAIN CONTROLLER]` the DNS zones, forward **and** reverse → `ppsm-results\inventory\`
```powershell
$recs = Get-DnsServerZone | Where-Object { -not $_.IsAutoCreated } | ForEach-Object {
  $z = $_.ZoneName
  Get-DnsServerResourceRecord -ZoneName $z |
    Where-Object RecordType -in 'A','PTR','CNAME' |
    Select-Object HostName, RecordType, @{n='ZoneName';e={$z}},
      @{n='IPv4Address';e={$_.RecordData.IPv4Address.IPAddressToString}},
      @{n='PtrDomainName';e={$_.RecordData.PtrDomainName}},
      @{n='HostNameAlias';e={$_.RecordData.HostNameAlias}}
}
$recs | Export-Csv -NoTypeInformation dns-records.csv
```
The record data has to be expanded like that; a plain `Export-Csv` writes `DnsServerResourceRecordA` where the address should be. Where PowerShell is not available, DNS Manager's right-click export or `dnscmd /zoneprint <zone> > zone.txt` are read just as well.

**4.** `[WINDOWS HOST]` the local hosts file → `ppsm-results\inventory\`
```
copy %SystemRoot%\System32\drivers\etc\hosts hosts-file.txt
```

### B. Linux hosts *(repeat per host)*

**5.** `[LINUX HOST]` ports and connections → `ppsm-results/hosts/`
```bash
H=$(hostname -s)
{ echo "# ppsm-host: $H"; sudo ss -tuanp; } > $H-sockets.txt
```
The `ppsm-host` line is not optional: it is how a `0.0.0.0` listener is attributed to this machine.

**6.** `[LINUX HOST]` the local name sources → `ppsm-results/inventory/`
```bash
cp /etc/hosts hosts-file.txt
ip neigh > arp-table.txt
```

**6b.** `[LINUX HOST]` the BIND zone files, if DNS runs on BIND
```bash
sudo cp /var/named/*.zone /var/named/*.rev ppsm-results/inventory/
sudo cp /etc/bind/db.* ppsm-results/inventory/
```
Take the reverse zones as well. Leave out the stock files (`db.local`, `db.127`, `db.root`) — they name nothing in your enclave.

### C. Network devices
**7.** `[FIREWALL]` connection table and ruleset (as scripted step 5) · **7b.** `[FMC CONSOLE]` FMC reports (as 5b) · **8.** `[SWITCH]` NetFlow (as 6)

### D. Finish in the tool
**9.** add the ACAS scan, import the folder · **10.** export the eMASS template, save the session

If you are short on time: group A or B, plus the firewall connection table. Everything else improves the result rather than making it possible.

---

## 6. What the collectors produce

```
ppsm-results\
  READ-ME.txt                    when, by whom, which hosts were reached and which
                                 were not, which shell and method each host used
  hosts\<host>-connections.csv   one row per socket: source and destination
                                 address/port, protocol, state, process, PID, Collector
  hosts\<host>-netstat.txt       the raw netstat -ano, for reference
  hosts\pfirewall.log            the Windows Firewall log, if enabled
  inventory\ad-computers.csv     every computer account with an address
  inventory\dns-records.csv      A, PTR and CNAME records from every zone
  inventory\hosts-file.txt       the local hosts file
  network\<device>-showconn.txt  the firewall connection table
  network\<device>-acl.txt       access-lists and object-groups
  network\<switch>-netflow.csv   the NetFlow cache
```

The connections CSV marks a client of a local listener as Inbound with the listener as the destination, however Windows reported it. The builder decides direction again from the listener list in the same file, so a hand-edited CSV still reads correctly.

---

## 7. Every file the builder reads

Each file is identified from its **contents**, never its name, and the import line says what it was taken to be. Drop the whole folder.

**Required — the register cannot be built without these**

| Input | Why |
|---|---|
| Host ports and connections | `<host>-connections.csv`, `-netstat.txt`, `-sockets.txt`, or raw `Get-NetTCPConnection` / `Get-NetUDPEndpoint` CSV exports |
| Firewall connection table | `show conn`, or FMC Connection Events — what actually crosses the boundary, with the interface names that set Boundary |
| At least one name source | AD export and/or DNS zones. Without one the register is a list of bare addresses |

**Strongly recommended — each closes gaps the required files leave**

| Input | Gives |
|---|---|
| DNS zones, forward + reverse | the authoritative hostname for every address |
| Firewall ruleset | access-list + object-group, or the FMC / Palo Alto policy export (PDF or CSV) — what is permitted |
| ACAS / Nessus scan | `.nessus` or the Tenable CSV findings export — credentialed listening services, products, and the installed-software inventory |
| NetFlow | switch cache CSV, or an analyser's Top Conversations report — east-west flows the firewall never sees |
| FMC Servers and Hosts | discovered services, hostnames and OS |

**When you have them — read automatically if dropped in:** FMC intrusion / security intelligence events, FMC site-to-site VPN report, Zeek `conn.log`, pcap / pcapng / tcpdump / pktmon, `pfirewall.log` and `Get-NetFirewallRule` exports, ASA syslog, `iptables-save`, Palo Alto config, nmap XML (`-oX`), hosts files, DHCP leases, neighbour tables, CMDB exports, DNS Manager exports, `dnscmd` output, a previous eMASS export, a register kept by hand, the filled-in work list.

**Formats.** Anything above as CSV, Excel `.xlsx`, text, or a text-based PDF. Not readable: older binary `.xls` (save as `.xlsx`), scanned or image-only PDFs, zip archives (extract first). Read-me and notes files are skipped rather than reported as errors. Tenable CSV exports with a multi-line Plugin Output column are read correctly.

---

## 8. How the sources fill each other's gaps

No single source has the whole picture. Dropping the folder runs all of this automatically.

**Which end is which.** PPSM registers the *listening service*, never the client. One rule covers every source: the end offering the service becomes the Destination, whoever reached it becomes the Source.

| Source | Destination | Source |
|---|---|---|
| ss / netstat / collector CSV | LISTEN line local address | the peer on ESTABLISHED |
| Get-NetTCPConnection | LocalAddress when State=Listen | the other end |
| show conn / syslog / pcap | the end with the better-known port | the other end |
| FMC connection events | Responder | Initiator |
| NetFlow | DST ADDR | SRC ADDR |
| Zeek conn.log | `id.resp_h` | `id.orig_h` |
| nmap / Nessus | the scanned host | (none) |
| access-list / policy | destination network | source network |

**Names.** Every name source feeds one inventory, applied to every row from every file, in both directions: an address gains its name, a name gains its address, a short name is completed with the zone, a CNAME leads to the address it points at. Where sources disagree the most authoritative wins (reverse DNS, forward DNS and AD, the host itself, scans, hosts files, FMC, DHCP, NetFlow labels, ARP last); the losers are kept as aliases. A name two hosts share is never guessed.

**Rules against traffic.** A rule written against a network gains the host that actually used it; an observed flow gains the rule that permits it. A rule nobody used keeps its network and is reported separately.

**Services and flows.** A listening service found by a scan folds into the traffic that used it, so one row carries both the product and the client. The review checks flag any row where a scan found the service but nothing shows who reaches it; the firewall connection table is what usually closes that gap.

**Repeat sightings.** The same flow usually appears in several sources. Those merge into one row as they import, and the Origin column lists every tool that saw it.

**Addresses that point nowhere.**

| Seen | Treated as |
|---|---|
| `0.0.0.0`, `*` | every interface on this host — attributed to the machine the file came from |
| `127.x`, `::1` | loopback — cannot leave the host, skipped with a count |
| IPv6 | set aside by default (Settings → IPv4 only) |
| ephemeral port as destination | 32768–60999 / 49152–65535 nearly always means the capture was read backwards; flagged |
| a network | stays a network: a valid rule source, never named |

---

## 9. Hostnames and addresses

The register works in addresses; names are kept beside them. A hostname anywhere — in the hostname cells, in the address cells (`wsus11` in Source IP), several per cell joined by semicolons, upper case, trailing dot — is recognised as a name and resolved through the inventory. A short name is completed with its zone, worked out from the export's SOA row, the file name, the suffix most rows share, or the DNS domain under Settings.

Resolve Hostnames (and the automatic pass on import) reports:

| Line | Meaning |
|---|---|
| N addresses found from a name | filled in from the zones |
| N short names completed | `WSUS11` → `wsus11.range2.mil` |
| N name/address mismatches | the row says one name, DNS says another for that address; the row is annotated, not rewritten |
| N hostnames still without an address | no source knows them; listed under Endpoints with a box to type the address into, and on the Unidentified Work List |

Nothing is guessed: a name no source knows stays as it is and is raised as a must-fix check rather than filled with a lookalike. On the eMASS export ([section 17](#17-the-emass-export)) only addresses are written.

---

## 10. Nessus / ACAS scans and the software inventory

A `.nessus` file is usually the richest input, and everything in it is used:

| From the scan | Used for |
|---|---|
| names | FQDN, reverse DNS and NetBIOS; the best becomes the hostname, the rest become aliases |
| OS and type | the endpoint list |
| interfaces | a host's other addresses resolve to the same name |
| the product | service detection fills Application/Software Record Name — "Oracle Database 19c Enterprise Edition", not `oracle_tns` |
| encryption | the SSL/TLS plugins mark the service as encrypted |
| connections | a credentialed scan's netstat output gives active connections, not just listeners |
| credentialed? | an uncredentialed scan sees only what answered from the network; those hosts are marked and a review check lists them |

**Software inventory.** A credentialed scan lists what is *installed* on each host — Windows "Installed Software Enumeration" (plugin 20811), "Software Enumeration (SSH)" for RPM/DEB packages (22869), the CPE list (45590) and the Windows service list (44401). The builder keeps that per address; the Endpoints table shows a Software count per host, with the list on hover.

During Fill In (and the automatic pass on import), where a row's Application/Software Record Name is blank, a bare process name (`svchost`, `sqlservr`, `w3wp`) or just a copy of the service, the installed product that serves that port is written instead — "Microsoft SQL Server 2019 (64-bit) 15.0.2000.5" behind 1433, "Splunk Universal Forwarder 9.1.2" behind 8089. A product is matched by the port it usually serves, by the service it implies, or by the process it runs as. A proper product name already in the row is never replaced; hosts the scan did not inventory are not touched.

Where a host has an inventory and nothing in it accounts for a port, the row is noted and the review checks list it under *worth a look* — that is a listener to name by hand.

The Tenable / ACAS CSV findings export is read the same way, inventory included.

---

## 11. Services and application names

**Data Service** carries the service's *name*, never a code or a bare acronym: "Remote Desktop Protocol (RDP)" rather than `ms-wbt-server`, "Lightweight Directory Access Protocol (LDAP)" rather than `ldap`, "Server Message Block (SMB)" rather than `microsoft-ds` or `cifs`. Codes that arrive from a scan (`www`, `cifs`, `epmap`), from Zeek (`ssl`) or from a CSV are upgraded when the register is finished. A name you type is left exactly as typed. Custom names under Settings (`9997 = splunk-forwarder`) override the built-in port table.

**Application/Software Record Name** is the *product*: from the scan's service detection, from the software inventory, or from a broad product directory that maps "Apache/2.4.57", "OpenSSH_8.7", "Microsoft SQL Server 2019" to their common service and usual port. A Windows process name is an application, not a service: it goes to this column and the service is named from the directory or the port. Version strings are ignored when comparing, so "Microsoft IIS/10.0" and "microsoft iis 10" are one application.

---

## 12. Boundaries

Internal traffic gets no B number. Internal means not crossing any CAL-defined boundary, and that explicitly includes the Enclave Gateway (B11/B12) and the Enclave DMZ (B9/B10). Traffic that stays inside is tagged **Internal** — a real value, not a gap.

| Observed | Derived |
|---|---|
| both ends inside your ranges | Internal |
| inside → outside your ranges | B14 |
| outside → inside | B13 |
| firewall interface names | the zone pair, where the data carried them (beats an address guess) |
| anything it cannot tell | left blank for you |

The tool never assigns a DISN boundary (B1–B8, B15) on its own. Nothing in host output, a connection table or a flow cache can prove a flow crossed the DISN. Those are yours to set where the architecture warrants it, and to confirm with your AO or ISSM. A review check flags any internal flow carrying a DISN boundary.

If your enclave uses address space outside RFC1918, set **Internal ranges** under Settings before importing; otherwise internal traffic reads as leaving the enclave and picks up a B14 it does not deserve.

On export, Internal rows carry an Internal Use Only boundary (18 by default — Settings) because the template has no plain "Internal".

---

## 13. What is left for you

Three fields are judgements rather than observations. Internal rows need none of them.

| Field | Who decides |
|---|---|
| **Mission Justification** | why the flow needs to exist. A reviewer reads this first. Fill In can draft it ([§15](#15-review-checks-and-rule-coverage)) and the export generates one where it is still blank ([§17](#17-the-emass-export)), but a real one from the system owner is what gets accepted. |
| **CAL / VA Reference** | look the service up on the PPSM CAL; if it is already listed, cite its VA report rather than raising a duplicate. |
| **Assurance Category** | the CAL entry for that service, or your own CLSA for a local one. |

| Assurance category | Meaning |
|---|---|
| Green | best practice, minimal mitigation |
| Yellow | conditional, documented mitigations |
| Red | banned; needs an exception |
| Orange | legacy exception, not for new systems |
| AO | local service, AO accepts the risk |
| Temporary | on the CAL pending a VA |

Fill them in the table, or select rows and use **Bulk-apply**. Submission path follows from the boundary: B1–B8 and B15 go VA / Further Action through the PPSM CCB/TAG; B9–B14 and B16 are CLSA, component local; B13 and B14 additionally need a DoDIN waiver; Internal needs no submission.

---

## 14. Dedupe and consolidation

**Dedupe.** Two rows are the same registration when they describe the same flow: same source, same destination, same protocol, same destination port(s). Direction as seen from one side or the other, the enclave one tool filled in, the origin tool and the counters are detail about the same flow, not a reason to keep two rows. The most complete row is kept, its blanks are filled from the rows that go, origin tools are merged and counters added. A row already covered by a consolidated row (`80; 443` from `10.1.1.5; 10.1.1.6` covers `10.1.1.5 → 443`) is removed; so is a bare listener covered by an observed flow to that port. Names and addresses are compared trimmed, case-folded and without a trailing dot. Rules and observations are never merged with each other. Dedupe runs on import, at Finish, at export, and on demand.

**Consolidate.** When the same source reaches the same destination on several ports, those collapse into one row with the ports joined by semicolons — `88; 389; 636` — and Data Service and application lined up with them. Then several sources reaching the same service become one row with the sources joined. A different destination, protocol, boundary or direction keeps its own row. On by default; the toggles are under Settings, and the **Consolidate** button runs it on demand.

---

## 15. Review checks and rule coverage

| Severity | Checks |
|---|---|
| **Must fix** | Boundary not set · Internal traffic tagged with a DISN boundary · Destination port in an ephemeral range · No CAL/VA reference · Assurance category Red / Banned · Hostname with no IP address · No source (a scan found the service but not who reaches it) · Observed traffic with no rule behind it |
| **Should fix** | No data service · No assurance category · No mission justification · Mission justification still a draft · Web service on a non-standard port (NSPU exception) · Boundary 13/14 needs a DoDIN waiver · Hostname disagrees with DNS · Rule permits traffic never seen · Found by an uncredentialed scan |
| **Worth a look** | Destination system not named · Duplicate registration · No installed software matches this port |

**Export Findings** writes the list to CSV with the offending rows.

**Rule coverage** (same card). Every observed flow is read against the imported ruleset on source scope, destination scope, protocol *and* port (single, range, list, or any) — not just the port, which would let any 443 rule anywhere cover every HTTPS flow. **Run Coverage** shows flows with no permitting rule, partly covered consolidated rows, flows matched only by a deny rule ("denied but seen"), and permit rules no flow used. **Coverage Report** writes it all to one CSV with rule references and per-rule usage counts. Needs both a ruleset and observed traffic.

**Mission Justification drafts.** Under Fill In, *Mission Justification (draft from service)* is off by default. Tick it and rows with no purpose get a sentence from the service need, both ends and the port, ending `[DRAFT — confirm the mission need and edit before submission]`. The automatic pass never writes it, and the review checks flag any row still carrying the marker.

---

## 16. Compare with the previous register

Load the register you submitted last time — the `allow.xlsx` the builder wrote, an eMASS PPSM export (CSV or workbook), a register CSV or a saved session — and the builder compares it with the current one:

| Result | Meaning |
|---|---|
| Added | in the current register, not the previous |
| Removed | in the previous register, not seen in this run |
| Changed | in both, but Data Service, Boundary, Direction, Type, Application, VPN, Purpose or a hostname differs |

Both sides are compared flow by flow, so a consolidated `80; 443` row in last quarter's file matches the two separate rows here, and a service written as a code then and a name now is not a change. Firewall rules are left out. **Diff Report** writes the three lists to CSV. **Mark Registered / New** sets Status on the current rows so what you submit is the delta.

---

## 17. The eMASS export

**Export → eMASS Import (allow.xlsx)** writes the PPSM Boundary Information Import Template exactly: the CUI banner and system header block in rows 1–7, the 21 columns A–U in the template's order on row 8, and the Glossary and Instructions sheets beside the PPSM sheet. Boundary, Direction, Type, both Connection Logical Point columns, VPN Tunnel Type and VPN / Encrypted Tunnel Traffic keep the template's dropdowns. The character limits from the Instructions sheet are enforced per value. Nothing is renamed, reordered or added. **eMASS CSV** writes the same 21 columns as CSV.

Before writing, the register is finished (names resolved, fields derived, duplicates merged, ports consolidated) and the *sheet* is shaped from it. **The working register is never changed by an export.**

### How the sheet is shaped — Settings › eMASS template, all on by default

| Setting | Effect |
|---|---|
| **One row per port** | A port is listed **once**. TCP and UDP on the same port are one row reading `TCP/UDP`; a port seen both ways is `Bi-Directional`; a port seen across more than one boundary carries them semicolon-joined, as the Instructions allow. Every service, application, source and destination seen on the port is joined in, each named once. A register row that Consolidate joined (`80; 443`) is split back so each port lands in exactly one exported row. |
| **IP addresses only** | No hostnames in Source / Destination; the FQDN columns carry the address. A name sitting in an address cell is resolved on the way out; a name no source can resolve is left in place and the export says how many there are. |
| **Order by port** | Rows run from the lowest port to the highest. |
| **Purpose on every row** | A row with no Mission Justification gets a statement built from the service need, both ends and the port, so column U is never blank. A written purpose is used as is (any `[DRAFT]` marker removed). |

### Set once — Settings › eMASS template

| Setting | Fills |
|---|---|
| DNS domain | completes short names into FQDNs |
| Physical location of this enclave | Source / Destination Physical Location for anything inside your ranges |
| Source / Destination Physical Location | the same, per end, where the two differ (falls back to the enclave location) |
| Label for anything outside the enclave | the location columns for outside addresses — a cloud provider, or "External (non-DoD)" |
| VPN / Encrypted Tunnel Traffic on export | per row from the boundary, or Yes / No for the whole export; a tick box applies it even over per-row values |
| VPN Tunnel Type on export | one of the Glossary values (AF VPN, GRE Tunnel, JRSS RA VPN, Type-1 Encrypted Tunnel, …) written where a row has none, or everywhere with the overwrite box; never onto a row whose VPN traffic is No |
| VPN Tunnel Type when there is no tunnel | blank by default; `N/A` if your reviewer wants it filled |
| Internal boundary | which Internal Use Only boundary internal rows export as (18 by default) |

Per-row values set in the table or by **Bulk-apply** (Source/Destination Physical Location, VPN / Encrypted Tunnel Traffic, VPN Tunnel Type from the Glossary list) win unless the overwrite boxes are ticked.

**System header** — System Identification › eMASS template header: Exported By, Office / Org, Information System Owner, System ID, DITPR ID, POC Name, Phone, E-Mail, Date Reviewed, Reviewed By. None of these can be derived from traffic.

On export the tool names any required field still empty and how many rows it affects, and how many rows still carry an unresolved hostname.

**Do not restructure the workbook.** The importer reads columns by position; the template says not to delete columns or sheets, remove the classification label, or add columns.

---

## 18. Other exports

| Export | Contents |
|---|---|
| Register (`.xlsx` / `.csv`) | the full working register, every column, including the ones eMASS does not take |
| Review findings (`.csv`) | every outstanding item with its rows |
| Coverage report (`.csv`) | flows and the rules that permit them; rules and the flows that used them |
| Diff report (`.csv`) | Added / Removed / Changed against a previous register |
| Endpoints (`.csv`) | every system, named and unnamed; the Unidentified Work List; the host map |
| Boundary diagram | SVG, PNG or print, drawn from the register |
| Printable report | the register as a document |
| Session (`.json`) | everything: rows, settings, host inventory, software inventory. **Save it before closing the tab**; the browser keeps nothing by itself |
| Profile (`.json`) | settings only, to carry to another system |

Register exports fold dashes, curly quotes and bullets to plain ASCII because they render as `?` in Excel on many desktops, and store Boundary as a bare code. The eMASS template always keeps its exact official wording.

---

## 19. Settings

Most can be left alone. Worth knowing about:

| Setting | Notes |
|---|---|
| Simple mode | on by default: the whole finish pass runs on import. Untick at the top for the endpoint list, review checks, diagram and per-field controls |
| IPv4 only | on. Turn off only for a dual-stack enclave |
| Internal ranges | CIDRs counted as inside on top of RFC1918 |
| Subnet to enclave map | optional; otherwise the enclave column falls back to the observed /24 |
| Ephemeral range | 49152–65535 Windows, 32768–60999 Linux. *Clear ephemeral source ports* is on; *Exclude ephemeral destination* is off |
| Custom service names | `9997 = splunk-forwarder`, `TCP/8443 = vendor-gui`; override the port table |
| Custom boundary tags | your own alongside the built-ins |
| Consolidate ports / sources | both on; the eMASS sheet is consolidated regardless |
| Fold to ASCII | on, for register exports |
| pcap limits | minimum packets per flow strips scan noise; max packets keeps the browser responsive |
| eMASS template | see [§17](#17-the-emass-export) |
| Visible columns | hide register columns you do not use |

---

## 20. Troubleshooting

| Symptom | What to do |
|---|---|
| A file imports as zero entries | Read the import line: it names what the file was taken to be and why it gave nothing. Usually the wrong export — nmap normal output instead of `-oX` XML, or a Tenable HTML report instead of `.nessus`. |
| An Excel file is refused | Older binary `.xls` cannot be read. Save as `.xlsx`. |
| A PDF gives nothing | A scan or image export with no text. Export the original; for a firewall policy a CSV export is most reliable. |
| Hostnames are missing / the export reports unresolved names | Add a name source, ideally the DNS zones forward and reverse. Names apply to every row whenever a new source arrives. Type the last few under Endpoints. |
| Internal traffic shows B13 or B14 | Non-RFC1918 space: add it under Settings › Internal ranges and import again. |
| Client and server look backwards | The data had nothing to say which side opened the connection. Fix the rows with Bulk-apply; the ephemeral-port check flags them. |
| Ports still appear more than once on the eMASS sheet | Check Settings › eMASS template › *One row per port* is on, and that the rows really share a port (a port seen on two destinations is still one row; look at the Destination cell). |
| The collector says a host is unreachable | The WinRM port did not answer within `-ProbeTimeout`. Check WinRM is enabled (`winrm quickconfig`), the firewall allows 5985, and the name resolves. `-ProbeTimeout 10` for a slow link. |
| The collector took the netstat path on a host | `Get-NetTCPConnection` is not available there. Results are complete; the Collector column says so. Run `-SelfTest` on that host once to confirm. |
| "Could not be read" | File locked, open elsewhere, or blocked by endpoint policy. Copy it to a local folder you own. |
| Nothing happens when you drop a file | Not a real browser tab (preview pane, Protected View). Check for `file://` in the address bar. |
| Shows "N entries, 0 shown" | A filter is hiding the rows. Clear the search box and dropdowns. |
| Anything else | Run **Self-Test**. All green means the tool is fine and the input is the problem. Red: note the browser and which checks failed. |

---

## 21. A few things worth remembering

The tool assembles a register. It does not certify compliance: the accuracy of the inputs, and the boundary and assurance analysis, remain the assessor's responsibility.

Every row records the tools and the host the data came from, so an entry can always be traced back to the sensor, firewall or workstation that observed it.

The working register and the eMASS sheet are different things. The register keeps one row per flow with everything known about it; the sheet is shaped from it on the way out — one row per port, addresses only, in port order. Export never changes the register.

Save the session before closing the tab. The browser keeps nothing by itself.
