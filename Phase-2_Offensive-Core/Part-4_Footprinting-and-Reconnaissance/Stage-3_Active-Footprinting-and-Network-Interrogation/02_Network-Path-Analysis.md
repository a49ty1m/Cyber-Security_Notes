# Network Path Analysis

**Roadmap:** Phase 2: Offensive Core → Part 4: Footprinting and Reconnaissance → Stage 3: Active Footprinting & Network Interrogation

# Section 1 — What it is and where it sits

Network path analysis maps the route that packets take from your machine to the target, identifying every intermediary router and network device along the way. The output is not just a list of hops — it is a topology inference: you can deduce where the perimeter firewall sits, what the DMZ structure looks like, whether a load balancer or CDN intercepts traffic, and what addressing scheme is used on the target's internal network.

The fundamental technique is traceroute — exploiting the IP TTL (Time to Live) field to elicit ICMP Time Exceeded responses from each hop along the path. The target never has to respond at all; intermediate routers do the revealing.

```text
Attack Chain
──────────────────────────────────────────────────────────────────────
Stage 3 (Active)
  Live Host Discovery  →  [Network Path Analysis]  →  Port Scanning
      ↑ done                  ↑ YOU ARE HERE            ↓ next
                                                    Service Enum
Tools: traceroute, tracert, mtr, hping3, nmap --traceroute
──────────────────────────────────────────────────────────────────────
```

The intelligence value of traceroute is proportional to your ability to interpret what the hops tell you about network architecture. Raw hop output is not useful. Classified, correlated hop output — "this RFC1918 address is the target's internal load balancer, and it's two hops behind the CDN edge" — directly shapes how you approach port scanning and service enumeration.

---

# Section 2 — How attackers actually use this

## 2.1 TTL mechanics and how traceroute works

Every IP packet contains a TTL field set by the sender (typically 64 for Linux, 128 for Windows). Each router that forwards the packet decrements the TTL by 1. When TTL reaches 0, the router discards the packet and sends an ICMP Time Exceeded (Type 11, Code 0) message back to the original sender — containing its own IP address in the response.

Traceroute exploits this by sending a sequence of probes with incrementing TTL values: TTL=1 expires at the first hop (router 1 responds), TTL=2 expires at the second hop (router 2 responds), and so on until a probe reaches the target, which responds normally.

```text
TTL=1:  Your machine → Router1 → [TTL expires] → Router1 sends ICMP Time Exceeded (reveals its IP)
TTL=2:  Your machine → Router1 → Router2 → [TTL expires] → Router2 sends ICMP Time Exceeded
TTL=3:  Your machine → ... → Router3 → [TTL expires] → Router3 reveals its IP
TTL=N:  Your machine → ... → Target → [Port closed/open] → Target responds → done
```

## 2.2 Reading hop output for topology inference

A traceroute to an external corporate target typically reveals:

```bash
$ traceroute 203.0.113.45
 1  192.168.1.1 (192.168.1.1)      1.2ms    ← your home/office router
 2  10.x.x.1                        8.4ms    ← ISP DSLAM/CMTS
 3  203.0.113.1                     14.2ms   ← ISP core router
 4  198.51.100.5                    16.1ms   ← ISP transit or peering point
 5  203.0.113.254                   18.9ms   ← target org's upstream router (perimeter)
 6  * * *                                    ← firewall drops probes (filtered hop)
 7  * * *                                    ← still filtered
 8  203.0.113.45                    22.4ms   ← target host responds
```

**Interpreting this output:**
- Hop 5 (`203.0.113.254`) is the last external hop before filtering begins — this is the target's upstream router/transit provider
- Hops 6–7 (`* * *`) are filtered: packets are reaching those devices but the devices don't send ICMP Time Exceeded back (firewall suppresses ICMP or routes change)
- The two `* * *` hops between the upstream router and the target suggest: perimeter firewall (hop 6) and possibly a DMZ router or load balancer (hop 7) before the actual target
- The target at hop 8 responds — it is not behind an ICMP-blocking host firewall

**RFC1918 hops in an external trace:**
```bash
 5  203.0.113.254   18.9ms   ← external
 6  10.10.0.1       19.2ms   ← RFC1918! Internal network address leaking into traceroute
 7  10.10.0.10      19.8ms   ← another internal hop
 8  203.0.113.45    22.4ms   ← target
```
An RFC1918 address appearing in a traceroute to an external target is a misconfiguration: the internal router is sending ICMP Time Exceeded replies that leak its internal IP. This reveals the internal IP addressing scheme. `10.10.0.1` and `10.10.0.10` are the internal router IPs between the perimeter and the target.

## 2.3 Probe types: UDP vs TCP vs ICMP traceroute

Standard traceroute uses UDP probes to high port numbers (33434+) by default on Linux. Windows `tracert` uses ICMP Echo. TCP-based traceroute sends TCP SYN packets. Each behaves differently at firewall boundaries:

| Probe type | Flag | Firewall behavior | Best for |
|------------|------|------------------|---------|
| UDP (default Linux) | `traceroute` | UDP to high ports often blocked — many hops show `* * *` | General path mapping |
| ICMP Echo (Windows default) | `tracert` or `-I` | ICMP often blocked at perimeters | Basic path check |
| TCP SYN to open port | `-T -p 80` | TCP SYN to allowed ports passes through firewalls | Firewall-bypass path mapping |
| TCP SYN to port 443 | `-T -p 443` | HTTPS almost always permitted | Mapping past strict firewalls |

TCP traceroute to an open port (typically 80 or 443) produces the most complete hop picture against hardened targets — because the firewall allows TCP SYN to that port, so each intermediate hop is visible rather than filtered.

```bash
# TCP traceroute to port 80 (passes through web-permitting firewalls)
$ sudo traceroute -T -p 80 203.0.113.45

# TCP traceroute to port 443
$ sudo traceroute -T -p 443 203.0.113.45

# ICMP traceroute
$ traceroute -I 203.0.113.45

# UDP (Linux default)
$ traceroute 203.0.113.45
```

## 2.4 Detecting network boundaries from hop analysis

The pattern of IP addresses across hops reveals network architecture segments:

```text
Perimeter boundary:
  → Last public IP hop → * * * (filtered) → target's public IP
  → Interpretation: firewall between upstream router and DMZ

DMZ detection:
  → Public IP → RFC1918 10.x.x.x hop → another RFC1918 → target's public IP
  → Interpretation: internal routing between firewall and server is RFC1918
  → The RFC1918 hops are the firewall's internal interface and DMZ switch

Load balancer detection:
  → Same destination IP, but different intermediate hops on repeated traces
  → Or: TTL changes unexpectedly, suggesting asymmetric routing
  → Use mtr to run continuous traces and observe hop consistency

CDN/anycast detection:
  → First 3-5 hops reach a CDN PoP IP (verify against CDN IP ranges)
  → Remaining hops reach the CDN PoP — the real origin is not in the path
  → Target is behind CDN: path analysis cannot reveal origin server location
```

## 2.5 mtr — continuous path + latency analysis

`mtr` (Matt's Traceroute) combines traceroute and ping into a real-time continuously updating display. It shows: per-hop packet loss percentage, average latency, best and worst latency, and standard deviation. This reveals latency spikes and asymmetric routing that single-pass traceroute misses.

```bash
$ mtr 203.0.113.45

                                  My traceroute [v0.94]
  Keys: Help Display mode Restart Statistics Order of fields quit
                                           Packets              Pings
 Host                                   Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. 192.168.1.1                          0.0%    20    1.2   1.3   1.0   1.8   0.2
 2. 10.200.0.1                           0.0%    20    8.1   8.4   7.9   9.2   0.3
 3. 203.0.113.1                          0.0%    20   14.1  14.3  13.8  15.0   0.3
 4. 198.51.100.5                        20.0%    20   15.9  16.0  15.5  17.2   0.4  ← 20% loss
 5. ???                               100.0%    20    0.0   0.0   0.0   0.0   0.0   ← filtered
 6. 203.0.113.45                         0.0%    20   22.1  22.4  21.8  23.5   0.4
```

Hop 4 showing 20% packet loss: this device is rate-limiting or deprioritizing ICMP Time Exceeded responses — a common configuration on router ACLs. It does not mean the path is dropping 20% of actual user traffic. The real target (hop 6) shows 0% loss.

Hop 5 showing 100% loss (`???`): filtered, as before. mtr makes it obvious that hop 5 is consistently filtered across all 20 probe packets — confirming a stateful firewall silently dropping TTL-expired responses.

## 2.6 Firewalk — firewall rule inference without traceroute

Firewalk is an advanced technique that uses TTL manipulation to probe whether specific ports are allowed through a firewall without actually reaching the target. By setting TTL to expire exactly at the firewall, you send probes that die at the firewall and observe whether the router behind the firewall (one hop further) sends an ICMP Time Exceeded.

```text
Logic:
  If TTL set to expire at router behind the firewall:
    → Firewall allows probe → packet reaches next router → ICMP Time Exceeded received
    → Firewall blocks probe → packet dies at firewall → no ICMP Time Exceeded
```

In practice, firewalk requires knowing the exact hop count to the gateway — which you get from standard traceroute. Modern firewalls often suppress ICMP Time Exceeded responses making this technique unreliable, but it is theoretically sound for older firewall architectures.

## 2.7 Interpreting latency and path anomalies

Beyond hop IP addresses, latency data reveals network characteristics:

- **Sudden latency increase at a hop:** The packet is crossing a geographic or network boundary. A jump from 5ms to 45ms indicates a transatlantic or transcontinental link — the server is in a different country or region from what DNS suggested.
- **Latency decrease at a later hop:** Asymmetric routing — the return path is different from the forward path. The hop showing lower latency than previous hops is responding from a different network segment on the return journey.
- **Large standard deviation at a hop:** Queue depth variability — the router at that hop is under variable load. Does not indicate a security finding but does indicate a congested or rate-limited path.
- **All hops after hop N show `* * *` but target responds:** Hops N+1 through target all suppress ICMP Time Exceeded (firewall rule: drop ICMP TTL-exceeded), but the target itself responds to the final probe. This means the path exists but the internal hops are invisible — common for well-configured corporate firewalls.

## 2.8 Load balancer and anycast detection from path inconsistency

Load balancers and anycast routing both produce inconsistent path data when the same destination is traced multiple times. A standard unicast destination routes the same way every time — the same intermediate hops, the same latency. A load-balanced or anycast destination may show different hop patterns on each trace.

**Load balancer detection:**
When a VIP (Virtual IP) sits in front of multiple real servers, successive connections may route to different backend hosts. This shows up in traceroute as: consistent hops up to the load balancer, then diverging final hops or different response RTTs on repeated traces.

```bash
# Run the same TCP traceroute 3 times in rapid succession
$ for i in 1 2 3; do
    sudo traceroute -T -p 80 -n 203.0.113.45 2>/dev/null | tail -5
    echo "--- run $i ---"
  done

--- run 1 ---
 6  10.10.0.5     19.2ms   ← internal hop before target
 7  203.0.113.45  22.4ms
--- run 2 ---
 6  10.10.0.7     19.8ms   ← DIFFERENT internal hop!
 7  203.0.113.45  22.6ms
--- run 3 ---
 6  10.10.0.5     19.1ms   ← back to first path
 7  203.0.113.45  22.3ms
# Conclusion: 10.10.0.5 and 10.10.0.7 are two backend servers
# behind a load balancer at 203.0.113.45
```

The alternating internal hop (`10.10.0.5` / `10.10.0.7`) confirms a load balancer with at least two backend servers. Both IPs are internal addresses of real application servers — the load balancer is the attack surface for initial access, but the backends are the actual targets for persistence and lateral movement post-exploitation.

**Anycast detection:**
CDN and DNS provider infrastructure uses anycast routing: the same IP is announced from dozens of global locations. A traceroute terminates at the nearest PoP, not the origin server. Anycast detection:

```bash
# Trace from two different source locations (use VPN/proxy on different continent)
# From Asia: trace terminates at Singapore PoP in 3 hops (12ms)
# From Europe: same IP, trace terminates at Frankfurt PoP in 4 hops (9ms)
# → IP is anycast — you are hitting the CDN edge, not the real origin

# Confirm with CDN IP range check
$ python3 -c "
import ipaddress, urllib.request, json
ip = ipaddress.ip_address('203.0.113.45')
data = json.loads(urllib.request.urlopen('https://ip-ranges.amazonaws.com/ip-ranges.json').read())
for p in data['prefixes']:
    if ip in ipaddress.ip_network(p['ip_prefix']):
        print(f'AWS CloudFront: {p[\"ip_prefix\"]} region={p[\"region\"]}')
"
```

If the IP is confirmed as an anycast CDN edge, path analysis cannot reach the origin server — all traceroute probes terminate at the nearest PoP. The origin IP must be found through historical DNS (Stage 2 SecurityTrails technique) or misconfigured headers.

## 2.9 Correlating traceroute data with BGP and Shodan intelligence

Traceroute hop IPs are rarely in isolation — each can be enriched with ASN ownership, geographic location, and Shodan service data to build a richer picture of the network architecture.

```bash
# Enrich each hop IP with ASN/org information
$ traceroute -n 203.0.113.45 | grep -oP '\d+\.\d+\.\d+\.\d+' \
  | while read ip; do
    asn=$(curl -s "https://api.hackertarget.com/aslookup/?q=$ip" 2>/dev/null)
    echo "$ip → $asn"
  done

203.0.113.1   → AS12345 | ISP Transit Corp | US
198.51.100.5  → AS12345 | ISP Transit Corp | US
198.51.100.1  → AS67890 | Corp-Target LLC | US         ← org's own ASN starts here
10.10.0.5     → private address (RFC1918)

# The transition from AS12345 (ISP) to AS67890 (target org) marks the network boundary
# 198.51.100.1 is the target org's upstream router — the perimeter

# Cross-reference each hop with Shodan
$ shodan host 198.51.100.1
IP: 198.51.100.1
Organization: Corp-Target LLC
Open ports: 22 (SSH), 179 (BGP)              ← BGP on port 179 = router confirmed
Hostname: router1.corp-target.com

# Traceroute hop shows BGP on port 179 → this is a BGP-peering edge router
# Understanding: AS67890 peers with ISP AS12345 at 198.51.100.1

# Geographic tracing from latency
# Rule of thumb: light travels ~200km per millisecond in fiber
# 45ms RTT to a hop → hop is within ~4500km of your source
# 4ms RTT → hop is within ~400km — likely in the same country/region
```

Combining ASN data with traceroute: the exact hop where the ASN transitions from the ISP's ASN to the target organization's ASN is the network perimeter — the outermost device the target organization controls. Every hop in the target's ASN is their infrastructure.

---

# Section 3 — Core concepts and terminology


| Term | Definition |
|------|-----------|
| **TTL (Time to Live)** | IP header field decremented by 1 at each router hop; when it reaches 0, the router sends ICMP Time Exceeded and discards the packet |
| **ICMP Time Exceeded (Type 11)** | The ICMP message sent by a router when it receives a packet with TTL=0; contains the router's IP address |
| **Traceroute** | A tool that sends successive probes with increasing TTL values (1, 2, 3...) to map each hop on the path to a destination |
| **tracert** | Windows implementation of traceroute; uses ICMP Echo probes rather than UDP |
| **mtr** | Matt's Traceroute — combines traceroute and ping into a continuously updating live display with per-hop loss and latency stats |
| **Hop** | Each router or network device that forwards a packet along the path from source to destination |
| **`* * *`** | Traceroute notation indicating a probe expired at a hop that did not send ICMP Time Exceeded — either filtered or silently dropped |
| **RFC 1918 leak** | An internal/private IP address (10.x, 172.16.x, 192.168.x) appearing in a traceroute to an external target, revealing internal network addressing |
| **DMZ (Demilitarized Zone)** | A network segment between the external internet and the internal private network, containing publicly accessible servers |
| **Asymmetric routing** | When the forward path (source to destination) and return path (destination to source) differ — common in large networks |
| **Anycast** | An IP addressing scheme where the same IP is announced from multiple locations globally; nearest-node routing; makes traceroute terminate at the nearest PoP rather than the actual target |
| **TCP traceroute** | Traceroute using TCP SYN probes instead of UDP/ICMP; more likely to pass through firewalls that allow specific TCP ports |
| **Firewalk** | Technique using TTL manipulation to probe whether a firewall allows specific ports, without packets reaching the target |
| **PoP (Point of Presence)** | A CDN or ISP network location where traffic is handled before forwarding to the origin |

---

# Section 4 — Tools and commands

| Tool | Command | What it shows | When to use |
|------|---------|--------------|------------|
| `traceroute` | `traceroute 203.0.113.45` | UDP-based path (Linux default) | Standard external path map |
| `traceroute -I` | `traceroute -I 203.0.113.45` | ICMP-based path | When UDP is more filtered than ICMP |
| `traceroute -T -p 80` | `sudo traceroute -T -p 80 203.0.113.45` | TCP path via port 80 | Firewall-bypass path mapping |
| `tracert` (Windows) | `tracert 203.0.113.45` | ICMP path (Windows) | Windows system tracing |
| `mtr` | `mtr 203.0.113.45` | Live continuous trace + loss/latency | Latency analysis, loop detection |
| `mtr -r -n` | `mtr -r -n -c 20 203.0.113.45` | Report mode, no DNS, 20 cycles | Non-interactive mtr for scripts |
| `hping3 --traceroute` | `sudo hping3 --traceroute -V -I 203.0.113.45` | ICMP traceroute via hping3 | Custom probe traceroute |
| `nmap --traceroute` | `nmap --traceroute -sn 203.0.113.45` | nmap-integrated path | Combine with host discovery |

**Standard traceroute + TCP comparison:**
```bash
# UDP traceroute (many hops show * * *)
$ traceroute 203.0.113.45
 1  192.168.1.1    1.1ms
 2  10.200.0.1     8.3ms
 3  203.0.113.1   14.1ms
 4  * * *                  ← UDP blocked
 5  * * *                  ← UDP blocked
 6  * * *                  ← UDP blocked

# TCP traceroute to port 80 (passes through web-allowing firewall)
$ sudo traceroute -T -p 80 203.0.113.45
 1  192.168.1.1    1.1ms
 2  10.200.0.1     8.3ms
 3  203.0.113.1   14.1ms
 4  198.51.100.5  16.0ms   ← ISP transit — now visible with TCP
 5  203.0.113.254 18.8ms   ← target org upstream router
 6  * * *                   ← firewall (silences ICMP TTL-exceeded)
 7  203.0.113.45  22.4ms   ← target responds
```
TCP traceroute reveals 2 more hops than UDP — the ISP transit (hop 4) and the target's upstream router (hop 5) — because the firewall allows TCP 80 through, so those hops can respond with ICMP Time Exceeded. Only the firewall itself (hop 6) is silent.

**mtr for latency and loss analysis:**
```bash
$ mtr -r -n -c 30 203.0.113.45

Start: 2024-08-27T22:15:00+0530
HOST: kali                         Loss%   Snt  Last   Avg  Best  Wrst StDev
  1.|-- 192.168.1.1                  0.0%    30   1.1   1.2   0.9   1.8   0.2
  2.|-- 10.200.0.1                   0.0%    30   8.1   8.3   7.8   9.5   0.3
  3.|-- 203.0.113.1                  0.0%    30  14.0  14.2  13.7  15.1   0.3
  4.|-- 198.51.100.5                16.7%    30  15.8  16.1  15.4  17.8   0.5  ← ICMP rate-limited
  5.|-- ???                        100.0%    30   0.0   0.0   0.0   0.0   0.0  ← firewall silent
  6.|-- 203.0.113.45                 0.0%    30  22.0  22.3  21.5  23.8   0.5  ← target healthy
```
Hop 4's 16.7% loss is ICMP rate-limiting on the transit provider's router — not actual path packet loss. Real end-to-end loss is 0% (target shows 0%). Topology: ISP → transit → firewall (silent) → target.

**RFC1918 internal leak detection:**
```bash
$ traceroute 203.0.113.45
 4  198.51.100.5   15.8ms   ← last public hop before the org
 5  10.0.0.1       16.1ms   ← INTERNAL IP — firewall's internal interface leaked
 6  10.10.0.5      16.4ms   ← DMZ switch or internal router
 7  203.0.113.45   22.4ms   ← target

# Document: internal range appears to be 10.0.0.0/8
# Firewall internal interface: 10.0.0.1
# DMZ segment: 10.10.0.0/24 (inferred from 10.10.0.5)
```

---

# Section 5 — Defender detection

- **ICMP traceroute visibility:** Every ICMP probe in a traceroute reaches the target's upstream routers and network edge. Routers log traffic — a burst of ICMP packets with TTL values incrementing from 1 to 20+ in rapid sequence is a distinctive traceroute signature visible in router syslog and any network flow collection system (NetFlow, sFlow, IPFIX).
- **TCP traceroute detection:** TCP SYN packets with incrementing TTL values are not a common traffic pattern in normal web traffic. A stateful IDS inspecting TCP can detect the TTL-manipulation pattern.
- **ICMP Time Exceeded suppression:** Well-configured network devices suppress ICMP Time Exceeded responses — the operator gets `* * *` instead of the router's IP. This is the correct defender behavior and prevents topology leakage from traceroute.
- **RFC1918 leak is a misconfiguration:** If a defender's routers are leaking RFC1918 addresses in ICMP Time Exceeded, it is a router ACL misconfiguration. Defenders should block outbound ICMP Time Exceeded from internal router interfaces.
- **mtr and continuous probing:** mtr sending 30+ packets per hop to the same destination is more detectable than a single-pass traceroute. Use `-c 10` to limit probes per hop.
- **Mitigation for operators:** Use TCP traceroute to a port confirmed open (from passive Shodan data) — this is the most firewall-friendly probe type and the least distinctively "recon" looking. Rate-limit with `--sendwait` in traceroute to space probes.

---

# Section 6 — Lab task

**Platform:** Kali Linux. Target: any public internet host you're authorized to test, or a VirtualBox internal network with a router VM between Kali and Metasploitable.

**Objective:** Map the network path to a target using multiple traceroute methods and produce an annotated topology diagram.

**Steps:**

1. **Standard UDP traceroute:** `traceroute -n 203.0.113.45 | tee udp_trace.txt` — note which hops show `* * *`
2. **ICMP traceroute:** `sudo traceroute -I -n 203.0.113.45 | tee icmp_trace.txt` — compare to UDP
3. **TCP traceroute to port 80:** `sudo traceroute -T -p 80 -n 203.0.113.45 | tee tcp80_trace.txt`
4. **TCP traceroute to port 443:** `sudo traceroute -T -p 443 -n 203.0.113.45 | tee tcp443_trace.txt`
5. **mtr report:** `mtr -r -n -c 20 203.0.113.45 | tee mtr_report.txt`
6. **Compare:** Which probe type revealed the most complete path? Which had the most `* * *` hops?
7. **RFC1918 check:** `grep "10\.\|172\.1[6-9]\.\|172\.2[0-9]\.\|172\.3[01]\.\|192\.168\." udp_trace.txt icmp_trace.txt tcp80_trace.txt` — any private IPs leaking?
8. **Identify the boundary hops:** What is the last public IP hop before the target? Is there a filtered section suggesting a firewall?
9. **Latency analysis from mtr:** Is there a hop with significantly higher latency than adjacent hops? What does that suggest about that network segment?
10. **Draw topology:** Produce a text diagram showing: your machine → ISP hops → last public hop → firewall (inferred) → target. Label each layer.

```bash
git commit -m "recon(stage3): network path analysis — topology map for <target>"
```

---

# Section 7 — Common mistakes

**1. Treating `* * *` hops as definitive evidence of a dead hop**
_Why it matters:_ `* * *` means no ICMP Time Exceeded was received — not that the hop doesn't exist. The device at that hop may be forwarding packets perfectly while suppressing ICMP TTL-exceeded replies. The path continues past the `* * *` hops to a responding target.
_Fix:_ Never conclude a device doesn't exist because it shows `* * *`. If the trace reaches the target eventually, all intermediate hops were functional — some just didn't send ICMP Time Exceeded.

**2. Using only UDP traceroute on a target with firewall-blocked UDP**
_Why it matters:_ Linux default traceroute uses UDP to high ports (33434+). Corporate firewalls commonly block outbound UDP to high ports. Every hop after the firewall shows `* * *`, giving the false impression of no path.
_Fix:_ Always try TCP traceroute (`-T -p 80` and `-T -p 443`) after UDP. TCP to allowed ports passes through the firewall and reveals hops hidden by UDP filtering.

**3. Not noticing RFC1918 addresses leaking in the trace**
_Why it matters:_ Internal private addresses in a traceroute to an external target reveal internal network topology. This is a high-value OSINT finding that operators frequently scroll past.
_Fix:_ After any traceroute, grep the output for RFC1918 ranges: `grep -E "^[[:space:]]+[0-9]" trace.txt | grep -E "10\.|172\.1[6-9]\.|172\.2[0-9]\.|172\.3[01]\.|192\.168\."`.

**4. Ignoring latency anomalies in mtr output**
_Why it matters:_ A sudden latency jump of 30-40ms at a single hop indicates a geographic or network boundary crossing. This tells you where the server actually is, which may differ from what whois records suggest.
_Fix:_ Note latency jumps in the mtr output. A 5ms → 45ms jump at a specific hop means the packet crossed an ocean or continent at that point.

**5. Concluding topology from a single probe type**
_Why it matters:_ UDP, ICMP, and TCP traceroutes all show different hops depending on what the firewall allows. Relying on one shows an incomplete picture.
_Fix:_ Run all three probe types and overlay the results. The combined view gives the most complete path picture.

---

# Section 8 — Move-on gate

1. A traceroute to a target shows 3 hops then `* * * * * *` then the target responds at hop 8. Draw (in text) the network topology you can infer from this output. Label each layer: your router, ISP, target upstream, filtered zone, target.

2. Explain why TCP traceroute to port 443 often reveals more hops than standard UDP traceroute to the same target. Name the exact nmap and traceroute commands to perform TCP traceroute.

3. A traceroute shows the IP `10.10.5.1` appearing at hop 6 in a trace to a public external IP. What is this called, what does it reveal about the target network, and what follow-on intelligence can you extract from this finding?
