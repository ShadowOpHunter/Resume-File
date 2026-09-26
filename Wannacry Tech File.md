Wannacry Tech look 
Structural Component & File Mapping
WannaCry does not operate out of a single monolithic script; it is a multi-stage modular application written in C/C++ that heavily relies on the native Windows CryptoAPI, WinINet APIs, and an embedded password-protected ZIP archive.
When the initial malware loader or dropper executes, it drops and interacts with several critical files:
Binary / File	Functional Map & Role in Source Code
mssecsvc.exe	The Dropper / Worm Component. Contains the initial execution logic, the hardcoded kill-switch domain check, and the SMB exploitation module.
tasksche.exe	
The Ransomware Initiator. 
Extracted from the binary's resource section. It initializes the local workspace, creates a hidden folder, and drops the payload.
taskdl.exe	The Cleanup Thread. 
Routinely scans the directory structure to identify and permanently delete temporary .WNCRYT backup files created during the encryption phase.
@WanaDecryptor@.exe	
The UI application. 
The customer-facing frontend interface that handles localized payment screens, timers, and communicating decryption checks.
c.wnry	Configuration file. 
Holds C2 routing information, hardcoded Tor node configuration addresses, and the master Bitcoin wallets.
t.wnry	Encrypted DLL Payload. 
Contains the core cryptographic loop logic responsible for iterating through local and network file paths to execute the encryption routines.

msg/m_*.wnry	
Localization files. Language packs spanning dozens of languages to dynamically render the ransom message globally depending on system locale.
2. High-Level Technical Execution Flow
The software architecture operates in three distinct, sequential phases:

[Infection/Exploit] ➔ [Kill-Switch & Dropper Verification] ➔ [Payload Extraction & Cryptographic Chain]
Phase 1: Network Propagation & Exploitation
The worm component scans random external IP addresses and internal subnets looking for exposed Port 445 (SMBv1). It utilizes the "EternalBlue" (MS17-010) vulnerability to achieve unauthenticated Remote Code Execution (RCE) via kernel-pool corruption. Once access is gained, it injects the "DoublePulsar" backdoor to reliably stream and execute the initial payload dropper directly into ring 0 memory space.

Phase 2: The Kill-Switch & Persistence Verification
Upon execution, the very first routine in the source main method performs a protection check to thwart dynamic malware analysis sandboxes

1. The WinINet Check: The binary utilizes InternetOpenA() and InternetOpenUrlA() to send an HTTP GET request to a hardcoded, unregistered domain:
hxxp://www[.]iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea[.]com
2. The Exit Branch: If the domain returns a valid HTTP response, the program cleanly terminates via ExitProcess(). If it fails 
(standard behavior on real unpatched networks), execution skips the safety check and carries forward.

3. Service Creation: It registers a persistent system service named mssecsvc2.0 
("Microsoft Security Center 2.0 Service") to secure ring-3 persistence across computer reboots.

Phase 3: Extraction and Environment Preparation
The program checks for the mutex Global\MsWinZonesCacheCounterMutexA. 
If missing, it claims the mutex, reads its own PE (Portable Executable) resource directory, and decrypts an embedded password-protected ZIP archive using a hardcoded password string (WNcry@2ol7). 
It drops tasksche.exe into the local path, flags the directory as hidden via attrib +h, and executes it with full administrative contexts using icacls . /grant Everyone:F.

3. Cryptographic Deep-Dive
WannaCry implements a complex, custom-tiered hybrid encryption architecture utilizing a combination of RSA-2048 and AES-128-CBC.

  [Attacker Private Key] (Kept on C2 Server)
          │
  [Attacker Public Key] (Hardcoded in Malware) ─── Encrypts ───┐
                                                               │
  [Victim Private Key] (.dky) ◄─────── Decrypts ───────────────┘
          │
  [Victim Public Key] (.eky) ─── Encrypts ───┐
                                             │
  [Per-File AES Key] ◄───────── Decrypts ────┘
          │
     [Target File]

1. Key Generation: On start, it generates a unique master Victim RSA-2048 keypair natively via the Windows CryptoAPI (CryptGenKey).

2. Asymmetric Protection:
	The newly generated Victim Private Key is immediately encrypted using a hardcoded master Attacker Public Key baked inside the malware body. 
It exports this file to disk as 00000000.eky.

The corresponding Victim Public Key is exported natively as a plaintext header token to track local encryption routines.
3. Symmetric File Processing: The file scanner routine targets a hardcoded array of over 170 productivity extensions (.docx, .xlsx, .pdf, .zip, .jpg, etc.). For every matched file:
	 It creates a distinct 128-bit AES key in CBC mode.
	 The individual file contents are encrypted via a statically linked, third-party AES library.
	 The unique AES key is then encrypted using the Victim Public Key and appended directly into a custom 8-byte header block (WANACRY!) stamped over the newly altered target file, which has its extension rewritten to .WNCRY.
Because the Victim Private Key is locked behind the Attacker Public Key, no computer can decrypt its own local database files without obtaining the master asymmetric private response key from the command-and-control server operators over Tor.

Below is the decompiled implementation of the WannaCry kill-switch function. In the original mssecsvc.exe source code, this logic resides at the very beginning of the initialization routine (often marked as entry or WinMain).
Decompiled C/C++ Source Code Logic

This code is represented exactly as it looks when extracted via reverse-engineering tools like Ghidra or IDA Pro:
cpp
#include <windows.h>
#include <wininet.h>

// Hardcoded kill-switch domain found in the binary's string table
const char* KILL_SWITCH_URL = "http://iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com";

int CheckKillSwitch() {
    HINTERNET hInternetSession = NULL;
    HINTERNET hUrlConnection = NULL;
    int status = 0;

    // 1. Initialize the Windows Internet (WinINet) API session
    hInternetSession = InternetOpenA(
        "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1)", // User-Agent
        INTERNET_OPEN_TYPE_PRECONFIG,                             // Use registry configuration
        NULL, 
        NULL, 
        0
    );

    if (hInternetSession != NULL) {
        // 2. Attempt to establish a connection to the hardcoded URL
        hUrlConnection = InternetOpenUrlA(
            hInternetSession, 
            KILL_SWITCH_URL, 
            NULL, 
            0, 
            INTERNET_FLAG_RELOAD, // Force a download from the origin server, bypass cache
            0
        );

        // 3. Evaluate the connection state
        if (hUrlConnection != NULL) {
            // The connection succeeded! (The domain returned a valid HTTP response code)
            InternetCloseHandle(hUrlConnection);
            InternetCloseHandle(hInternetSession);
            return 1; // Signal to terminate the process
        }
        
        InternetCloseHandle(hInternetSession);
    }
    
    // Connection failed (Domain is unregistered or offline)
    return 0; // Signal to continue execution of the ransomware
}

int main() {
    // Execute the kill-switch routine before dropping payloads or scanning network ports
    if (CheckKillSwitch() == 1) {
        // If the domain is live, clean exit. The worm deactivates immediately.
        ExitProcess(0); 
    }

    // --- Ransomware & Propagation Logic Continues Below ---
    // Start_Exploitation_Routine();
    // Drop_Payloads();
    return 0;
}

Critical Code Analysis
• The Logical Flaw: 
The malware authors intended for this routine to verify if the malware was running inside a sandboxed malware-analysis environment. 
Many isolated sandboxes are configured to spoof the internet by responding with a generic HTTP 200 OK to every outbound web request.
If the authors saw a connection succeed to an absurd, random string, they assumed they were being analyzed and shut down to hide their true payload.
The "Sinkhole" Fix: On May 12, 2017, security researcher Marcus Hutchins (MalwareTech) found this string in the binary, realized it wasn't registered, and bought the domain for roughly $10.66. 
As soon as the domain became live on the public internet, any regular computer infected with WannaCry would suddenly get a successful hUrl Connection block, forcing the program into the ExitProcess(0) branch and stopping the worm mid-propagation





When analyzing the WannaCry execution map, researchers focus on a highly specific chain of events known as the Propagation & Initialization Pipeline. This sequence dictates exactly how the binary checks its environment, reads local host data, and maps its lateral trajectory across a network. 
Below is the complete Read-Out and Logical Map detailing exactly how the payload behaves line-by-line during triage. Use your three-dot menu (...) to save this final technical document as wannacry-execution-map.md inside your portfolio repository:
🛡️ Copy and Paste This Analysis Report to Your GitHub
markdown
# 🔍 Threat Blueprint: WannaCry Runtime Logic & Propagation Map
*A definitive read-out and logical mapping of initial access strings, infrastructure queries, and thread loops.*

## 1. The Pre-Infection Execution Map
When the binary payload compiles and executes inside a target operating system, it routes its core function loops through a strict, hardcoded sequence:

```text
[Runtime Stream Mapping]
       [Start: Binary Execution Triggered]
                        |
                        v
         +------------------------------+

         | Phase 1: Inbound HTTP Query  | ---> (Domain Resolved / 200 OK Status)
         | Check to Kill-Switch Domain  |                      |
         +------------------------------+                      v
                        |                         [Kill-Switch Active: Abort]
         (Connection Fails / 404 Status)                       |
                        |                                      v
                        v                              [ExitProcess(0)]
       +----------------------------------+

       | Phase 2: Resource Extraction     |
       | Unpacks Dropper & Drops Services |
       +----------------------------------+
                        |
                        v
       +----------------------------------+

       | Phase 3: Split execution threads |
       +----------------------------------+
          /                            \
         v                              v
[Thread A: Local Encryption]   [Thread B: SMBv1 Worm Propagation]
- Scans user directories.      - Generates target subnet IP arrays.
- AES-128 locks target extensions.  - Weaponizes EternalBlue over Port 445.
```

---

## 2. Technical Read-Out of the Core Logic Phases

### Phase 1: The Infrastructure Check Read-Out
The payload initializes a background network socket connection using standard Windows API protocols (`InternetOpenUrlA`) targeting the static unregistered domain string:
* **The Telemetry Check:** `hxxp://www[.]iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea[.]com`
* **The Logic Break:** If the domain is successfully registered (Sinkholed) and responds with an active HTTP status code, the program triggers an unconditional `ExitProcess(0)` sequence, rendering the malware dormant. If it receives a network error or connection failure (the expected state during the original deployment), the execution loop shifts to local payload unpacks.

### Phase 2: Local Resource Decompression & Persistence
Once the network verification boundary drops negatively, the payload drops localized structural controls:
* **The Unpacking Layer:** The base file extracts compressed zip arrays directly from its inner `.rsrc` compilation block. 
* **The System Elevation:** It drops a customized configuration layout containing localized desktop language translations, an encryption configuration registry matrix (`t.wnry`), and creates a dynamic, persistent Windows background execution service (often labeled `mssecsvc2.0`).

### Phase 3: The Lateral Movement Routing Engine
The payload drops two concurrent, separate execution threads to maximize impact across the enterprise domain:

* **The Encryption Loop (Thread 1):** Scans the local hard disk tables recursively, reading standard document envelopes, images, compressed ZIP blocks, and critical source code files. It locks the files with an asymmetric encryption key framework, appending the hard `.WNCRY` file extension header to block local user tracking.
* **The Propagation Loop (Thread 2):** Wakes up the embedded **EternalBlue exploit code modules (CVE-2017-0144)**. It pulls down the local network cache, isolates the immediate subnet parameters, and runs a rapid network thread scan that attempts to communicate over **Port 445 (SMBv1)**. If an unpatched machine is found, the worm pushes raw buffer overflow packet strings down the line, hijacking the target machine's system root memory to copy and trigger the payload automatically on the adjacent device.
markdown
# 🔬 Sandbox Triage: Structural Analysis of WannaCry.exe.sample
*A technical threat intelligence report extracting cryptographic signatures, static indicators, and vendor telemetry from sandboxed binary executions.*
## 1. Threat Profile & Core Metadata
This triage report details the static and behavioral parameters of a verified WannaCry ransomware artifact. The payload is heavily associated with the *Stardust Chollima* (Lazarus Group) threat matrix and targets legacy enterprise system architecture.
* **File Name:** `WannaCry.exe.sample`
* **File Size:** 3.4 MiB (3,514,368 bytes)
* **Threat Score:** 100 / 100 (Critical Malicious Verdict)
* **Total Behavior Indicators Indexed:** 309 Indicators (22 Malicious, 32 Suspicious)

### Cryptographic Hashes (Indicators of Compromise)
```text
[IoC Registry Table] Binary Identifiers:
MD5:    84c82835a5d21bbcf75a61706d8ab549
SHA-1:  5ff465afaabcbf0150d1a3ab2c2e74f3a4426467
SHA-256: ed01ebfbc9eb5bbea545af4d01bf5f1071661840480439c6e5babe8e080e41aa
imphash: 68f013d7437aa653a8a98a05807afeb1
```
## 2. Cross-Vendor Detection Telemetry
Static signatures and machine learning (ML) engines display near-unanimous classification of this sample as an active file-encoding ransomware threat:

* **CrowdStrike Falcon:** Malicious (100% Confidence) -> `win/malicious_confidence_100%`
* **ESET:** Classified as `Win32/Filecoder.WannaCryptor.D` (Active Filecoder Trojan framework)
* **Bitdefender / Emsisoft:** Flags exploitation vector referencing `Exploit.CVE-2017-0147` (Targeting memory-handling flaws within network communication protocols)
* **ClamAV:** `Win.Ransomware.Wannacryptor-9940180-0`

---

## 3. Technical Analyst Observations

### A. The Exploitation Hook (CVE-2017-0147 / MS17-010)
As shown in the vendor signatures, the payload’s primary propagation engine relies on weaponizing critical network protocol flaws. Rather than relying on standard user interaction, the binary drops a network driver module that scans volatile environments, executes remote buffer overflows, and drops full administrative privileges across vulnerable endpoints.

### B. High Entropy Bounds
The report indexes file entropy close to **8** (the maximum possible score). High structural entropy indicates that the data inside the sample is heavily compressed or encrypted. Malware writers use high entropy packing layers to mask their functional code logic strings from standard keyword scanners, forcing defenders to rely on dynamic sandbox execution and behavior tracking to map the true threat profile.

### C. Sandbox Environment Resilience
The multi-environment source data notes that the payload was executed across isolated instances of Windows 7, Windows 10, and Windows 11. The rising indicator count across modern operating systems (scaling from 242 up to 272 total indicators) demonstrates that while the core network worm exploit may be mitigated by patches on modern systems, the underlying execution logic and payload persistence structures still trigger heavy host system alerts



WannaCry.exe.sample
peexe
executable
·
3.4MiB (3514368 bytes)

THREAT SCORE

100
/ 100
MALICIOUS
AV DETECTIONS

24
/25

24
Mal

0
Sus

1
Clean

0
None
INDICATORS

309
total

22
Mal

32
Sus

255
Info


0

0

FILE METADATA
SHA-256

ed01ebfbc9eb5bbea545af4d01bf5f1071661840480439c6e5babe8e080e41aa
SHA-1

5ff465afaabcbf0150d1a3ab2c2e74f3a4426467
MD5

84c82835a5d21bbcf75a61706d8ab549
imphash

68f013d7437aa653a8a98a05807afeb1
PEhash

b08b0f894f904e0159289e0ba1fcffa9d85e3347
ssdeep

98304:QqPoBhz1aRxcSUDk36SAEdhvxWa9P593R8yAVp2g3x:QqPe1Cxcxk3ZAEUadzR8yc4gB
FILE INFO
Entropy
8
First Seen
2017-07-05 20:35:07 UTC
Last Analyzed
2026-09-24 17:09:46 UTC
Last Anti-Virus Scan
2026-09-24 17:09:46 UTC
MALICIOUS

WannaCry.exe.sample



Detection



Activity



Metadata



Comments

119
SANDBOX SOURCES

R1

Win7
242 indicators


R2

Win10
257 indicators


R3

Win11
272 indicators


R4

Win7
78 indicators


R5

Win7 (HWP Support)
76 indicators

Click chips to filter · select multiple to combine
Anti-Virus Results
CrowdStrike Falcon


Static Analysis and ML

Malicious (100%)
win/malicious_confidence_100% (W)


MetaDefender - Multi Scan Analysis
Learn more


23 / 24 FLAGGED

Search vendors...
AhnLab
malicious
Trojan/Win32.WannaCryptor
Antiy
malicious
Trojan[Ransom]/Win32.Wanna
Aurora
malicious
Malware_-10
Avira
malicious
TR/Ransom.Gen
Bitdefender
malicious
Exploit.CVE-2017-0147.13
ClamAV
malicious
Win.Ransomware.Wannacryptor-9940180-0
CMC
malicious
Win32_Filecoder_WannaCryptor_D
Emsisoft
malicious
Exploit.CVE-2017-0147.13 (B)
ESET
malicious
Win32/Filecoder.WannaCryptor.D trojan
Filseclab
malicious
Win.WanaDecryptor.vrsr
Gridinsoft
malicious
Malware.Win32.Gen.bot!se54409
Huorong
malicious
Ransom/Wannacry.j
K7
malicious
Trojan ( 0050d7171 )
Lionic
malicious
Trojan.Win32.Wanna.toNn
NETGATE
malicious
Trojan.Win32.Malware
SentinelOne
malicious
Windows_Malware_GenericRansomware_PE_2
Sophos
malicious
Troj/Ransom-EMG
TACHYON
malicious
Ransom/W32.WannaCry.Zen
Trellix
malicious
Ransom-O.g
Varist
malicious
W32/Trojan.ZTSA-8671
Vir.IT eXplorer
malicious
Trojan.Win32.WannaCry.B
Xcitium
malicious
Malware
Zillya!

malicious
Trojan.WannaCry.Win32.2
Intel Graph

All Nodes


All Connections

markdown
#  WannaCry (WanaCrypt0r 2.0) Technical Mapping & Writeup

##  Lab Environment & Malware Artifacts
* **Analysis OS:** Windows 7 SP1 (X64) / Flare VM (Isolated Host-Only Network)
* **Tools Used:** Ghidra v11.x, x64dbg, Wireshark, Process Monitor
* **Malware Sample Hashes:**
  * **MD5:** `db349b97c37d22f5ea1d1841e3c89eb4`
  * **SHA-256:** `24d004a104d4d540340c2a1c84938fa85b2e926a95f35081cca1a05220d91256`

---

## 1. Structural Component & File Mapping
WannaCry is a **multi-stage modular application written in C/C++** that relies on the native Windows CryptoAPI, WinINet APIs, and an embedded password-protected ZIP archive.

| Binary / File | Functional Map & Role in Source Code |
| :--- | :--- |
| **`mssecsvc.exe`** | **The Dropper / Worm Component.** Contains the initial execution logic, the hardcoded kill-switch domain check, and the SMB exploitation module. |
| **`tasksche.exe`** | **The Ransomware Initiator.** Extracted from the binary's resource section. It initializes the local workspace, creates a hidden folder, and drops the payload. |
| **`taskdl.exe`** | **The Cleanup Thread.** Routinely scans the directory structure to identify and permanently delete temporary `.WNCRYT` backup files created during encryption. |
| **`@WanaDecryptor@.exe`**| **The UI Application.** The customer-facing frontend interface that handles localized payment screens, timers, and decryption communication. |
| **`c.wnry`** | **Configuration File.** Holds C2 routing information, hardcoded Tor node configuration addresses, and the master Bitcoin wallets. |
| **`t.wnry`** | **Encrypted DLL Payload.** Contains the core cryptographic loop logic responsible for iterating through file paths to execute encryption routines. |
| **`msg/m_*.wnry`** | **Localization Files.** Language packs spanning dozens of languages to dynamically render the ransom message globally depending on system locale. |

---

## 2. High-Level Technical Execution Flow

Use code with caution.
[Infection/Exploit] ➔ [Kill-Switch & Dropper Verification] ➔ [Payload Extraction & Cryptographic Chain]

### Phase 1: Network Propagation & Exploitation
The worm component scans random external IP addresses and internal subnets looking for exposed **Port 445 (SMBv1)**. It utilizes the **"EternalBlue" (MS17-010)** vulnerability to achieve unauthenticated Remote Code Execution (RCE) via kernel-pool corruption. Once access is gained, it injects the **"DoublePulsar"** backdoor to reliably stream and execute the initial payload dropper directly into ring 0 memory space.

### Phase 2: The Kill-Switch & Persistence Verification
Upon execution, the very first routine in the source main method performs a protection check to thwart dynamic malware analysis sandboxes:
1. **The WinINet Check:** The binary utilizes `InternetOpenA()` and `InternetOpenUrlA()` to send an HTTP GET request to a hardcoded, unregistered domain:
   `hxxp://www[.]iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea[.]com`
2. **The Exit Branch:** If the domain returns a valid HTTP response (or points to a live sinkhole server), the program cleanly terminates via `ExitProcess(0)`. If it fails, execution skips the safety check and carries forward.
3. **Service Creation:** It registers a persistent system service named **`mssecsvc2.0`** ("Microsoft Security Center 2.0 Service") to secure ring-3 persistence across computer reboots.

### Phase 3: Extraction and Environment Preparation
The program checks for the mutex `Global\MsWinZonesCacheCounterMutexA`. If missing, it claims the mutex, reads its own **PE (Portable Executable) resource directory**, and decrypts an embedded password-protected ZIP archive using a hardcoded password string (`WNcry@2ol7`). It drops `tasksche.exe` into the local path, flags the directory as hidden via `attrib +h`, and executes it with full administrative contexts using `icacls . /grant Everyone:F`.

---

## 3. Decompiled Kill-Switch Source Code Logic

This logic resides at the very beginning of the initialization routine (`WinMain`) inside `mssecsvc.exe`:

```cpp
#include <windows.h>
#include <wininet.h>

const char* KILL_SWITCH_URL = "http://iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com";

int CheckKillSwitch() {
    HINTERNET hInternetSession = NULL;
    HINTERNET hUrlConnection = NULL;

    hInternetSession = InternetOpenA(
        "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1)", 
        INTERNET_OPEN_TYPE_PRECONFIG,                             
        NULL, NULL, 0
    );

    if (hInternetSession != NULL) {
        hUrlConnection = InternetOpenUrlA(
            hInternetSession, KILL_SWITCH_URL, NULL, 0, INTERNET_FLAG_RELOAD, 0
        );

        if (hUrlConnection != NULL) {
            InternetCloseHandle(hUrlConnection);
            InternetCloseHandle(hInternetSession);
            return 1; // Signal to terminate process (Kill-switch triggered)
        }
        InternetCloseHandle(hInternetSession);
    }
    return 0; // Signal to continue execution
}

int main() {
    if (CheckKillSwitch() == 1) {
        ExitProcess(0); // Clean exit. Worm deactivates immediately.
    }
    // --- Ransomware & Propagation Logic Continues Below ---
    return 0;
}

## 4. Cryptographic Deep-Dive
WannaCry implements a complex, custom-tiered **hybrid encryption architecture** utilizing a combination of **RSA-2048** and **AES-128-CBC**.

[Attacker Private Key] (Kept on C2 Server)
│
[Attacker Public Key] (Hardcoded in Malware) ─── Encrypts ───┐
│
[Victim Private Key] (.dky) ◄─────── Decrypts ───────────────┘
│
[Victim Public Key] (.eky) ─── Encrypts ───┐
│
[Per-File AES Key] ◄───────── Decrypts ────┘
│
[Target File]

1. **Key Generation:** On start, it generates a unique master **Victim RSA-2048 keypair** natively via the Windows CryptoAPI (`CryptGenKey`).
2. **Asymmetric Protection:** 
   * The newly generated **Victim Private Key** is immediately encrypted using a hardcoded master **Attacker Public Key** baked inside the malware body. It exports this file to disk as `00000000.eky`.
   * The corresponding **Victim Public Key** is exported natively as a plaintext header token to track local encryption routines.
3. **Symmetric File Processing:** The file scanner routine targets a hardcoded array of over **170 productivity extensions** (.docx, .xlsx, .pdf, etc.). For every matched file:
   * It creates a distinct **128-bit AES key** in CBC mode.
   * The unique AES key is then encrypted using the **Victim Public Key** and appended directly into a custom **8-byte header block (`WANACRY!`)** stamped over the newly altered target file, which has its extension rewritten to **`.WNCRY`**.

## 🔍 Indicators of Compromise (IoCs)

### Host Indicators
* **Created Services:** `mssecsvc2.0`
* **Mutex Created:** `Global\MsWinZonesCacheCounterMutexA`
* **File Extensions Appended:** `.WNCRY`, `.WNCRYT`

### Network Indicators
* **Kill-Switch Domain:** `hxxp://www[.]iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea[.]com`
* **Target Ports:** TCP Port 445 (SMB over IP)

---

## 🛡️ Detection Engineering (YARA Rule)

```yara
rule WannaCry_KillSwitch_Detector {
    meta:
        description = "Detects WannaCry loader worm component based on kill-switch domain strings"
        author = "Malware Analysis Lab Portfolio"
    
    strings:
        \$kill_domain = "iuqerfsodp9ifjaposdfjhgosurijfaewrwergwea.com" ascii wide
        \$user_agent = "Mozilla/4.0 (compatible; MSIE 6.0; Windows NT 5.1; SV1)" ascii
        \$mutex = "MsWinZonesCacheCounterMutexA" ascii wide

    condition:
        uint16(0) == 0x5A4D and (\$kill_domain or (\(user_agent and\)mutex))
}
```
