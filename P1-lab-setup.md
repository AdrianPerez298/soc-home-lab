# Project 1: Home SOC Lab with Wazuh SIEM

**Goal:** Build an isolated home lab with a free, open-source SIEM, connect an endpoint, and verify that real security events (failed logins) are collected, correlated, and alerted on.

**Skills shown:** virtualization · Linux server administration · SIEM deployment · log ingestion · troubleshooting from logs · alert review · network isolation · backup/baseline

---

## Environment

| Component | Details |
|---|---|
| Host | Apple silicon Mac (M4, 16 GB RAM) |
| Hypervisor | UTM (type 2 hypervisor) |
| SIEM VM | Ubuntu Server 24.04.5 LTS (ARM64): 4 vCPU, 8 GB RAM, 64 GB disk |
| SIEM | Wazuh 4.14.9 all-in-one (server/manager, indexer, dashboard) |
| Endpoint | macOS host running the Wazuh agent (`mac-endpoint`) |
| Network | UTM Shared Network (NAT behind the host), private range 192.168.64.0/24 |

## Architecture

```
 mac-endpoint (Wazuh agent)
        │  1515/tcp  enrollment (once)
        │  1514/tcp  log forwarding (ongoing)
        ▼
 ┌──────────── soc-lab VM (192.168.64.2) ────────────┐
 │ Wazuh server: decodes logs, applies detection rules │
 │ Wazuh indexer: stores events (local only, 9200)    │
 │ Wazuh dashboard: HTTPS 443, alerts + search        │
 └────────────────────────────────────────────────────┘
```

## Build steps

1. **Verified the installer image.** Downloaded the Ubuntu Server ARM64 ISO from Canonical's official release server and confirmed its SHA-256 hash matched the published `SHA256SUMS` list (integrity check before installing anything).
2. **Built the VM** in UTM and expanded the LVM root volume to use the full disk (Ubuntu's installer only allocates about half by default).
3. **Installed Wazuh all-in-one** using the official installation script from `packages.wazuh.com` (HTTPS, official domain, matches the vendor documentation).
4. **Froze the Wazuh package repository** so automatic OS updates can't upgrade Wazuh components out of sync.
5. **Verified listening services** with `ss -tlnp`: dashboard on 443, agent enrollment on 1515, agent traffic on 1514, server API on 55000. The indexer (9200/9300) listens on 127.0.0.1 only, so it is not reachable from the network (smaller attack surface).
6. **Enrolled an endpoint:** deployed the Wazuh agent to the macOS host via the dashboard's deployment wizard (agent version matched to the server). Status: **active**.

## Troubleshooting: failed first install

The first installation failed at the Wazuh manager step. Instead of re-running blindly, I read the installer log:

```
sudo tail -n 40 /var/log/wazuh-install.log
```

**Root cause:** `Could not get lock /var/lib/dpkg/lock-frontend. It is held by process … (unattended-upgr)`. Ubuntu's automatic security updates were using the package manager at the same moment, and only one process can hold the dpkg lock.

**Fix:** waited until the lock was free (`sudo fuser /var/lib/dpkg/lock-frontend` returned nothing), then re-ran the installer with `-o` to overwrite the partial install. The second install succeeded.

## Validation: generating and finding real alerts

I generated two harmless failure patterns and confirmed the SIEM detected them:

| Test | Where it was logged | Wazuh rules triggered |
|---|---|---|
| Repeated SSH logins as a non-existent user (host → VM) | `soc-lab` (the machine being logged into) | 5710 "sshd: Attempt to login using a non-existent user" (level 5) ×4 · 5503 "PAM: User login failed" (level 5) · **2502 "User missed the password more than one time" (level 10)** |
| Wrong `sudo` password ×3 on the macOS endpoint | `mac-endpoint` | sudo authentication failure events |

**Takeaways:**
- An alert is attributed to the **target** system. The **source** of the attempt is in the event's `data.srcip` field.
- The SIEM **correlated** several level-5 failures into a single level-10 alert (rule 2502). That's the difference between collecting logs and detecting a pattern.

## Isolation and safety

- **Network:** the VM sits on a NAT network behind the host. The host can reach the VM, and the VM has outbound internet for updates, but other devices on the home LAN **cannot** initiate connections to it (no port forwarding is configured).
- **Lab rules:** logs and public packet captures only; no live malware is ever run. For riskier work, the VM would move to a host-only network with no internet access.
- **Credentials:** no passwords or secrets are stored in this repository. The dashboard uses Wazuh's self-signed certificate, which is acceptable only because the server is under my control on a private network.

## Backup / baseline

The VM was powered off and cloned in UTM (`soc-lab-baseline-S20`) as a known-good baseline. UTM has no snapshot button for this VM type, so a powered-off clone serves the same purpose: any later lab mistake can be rolled back in minutes.

## What I learned

Throughout this project, I built a Linux VM and a Wazuh SIEM. When I downloaded the Linux virtual machine image, I compared its SHA-256 hash digest with the published hash list to make sure no one had tampered with the file. A problem I faced while installing Wazuh was that the installation failed, and instead of rerunning it and praying, I read the installation logs and found that Ubuntu's automatic security updates were running at the same time and holding the package manager's lock. It taught me to read the logs before retrying, and that two package installs can't run at once. One specific insight from this project is how NAT hides a computer's private IP by translating it into the router's public IP (or another NAT device's IP, like how my Mac is the NAT device for my VM). This is significant because the source IP of an alert might be the NAT device, not the actual machine behind it. Up next is Project 2, log analysis, where I'll work through Windows event IDs, Linux authentication logs, and spotting brute-force patterns.

## Next

**Project 2: log analysis.** Windows event IDs, Linux authentication logs, and brute-force patterns.
