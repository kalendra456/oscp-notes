# OSCP / PEN-200 Information Gathering Notes

> Detailed, process-oriented notes for OSCP/PEN-200 study.  
> Focus: **what the question is asking, what command to run, what output matters, and how to derive the answer.**

---

## Table of Contents

1. [Information Gathering Mindset](#1-information-gathering-mindset)
2. [WHOIS Enumeration](#2-whois-enumeration)
3. [Google Hacking / Search Engine Recon](#3-google-hacking--search-engine-recon)
4. [Passive LLM-Aided Enumeration](#4-passive-llm-aided-enumeration)
5. [DNS Enumeration](#5-dns-enumeration)
6. [TCP/UDP Scanning with Netcat](#6-tcpudp-scanning-with-netcat)
7. [Port Scanning with Nmap](#7-port-scanning-with-nmap)
8. [SMB Enumeration](#8-smb-enumeration)
9. [SMTP Enumeration](#9-smtp-enumeration)
10. [SNMP Enumeration](#10-snmp-enumeration)
11. [Windows Living-Off-the-Land Enumeration](#11-windows-living-off-the-land-enumeration)
12. [Important Ports to Memorize](#12-important-ports-to-memorize)
13. [Lab Question → Solving Process](#13-lab-question--solving-process)
14. [Full OSCP Information Gathering Workflow](#14-full-oscp-information-gathering-workflow)

---

# 1. Information Gathering Mindset

Information gathering is **iterative**.

A useful mental model:

```text
Initial clue
  ↓
Identify hostname / IP / person / service
  ↓
Run focused enumeration
  ↓
Extract useful data
  ↓
Use the new information as the next pivot
  ↓
Repeat
```

Examples:

```text
WHOIS → nameserver → IP → scan → DNS service
DNS → hostname → web server → HTTP enumeration
Nmap → 445/tcp → SMB enumeration
Nmap → 161/udp → SNMP enumeration
SNMP → software/process → version research
```

## Passive vs Active Recon

### Passive

No direct probing of the target infrastructure.

Examples:

- WHOIS
- Search engines
- Google dorks
- Public documents
- Public repositories
- Social media
- LLM-assisted OSINT organization

### Active

Direct interaction with target systems/services.

Examples:

- DNS queries
- Netcat/Nmap scans
- SMB
- SMTP
- SNMP

## OSCP habit

**Save scan output.**

Do not repeatedly scan a network if the answer can be extracted from an existing file.

Example:

```bash
nmap -sS 192.168.50.0/24 -oG scan.gnmap
grep "25/open" scan.gnmap
grep "445/open" scan.gnmap
```

---

# 2. WHOIS Enumeration

WHOIS commonly uses:

```text
TCP/43
```

It can reveal:

- Registrant
- Organization
- Registrar
- Registrar WHOIS server
- Administrative contact
- Technical contact
- Email addresses
- Name servers
- Creation/expiration dates
- Domain status
- NetRange / CIDR / network owner

---

## Forward WHOIS

You know the **domain** and want registration data.

```bash
whois megacorpone.com -h <WHOIS_SERVER_IP>
```

Example:

```bash
whois megacorpone.com -h 192.168.50.251
```

Useful filters:

```bash
whois megacorpone.com -h <IP> | grep -i "Name Server"
whois megacorpone.com -h <IP> | grep -i "Registrar WHOIS Server"
whois megacorpone.com -h <IP> | grep -i "Tech"
whois megacorpone.com -h <IP> | grep -i "Email"
whois megacorpone.com -h <IP> | grep -i "OS{"
```

---

## Lab Process: Find the Third Name Server

Question style:

> What is the hostname of the third MegaCorp One name server?

Run:

```bash
whois megacorpone.com -h <WHOIS_SERVER_IP>
```

Filter:

```bash
whois megacorpone.com -h <WHOIS_SERVER_IP> | grep -i "Name Server"
```

Expected style:

```text
Name Server: NS1.MEGACORPONE.COM
Name Server: NS2.MEGACORPONE.COM
Name Server: NS3.MEGACORPONE.COM
```

Answer:

```text
NS3.MEGACORPONE.COM
```

### Takeaway

If a question asks for **one specific field**, grep that field.

---

## Lab Process: Registrar WHOIS Server

Reuse the same WHOIS output:

```bash
whois megacorpone.com -h <IP> | grep -i "Registrar WHOIS Server"
```

Expected:

```text
Registrar WHOIS Server: whois.gandi.net
```

Answer:

```text
whois.gandi.net
```

---

## Lab Process: Flag in WHOIS DNS Section

Run:

```bash
whois offensive-security.com -h <IP>
```

Search:

```bash
whois offensive-security.com -h <IP> | grep -iE "dns|name server|OS\{"
```

If the question already tells you the flag is in the **DNS section**, search there first.

---

## Reverse WHOIS

You know an IP and want ownership/network information.

```bash
whois <TARGET_IP> -h <WHOIS_SERVER_IP>
```

Look for:

```text
NetRange
CIDR
NetName
OrgName
Country
```

### Pivot idea

```text
IP
 ↓
Network owner / CIDR
 ↓
Related IP space
 ↓
Additional recon
```

---

# 3. Google Hacking / Search Engine Recon

Google hacking means combining search operators to reduce irrelevant results.

---

## `site:`

Restrict results to one domain:

```text
site:megacorpone.com
```

---

## `filetype:` / `ext:`

Find specific file types:

```text
site:megacorpone.com filetype:pdf
site:megacorpone.com ext:php
site:megacorpone.com ext:txt
```

Useful file types:

```text
pdf
doc
docx
xls
xlsx
txt
log
conf
ini
xml
json
sql
db
php
```

---

## `inurl:`

Find URLs containing a keyword:

```text
site:megacorpone.com inurl:admin
site:megacorpone.com inurl:login
site:megacorpone.com inurl:upload
```

---

## `intitle:`

Directory listings:

```text
site:megacorpone.com intitle:"index of"
```

Generic:

```text
intitle:"index of" "parent directory"
```

---

## `intext:`

Search page content:

```text
site:megacorpone.com intext:"password"
site:megacorpone.com intext:"github.com"
```

---

## Exclusion with `-`

Exclude target-owned results:

```text
"MegaCorp One" -site:megacorpone.com
```

Useful when the question asks for information **not listed on the company's own website**.

---

## Lab Process: Find VP of Legal

Search:

```text
"VP of Legal" "MegaCorp One"
```

Or:

```text
site:megacorpone.com "VP of Legal"
```

Result:

```text
Mike Carlow
```

---

## Lab Process: Find Employee Email

Pivot from the discovered name:

```text
"Mike Carlow" "megacorpone.com"
"Mike Carlow" email
site:megacorpone.com "@megacorpone.com"
```

Result:

```text
mcarlow@megacorpone.com
```

---

## Lab Process: Find Employees Not Listed on Main Site

Use:

```text
"MegaCorp One" -site:megacorpone.com
```

Potential external sources:

- LinkedIn
- GitHub
- GitLab
- Cached/archived pages
- Third-party articles
- Public documents

---

# 4. Passive LLM-Aided Enumeration

Use an LLM to:

- Brainstorm recon ideas
- Generate target-specific dorks
- Organize WHOIS information
- Correlate employee data
- Suggest naming patterns
- Generate likely subdomain lists
- Build recon checklists

Example prompts:

```text
What is the WHOIS information for megacorpone.com?

What public information is available about the leadership of MegaCorpOne and their social media presence?

Generate Google dorks for exposed repositories related to megacorpone.com.

Suggest passive subdomain-enumeration techniques for megacorpone.com.
```

## Important Rule

Treat LLM output as a **lead**, not authoritative proof.

Validation flow:

```text
LLM suggestion
  ↓
host / dig / dnsrecon / subfinder
  ↓
Confirm target
  ↓
Nmap / protocol enumeration
```

---

# 5. DNS Enumeration

DNS records can reveal significant infrastructure.

## Important Record Types

| Record | Meaning |
|---|---|
| A | IPv4 address |
| AAAA | IPv6 address |
| NS | Authoritative name server |
| MX | Mail exchanger |
| PTR | Reverse lookup / IP-to-hostname |
| CNAME | Alias |
| TXT | Arbitrary text / verification / policy data |

---

## Basic `host` Queries

A record:

```bash
host www.megacorpone.com
```

MX:

```bash
host -t mx megacorpone.com
```

TXT:

```bash
host -t txt megacorpone.com
```

---

## MX Priority

Example:

```text
10 fb.mail.gandi.net
20 spool.mail.gandi.net
50 mail.megacorpone.com
60 mail2.megacorpone.com
```

Rule:

```text
LOWER number = HIGHER preference
```

So:

```text
10 = best
20 = second-best
```

---

## Forward DNS Brute Force

Create a small wordlist:

```bash
cat > list.txt << EOF
www
ftp
mail
owa
proxy
router
EOF
```

Run:

```bash
for sub in $(cat list.txt); do
    host $sub.megacorpone.com
done
```

Valid:

```text
mail.megacorpone.com has address 167.114.21.68
```

Invalid:

```text
NXDOMAIN
```

---

## Reverse DNS Brute Force

If you discover hosts around:

```text
167.114.21.x
```

Probe nearby PTR records:

```bash
for ip in $(seq 64 79); do
    host 167.114.21.$ip
done | grep -Ev "not found|timed out"
```

Possible discoveries:

```text
admin.megacorpone.com
beta.megacorpone.com
fs1.megacorpone.com
intranet.megacorpone.com
mail.megacorpone.com
router.megacorpone.com
siem.megacorpone.com
snmp.megacorpone.com
vpn.megacorpone.com
```

### Takeaway

```text
One IP
 ↓
Nearby PTR records
 ↓
Many hostnames
 ↓
New scanning targets
```

---

## DNSRecon

Standard:

```bash
dnsrecon -d megacorpone.com -t std
```

Brute force:

```bash
dnsrecon -d megacorpone.com -D ~/list.txt -t brt
```

Options:

```text
-d    domain
-D    wordlist
-t    enumeration type
std   standard
brt   brute force
```

---

## DNSEnum

```bash
dnsenum megacorpone.com
```

May reveal:

- A records
- NS
- MX
- Subdomains
- Zone transfer results
- Network ranges

---

## Windows DNS Enumeration

Resolve host:

```cmd
nslookup mail.megacorptwo.com
```

Use a specific DNS server:

```cmd
nslookup mail.megacorptwo.com 192.168.50.151
```

TXT:

```cmd
nslookup -type=TXT info.megacorptwo.com 192.168.50.151
```

### Lab solving rule

If the question asks for a TXT record, query TXT directly.

---

# 6. TCP/UDP Scanning with Netcat

Netcat is not primarily a scanner, but it can perform simple port discovery.

---

## TCP Scan

```bash
nc -nvv -w 1 -z TARGET 3388-3390
```

Options:

```text
-n      no DNS resolution
-v/-vv  verbose
-w 1    one-second timeout
-z      zero-I/O scanning
```

Example:

```text
3389 open
3388 Connection refused
3390 Connection refused
```

---

## Lab: Lowest Open TCP Port

```bash
nc -nv -z -w 1 <TARGET_ENDING_151> 1-10000
```

Process:

1. Scan the required range.
2. Collect all `open` results.
3. Submit the **lowest** open port.

---

## Lab: Highest Open TCP Port

Use the same scan:

```bash
nc -nv -z -w 1 <TARGET_ENDING_151> 1-10000
```

Submit the **highest** open result.

### Efficiency

Do not rerun a scan if you already have all the required output.

---

## UDP Scan

```bash
nc -nv -u -z -w 1 TARGET 120-200
```

`-u` means UDP.

### Important UDP Behavior

```text
Closed UDP
  -> often ICMP Port Unreachable

No response
  -> may be open
  -> may be filtered
  -> may simply ignore empty packets
```

UDP scans are less reliable than TCP scans.

---

# 7. Port Scanning with Nmap

Nmap is the main OSCP scanning tool.

---

## Basic Commands

Default:

```bash
nmap TARGET
```

Treat host as online:

```bash
nmap -Pn TARGET
```

SYN scan:

```bash
sudo nmap -sS TARGET
```

TCP Connect:

```bash
nmap -sT TARGET
```

UDP:

```bash
sudo nmap -sU TARGET
```

Combined TCP SYN + UDP:

```bash
sudo nmap -sS -sU TARGET
```

All TCP ports:

```bash
sudo nmap -sS -Pn -p- TARGET
```

Service/version:

```bash
nmap -sV TARGET
```

OS fingerprinting:

```bash
sudo nmap -O TARGET --osscan-guess
```

Aggressive scan:

```bash
nmap -A TARGET
```

---

## SYN Scan Concept

```text
Kali   -> SYN
Target -> SYN/ACK if open
Scanner does not complete the normal connection
Closed target port -> RST
```

The word **stealth** is historical. Modern IDS/IPS can detect SYN scans.

---

## Network Sweep

```bash
nmap -v -sn 192.168.50.1-253 -oG ping-sweep.txt
```

Extract live hosts:

```bash
grep Up ping-sweep.txt | cut -d " " -f 2
```

---

## Scan One Service Across a Network

SMTP:

```bash
nmap -p25 --open SUBNET
```

WHOIS:

```bash
nmap -p43 --open SUBNET
```

SMB:

```bash
nmap -p445 --open SUBNET
```

HTTP:

```bash
nmap -p80 SUBNET -oG web-sweep.txt
grep open web-sweep.txt | cut -d" " -f2
```

---

## Lab: Which Host Has Port 25 Open?

Run:

```bash
sudo nmap -sS <SUBNET> -oG scan.gnmap
```

Filter:

```bash
grep "25/open" scan.gnmap
```

Then submit the matching host.

If the lab requires replacing the dynamically assigned third octet with `50`, change only the final submitted IP format.

Example answer style:

```text
192.168.50.8
```

---

## Lab: Which Host Runs WHOIS?

WHOIS = TCP/43.

```bash
nmap -p43 --open <SUBNET>
```

The matching host is the answer.

Example walkthrough answer:

```text
192.168.50.251
```

---

## Windows Port Discovery Against the DC

PowerShell:

```powershell
1..1024 | % {
    echo ((New-Object Net.Sockets.TcpClient).Connect("192.168.50.151", $_)) "TCP port $_ is open"
} 2>$null
```

Read ports in ascending order.

Walkthrough first four:

```text
53,88,135,139
```

---

## Full-Port / High-Range Challenge

Default Nmap may miss high ports.

Start:

```bash
sudo nmap -sS -Pn -p- TARGET
```

Then inspect the unusual port:

```bash
sudo nmap -sV -p <HIGH_PORT> TARGET
```

### Important

Read the **entire service fingerprint/banner**.

An `unknown` service may still return useful text or a flag.

---

## Nmap Scripting Engine (NSE)

List scripts:

```bash
ls /usr/share/nmap/scripts/
```

Search:

```bash
ls /usr/share/nmap/scripts/ | grep http
```

Help:

```bash
nmap --script-help http-headers
```

Run:

```bash
nmap --script http-headers TARGET
```

---

## Lab: Find HTTP Title "Under Construction"

```bash
sudo nmap -sV --script http-title \
-p80,443,8000,8080,8443 \
<SUBNET>
```

Look for:

```text
http-title: Under Construction
```

Then:

```bash
curl http://<MATCHING_HOST>/
```

Read the flag from the index page.

---

# 8. SMB Enumeration

Important ports:

```text
139/tcp = NetBIOS Session
445/tcp = SMB
```

---

## Find SMB Hosts

```bash
nmap -Pn -p139,445 --open <SUBNET> -oG smb.gnmap
```

Find 445:

```bash
grep "445/open" smb.gnmap
```

Count:

```bash
grep "445/open" smb.gnmap | wc -l
```

---

## NetBIOS Enumeration

```bash
sudo nbtscan -r 192.168.50.0/24
```

May reveal:

- NetBIOS name
- Host role
- Naming convention
- Server identity

---

## SMB NSE Scripts

```bash
ls -1 /usr/share/nmap/scripts/smb*
```

Useful examples:

```text
smb-enum-users
smb-enum-shares
smb-enum-groups
smb-enum-sessions
smb-os-discovery
smb2-security-mode
```

Run OS discovery:

```bash
nmap -p139,445 --script smb-os-discovery TARGET
```

Potential output:

```text
OS
Computer name
NetBIOS name
Domain
Forest
FQDN
```

---

## Windows `net view`

```cmd
net view \\dc01 /all
```

Administrative shares:

```text
ADMIN$
C$
IPC$
```

AD-related shares:

```text
NETLOGON
SYSVOL
```

---

## `enum4linux`

```bash
enum4linux -a TARGET
```

Modern:

```bash
enum4linux-ng -A TARGET
```

Look for:

- Users
- Groups
- Shares
- Domain/workgroup
- Password policy
- Share comments

---

## Lab: Find User `alfred` and the Flag

Step 1: find SMB hosts.

```bash
nmap -Pn -p445 --open <SUBNET>
```

Step 2: enumerate each host.

```bash
for ip in <SMB_HOSTS>; do
    echo "===== $ip ====="
    enum4linux -a "$ip"
done
```

Step 3: find `alfred`.

```bash
enum4linux -a TARGET | grep -i alfred
```

Step 4: once the correct host is found, inspect:

```text
Share Enumeration
```

Especially:

```text
Comment
```

The flag is stored in a share comment.

### Takeaway

Do not look only at **share names**. Read descriptions/comments too.

---

# 9. SMTP Enumeration

SMTP normally uses:

```text
TCP/25
```

Useful commands:

```text
HELO
EHLO
VRFY
EXPN
MAIL FROM
RCPT TO
```

---

## Find SMTP Server

```bash
sudo nmap -Pn -p25 --open <SUBNET>
```

Connect:

```bash
nc -nv <SMTP_HOST> 25
```

Test:

```text
HELO kali
VRFY root
VRFY idontexist
```

Possible responses:

```text
252 2.0.0 root
550 ... User unknown
```

---

## Lab: VRFY Root Response Code

Process:

1. Scan subnet for TCP 25.
2. Connect with Netcat.
3. Issue:

```text
VRFY root
```

4. Read the **first three digits**.

Walkthrough answer:

```text
252
```

---

## Windows SMTP Check

```powershell
Test-NetConnection -Port 25 TARGET
```

If Telnet is available:

```cmd
telnet TARGET 25
```

Then:

```text
VRFY root
```

---

# 10. SNMP Enumeration

SNMP commonly uses:

```text
UDP/161
```

Common weak community strings:

```text
public
private
manager
```

SNMP may expose:

- OS
- Hostname
- Contact information
- Users
- Processes
- Process paths
- Installed software
- Interfaces
- Storage
- Listening TCP ports

---

## Find SNMP with Nmap

```bash
sudo nmap -sU --open -p161 <SUBNET> -oG open-snmp.txt
```

---

## Find SNMP with `onesixtyone`

Create community list:

```bash
echo public > community.txt
echo private >> community.txt
echo manager >> community.txt
```

Run:

```bash
onesixtyone -c community.txt <SUBNET>
```

Alternative with host list:

```bash
for ip in $(seq 1 254); do
    echo 192.168.50.$ip
done > ips

onesixtyone -c community.txt -i ips
```

---

## General `snmpwalk`

```bash
snmpwalk -v2c -c public TARGET
```

Or:

```bash
snmpwalk -c public -v1 -t 10 TARGET
```

---

## Useful SNMP OIDs

| OID | Meaning |
|---|---|
| `1.3.6.1.2.1.25.1.6.0` | System process count |
| `1.3.6.1.2.1.25.4.2.1.2` | Running programs |
| `1.3.6.1.2.1.25.4.2.1.4` | Process paths |
| `1.3.6.1.2.1.25.2.3.1.4` | Storage units |
| `1.3.6.1.2.1.25.6.3.1.2` | Installed software |
| `1.3.6.1.4.1.77.1.2.25` | User accounts |
| `1.3.6.1.2.1.6.13.1.3` | TCP local ports |

---

## Enumerate Users

```bash
snmpwalk -c public -v1 TARGET \
1.3.6.1.4.1.77.1.2.25
```

---

## Enumerate Running Programs

```bash
snmpwalk -v2c -c public TARGET \
1.3.6.1.2.1.25.4.2.1.2
```

Search for SNMP process:

```bash
snmpwalk -v2c -c public TARGET \
1.3.6.1.2.1.25.4.2.1.2 | grep -i snmp
```

Lab answer:

```text
snmp.exe
```

---

## Enumerate Installed Software

```bash
snmpwalk -c public -v1 TARGET \
1.3.6.1.2.1.25.6.3.1.2
```

Useful for product/version research.

---

## Enumerate TCP Listening Ports via SNMP

```bash
snmpwalk -c public -v1 TARGET \
1.3.6.1.2.1.6.13.1.3
```

This can reveal ports listening **only locally**, which might not appear in a remote Nmap scan.

---

## Enumerate Interfaces with `-Oa`

```bash
snmpwalk -v2c -c public -Oa TARGET \
1.3.6.1.2.1.2.2.1.2
```

`-Oa` attempts to convert hexadecimal strings to ASCII.

Lab process:

1. Run the interface OID query.
2. Read the **first returned STRING**.

Walkthrough answer:

```text
Software Loopback Interface 1
```

---

# 11. Windows Living-Off-the-Land Enumeration

When Kali tools are unavailable, use native Windows utilities.

Useful tools:

```text
whoami
ping
netstat
nslookup
net
PowerShell
Test-NetConnection
telnet
```

Examples:

```cmd
nslookup mail.megacorptwo.com
nslookup -type=TXT info.megacorptwo.com 192.168.50.151
net view \\dc01 /all
```

PowerShell:

```powershell
Test-NetConnection -Port 445 192.168.50.151
```

---

# 12. Important Ports to Memorize

| Port | Service |
|---:|---|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 25 | SMTP |
| 43 | WHOIS |
| 53 | DNS |
| 80 | HTTP |
| 88 | Kerberos |
| 110 | POP3 |
| 111 | RPCbind |
| 123 | NTP |
| 135 | MSRPC |
| 137 | NetBIOS Name |
| 139 | NetBIOS Session |
| 143 | IMAP |
| 161 | SNMP |
| 389 | LDAP |
| 443 | HTTPS |
| 445 | SMB |
| 464 | Kerberos password change |
| 636 | LDAPS |
| 1433 | MSSQL |
| 2049 | NFS |
| 3268 | Global Catalog LDAP |
| 3269 | Global Catalog LDAPS |
| 3306 | MySQL |
| 3389 | RDP |
| 5432 | PostgreSQL |
| 5985 | WinRM HTTP |
| 5986 | WinRM HTTPS |
| 8080 | Alternate HTTP |

## Memorize the Next Action Too

```text
25  -> SMTP enumeration
43  -> WHOIS
53  -> DNS enumeration
161 -> SNMP enumeration
445 -> SMB enumeration
80/443 -> Web enumeration
```

---

# 13. Lab Question → Solving Process

## "Which host has port 25 open?"

```bash
nmap -p25 --open SUBNET
```

Submit the matching host.

---

## "Which host runs WHOIS?"

```bash
nmap -p43 --open SUBNET
```

WHOIS = TCP/43.

---

## "Which is the lowest/highest TCP open port?"

```bash
nc -nv -z -w 1 TARGET 1-10000
```

Read only ports marked open.

---

## "What is the first returned UDP port?"

```bash
nc -nv -u -z -w 1 TARGET <RANGE>
```

Follow the exact exclusions/range stated in the question.

---

## "How many SMB hosts are there?"

```bash
nmap -p445 --open SUBNET -oG smb.txt
grep "445/open" smb.txt | wc -l
```

---

## "What are the administrative shares?"

From Windows:

```cmd
net view \\dc01 /all
```

Answer:

```text
ADMIN$,C$,IPC$
```

---

## "Find user alfred and the flag"

1. Find SMB hosts.
2. Run `enum4linux -a` against each.
3. Identify host containing `alfred`.
4. Inspect share comments.

---

## "What SMTP code does VRFY root return?"

```bash
nc TARGET 25
```

Then:

```text
VRFY root
```

Read the first three digits.

---

## "What is the SNMP process?"

```bash
snmpwalk -v2c -c public TARGET \
1.3.6.1.2.1.25.4.2.1.2 | grep -i snmp
```

---

## "What is the first interface?"

```bash
snmpwalk -v2c -c public -Oa TARGET \
1.3.6.1.2.1.2.2.1.2
```

Read the first `STRING`.

---

## "Find the site titled Under Construction"

```bash
nmap --script http-title \
-p80,443,8000,8080,8443 \
SUBNET
```

Then:

```bash
curl http://MATCHING_IP/
```

---

# 14. Full OSCP Information Gathering Workflow

## Step 1 — Passive Recon

Gather:

```text
Domains
Nameservers
Employees
Emails
Technologies
Public documents
Public repositories
Naming conventions
```

---

## Step 2 — DNS Enumeration

```bash
host DOMAIN
host -t mx DOMAIN
host -t txt DOMAIN
dnsrecon -d DOMAIN -t std
dnsenum DOMAIN
```

Then brute-force likely subdomains.

---

## Step 3 — Discover Hosts

```bash
nmap -sn SUBNET -oG hosts.txt
grep Up hosts.txt
```

---

## Step 4 — Initial TCP Scan

```bash
sudo nmap -sS -Pn --top-ports 1000 SUBNET -oA initial
```

---

## Step 5 — Full TCP Scan on Interesting Hosts

```bash
sudo nmap -sS -Pn -p- TARGET -oA full
```

---

## Step 6 — Service Enumeration

```bash
sudo nmap -sC -sV -Pn -p <OPEN_PORTS> TARGET -oA services
```

---

## Step 7 — UDP

```bash
sudo nmap -sU -Pn --top-ports 100 TARGET
```

Pay special attention to:

```text
53
123
161
```

---

## Step 8 — Protocol-Specific Enumeration

```text
25  -> SMTP
53  -> DNS
161 -> SNMP
445 -> SMB
80  -> HTTP
443 -> HTTPS
```

---

## Step 9 — Save and Correlate

Keep notes on:

```text
Hostnames
IPs
Ports
Versions
Users
Domains
Shares
Processes
Software
Credentials/clues
```

---

## Step 10 — Repeat

```text
New hostname
  -> resolve
  -> scan
  -> enumerate service

New username
  -> SMB / SMTP / AD / password reuse later

New software/version
  -> vulnerability research

New IP range
  -> additional DNS / port discovery
```

---

# Final Rule

> **Do not stop at "port open".**

An open port is the **start** of enumeration.

```text
445 open
  -> SMB enumeration

161 open
  -> community string testing
  -> snmpwalk

25 open
  -> SMTP interaction
  -> VRFY / EXPN

53 open
  -> DNS enumeration
  -> records / subdomains / PTR

HTTP discovered
  -> titles / headers / directories / technologies
```

The OSCP-style answer is often found **one or two protocol-specific enumeration steps after the initial scan**.

---

## Disclaimer

Use these techniques only on systems you own or are explicitly authorized to test, such as PEN-200/OSCP labs, CTFs, and approved penetration-testing scopes.
