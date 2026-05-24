# Milestone 2: Windows Endpoint with Sysmon and Splunk Forwarder

## Objective

Add a Windows Server endpoint to the lab, install Sysmon for rich security telemetry, and configure the Splunk Universal Forwarder to ship those logs to the Splunk SIEM. By the end,
Sysmon events flow from the Windows VM into Splunk in near-real-time, queryable by sourcetype.

## Architecture Decision

Chose Windows Server 2022 over Windows 10 for the endpoint.
Reasoning:

- Most high-value attack targets in real environments are servers, not desktops — domain controllers, file servers, web servers
- Server 2022 supports installing the Active Directory role, which opens up AD attack/detection scenarios in later  milestones
- 180-day evaluation period versus 90 days for Windows 10 —  more runway for the project
- My offensive cert background (CRTP) is specifically about AD exploitation, so a domain-capable lab puts those skills to defensive use

## Environment

- **Splunk VM:** Ubuntu 24.04, 192.168.68.130, Splunk 9.4.x
  (from Milestone 1)
- **Windows VM (new):**
  - Name: SOC-Lab-DC
  - OS: Windows Server 2022 Standard Evaluation (Desktop Experience)
  - RAM: 4 GB
  - CPU: 2 cores
  - Disk: 60 GB
  - Network: NAT
  - IP: 192.168.68.131
- **Hypervisor:** VMware Workstation Pro 17
- **Sysmon version:** 15.x with SwiftOnSecurity config
- **Universal Forwarder version:** 10.2.3

## Setup Steps

### 1. Windows Server 2022 VM Creation

Created the VM in VMware. Hit the Easy Install product key prompt on first attempt — VMware tries to auto-install Windows and demands a license key. Worked around it by cancelling and restarting the wizard in Custom mode, choosing "I will install
the operating system later" so the ISO would be attached but not auto-run.

After VM creation, attached the Server 2022 evaluation ISO via VM Settings → CD/DVD → Use ISO image file.

### 2. Windows Server 2022 Installation

Installed Windows Server 2022 Standard Evaluation with Desktop Experience — the GUI version, not Server Core. Took ~20 minutes with several auto-reboots during install.

Created the local Administrator password during setup.

### 3. Post-Install Network Verification

After first login, confirmed network connectivity to the Splunk VM:

```
ipconfig          # confirmed VM IP: 192.168.68.131
ping 192.168.68.130   # confirmed reachable from Splunk VM
```

Both VMs on the same VMware NAT network, able to communicate.

### 4. VMware Tools Installation

Installed VMware Tools to enable host/guest integration — specifically clipboard sharing, which makes copy/paste of commands from the host into the VM possible. Critical for productivity during the rest of the milestone.

First install attempt didn't complete properly; VMware Tools wasn't showing in Programs and Features after reboot. Redid the install: VM menu → Install VMware Tools, then ran
setup64.exe as administrator from the mounted DVD drive.
Reboot required.

### 5. Sysmon Installation

Downloaded two things inside the Windows VM:

- **Sysmon** from Microsoft Sysinternals
- **SwiftOnSecurity Sysmon config** from GitHub — the industry-standard starting config for SOC-friendly logging

Extracted both, copied the config XML into the Sysmon folder alongside `Sysmon64.exe`, then installed from an Administrator Command Prompt:

```
cd C:\Users\Administrator\Downloads\Sysmon
Sysmon64.exe -accepteula -i sysmonconfig-export.xml
```

Output confirmed: `Sysmon started.`

Verified Sysmon was logging by querying its event channel directly:

```
wevtutil qe Microsoft-Windows-Sysmon/Operational /c:3 /f:text
```

Returned events with process creation details — Sysmon was working.

### 6. Splunk Universal Forwarder Installation

Downloaded the 64-bit MSI installer from splunk.com. Walked through the installer:

- Accepted the license
- Left deployment server blank (not used in lab)
- Set receiving indexer to `192.168.68.130:9997` (the Splunk VM)
- Created an admin credential for the forwarder

Installer completed and the SplunkForwarder service started automatically.

### 7. Configuring Windows Event Log Inputs

This is where the install hit its first real problem. Splunk Universal Forwarder 10.x changed how Windows event log inputs work via CLI — the standard `splunk add monitor "WinEventLog:..."`
command throws a confusing error: `Parameter name: Path must be a file or directory.`

Switched approach to creating the input configuration via `inputs.conf` directly, which is the production-grade approach anyway.

Tried Notepad first, but the file wasn't ending up in the right location — likely due to UAC or Notepad's path resolution sending it to a different folder. Bypassed that by writing the config straight from cmd:

```
(
echo [WinEventLog://Security]
echo disabled = 0
echo index = main
...
echo [WinEventLog://Microsoft-Windows-Sysmon/Operational]
echo disabled = 0
echo index = main
echo renderXml = 1
echo sourcetype = XmlWinEventLog:Microsoft-Windows-Sysmon/Operational
) > "C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf"
```

Verified the file content with `type` then restarted the forwarder:

```
splunk restart
```

### 8. Enabling Receiving on Splunk

The Splunk server also needs to be configured to listen for incoming forwarder traffic. From the Splunk web UI:

Settings → Forwarding and receiving → Configure receiving → New Receiving Port → `9997`.

### 9. First Verification — Most Logs Flow, Sysmon Doesn't

First search in Splunk showed 1,846 events arriving — but breaking down by sourcetype revealed only Security, System, and Application Windows logs. **Zero Sysmon events.**

The config was in place. Splunk could see the input definition in `btool` output. The Sysmon event channel existed and was populated. But the forwarder wasn't shipping it.

### 10. Root Cause — Forwarder Service Account Permissions

Checked which account the SplunkForwarder service runs as:

```
sc qc SplunkForwarder
```

Result: `SERVICE_START_NAME : NT SERVICE\SplunkForwarder`

This is a limited virtual service account. By default it doesn't have permission to read certain Microsoft-Windows-* event channels, including the Sysmon channel.

**Fix:** change the service to run as `LocalSystem`:

```
sc stop SplunkForwarder
sc config SplunkForwarder obj= "LocalSystem"
sc start SplunkForwarder
```

After the change, Sysmon events started flowing within 90 seconds.

This is a real-world gotcha that catches a lot of people installing Splunk on Windows for the first time. In a production environment, you'd use a dedicated AD service account with explicit permissions to the event channels rather than LocalSystem, but for a home lab LocalSystem is appropriate.

### 11. Final Verification

In Splunk search:

```
index=main sourcetype="XmlWinEventLog:Microsoft-Windows-Sysmon/Operational"
```

Last 15 minutes: 363+ Sysmon events.

Expanded events showed the rich Sysmon fields — Image, CommandLine, ParentImage, ParentCommandLine, ProcessGuid, User, IntegrityLevel — exactly the telemetry a SOC analyst needs for behavioural investigation.

## VM Snapshots

After confirming the end-to-end pipeline was working, took snapshots on both VMs named `M2 - Sysmon logs flowing` so the working state is preserved before any further changes.

## What Went Wrong / What I Learned

- **VMware's Easy Install gets in the way for proper lab work.** It tries to automate the install and demands a license key even for evaluation ISOs. Cancelling and using Custom mode with manual ISO attachment is the cleaner path.
- **Splunk Universal Forwarder 10.x doesn't accept the old `splunk add monitor "WinEventLog:..."` CLI syntax.** Almost every tutorial online still references that command because it worked in 9.x. The current way is via `inputs.conf`.
- **Notepad's "Save As" with paths in Program Files can fail silently.** The file ends up somewhere else without an obvious error. Writing config files directly from cmd is more reliable.
- **The biggest gotcha: SplunkForwarder service account permissions.** This isn't documented prominently and the symptom (some Windows event logs arriving, others not) doesn't immediately point to the cause. The fix is one command but finding it took methodical debugging — checking inputs were defined, checking the channel existed, finally checking the service account.
- **`btool inputs list --debug | findstr Sysmon`** is the diagnostic command that proved the config was being read Useful pattern for any future "is my config actually loading" question.

## What's Next

Milestone 3: install Kali Linux as the attacker VM, run a first attack (likely an Nmap scan or basic enumeration) from Kali against the Windows endpoint, find the evidence in Splunk, and write the first detection query.

## Screenshots

- `Screenshots/03-sysmon-events-in-splunk.png` — Splunk search showing Sysmon events flowing in
- `Screenshots/04-sysmon-event-details.png` — Expanded view of a single Sysmon event showing the rich field set
