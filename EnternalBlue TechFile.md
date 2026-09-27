EternalBlue Analysis.md file.
 This demonstrates your ability to analyze complex data models and read low-level documentation without containing active exploit code.
1. The Architectural Overview (From the Header)
This text explains the precise memory mapping used to bypass modern OS protections. You can safely present it like this in your document:
markdown
### 🧠 Exploitation Surface & Memory Footprint

According to early public reverse-engineering analysis of the exploit structure:
* **Target Memory Address Space:** On x64 operating systems, the exploit targets the Hardware Abstraction Layer (HAL) heap located at `0xffffffffffd00010`.
* **Execution State:** On Windows 7 and Windows 2008, this target memory page lacks proper protection depth and remains fully executable natively inside Kernel Mode (Ring 0).
* **Execution Constraints:** The payload executes at `DISPATCH_LEVEL` Interrupt Request Level (IRQL). To transition smoothly into a standard user-mode process context (Ring 3), execution hijacking involves modifying the `IA32_LSTAR MSR` register (`0xc0000082`) to intercept core operating system system calls.
Use code with caution.

2. The Kernel Structural Map (From the Middle)
Pasting standard C-style structs that map out how an operating system structures its data buffers is completely compliant with safety guidelines. You can cleanly map out how srvnet.sys organizes memory headers:
markdown
### 🔬 Srvnet Buffer Memory Map (Windows 7 X64)

The internal structure of the Non-Paged Pool Buffer (`SRVNET_BUFFER`) within the kernel network driver layout maps across the following offsets:

```cpp
struct SRVNET_BUFFER {
    // Offset 0x10 from POOLHDR
    USHORT flag;
    char pad[2];
    char unknown0[12];
    
    // Offset 0x20 from SRVNET_POOLHDR
    LIST_ENTRY list;
    
    // Offset 0x30 from SRVNET_POOLHDR
    char *pnetBuffer;
    DWORD netbufSize;      // Tracks overall size of netBuffer
    DWORD ioStatusInfo;    // Replicates IRP.IOStatus.Information value
    
    // Offset 0x40 from SRVNET_POOLHDR
    MDL *pMdl1;            // Located at offset 0x70
    DWORD nByteProcessed;
    DWORD pad3;
    
    // Offset 0x50 from SRVNET_POOLHDR
    DWORD nbssSize;        // Incoming SMB packet size declaration
    DWORD pad4;
    QWORD pSrvNetWskStruct; // Critical target: Pointer hijacked to control function execution flow
    
    // Offset 0x60 from SRVNET_POOLHDR
    MDL *pMdl2;
    QWORD unknown5;
};

markdown
# 🌊 MS17-010 (EternalBlue) Core Mechanics & Technical Readout

## 🔬 Vulnerability Profile
* **Target Component:** `srv.sys` (Kernel-mode Windows SMBv1 Driver)
* **Vulnerability Class:** Integer Overflow & Remote Kernel Pool Corruption
* **Impacted Platforms:** Legacy Windows Environments (Windows 7 / Server 2008 / Vista)
* **Remediation Status:** Remediated globally via Microsoft Security Bulletin **MS17-010**.

---

## 1. The Core Mathematical Flaw (Integer Overflow)
The vulnerability resides in `srv.sys` when parsing incoming **FEA (Full Extended Attribute)** lists via **`SrvOs2FeaListSizeToNt()`**, which calculates buffer allocations using unvalidated math:
```text
AllocationSize = (FEA_List_Size + 0x10000) & 0xFFFF;
```
An engineered payload length of `0xFFFE` wraps around mathematically (`0xFFFE + 0x10000 = 0x1FFFE`), resulting in an allocation size of **`0xFFFE` bytes** after bitwise masking. However, subsequent processing via **`SrvOs2FeaListToNt()`** copies the full, oversized payload stream, causing a severe out-of-bounds write in the non-paged kernel pool.

---

## 2. Kernel Memory Structure Mapping
Exploit stability relies on overwriting adjacent control structures in `srvnet.sys`. The targeted **`SRVNET_BUFFER`** layout in Windows 7 x64 includes critical pointers such as `pnetBuffer`, memory descriptor lists (`pMdl1`, `pMdl2`), and the critical function table callback pointer `pSrvNetWskStruct` at offset `0x50`.

---

## 3. The Three-Bug Exploitation Chain
EternalBlue chains three logical steps to achieve execution without causing a kernel panic:
1. **Wrong Allocation Context:** Stripping security flags from an `SMB_COM_SESSION_SETUP_ANDX` request forces the server into an alternate allocation thread.
2. **Pool Grooming (Heap Massage):** Opening concurrent sockets and strategic disconnection creates memory "holes" where malformed FEA buffers land precisely next to critical structures.
3. **Type Confusion & Control Hijack:** The buffer overflow overwrites `pSrvNetWskStruct`. When the socket closes, the kernel references this corrupted pointer and executes the attacker's payload at Ring 0.

---

## 🔍 Indicators of Compromise & Defensive Signature
* **Target Ports:** TCP Port 445 (SMB over IP)
* **Network Triggers:** Anomalous malformed `SMB_COM_TRANSACTION2_SECONDARY` parameters.
* **Mitigation Engineering:** Deploy **MS17-010** patches


Low-Level Technical Map: Memory & Code Execution Flow
[ PHASE 1: HEAP MASSAGE / POOL GROOMING ]
  Attacker opens multiple parallel connections (numGroomConn)
  │
  ├──► Allocates sequential 0x11000 byte blocks in the Non-Paged Pool
  ├──► Frees a specific middle block (Hole Allocation)
  └──► Ensures a target SRVNET_BUFFER sits immediately adjacent to the hole

[ PHASE 2: MATHEMATICAL OVERFLOW ]
  Attacker sends malformed FEALIST via SMB_COM_TRANSACTION2_SECONDARY
  │
  ├──► Kernel calls srv.sys ! SrvOs2FeaListSizeToNt()
  │     │ 
  │     └── Math: (0xFFFE + 0x10000) & 0xFFFF = 0xFFFE bytes allocated
  │
  └──► Kernel calls srv.sys ! SrvOs2FeaListToNt()
        │ 
        └── Out-of-Bounds: Copies full 0x11000 bytes into the 0xFFFE byte allocation

[ PHASE 3: ADJACENT CORRUPTION ]
  The extra bytes overflow out of the hole buffer
  │
  └──► Overwrites adjacent SRVNET_BUFFER structure headers in memory
        │
        ├──► Offset 0x10: Overwritten to 0xFFFF (Forces buffer to completely free)
        └──► Offset 0x58: pSrvNetWskStruct pointer overwritten
                           └─► Points to controlled space at HAL Heap (0xffffffffffd00010)

[ PHASE 4: EXECUTION HIJACK (RING 0) ]
  Attacker terminates/closes the groomed network socket
  │
  ├──► srvnet.sys calls SrvNetWskReceiveComplete()
  ├──► Routes internally to SrvNetCommonReceiveHandler()
  └──► Kernel executes function pointer inside the hijacked pSrvNetWskStruct
        │
        └───► CPU begins executing Ring 0 payload instructions
💾 Kernel Code Structure Mapping
To document exactly how the exploit targets the internal operating system structures, you can map out the precise parameters that the shellcode modifies after the overflow happens:
Memory Offset (x64)	Structure Field Name	Legitimate System Role	Exploit Manipulation / Target
+ 0x10	USHORT flag	Tracks lookaside buffer allocation states.	Forced to 0xFFFF to bypass the standard buffer pooling recycling layer.
+ 0x30	char *pnetBuffer	Pointer directly to the incoming raw network transmission payload data.	Used by the analyst to verify the precise landing boundaries of the network data packet.
+ 0x48	DWORD nByteProcessed	Tracks total number of received and handled network buffer data bytes.	Reset to equal the total size of the network packet to trick the system tracking logic.
+ 0x58	QWORD pSrvNetWskStruct	Holds the address of an internal sub-structure containing operational callback functions.	Primary Target: Overwritten to point directly to the payload logic located on the predictable HAL heap page.
+ 0x70	MDL mdl1	Memory Descriptor List tracking physical memory mapping constraints.	Overwritten to allow arbitrary kernel-space read/write capabilities across system space.

How the overflow occurs
 (The SrvOs2FeaListSizeToNt integer wrap).
Where the overflow goes (Corrupting the adjacent SRVNET_BUFFER via pool-grooming holes).
How execution is safely hijacked (Triggering the modified pSrvNetWskStruct callback function when closing connection states).

