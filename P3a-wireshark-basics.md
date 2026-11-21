# Project 3a: Wireshark Basics (HTTP and Telnet Captures)

**Goal:** Learn to read packet captures in Wireshark: navigate the panes, filter traffic, rebuild conversations with Follow TCP Stream, and show why cleartext protocols expose credentials.

**Skills shown:** packet analysis · Wireshark display filters · TCP handshake · Follow TCP Stream · HTTP request/response reading · cleartext vs encrypted protocols · traffic overview with Statistics

---

## Setup

- **Tool:** Wireshark (installed with Homebrew: `brew install --cask wireshark`) on macOS.
- **Data:** two public sample captures from the Wireshark project's wiki: `http.cap` (one web page load) and `telnet-cooked.pcap` (a Telnet login).
- **Safety:** benign public captures only. Wireshark only reads them; nothing was executed or extracted.

## Capture 1: http.cap (a web page load)

| Question | Finding |
|---|---|
| Packets in the capture | 43 |
| Client (sends the first SYN) | 145.254.160.237 |
| Server | 65.208.228.223 |
| Handshake | SYN → SYN, ACK → ACK, then the HTTP request |
| Request | `GET /download.html HTTP/1.1` (method, path, protocol version) |
| Host | `www.ethereal.com` (Ethereal was Wireshark's original name) |
| User-Agent | `Mozilla/5.0 (...)`: identifies the browser and OS |
| Response | `HTTP/1.1 200 OK` |
| Statistics → Conversations (IPv4) | 3 conversations; the largest (~21 KB) was 145.254.160.237 ↔ 65.208.228.223 (the web page itself) |

**Filters used:**
```
http                 requests and responses
http.request         only the client's GET/POST requests
ip.addr == <IP>      traffic to or from one host
tcp.port == 80       one port, either direction
```

**Follow TCP Stream** rebuilt the whole conversation in order: red = what the client sent, blue = what the server sent back.

## Capture 2: telnet-cooked.pcap (cleartext login)

Filtered on `telnet` and followed the TCP stream.

- The **username and password were fully readable** in plain text (public sample values, redacted here).
- The login **succeeded**: a shell prompt followed, and the user ran commands such as `/sbin/ping www.yahoo.com`.
- Anyone who can capture this traffic gets the same credentials. Telnet (port 23) encrypts nothing.

**Comparison:** over SSH (port 22), the same capture would show only **metadata** (IPs, ports, timing, amount of data). The username, password, and commands would be encrypted.

## SOC takeaways

- Logs show **that** something happened; a PCAP shows **what** was actually sent.
- Order of work on a big capture: **overview** (Statistics → Conversations, Protocol Hierarchy) → **filter** to the suspicious host → **follow** the stream.
- Credentials seen in cleartext are **exposed** and should be reset, whoever used them. The fix is to disable the cleartext protocol and use the encrypted version (Telnet → SSH, FTP → SFTP, HTTP → HTTPS).

## What I learned

This project taught me how to analyze captured packets. I learned how to filter for addresses, ports, and protocols. I learned that packets are like pieces of a sentence: by following the TCP stream, you can read the entire conversation, and if the protocol uses cleartext, that includes usernames and passwords.
