# Project 2a: Windows Event Log Investigation (Password Spray → Persistence)

**Goal:** Reconstruct an overnight intrusion on a Windows workstation using only Security event log data, then recommend mitigations.

**Data:** A simulated Windows Security log (38 events) for a finance workstation, `FIN-WS07`, built for training in the real Windows event format. All names and IPs are fictional; `203.0.113.0/24` is a documentation-only range.

**Skills shown:** Windows Event ID analysis · logon-type interpretation · separating noise from signal · timeline reconstruction · ATT&CK-style attack-chain thinking · mitigation recommendations

---

## Event IDs used

| Event ID | Meaning |
|---|---|
| 4624 / 4625 | Successful / failed logon (with logon type: 2 keyboard, 3 network, 5 service, 10 RDP) |
| 4672 | Special (admin-level) privileges assigned to a new logon |
| 4688 | New process created (with command line) |
| 4720 | User account created |
| 4732 | Member added to a security-enabled group |
| 1102 | Security audit log cleared |

## Findings

**1. Password spray, not brute force.** Thirteen 4625 events from a single external IP (`203.0.113.45`), each against a **different** account, spaced exactly **8 seconds apart** at 02:10 AM. One attempt per account stays under any lockout threshold, and the steady 8-second interval points to an automated tool. One other 4625 (jlopez, logon type 2, 6:05 PM the previous evening, followed seconds later by a successful unlock) was ruled out as a normal typo.

**2. Service account takeover.** The spray succeeded against `svc_backup`, a service account, with no prior failures (its password was on the spray list):
- 02:12:07: 4624 **type 3** (credentials validated over the network)
- 02:14:30: 4624 **type 10** (RDP session) from the same external IP
- 02:14:30: **4672**: the account already held admin-level privileges, so no privilege escalation was needed.

**3. Reconnaissance.** 4688 process events show `whoami /priv`, `net user`, and `net localgroup administrators`: the attacker checked their own rights and enumerated users and admins.

**4. Persistence.** `net user support_tmp * /add` (**4720**), then `net localgroup administrators support_tmp /add` (**4732**): a new backdoor admin account.

**5. Hidden execution.** `powershell.exe -NoProfile -WindowStyle Hidden -EncodedCommand …`: a hidden, Base64-encoded script. Next step: decode it in a safe tool to identify the payload (never execute it).

**6. Anti-forensics.** `wevtutil cl Security` (4688) at 02:30:55, followed by **1102** at 02:30:56. The local Security log was wiped, but every earlier event had already been forwarded to the SIEM, which is why this investigation was possible.

## Timeline

| Time | Event | Activity |
|---|---|---|
| 02:10:03–02:11:39 | 4625 ×13 | Password spray from 203.0.113.45 (8 s intervals) |
| 02:12:07 | 4624 type 3 | svc_backup password guessed |
| 02:14:30 | 4624 type 10 + 4672 | RDP logon as svc_backup (already admin) |
| 02:15:10–02:16:05 | 4688 | Recon: whoami /priv, net user, net localgroup administrators |
| 02:18:12–02:18:30 | 4720 + 4732 | Backdoor admin `support_tmp` created |
| 02:22:47 | 4688 | Hidden encoded PowerShell |
| 02:30:55–02:30:56 | 4688 + 1102 | Local Security log cleared |

## Recommendations

1. **Least privilege for service accounts:** `svc_backup` should not hold admin rights it doesn't need, and should be denied interactive/RDP logon entirely.
2. **Strong, unique service-account passwords + MFA on remote access:** a complex password wouldn't appear on a spray list, and MFA would have stopped the RDP logon even with the right password.
3. **Never expose RDP (3389) to the internet:** disable it where unused, or restrict it to a VPN.
4. **Immediate response:** disable `support_tmp` and `svc_backup`, reset credentials, isolate FIN-WS07 after verification, and decode the PowerShell payload to scope impact.

## What I learned

This lab taught me how to tell normal user noise apart from signs of an attack. For example, there was a failed login the evening before the password spray, but the main differences were that it was a type 2 (keyboard) logon, and that right after the failed authentication the user successfully authenticated and didn't show any signs of malicious activity, unlike the attacker's one successful authentication. I also learned why central logging matters: during the breach, the attacker tried to erase their steps by clearing the logs, which on the host machine would look like nothing happened, but in the SIEM the logs weren't deleted, and we could even see in the SIEM that the attacker cleared the host machine's logs. The best fix would be least privilege: making sure that svc_backup, a service account, doesn't have admin rights, because it doesn't need them to do its job. Accounts should only be given the permissions necessary to do their jobs; everything else should be denied.
