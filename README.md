# Network Assessment Lab

A small network of my own machines, built in Docker on an isolated network. I scanned it with Nmap the way an attacker would, verified every finding by hand, rated the risks, fixed them and scanned again to prove the fixes worked.

## Why

Attackers start by mapping a network to see what is running and what is exposed. Forgotten test servers, default settings and services with no password are some of the most common ways into a company. This project shows how to find them, judge how serious they are and remove them, with evidence at every step.

## Lab Setup

All machines sit on an internal Docker network with no route to the internet or my home network.

| Machine | Address | Role |
|---|---|---|
| scanner | 172.30.0.5 | Debian with Nmap, curl and redis-cli, the assessor's machine |
| web | 172.30.0.10 | nginx web server |
| cache | 172.30.0.20 | Redis key value store, started with protection turned off |
| files | 172.30.0.30 | A quick Python file server left running, holding an old config backup |
| app | 172.30.0.40 | My hardened Flask app from the secure container project |

The weaknesses were planted on purpose, and the password in the backup file is fake.

## Method

1. Host discovery to find which machines are up
2. Port scan of all 65,535 ports on every machine
3. Service and version detection on the open ports
4. Manual checks to confirm each finding instead of trusting the scan alone
5. Risk rating, fixes and a full rescan

## Results

Scanned with Nmap 7.95 in November 2026.

| Measure | Before | After |
|---|---|---|
| Target hosts up | 4 | 3 |
| Open ports | 4 | 3 |
| Versions exposed | 3 | 0 |
| Critical findings | 2 | 0 |
| High findings | 1 | 0 |
| Password readable without logging in | Yes | No |
| Redis commands accepted from a stranger | Yes | No, NOAUTH Authentication required |

## Findings and Risk Ratings

Risk is likelihood multiplied by impact, each scored 1 to 3.

| # | Finding | Likelihood | Impact | Risk | Fix | Proof |
|---|---|---|---|---|---|---|
| 1 | Redis accepted reading and writing with no password | 3 | 3 | 9 Critical | Password from the environment, protected mode on | Stranger gets NOAUTH, the right password gets PONG |
| 2 | Database password readable in config.bak | 3 | 3 | 9 Critical | File deleted, the password would be changed in a real company | Download times out |
| 3 | Test file server left running and listing its folder | 3 | 2 | 6 High | Service removed | Host no longer answers |
| 4 | nginx version shown and default page never replaced | 3 | 1 | 3 Low | server_tokens off and a custom page | Server header shows nginx with no version |
| 5 | App shows the name Gunicorn | 3 | 1 | 3 Low | Accepted as residual risk | No version shown |

## Key Findings

**Findings 2 and 3 formed a chain.** The test file server is how the password leaked. Removing the server fixed the high risk and cut off the critical one in the same move.

**An open port does not mean a service is vulnerable, and a clean scan does not mean it is safe.** After the Redis fix, Nmap still showed port 6379 open, because the service still has to talk to the apps that use it. Only testing the service itself proved the password was now required.

**I tested for path traversal and it was blocked.** I suspected the file server could be tricked into reading files outside its folder. The test returned 404, so I recorded it as checked and not vulnerable instead of reporting a guess.

**The scan found a port the image never declared.** Docker listed no port for the file server, yet Nmap found 8000 open. Declared ports are documentation, not security, which is why scanning matters.

**Deleting a leaked password is not enough.** The password is gone from the folder but still sits in the project's Git history. In a real company the password would be changed, so the old leak becomes useless.

**One fix hid two versions.** Once Redis required a password, Nmap could no longer ask it for its version.

## Principles Applied

| Principle | Where | Proof |
|---|---|---|
| Reducing the attack surface | Unneeded file server removed | Hosts down from 4 to 3 |
| Authentication | Redis demands a password | NOAUTH for strangers |
| Fail securely | Lab refuses to start without its secrets | Compose stops with a clear error |
| No secrets in code | Passwords come from the environment | No password in any project file |
| Least information | Version numbers hidden | 0 versions exposed |
| Immutability | nginx settings mounted read only | Mounted with ro |
| Verify, do not assume | Every finding and fix confirmed by hand | Evidence files 04 to 11 |

## Standards

MITRE ATT&CK T1046 Network Service Discovery, the technique used in the scans, and T1552.001 Credentials in Files, finding 2.

OWASP Top 10 Security Misconfiguration, findings 1, 3 and 4.

Cyber Essentials Secure Configuration, by removing unneeded services and changing default settings.

The risk ratings follow the likelihood and impact approach used in NIST SP 800 30.

## Residual Risk

Traffic inside the lab is plain HTTP, so anyone listening on the network could read it. This is the focus of my next project.

The Redis password is passed at startup, so anyone with Docker access on the host can read it. A secrets manager would fix this.

nginx and Gunicorn still show their names, although no versions. A reverse proxy could hide them.

The fake password from the first commit remains in Git history.

## Legal Note

I only scanned machines I own, on a network I built and isolated. Scanning systems without permission can be an offence under the Computer Misuse Act 1990.

## Tech Stack

Docker, Docker Compose, Nmap 7.95, curl, redis-cli, nginx, Redis, Python, Git

## How to Run

You need Docker Desktop and the flask-app:hardened image from my [secure-flask-app](https://github.com/MuskanSayyedCySec/secure-flask-app) project.

```bash
git clone https://github.com/MuskanSayyedCySec/network-assessment-lab.git
cd network-assessment-lab
export LAB_SECRET_KEY=$(python3 -c "import secrets; print(secrets.token_hex(32))")
export LAB_REDIS_PASSWORD=$(python3 -c "import secrets; print(secrets.token_hex(16))")
docker compose up -d --build
```

To repeat the scans.

```bash
docker compose exec scanner nmap -sn 172.30.0.0/24
docker compose exec scanner nmap -sV -p- --open 172.30.0.10 172.30.0.20 172.30.0.30 172.30.0.40
```

To stop the lab.

```bash
docker compose down
```

## Project Structure

| File | Purpose |
|---|---|
| `compose.yaml` | The whole lab as code, network and machines |
| `scanner/Dockerfile` | The assessor's machine with Nmap, curl and redis-cli |
| `nginx/` | Settings that hide the version and replace the default page |
| `evidence/` | Every scan and check, before and after |

## Next Steps

Capture the lab's traffic with Wireshark to show what an attacker could read from plain HTTP, and detect a scan as it happens.
