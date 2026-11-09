# Project 2c: Detecting a Brute Force in Wazuh and Building a Timeline

**Goal:** Run a simulated attack against my lab VM (wrong passwords only, inside the lab), find it in the Wazuh SIEM, read the alert fields an analyst uses, and turn the events into an incident timeline.

**Skills shown:** SIEM alert analysis · timeline building · MITRE ATT&CK mapping · time zone handling · separating evidence from inference · incident summary for a manager

---

## Setup

- **Target:** Ubuntu Server 24.04 VM running Wazuh 4.14 all-in-one, on a NAT network in UTM.
- **"Attacker":** my Mac, reaching the VM through port-forward rules (SSH 2222 → 22; dashboard 8443 → 443). The source IP in every event is `10.0.2.2`, UTM's NAT gateway.

## Timeline (all times PST, Nov 8, 2026)

| Time | What happened | Rule (level) | Account | Source |
|---|---|---|---|---|
| 19:50:52–19:51:00 | Username guessing: attempts on a non-existent account | sshd: non-existent user (5) | fakeadmin | 10.0.2.2 |
| 19:51:02 | Wazuh correlated the repeated failures | User missed the password more than one time (10) | fakeadmin | 10.0.2.2 |
| 19:51:22–19:51:32 | Brute force against a real account | Authentication failed (5) | adrian | 10.0.2.2 |
| 19:51:34 | Wazuh correlated the repeated failures | User missed the password more than one time (10) | adrian | 10.0.2.2 |
| 19:52:40 | **Successful login after the failures** | sshd: authentication success (3) | adrian | 10.0.2.2 |
| 19:53:10 | `sudo whoami`: confirmed admin rights | Successful sudo to ROOT (3) | adrian | Not recorded; linked to the SSH session by account and time |
| 19:53:28 | Session logged off | Session closed (3) | adrian | Not recorded |

**MITRE ATT&CK:** Brute Force (T1110), as tagged by Wazuh on the correlated alert.

## Conclusion

**Account compromise** of `adrian`, an account with sudo rights. There is no evidence of data loss, so "data breach" wouldn't be supported by this evidence.

## Summary for a manager

An attacker guessed usernames, then brute-forced the `adrian` account and logged in. They confirmed admin rights with `sudo whoami`, then logged off about 30 seconds later. It's urgent because the account has admin access and they could have left a backdoor. Next steps: confirm with the account owner, block the source, reset the password, and check for new accounts, SSH keys, and scheduled tasks. Long term: restrict SSH to a VPN and require keys or MFA.

## Observations

- **Levels matter:** single failures are level 5, but repeated failures were correlated into one level 10 alert. A successful login is only level 3, so the most important event in this attack had the *lowest* severity. An analyst has to read the sequence, not just sort by level.
- **Time zones:** `/var/log/auth.log` records UTC, while the dashboard shows my local time, 8 hours apart. Every timeline states its time zone.
- **Evidence vs. inference:** the sudo event has no source IP. I linked it to the session by account and time, and the timeline says so.

## What I learned

During this project, I learned how to create a timeline. While I was creating it, I noticed that one of the logs with the lowest alert level was the most serious: the successful authentication, at level 3. It taught me that sorting alerts by level alone can hide the most important event, so I have to read the sequence of events too.
