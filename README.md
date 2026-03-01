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
*Engagement ongoing. Further phases will be documented as vulnerabilities are mapped and exploited.*
