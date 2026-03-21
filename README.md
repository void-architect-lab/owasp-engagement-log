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

---

### Phase 2: Passive Reconnaissance & Application Mapping
**Objective:** Map the application's attack surface and underlying architecture without triggering aggressive intrusion detection thresholds.

* **Network State:** External VPNs and secure tunnels (Cloudflare WARP) were deliberately disabled prior to execution to prevent local routing conflicts and ensure pure, uninterrupted Man-in-the-Middle (MITM) packet capture.
* **Mapping Technique:** Executed manual spidering and site navigation utilizing the Burp Suite proxy with Intercept toggled OFF.
* **Architectural Findings:**
  * Successfully mapped the target's primary directory structure.
  * Identified heavy reliance on backend API routing (`/rest/` and `/api/` endpoints), confirming a decoupled frontend/backend architecture.
  * Discovered real-time WebSocket/polling endpoints (`/socket.io/`), expanding the potential attack surface beyond standard HTTP stateless requests.

---

### Phase 3: Active Exploitation & Authentication Bypass
**Objective:** Compromise administrative credentials via SQL Injection (SQLi) at the authentication layer.

* **Attack Vector:** Targeted the REST API login endpoint (`/rest/user/login`).
* **Execution:** Intercepted the POST request using Burp Suite and manipulated the `email` parameter with a crafted SQL injection payload to alter the backend query logic.
* **Findings:** Successfully bypassed the authentication mechanism, granting unauthorized administrative access to the application dashboard.
* **Post-Exploitation Note:** While this specific vector relied on a known administrative email, real-world execution would utilize OSINT for email enumeration or a tautology payload (`' OR 1=1 --`) to force evaluation of the first database index (typically UID 1 / Admin).

---

### Phase 4: Active Exploitation & Local File Inclusion (LFI) via XXE
**Objective:** Exploit an insecure XML parser to read arbitrary local files from the host container, demonstrating a path to infrastructure compromise.

* **Target Endpoint:** B2B Customer Complaint File Upload (`POST /file-upload`)
* **Vulnerability Type:** XML External Entity (XXE) Injection (CWE-611)
* **CVSS v3.1 Score:** 7.5 (High)
* **Vulnerable Component:** Node.js `libxmljs` version ~0.18
* **Attack Vector:**
  * The `/file-upload` endpoint's core business logic is deprecated and programmed to return an HTTP `410 Gone` error.
  * However, the underlying application framework parses incoming XML payloads *before* generating this rejection response. Furthermore, the error handler explicitly reflects the parsed XML string back to the client.
* **Execution:**
  1. Intercepted a standard `multipart/form-data` file upload request to `/file-upload` using Burp Suite.
  2. Manipulated the `Content-Type` of the specific form boundary, altering it to `text/xml`.
  3. Injected a crafted XML payload containing a malicious Document Type Definition (DTD). The DTD defined an external entity (`&xxe;`) pointing to the container's local file system (`file:///etc/passwd`).
* **Findings:** The backend `libxmljs` parser failed to disable external entity resolution by default. It parsed the malicious DTD, retrieved the contents of the target file, and reflected the contents of `/etc/passwd` in the HTTP `410` error response body. This confirms successful Local File Inclusion (LFI).

---

### Phase 5: Active Exploitation & DOM-Based Cross-Site Scripting (XSS)
**Objective:** Execute arbitrary client-side code to demonstrate the potential for complete session hijacking via JWT exfiltration.

* **Target Component:** Angular Frontend Router (`/#/search?q=` parameter)
* **Vulnerability Type:** DOM-Based Cross-Site Scripting (DOM XSS) (CWE-79)
* **CVSS v3.1 Score:** 8.8 (High)
* **Attack Vector:**
  * The SPA routing protocol extracts the search query directly from the URL and reflects it into the Document Object Model (DOM) without proper sanitization.
  * While the primary search input UI implements strict contextual escaping, injecting directly into the URL parameter bypasses these frontend controls.
* **Execution:**
  1. Bypassed the primary UI search input field to avoid immediate Angular SCE sanitization.
  2. Appended a URL-encoded `<iframe>` payload directly to the application's URL routing parameter.
  3. Forced a page load, causing the router to parse the poisoned parameter and inject the malicious execution context into the search results display area.
* **Injected Payload:**
  ```html
  <iframe src="javascript:alert('XSS')"></iframe>
  ```

---

### Phase 6: Active Exploitation & Broken Object Level Authorization (BOLA/IDOR)
**Objective:** Access unauthorized data by manipulating resource identifiers, demonstrating a failure in backend access control validation.

* **Target Endpoint:** REST API Basket Retrieval (`GET /rest/basket/{id}`)
* **Vulnerability Type:** Insecure Direct Object Reference (IDOR) / Broken Object Level Authorization (BOLA) (CWE-285, CWE-639)
* **CVSS v3.1 Score:** 6.5 (Medium)
* **Attack Vector:**
  * The application API fails to validate whether the authenticated user making the request actually owns or has authorization to access the specified basket ID.
  * This allows any authenticated, low-privileged user to read the contents of any other user's shopping cart, including those belonging to Administrative accounts.
* **Execution:**
  1. Authenticated into the application as a standard, low-privileged user (User ID: 23, Basket ID: 6).
  2. Intercepted the legitimate outbound GET request intended to retrieve the user's own basket (`/rest/basket/6`).
  3. Modified the direct object reference in the URI path, changing the ID from `6` to `1` (the known/assumed Admin basket).
  4. Stripped HTTP caching headers (such as `If-None-Match`) from the intercepted request to force a fresh response from the backend server.
* **Injected Request:**
  ```http
  GET /rest/basket/1 HTTP/1.1
  Host: 192.168.X.X:3000
  Authorization: Bearer <user_jwt_token>
  ```
* **Findings:** The server processed the manipulated request without verifying authorization, successfully returning the contents of the targeted user's basket.

---

### Phase 7: Active Exploitation & HTTP Parameter Pollution (HPP) leading to BOLA
**Objective:** Exploit a parser discrepancy using duplicate JSON keys to bypass authorization middleware and manipulate unauthorized data.

* **Target Endpoint:** REST API Basket Item Modification (`POST /api/BasketItems/`)
* **Vulnerability Type:** HTTP Parameter Pollution (HPP) / Broken Object Level Authorization (BOLA) (CWE-236, CWE-285)
* **CVSS v3.1 Score:** 7.1 (High)
* **Attack Vector:**
  * The application implements a security filter to prevent users from adding items to unauthorized baskets.
  * However, this filter is vulnerable to HTTP Parameter Pollution (HPP) via duplicate JSON keys. An attacker can exploit a parser discrepancy between the authorization middleware (which validates the **first** key) and the database Object-Relational Mapper (ORM) (which prioritizes the **last** key) to forcefully modify other users' shopping carts, including Administrative accounts.
* **Execution:**
  1. Intercepted a legitimate POST request to add an item to the attacker's own basket (Basket ID: `6`).
  2. Modified the JSON body to include the target victim's basket ID (Basket ID: `1`) as a duplicate key at the end of the object.
  3. Forwarded the polluted payload to the API.
* **Injected Payload:**
  ```json
  {
    "ProductId": 1,
    "BasketId": "6",
    "quantity": 1,
    "BasketId": "1"
  }
  ```
* **Findings:** The validation middleware approved the request based on the first `BasketId` (`6`), but the backend ORM processed the addition into the second `BasketId` (`1`). This confirms the ability to arbitrarily alter the contents of unauthorized users' shopping baskets.

---

### Phase 8: Active Exploitation & Authentication Bypass via SQL Injection (Tautology)
**Objective:** Bypass the primary authentication mechanism and gain unauthorized administrative access by manipulating backend SQL query logic.

* **Target Endpoint:** REST API Login (`POST /rest/user/login`)
* **Vulnerability Type:** SQL Injection (Authentication Bypass) (CWE-89)
* **CVSS v3.1 Score:** 9.8 (Critical)
* **Attack Vector:**
  * The application dynamically concatenates un-sanitized user input directly into the backend SQL query.
  * This allows an attacker to manipulate the query's boolean logic (using a tautology) combined with a comment operator to truncate the remainder of the developer's intended query, completely bypassing password verification.
* **Execution:**
  1. Navigated to the application's login portal.
  2. Injected the payload `' OR 1=1 --` into the email field.
  3. Input arbitrary data into the password field and submitted the authentication request.
* **Query Manipulation Analysis:**

  * **Intended Query:**
    ```sql
    SELECT * FROM Users WHERE email = '[INPUT]' AND password = '[PASSWORD]' AND deletedAt IS NULL
    ```
  * **Executed Query:**
    ```sql
    SELECT * FROM Users WHERE email = '' OR 1=1 -- ' AND password = '...' AND deletedAt IS NULL
    ```
  * The database evaluates the `WHERE` clause as `FALSE OR TRUE`, resulting in a `TRUE` condition for the entire table. The `--` comment drops the password check entirely. The database returns the first record (User ID 1 / Admin), granting immediate session access.

* **Findings (Evidence of Compromise):** The server responded with an HTTP `200 OK` and returned a valid JSON Web Token (JWT) containing the `admin@juice-sh.op` identity, granting full administrative privileges within the application UI.

* **Remediation & Mitigation Strategy:**
  * **Parameterized Queries (Prepared Statements):** The application must completely cease the practice of concatenating strings to build database queries. Implement Parameterized Queries via the Sequelize ORM.
  * **Separation of Data and Code:** By using prepared statements, the database engine compiles the SQL query structure before inserting the user input. The input is treated strictly as a literal string value, completely neutralizing malicious SQL operators (like `'`, `OR`, and `--`).

---

## 🏆 State of the Dojo

| Layer | Technique | Outcome |
|---|---|---|
| **Network Layer** | HTTP Intercept & Header Injection | Full read/write control over raw HTTP traffic |
| **Frontend (XSS)** | DOM-Based XSS via URL Parameter | Arbitrary JS execution, session hijack vector |
| **Backend Parser (XXE)** | XML External Entity Injection | Local file read (`/etc/passwd`) from container OS |
| **Business Logic (BOLA/HPP)** | IDOR + HTTP Parameter Pollution | Unauthorized read/write on restricted user baskets |
| **Database Engine (SQLi)** | Tautology-Based SQL Injection | Total authentication bypass, Admin JWT obtained |
