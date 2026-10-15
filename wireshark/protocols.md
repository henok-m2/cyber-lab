# Wireshark Protocol Analysis

Three protocols captured between Kali (`10.66.66.195`) and Metasploitable2 (`10.66.66.151`)
on the isolated lab network.

## ICMP — Ping

**Filter:** `icmp`

**Capture:** `ping -c 4 10.66.66.151` from Kali.

**What's on the wire:** Four ICMP echo requests (Type 8) from `.195` to `.151`,
each followed by an echo reply (Type 0) in the opposite direction. The identifier
and sequence number fields match request/reply pairs.

**Annotation:** ICMP is a Layer 3 diagnostic protocol — it carries no application
data. Attackers use it for host discovery (ping sweeps). Defenders often block it
at the perimeter to frustrate scanning, which is why modern discovery also relies
on TCP SYN pings or ARP requests.

## TCP Three-Way Handshake — HTTP

**Filter:** `tcp.port == 80 && ip.addr == 10.66.66.151`

**Capture:** `curl -v http://10.66.66.151/` from Kali.

**What's on the wire:**

1. **SYN** — Kali to Meta2, random ISN (`X`), flags `0x002`
2. **SYN-ACK** — Meta2 to Kali, own random ISN (`Y`), ack `X+1`, flags `0x012`
3. **ACK** — Kali to Meta2, seq `X+1`, ack `Y+1`, flags `0x010`
4. **HTTP GET** — Kali to Meta2, port 80, the request
5. **HTTP 200 OK** — Meta2 to Kali, the response
6. **FIN/ACK teardown** — both directions

**Annotation:** The handshake is the substrate for TCP SYN scanning (`nmap -sS`),
which sends a SYN and infers port state from the response (SYN-ACK = open, RST =
closed) without ever completing the connection. The application never sees a full
connection, so nothing is logged at the application layer. Random ISNs (RFC 6528)
prevent attackers from predicting sequence numbers and injecting forged segments.

## DNS — Query and Negative Response

**Filter:** `dns`

**Capture:** `nslookup google.com 10.66.66.1` from Kali.

**What's on the wire:**

1. **Query** — Kali to `10.66.66.1:53`, QNAME `google.com`, QTYPE A, QCLASS IN
2. **Response** — dnsmasq to Kali, RCODE `REFUSED` (5)

**Annotation:** `REFUSED` means the DNS server received the query but is configured
not to answer. dnsmasq has no upstream resolvers — the isolated network has no path
to the internet, so external names cannot be resolved. This is a design property,
not a failure.

Compare with a **successful local resolution:** `nslookup metasploitable2 10.66.66.1`
returns an A record `10.66.66.151`, sourced from the DHCP lease table.

**Security relevance:** DNS is a frequent exfiltration channel (tunneling) and a
reconnaissance tool (zone transfers via AXFR, subdomain enumeration). Reading its
wire format fluently is a prerequisite for spotting malicious DNS traffic in
packet captures.
