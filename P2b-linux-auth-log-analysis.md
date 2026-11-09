# Project 2b: Linux auth.log Analysis (Brute Force vs Password Spray)

**Goal:** Generate brute-force and password-spray patterns against my own lab VM (wrong passwords only, inside the lab), then find and tell them apart in `/var/log/auth.log` using only the command line.

**Skills shown:** Linux log analysis · grep and shell pipelines · attack pattern recognition · reading source IPs behind NAT · lab troubleshooting

---

## Setup

- **Target:** Ubuntu Server 24.04 VM (`soc-lab`) in UTM, running OpenSSH and Wazuh.
- **"Attacker":** my Mac, connecting through a UTM port-forward rule (host 2222 → guest 22).
- **Safety:** everything stayed inside the lab. The only "attack" was typing wrong passwords.

## What I generated

| Pattern | What I did | Expected signature |
|---|---|---|
| Brute force | 6 wrong passwords against one real account (`adrian`) | Many failures, one account |
| Password spray | 1 wrong password each against 4 non-existent accounts | One failure each, many accounts |
| Success | 1 correct login after the failures | `Accepted password` from the same source |

## Analysis

**Failed logins:**
```
sudo grep "Failed password" /var/log/auth.log
sudo grep -c "Failed password" /var/log/auth.log     → 10
```
```
Failed password for adrian from 10.0.2.2 port 49623 ssh2
Failed password for invalid user alice from 10.0.2.2 port 49628 ssh2
```
`invalid user` means the account doesn't exist, so the attacker is guessing usernames. A line without it means a **real** account is under attack, which is more dangerous.

**Which accounts, and how many times:**
```
sudo grep "Failed password" /var/log/auth.log | awk '{print $(NF-5)}' | sort | uniq -c | sort -rn
      6 adrian
      1 carlos
      1 dave
      1 bob
      1 alice
```
One high count = **brute force**. Many counts of 1 = **spray**. The pattern is visible from the numbers alone.

**The success:**
```
sudo grep "Accepted" /var/log/auth.log
05:51:34 Accepted password for adrian from 10.0.2.2 port 49603 ssh2   ← my earlier analyst session
05:59:29 Accepted password for adrian from 10.0.2.2 port 49652 ssh2   ← right after the failures
```
Lining up timestamps matters: only the second login follows the failures. Failures, then a success, from the same source (which also tried fake usernames) is the signature of a likely **account takeover**.

## NAT and source IPs

Every source IP in the log is `10.0.2.2`: UTM's NAT gateway, not my Mac's real address. In a real investigation, the source IP can be a NAT device, and you have to look behind it to find the actual machine.

## Troubleshooting

UTM's Shared Network mode hung both VMs at "Starting," and updating UTM didn't help. I isolated the cause by testing one change at a time on my backup VM: switching to Emulated VLAN let it boot, which pointed to networking. That change made the VM unreachable at its old IP, so I added a port-forward rule to reach SSH.

## What I learned

I simulated an attack by imitating a password spray against invalid accounts and a brute-force attack against my own account. I already knew grep and pipeline commands from Coursera; this lab showed me how to use them on /var/log/auth.log to investigate attacks. My VM only has one real account, so I used fake accounts for the password spray, and I noticed that the logs specified the failed authentication was for an invalid user. That one word tells an analyst whether the attacker is guessing usernames or attacking a real account.
