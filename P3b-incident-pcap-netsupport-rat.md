# Project 3b: Incident PCAP Investigation (NetSupport RAT)

**Goal:** Confirm an IDS alert using a packet capture, then identify the infected Windows host and its user for an incident report.

**Skills shown:** alert verification · Wireshark display filters · Follow TCP Stream · C2 beaconing analysis · host and user identification (DHCP, NBNS, Kerberos, SAMR) · observed vs inferred reporting · IoC handling

---

## Scenario

A public training exercise from malware-traffic-analysis.net ("Easy As 123", 2026-02-28).

The SIEM showed several signature hits for **NetSupport Manager RAT** from `45.131.214[.]85` over **TCP 443**, starting **2026-02-28 19:55 UTC**. I had a PCAP of the internal host that triggered them.

| Environment | Value |
|---|---|
| LAN | 10.2.28.0/24 |
| Domain | easyas123[.]tech |
| Domain controller | 10.2.28.2 (EASYAS123-DC) |
| Gateway | 10.2.28.1 |

**Safety:** public PCAP only, opened read-only in Wireshark. No files were exported from the capture (no Export Objects), and no IoC was visited in a browser.

## Step 1: Confirm the alert

1. **Statistics → Conversations (IPv4):** only one internal host talked to the C2 IP: **10.2.28.88**.
2. **Filter** `ip.addr == 45.131.214.85` with UTC time display. First packet: **2026-02-28 19:55:51 UTC**, a SYN *from* 10.2.28.88, so the infected host started the connection (outbound).
3. **Follow TCP Stream:** the traffic on port 443 was **not TLS**. It was readable HTTP:

```
POST http://45.131.214.85/fakeurl.htm HTTP/1.1
User-Agent: NetSupport Manager/1.3
```

4. **Pattern:** 264 POST requests to the same URL, **one every 60 seconds**, for about 4.3 hours. That steady interval is C2 **beaconing**.

**Finding:** port 443 doesn't mean HTTPS. The RAT used the port because firewalls allow it outbound, but sent cleartext HTTP through it.

## Step 2: Identify the host and user

| Question | Answer | Where I found it | Filter / field |
|---|---|---|---|
| IP address | 10.2.28.88 | Conversations, C2 filter | `ip.addr == 45.131.214.85` |
| MAC address | 00:19:d1:b2:4d:ad | Ethernet II source | `eth.src` |
| Host name | DESKTOP-TEYQ2NR | DHCP Request (Option 12) and NBNS registrations | `dhcp`, `nbns` |
| User account | brolf | Kerberos client name | `kerberos.CNameString` |
| Full name | Becka Rolf | SAMR QueryUserInfo **response** (frame 339) | `samr.samr_UserInfo21.full_name` |

**Notes:**
- Kerberos names ending in `$` are **computer accounts** (the PC itself), not users.
- **LDAP was a dead end**: it was wrapped in Kerberos encryption (`SASL GSS-API Privacy`). The full name came from **SAMR** instead, where Windows asked the domain controller about the account. If one source is encrypted, look for another protocol that carries the same fact.
- The answer was in the **response** packet, not the request. Request = the question, response = the answer.

## Observed vs inferred

| Observed (in the packets) | Inferred (my conclusion) |
|---|---|
| 10.2.28.88 sent 264 cleartext POSTs to 45.131.214[.]85:443 every 60 s, User-Agent "NetSupport Manager/1.3" | DESKTOP-TEYQ2NR is infected with NetSupport Manager used as a RAT |
| User brolf (Becka Rolf) was logged in on that host | The attacker may have seen anything on Becka's screen or typed on that PC |

## IoCs (defanged)

- C2: `45.131.214[.]85` TCP 443
- URL: `hxxp://45.131.214[.]85/fakeurl.htm`
- User-Agent: `NetSupport Manager/1.3`

## Recommendations

**Right now:** isolate DESKTOP-TEYQ2NR · block **outbound** traffic to 45.131.214[.]85 · search SIEM and firewall logs for any other host contacting it (scope) · reset brolf's password · escalate to IR with these findings.

**Next:** find the initial infection (the first suspicious download or email before 19:55 UTC) and capture the RAT's process details from the host (EDR) before removing it.

## What I learned

This project taught me how to use a packet capture to trace network activity back to a specific device and user. I also learned that a port number doesn't prove which service was used; it only shows the expected service, which is why I saw plaintext when I followed the traffic stream on port 443. The PCAP was from a real attack, so I learned what real C2 beaconing looks like.
