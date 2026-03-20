# Penetration Testing Engagement: Local Cyber Range

**Operator:** null  
**Target Architecture:** Containerized Local Vulnerable Application 
**Application:** OWASP Juice Shop
**Network Scope:** `192.168.X.X:3000` (Internal Host Node)

## 🎯 Executive Summary
This repository documents a progressive, multi-phase penetration testing engagement against a locally hosted, containerized instance of OWASP Juice Shop. The objective is to systematically identify, exploit, and document web application vulnerabilities within a controlled, zero-risk infrastructure.

## 🧰 Arsenal & Infrastructure
* **Host Server:** Ubuntu 24.04 LTS (Headless Node)
* **Container Engine:** Docker Engine (Isolated Subnet)
* **Offensive OS:** Kali Linux (Virtual Environment)
* **Primary Tooling:** Burp Suite Community Edition (HTTP Proxy)

---

## 📝 Execution Log

### Phase 1: Initial Reconnaissance & Proxy Configuration
**Objective:** Establish traffic interception and verify packet manipulation capabilities.

* **MITM Establishment:** Successfully established a Man-in-the-Middle (MITM) proxy between the local browser and the target container network.
* **Traffic Control:** Verified the capability to freeze, inspect, and actively modify raw HTTP GET/POST requests prior to server contact.
* **Proof of Concept (Header Injection):** Successfully intercepted an outbound request and spoofed the HTTP `User-Agent` header to a custom signature (`PhaseNull-Agent/1.0`). The manipulated traffic was successfully routed to and accepted by the target server, confirming read/write control over the HTTP stream.

### Phase 2: Passive Reconnaissance & Application Mapping
**Objective:** Map the application's attack surface and underlying architecture without triggering aggressive intrusion detection thresholds.

* **Network State:** External VPNs and secure tunnels (Cloudflare WARP) were deliberately disabled prior to execution to prevent local routing conflicts and ensure pure, uninterrupted Man-in-the-Middle (MITM) packet capture.
* **Mapping Technique:** Executed manual spidering and site navigation utilizing the Burp Suite proxy with Intercept toggled OFF.
* **Architectural Findings:** * Successfully mapped the target's primary directory structure. 
  * Identified heavy reliance on backend API routing (`/rest/` and `/api/` endpoints), confirming a decoupled frontend/backend architecture. 
  * Discovered real-time WebSocket/polling endpoints (`/socket.io/`), expanding the potential attack surface beyond standard HTTP stateless requests.

### Phase 3: Active Exploitation & Authentication Bypass
**Objective:** Compromise administrative credentials via SQL Injection (SQLi) at the authentication layer.

* **Attack Vector:** Targeted the REST API login endpoint (`/rest/user/login`).
* **Execution:** Intercepted the POST request using Burp Suite and manipulated the `email` parameter with a crafted SQL injection payload to alter the backend query logic.
* **Findings:** Successfully bypassed the authentication mechanism, granting unauthorized administrative access to the application dashboard.
* **Post-Exploitation Note:** While this specific vector relied on a known administrative email, real-world execution would utilize OSINT for email enumeration or a tautology payload (`' OR 1=1 --`) to force evaluation of the first database index (typically UID 1 / Admin).

### Phase 4: Active Exploitation & Local File Inclusion (LFI) via XXE
**Objective:** Exploit an insecure XML parser to read arbitrary local files from the host container, demonstrating a path to infrastructure compromise.

* **Target Endpoint:** B2B Customer Complaint File Upload (`POST /file-upload`)
* **Vulnerability Type:** XML External Entity (XXE) Injection (CWE-611)
* **CVSS v3.1 Score:** 7.5 (High)
* **Vulnerable Component:** Node.js `libxmljs` version ~0.18
* **Attack Vector:** * The `/file-upload` endpoint's core business logic is deprecated and programmed to return an HTTP `410 Gone` error. 
  * However, the underlying application framework parses incoming XML payloads *before* generating this rejection response. Furthermore, the error handler explicitly reflects the parsed XML string back to the client.
* **Execution:**
  1. Intercepted a standard `multipart/form-data` file upload request to `/file-upload` using Burp Suite.
  2. Manipulated the `Content-Type` of the specific form boundary, altering it to `text/xml`.
  3. Injected a crafted XML payload containing a malicious Document Type Definition (DTD). The DTD defined an external entity (`&xxe;`) pointing to the container's local file system (`file:///etc/passwd`).
* **Findings:** The backend `libxmljs` parser failed to disable external entity resolution by default. It parsed the malicious DTD, retrieved the contents of the target file, and reflected the contents of `/etc/passwd` in the HTTP `410` error response body. This confirms successful Local File Inclusion (LFI).

---
*Engagement ongoing. Further phases will be documented as vulnerabilities are mapped and exploited.*
