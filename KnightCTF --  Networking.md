# KnightCTF :  Networking/Forensics Incident Reconstruction (PCAP Series)

Category: **Networking / DFIR / PCAP Forensics**  
Artifacts: `pcap1.pcapng`, `pcap2.pcapng`, `pcap3.pcapng`

---

## Table of Contents

1. [Scope and Approach](#scope-and-approach)  
2. [Tooling and Workflow](#tooling-and-workflow)  
3. [Challenge 1 : Reconnaissance (Open Ports)](#challenge-1--reconnaissance-open-ports)  
4. [Challenge 2 : Gateway Identification (Vendor)](#challenge-2--gateway-identification-vendor)  
5. [Challenge 3 : Exploitation (App Version + Username)](#challenge-3--exploitation-app-version--username)  
6. [Challenge 4 : Vulnerability Exploitation (Plugin + Version)](#challenge-4--vulnerability-exploitation-plugin--version)  
7. [Challenge 5 : Post-Exploitation (HTTP Payload Port + Reverse Shell Port)](#challenge-5--post-exploitation-http-payload-port--reverse-shell-port)  
8. [Challenge 6 : Database Credentials Theft (DB Username + Password)](#challenge-6--database-credentials-theft-db-username--password)  
9. [Automation / Script Snippets Used](#automation--script-snippets-used)  
10. [Final Flags](#final-flags)

---

## Scope and Approach

The challenge set simulates a realistic compromise chain:

- **Reconnaissance**: attacker scans for open services  
- **Enumeration**: attacker fingerprints network infrastructure + web application  
- **Exploitation**: vulnerable WordPress plugin is leveraged  
- **Post-exploitation**: payload is delivered and a reverse shell is established  
- **Credential theft**: database credentials are extracted from the server  

My approach for each PCAP:

1. Identify involved hosts  
2. Identify attacker activity (scan / exploit / shell / exfiltration)  
3. Confirm with protocol-level evidence  
4. Extract values required by the flag format  

---

## Tooling and Workflow

### Tools used

- **Wireshark** (primary analysis)
- **tshark** (CLI validation)
- **Python + scapy** (automation for counting / extraction)

### Standard Wireshark workflow I followed

1. **Statistics → Conversations**
   - Identify talkers, port patterns, scan behavior  
2. **Statistics → Endpoints**
   - Confirm all IPs participating  
1. Apply **progressive filters**
   - Start broad then narrow: `arp`, `tcp`, `http`, `tcp.flags.syn==1`  
4. Use **Follow Stream**
   - `Follow → TCP Stream` and `Follow → HTTP Stream` to extract human-readable content  
5. Use **Decode As…**
   - If HTTP runs on non-standard ports (payload delivery), force decode  

---

# Challenge 1 : Reconnaissance (Open Ports)
**PCAP:** `pcap1.pcapng`  
**Prompt:** Determine how many ports were found open.

## Exact steps I took

### Step 1 : Confirm scanning behavior
I suspected scanning, so I looked for SYN fan-out:

```wireshark
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

This showed one host probing many ports on a single destination.

### Step 2 : Identify scanner and target
From SYN packets:

- Scanner: `192.168.1.104`
- Target: `192.168.1.102`

### Step 3 : Determine what ports were actually open
This is the key step: **counting SYN packets is wrong**.
Open ports show **SYN/ACK** from the target.

Filter used:

```wireshark
ip.src == 192.168.1.102 && tcp.flags.syn == 1 && tcp.flags.ack == 1
```

### Step 4 : Count unique open ports
Looking at `tcp.srcport` values from SYN/ACK responses:

- 22  
- 80  

 Open ports discovered: **2**

### Flag
`KCTF{2}`

---

# Challenge 2 : Gateway Identification (Vendor)
**PCAP:** `pcap1.pcapng`  
**Prompt:** Identify the default gateway vendor.

## Exact steps I took

### Step 1 : Use ARP to find gateway MAC
Filter:

```wireshark
arp
```

Then locate `.1` mapping:

- `192.168.1.1 is-at 88:bd:09:38:d7:a0`

### Step 2 : Resolve vendor via OUI
OUI prefix:

- `88:BD:09`

Wireshark vendor resolution + OUI lookup indicates:

Vendor: **Netis**

### Flag
`KCTF{Netis}`

---

# Challenge 3 : Exploitation (App Version + Username)
**PCAP:** `pcap2.pcapng`  
**Prompt:** Determine the application targeted, then extract version + username.

## Exact steps I took

### Step 1 : Confirm web traffic exists
Filter:

```wireshark
http
```

HTTP was present.

### Step 2 : Identify application
I searched for WordPress patterns:

```wireshark
http.request.uri contains "wp-"
```

and

```wireshark
http.request.uri contains "wordpress"
```

Hits included:

- `/wordpress/`
- `/wordpress/wp-login.php`

Application = **WordPress**

### Step 3 : Extract username from login attempt
Filter POSTs:

```wireshark
http.request.method == "POST"
```

Then:

- Right click request → **Follow → HTTP Stream**

In form data:

- `log=kadmin_user`

Username = `kadmin_user`

### Step 4 : Extract WordPress version
My first attempt was to look for `readme.html`, but the quickest evidence was in the HTML fingerprint.

I followed the HTTP response to `/wordpress/`:

- **Follow HTTP Stream**
- Ensure decompression (gzip) handled by Wireshark

In the returned HTML:

Version observed = **6.9**

### Flag
`KCTF{6.9_kadmin_user}`

---

# Challenge 4 : Vulnerability Exploitation (Plugin + Version)
**PCAP:** `pcap2.pcapng`  
**Prompt:** Identify exploited plugin and its version.

## Exact steps I took

### Step 1 : Search for plugin enumeration
Filter for plugin paths:

```wireshark
http.request.uri contains "/wp-content/plugins/"
```

Found:

- `/wordpress/wp-content/plugins/social-warfare/readme.txt`

### Step 2 : Extract plugin version from readme
I followed the stream for that request:

- **Follow → HTTP Stream**

In `readme.txt` response:

- `Stable tag: 3.5.2`

Plugin = Social Warfare  
Version = 3.5.2

### Step 3 : Adjust for platform formatting
I initially used `social-warfare`, but platform expects:

`social_warfare`

### Flag
`KCTF{social_warfare_3.5.2}`

---

# Challenge 5 : Post-Exploitation (HTTP Payload Port + Reverse Shell Port)
**PCAP:** `pcap3.pcapng`  
**Prompt:** Identify payload delivery HTTP port and reverse shell port.

---

## A) Payload Delivery Port (HTTP)

### Exact steps I took

1. **Statistics → Conversations → TCP**  
   - Identify unusual non-standard ports.

2. Filter for GET strings even if not decoded as HTTP:

```wireshark
tcp contains "GET"
```

3. Found payload download:

- `192.168.1.102:40676 → 192.168.1.104:8767`
- `GET /payload.txt?...`
- Response: `HTTP/1.0 200 OK`

HTTP payload port = **8767**

---

## B) Reverse Shell Port

### Exact steps I took

1. Filter for shell-like artifacts:

```wireshark
tcp contains "bash"
```

2. Found classic reverse shell output:

- `bash: cannot set terminal process group...`

3. Verified stream endpoints:

- `192.168.1.102:39582  <->  192.168.1.104:9576`

Reverse shell port = **9576**

### Flag
`KCTF{8767_9576}`

---

# Challenge 6 : Database Credentials Theft (DB Username + Password)
**PCAP:** `pcap3.pcapng`  
**Prompt:** Extract DB username and password.

## Exact steps I took

### Step 1 : Focus on reverse shell traffic
Credentials are typically pulled via:

- `cat wp-config.php`

So I followed the reverse shell TCP stream.

### Step 2 : Search for DB config markers
Within stream I searched for:

- `DB_USER`
- `DB_PASSWORD`

I observed:

- Username: `wpuser`
- Password: `wp@user123`

### Flag
`KCTF{wpuser_wp@user123}`

---

## Automation / Script Snippets Used

### tshark : enumerate SYN/ACK open ports

```bash
tshark -r pcap1.pcapng -Y "ip.src==192.168.1.102 && tcp.flags.syn==1 && tcp.flags.ack==1"   -T fields -e tcp.srcport | sort -n | uniq -c
```

### Python (scapy) : validate open port list

```python
from scapy.all import rdpcap, TCP, IP

pkts = rdpcap("pcap1.pcapng")
target = "192.168.1.102"

open_ports = {p[TCP].sport for p in pkts
              if p.haslayer(IP) and p.haslayer(TCP)
              and p[IP].src == target
              and p[TCP].flags == "SA"}

print(sorted(open_ports))
print(len(open_ports))
```

---

## Final Flags

1. Recon open ports: `KCTF{2}`
2. Gateway vendor: `KCTF{Netis}`
3. WordPress exploitation: `KCTF{6.9_kadmin_user}`
4. Plugin exploited: `KCTF{social_warfare_3.5.2}`
5. Post-exploitation ports: `KCTF{8767_9576}`
6. DB credentials: `KCTF{wpuser_wp@user123}`

---

*End of submission.*
