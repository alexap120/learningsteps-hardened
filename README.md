# LearningSteps Lockdown: End-to-End Infrastructure Hardening

## Overview
This repository documents the security engineering, threat modeling, and infrastructure hardening for the **LearningSteps** cloud application. Originally deployed in a vulnerable 2-tier baseline state (FastAPI VM and Azure PostgreSQL Flexible Server), the environment is iteratively hardened against real-world attack vectors across the management plane, perimeter network, transport layer, data persistence, and security operations.

---

## Threat Model & Risk Assessment

| Scenario | Likelihood | Impact | Priority | Justification |
| :--- | :--- | :--- | :--- | :--- |
| **1. Leaked Static SSH Key via Public Repository** | High | Critical | **P1** | Automated bots scrape public Git repositories within seconds; compromised keys allow complete root access and database destruction. |
| **2. Public PostgreSQL Port Enumeration & Brute Force** | High | Critical | **P1** | Exposing port 5432 directly to `0.0.0.0/0` invites continuous automated scanning (Shodan/Masscan) and credential brute-forcing. |
| **3. Anonymous Data Tampering via Unauthenticated API** | High | High | **P2** | Without authentication middleware on FastAPI routes, unauthorized internet traffic can alter, create, or delete learning records. |
| **4. Man-in-the-Middle (MitM) Credential Interception** | Medium | High | **P2** | Transmitting data over unencrypted HTTP (port 8000) exposes session tokens, query data, and user payloads to network sniffing. |
| **5. Lateral Movement via Plaintext DB Secrets** | Medium | Critical | **P3** | Storing database credentials in `.env` files or plaintext templates allows any initial command execution on the VM to escalate into a full database takeover. |

---

## Security Milestones

### Milestone 1: Management Plane & Identity Hardening
* **Identified Vulnerabilities:**
  * Inbound port 22 exposed to `0.0.0.0/0`, inviting global brute-force discovery.
  * Static file-based `.pem` keys without MFA, lifecycle rotation, or individual user audit trails.
* **Remediations Applied:**
  * Restricted `allow-ssh` inbound Network Security Group (NSG) rule on `nsg-app` strictly to the administrative source IPv4 address (`94.134.108.204/32`).
  * Deployed the `AADSSHLoginForLinux` VM extension on `vm-learningstepsalex` to replace static private keys with short-lived, identity-backed OpenSSH certificates.
  * Assigned the `Virtual Machine Administrator Login` RBAC role to the authorized corporate identity.
* **Verification:**
  * Validated zero-trust identity authentication using `az ssh vm --resource-group rg-learningstepsalex --name vm-learningstepsalex`.

---

### Upcoming Milestones
- [ ] **Milestone 2: Ingress Security & TLS Enforcement** (Reverse proxy, HTTPS/TLS 1.3 termination).
- [ ] **Milestone 3: Zero-Trust API Authentication** (Identity proxy / OAuth2 enforcement).
- [ ] **Milestone 4: Database Isolation & Private Networking** (PostgreSQL VNet integration / Private Link).
- [ ] **Milestone 5: Visibility & Threat Response** (Azure Monitor Agent, Syslog streaming, and Sentinel playbooks).