The Technical Map & Write-up
Below is the complete, structurally safe layout for your Kronos report. Copy this Markdown block and paste it directly into your new file:
markdown
# 👑 Kronos Banking Trojan: Architectural Map & C2 Protocol Review

## 🔬 Threat Profile
* **Malware Classification:** Financial Banking Trojan (Man-in-the-Browser / MitB)
* **First Discovered:** Mid-2014
* **Primary Target Surface:** Major Web Browsers (Chrome, Firefox, Internet Explorer)
* **Core Mechanisms:** Form Grabbing, Hook-based Webinjects, Sandbox Evasion

---

## 1. High-Level Core Execution Map

Kronos operates via an injection framework that targets active browser processes rather than relying on heavy network manipulation protocols.

Use code with caution.

[ STAGE 1: INGESTION & COLD INJECTION ]
User runs dropper ➔ Obfuscated shellcode unpacks in memory space
│
└──► Scans running tasks for explorer.exe or runtimebroker.exe
└──► Executes Process Hollow / Remote Thread Injection
└──► Achieves Ring 3 persistent surveillance hooking handlers 
[ STAGE 2: PROCESS INTERCEPTION (MitB) ]
Malware monitors process creation structures waiting for a target browser
│
├──► Hooks low-level browser APIs (e.g., chrome.dll, ssl3.dll)
├──► Monitors web traffic natively before encryption/after decryption layers
└──► Compares requested target domains against a locally stored config.txt 
[ STAGE 3: DATA GRABBING & EXFILTRATION ]
User visits a matching target financial institution domain
│
├──► Form Grabber: Intercepts POST data directly out of form submission strings
├──► Webinject: Dynamically modifies HTML structure to phish for secondary keys/PINs
└──► Serializes stolen data and dispatches outbound payload packets to C2 

---

## 2. Low-Level Control Panel Protocol Analysis

Analysis of the leaked Kronos back-end administration configuration shows that the bot agent interacts with the PHP/MySQL panel infrastructure using a minimized command variable flag structure.

### Core C2 Parameter Commands
When transmitting telemetry, the malware passes commands to the control interface using specific operational parameters:

| Command Parameter | Backend Database Action | Telemetry Objective |
| :--- | :--- | :--- |
| **`a = 0`** | **Page Grab Ingestion Thread** | Tells the web panel to accept, process, and store raw intercepted HTML or form data stolen from the victim. |
| **`a = 1`** | **Configuration Retrieval** | Requests the latest target list from the database server containing the active financial webinject configurations. |
| **`a = 2`** | **Window Log Dump** | Streams active keystroke tracking logs or localized system event tracking back to the operator. |

### Local Encryption States
* **Network in Transit:** Configuration data and bot communications are securely routed using **AES in CBC Mode** to blend into standard application traffic profiles.
* **Local Storage on Disk:** Once the bot agent downloads target configurations to the local installation path (`%AppData%/Microsoft`), it converts and stores them using **AES in ECB Mode** or custom Blowfish arrays.

---

##  Key Indicators of Compromise (IoCs)

### Cryptographic Signatures & API Calls
Malware analysts look for specific cryptographic library references inside suspicious processes to track Kronos variants:
* `RtlComputeCrc32` (Commonly abused for quick integrity validation strings)
* Injection triggers tracking `chrome.dll`, `firefox.exe`, or standard process-creation library loops.

### Persistent Sandbox Evasion
The source code contains extensive loop checks evaluating 


markdown
#  The Marcus Hutchins Malware Analysis & History Index

This directory serves as a structured timeline and threat 
engineering index tracing the historical evolution of the malware frameworks linked to reverse-engineer and former 
developer Marcus "MalwareTech" Hutchins.

##  Chronological Threat Maps

### 1. The Precursor Framework (2012)
* **File Reference:** `[upas-kit-mapping.md](./upas-kit-mapping.md)`
* **Threat Profile:** A user-mode Ring 3 rootkit designed around modular 
browser surveillance.
* **Core Focus:** Low-level documentation on manipulating `NtQueryDirectoryFile` 
pointers within the Windows kernel file system structure to hide malicious file tracks.

### 2. The Commercial Banking Variant (2014)
* **File Reference:** `[kronos-trojan-mapping.md](./kronos-trojan-mapping.md)`
* **Threat Profile:** An advanced financial surveillance package engineered using Man-in-the-Browser (MitB) injection engines.
* **Core Focus:** Detailed configuration breakdowns tracking command parameters (`a=0`, `a=1`, `a=2`) used to intercept data prior to layer-7 transport encryption.

### 3. The Global Containment Operation (2017)
* **File Reference:** `[Wannacry-TechMap.md](./Wannacry-TechMap.md)`
* **Threat Profile:** The historic MS17-010 (EternalBlue) network worm execution chain that impacted global enterprise architectures.
* **Core Focus:** Decompiled C/C++ architecture tracking the unregistered `WinINet` Internet API request sequence that acted as the dynamic global kill-switch.

Marcus Hutchins Story 
https://youtu.be/0DoJQ4Cov9I?is=XDQrJqC-Ghgwd7fx

## Media & Biographical Reference Documentation
For an in-depth breakdown of the investigative operations, legal trials, and security engineering milestones associated with these files, reference the historical review map below:

* **Documentary File:** [The Hacker Who Accidentally Saved The Internet – Narrative Review](https://youtu.be/0DoJQ4Cov9I?is=XDQrJqC-Ghgwd7fx)
* **Timeline Metrics:** Tracks the operational evolution from 
