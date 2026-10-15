# Nmap Scan Notes — Metasploitable2

Reconnaissance against `10.66.66.151` from Kali (`10.66.66.195`) on the isolated `labnet`.
Reference: `man nmap`, Nmap Network Scanning (Lyon).

## Scan Types — What Each Flag Actually Does

| Flag | Name | Mechanism on the wire | Root? | App logs it? |
|---|---|---|---|---|
| `-sn` | Host discovery | ARP request on same subnet; ICMP echo + TCP SYN/ACK to ports 80/443 on remote subnets | No | No |
| `-sS` | SYN / half-open | SYN → SYN-ACK → **RST**. Never completes the handshake. | Yes | No |
| `-sT` | TCP connect | Full handshake via `connect(2)` syscall: SYN → SYN-ACK → **ACK** → FIN teardown | No | Yes |
| `-sU` | UDP | UDP datagram → ICMP port-unreachable for closed; silent for open\|filtered | Yes | Varies |
| `-sV` | Version detection | NULL probe + port-specific probes, matched against nmap-service-probes DB | No | Yes |
| `-O` | OS detection | TCP/IP stack fingerprint — initial TTL, window size, option ordering, ICMP quirks | Yes | No |
| `-sC` | Default NSE scripts | Category `default` scripts: banners, shares, titles, keys | No | Yes |
| `-p-` | All ports | Scans 1–65535 TCP, not the default top 1000 | — | — |
| `-oA <prefix>` | All output formats | Writes `<prefix>.nmap`, `<prefix>.gnmap`, `<prefix>.xml` | — | — |

### `-sn` — Host discovery only

Tells nmap "find live hosts, don't scan ports." On the local `/24`, nmap uses **ARP requests** (`-PR`) rather than IP probes — Layer 2, faster, can't be blocked at the IP layer. For each of the 256 addresses, it broadcasts `who-has 10.66.66.X tell 10.66.66.195`. A MAC reply means the host is alive. Filter in Wireshark: `arp`.

### `-sS` — SYN / half-open scan

nmap's default scan. Constructs raw TCP packets with a SYN, waits for SYN-ACK, then sends **RST** instead of ACK. The server kernel sees the SYN-ACK go out but never a completed connection, so the application's `accept()` never returns and no connection is logged. Requires root because raw sockets need `CAP_NET_RAW`. This is the scan whose handshake I captured in Day 2.

Filter in Wireshark: `tcp.flags.syn==1 && ip.addr==10.66.66.151`

### `-sT` — TCP connect scan

Uses the OS's `connect()` syscall — the kernel performs a normal handshake. No raw packets, so no root required, but the application sees a completed TCP connection and logs it. This is the fallback when you don't have raw socket privileges. The wire sequence is identical to `curl` or any client connecting — SYN, SYN-ACK, ACK, then data or immediate teardown.

### `-sU` — UDP scan

Connectionless, so no handshake to observe. nmap sends a UDP datagram and interprets:

- **ICMP port unreachable** → port closed
- **No response** → `open|filtered` (can't distinguish firewall drop from silent service)
- **Any application reply** → open

Slow and unreliable against rate-limited stacks. Still critical because DNS, SNMP, and DHCP live on UDP.

### `-sV` — Service version detection

Three-phase probe per open port:

1. NULL probe — connect, listen 5 s, catch banners (FTP/SSH/SMTP announce themselves)
2. Port-probable probe — send a protocol-appropriate request (HTTP for :80, `EHLO` for :25)
3. Fallback probes — try everything else

Responses are matched against `nmap-service-probes`, a signature database. Turns "port 21 open" into "vsftpd 2.3.4" — the version string that lets you search CVE databases.

### `-O` — OS detection

Fingerprints the kernel by comparing responses to unusual TCP flag combinations, initial TTL, window size, and IP options ordering against `nmap-os-db`. Useful for choosing exploit strategy — a Linux target with Samba calls for `usermap_script`, a Windows target with SMB calls for EternalBlue.

### `-sC` — Default NSE scripts

Runs Nmap Scripting Engine scripts in the `default` category — safe, non-intrusive checks. Examples that fire on Metasploitable2:

- `ftp-anon` — anonymous FTP login check
- `smb-enum-shares` — Samba share listing
- `http-title` — page title of web roots
- `ssh-hostkey` — SSH host key fingerprint

Not exploits — misconfiguration detection. But they reveal the paths that lead to exploits.

## `-sS` vs `-sT` — The Wire-Level Difference

The comparison exercise for Block 1. Both scans were captured with `tcpdump -i eth0 -w <file>.pcap` while running.

### `-sS` (half-open)

```
Kali                                    Meta2:80
  │──── SYN, seq=X ─────────────────────▶│
  │◀─── SYN-ACK, seq=Y, ack=X+1 ─────────│
  │──── RST ─────────────────────────────▶│  ← nmap aborts; never sends ACK
```

Packet 3 is **RST**, not ACK. This is the defining feature. The server never sees a completed handshake, so the application's connection is never accepted and never logged.

### `-sT` (full connect)

```
Kali                                    Meta2:80
  │──── SYN, seq=X ─────────────────────▶│
  │◀─── SYN-ACK, seq=Y, ack=X+1 ─────────│
  │──── ACK, seq=X+1, ack=Y+1 ──────────▶│  ← handshake completes
  │──── FIN ─────────────────────────────▶│  ← graceful teardown
```

Packet 3 is **ACK**. The handshake completes exactly as it does with `curl`. The application sees an accepted connection and (depending on config) logs the source IP.

**Screenshots:** `screenshots/nmap-sS-rst.png`, `screenshots/nmap-sT-ack.png`

## Open Ports on Metasploitable2

Result of `sudo nmap -sC -sV -p- -oA scans/msf2-full 10.66.66.151`.

| Port | Proto | Service | Version | Vulnerable? Why |
|---|---|---|---|---|
| 21 | tcp | ftp | vsftpd 2.3.4 | **CVE-2011-2523** — backdoor triggered by `:)` in username. Root shell on port 6200. |
| 22 | tcp | ssh | OpenSSH 4.7p1 (Debian) | Old but hardened. Default-credential and key-reuse attack surface. |
| 23 | tcp | telnet | Linux telnetd | Plaintext protocol — creds sniffable. No cryptographic auth. |
| 25 | tcp | smtp | Postfix smtpd | `VRFY`/`EXPN` for user enumeration. Open relay check. |
| 53 | tcp/udp | domain | ISC BIND 9.4.2 | Check AXFR zone transfer; enumeration of internal names. |
| 80 | tcp | http | Apache 2.2.8 (Ubuntu) | Hosts DVWA, Mutillidae, phpMyAdmin, TWiki. Web app surface. |
| 111 | tcp | rpcbind | 2 (RPC #100000) | Enumerate RPC services via `rpcinfo -p` for NFS, NIS. |
| 139 | tcp | netbios-ssn | Samba 3.x | Legacy NetBIOS. Same Samba codebase as 445. |
| 445 | tcp | netbios-ssn | Samba 3.0.20-Debian | **CVE-2007-2447** — `usermap_script` username field command injection. Root shell. |
| 512 | tcp | exec | netkit-rsh rexecd | Trust-based auth. `rlogin`/`rsh` often permit passwordless root. |
| 513 | tcp | login | rlogind | Same trust model. |
| 514 | tcp | shell | Netkit rshd | Same. |
| 1099 | tcp | java-rmi | GNU Classpath grmiregistry | Java RMI deserialization surface. |
| 1524 | tcp | bindshell | Metasploitable root shell | **Already a root shell.** `nc 10.66.66.151 1524` gives root. Intentional free win. |
| 2049 | tcp | nfs | 2-4 (RPC #100003) | Share often mounted `no_root_squash` — mount and become root on the export. |
| 2121 | tcp | ftp | ProFTPD 1.3.1 | **CVE-2010-4221** — stack buffer overflow in `pr_netio_telnet_gets`. |
| 3306 | tcp | mysql | 5.0.51a-3ubuntu5 | Default `root:` (empty) on some builds. |
| 5432 | tcp | postgresql | 8.3.0-8.3.7 | Default `postgres:postgres`. |
| 5900 | tcp | vnc | VNC protocol 3.3 | Default password `password`. |
| 6000 | tcp | X11 | Access denied | Exposed X server — attack if auth is weak. |
| 6667 | tcp | irc | UnrealIRCd | **CVE-2010-2075** — backdoored build; `AB;` in payload triggers command execution. |
| 8009 | tcp | ajp13 | Apache Jserv 1.3 | Tomcat AJP protocol — path to Tomcat internals. |
| 8180 | tcp | http | Apache Tomcat/Coyote 1.1 | Manager app `tomcat:tomcat` — WAR deployment → code exec. |

## Commands Run

```bash
# Host discovery
sudo nmap -sn 10.66.66.0/24

# Basic scan types
sudo nmap -sS 10.66.66.151
nmap -sT 10.66.66.151
sudo nmap -sU --top-ports 20 10.66.66.151

# Detection
sudo nmap -sV 10.66.66.151
sudo nmap -O 10.66.66.151

# Full scan, all ports, default scripts, three output formats
mkdir -p scans
sudo nmap -sC -sV -p- -oA scans/msf2-full 10.66.66.151
```

## Scan Output Files

- `scans/msf2-full.nmap` — human-readable
- `scans/msf2-full.gnmap` — greppable
- `scans/msf2-full.xml` — structured (importable to Metasploit, Faraday)

## Key Takeaways

- **`-sS` leaves no application-layer trace** because the handshake never completes. Preferred for stealth.
- **`-sT` is the fallback without root** but logs a connection for every open port.
- **`-sV` is what turns ports into targets** — the version string is the pivot to CVE lookup.
- **Always use `-p-`** against targets like Metasploitable2. Ports 1524, 6667, 8180 are only found on a full scan.
- **`-sC` is safe.** It doesn't exploit anything, only enumerates. It should always be in a first-pass scan.

## Next Steps

Exploitation phase, one technique per file under `metasploitable2/`. First target: vsftpd 2.3.4 (CVE-2011-2523).
