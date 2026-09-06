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

### Milestone 2: Ingress Security, TLS Termination & Edge WAF
* **Identified Vulnerabilities:**
  * Application lacked a managed ingress boundary, exposing backend ports directly.
  * Risk of Adversary-in-the-Middle (AitM) interception over unencrypted HTTP.
  * Uninspected application entry points exposed to web exploit payloads (SQLi, XSS, directory traversal).
* **Remediations Applied:**
  * Deployed an NPMplus container as an edge reverse proxy forwarding clean traffic to `127.0.0.1:8000`.
  * Provisioned an automated, CA-signed Let's Encrypt TLS certificate for `learningstepsalex.westeurope.cloudapp.azure.com`.
  * Enforced HTTP-to-HTTPS redirection (301), HSTS headers (max-age 2 years), and HTTP/2 transport.
  * Integrated CrowdSec AppSec (running the OWASP Core Rule Set) as an active bouncer to drop malicious payloads at the perimeter.
* **Verification:**
  * Validated automatic TLS upgrade and valid certificate chain via `curl -I`.
  * Verified that SQL injection and cross-site scripting payloads return `HTTP/2 403 Forbidden` at the proxy layer.

### Milestone 3: Zero-Trust API Authentication & Identity Proxy
* **Identified Vulnerabilities:**
  * Application API endpoints exposed as open public utilities with zero client identity verification or authorization barriers.
  * Anonymous internet traffic could read and manipulate stored journal entries without auditability.
* **Remediations Applied:**
  * Registered an Entra ID enterprise application (`learningsteps-oauth2-proxy-alex`) enforcing v2.0 token issuance and exposing the `access_as_user` delegated permission scope.
  * Pre-authorized the Microsoft Azure CLI client ID (`04b07795-8ddb-461a-bbee-02f9e1bf7b46`) for headless token acquisition without interactive consent prompt failures.
  * Deployed `oauth2-proxy` upstream on the VM listening on `127.0.0.1:4180`, integrating with Entra ID OpenID Connect discovery and bearer token validation.
  * Configured Nginx `auth_request` subrequest routing in NPMplus (`npmplus_auth_request: oauth2proxy`) to gate all inbound requests at the reverse proxy.
* **Verification:**
  * Confirmed unauthenticated requests are intercepted and redirected (`HTTP/2 302 Found` to `/oauth2/sign_in`).
  * Verified authenticated API queries presenting a valid Entra ID bearer token return `HTTP/2 200 OK` with JSON data.
  * Confirmed defense-in-depth: authenticated requests containing exploit payloads (SQLi / XSS) are still caught and dropped with `HTTP/2 403 Forbidden` by CrowdSec AppSec.

---

### Upcoming Milestones
- [x] **Milestone 1: Management Plane & Identity Hardening**
- [x] **Milestone 2: Ingress Security, TLS Termination & Edge WAF**
- [x] **Milestone 3: Zero-Trust API Authentication** (Identity proxy / OAuth2 enforcement).
- [ ] **Milestone 4: Database Isolation & Private Networking** (PostgreSQL VNet integration / Private Link).
- [ ] **Milestone 5: Visibility & Threat Response** (Azure Monitor Agent, Syslog streaming, and Sentinel playbooks).