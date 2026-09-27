UPAS Kit Malware: Architectural Blueprint & Rootkit Review

## Threat Profile
* **Malware Classification:** Modular Banking Spy Tool / Ring 3 Rootkit
* **First Discovered:** Mid-2012
* **Core Functionality:** Silent Form Grabbing, Inline API Hooking, Stealth Persistence
* **Historical Significance:** Developed as the architectural structural precursor to the Kronos banking framework.

---

## 1. High-Level Modular Execution Map

UPAS Kit was engineered as a lightweight, highly modular injection kit that focused heavily on stealth deployment and hiding from local antivirus engines.

Use code with caution.

[ PHASE 1: SILENT INGESTION & ROOTKIT PERSISTENCE ]
Dropper executes silently ➔ Checks environment for sandbox signatures
│
└──► Unpacks Ring 3 User-Mode Rootkit functions
└──► Hooks local directory enumeration APIs (Hides files from the OS view)
└──► Secures local execution persistence within user-space folders
[ PHASE 2: PROCESS INTERCEPTION & COMPONENT HOOKING ]
Monitors target web browsers (Internet Explorer, early Firefox/Chrome variants)
│
├──► Injects payload modules directly into active browser memory threads
└──► Intercepts internal HTTP POST data streams before layer-7 transmission
[ PHASE 3: TELEMETRY DISPATCH ]
Serializes data packets (Usernames, passwords, session state cookies)
│
└──► Encrypts the local stash payload using static cipher blocks
└──► Dispatches an obfuscated outbound POST connection to the C2 admin panel
---

## 2. Structural Rootkit Mechanics (Process Hiding)

The definitive feature of the UPAS Kit source code layout is its user-mode stealth layer. 
Instead of modifying kernel drivers, 
it accomplishes file and process hiding entirely inside Ring 3 by manipulating core Windows API responses:

### The API Interception Flow
1. **Targeting the Handler:** 
The malware injects itself into target system apps and searches for **`ntdll.dll`** or **`kernel32.dll`**.
2. **Hooking `NtQueryDirectoryFile`:** It intercepts the native system call `NtQueryDirectoryFile` 
(the low-level function Windows uses to look up files inside a folder).
3. **Altering the Structure Output:** When a user or an antivirus scanner opens a folder containing the malware, 
the hooked function modifies the returned directory structure on the fly. It loops through the file names, 
checks for its own file footprint, and completely slices its own name out of the linked list before 
Passing the data back to the user interface. 

As a result, the files remain completely invisible to standard Windows Explorer and basic security checkers.

---

## 🔍 Key Indicators of Compromise (IoCs)

### Behavioral Triggers
* **Process Interception:** Spawning remote threads or memory page manipulations 
(`VirtualAllocEx` / `WriteProcessMemory`) targeting core internet application executables.
* **Network Beacons:** Unencrypted or encoded outbound form 
