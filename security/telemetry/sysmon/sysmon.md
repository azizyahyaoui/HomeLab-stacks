# Sysmon — Host Telemetry

Sysmon (System Monitor) is a host telemetry tool that records detailed system activity so other security tools can collect, correlate, and investigate it.

This document is the **core Sysmon reference for this homelab**. It focuses on:

- What Sysmon is and where it fits
- Windows and Linux telemetry
- Installation and verification
- Configuration and filtering
- Useful event types
- Operational practices
- Small hands-on telemetry experiments

Wazuh and ELK are downstream consumers of this telemetry. Their detailed integration work belongs outside this core document.

## Official documentation

- [Sysmon for Windows](https://learn.microsoft.com/sysinternals/downloads/sysmon)
- [Sysmon overview](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/overview)
- [Enable and configure Sysmon](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/how-to-enable-sysmon)
- [Sysmon configuration files](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-configuration-files)
- [Sysmon events](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-events)
- [Sysmon for Linux](https://github.com/microsoft/SysmonForLinux)

---

## 1. What Sysmon is

Sysmon provides detailed, low-level telemetry about activity on a host.

Depending on the platform and configuration, it can record information such as:

- Process creation and termination
- Parent/child process relationships
- Command lines
- Process and file hashes
- Network connections
- DNS queries
- Driver and image loading
- File creation and deletion
- Registry activity
- Named pipes
- WMI activity
- Process tampering
- Sysmon configuration changes

The exact events available and the way they are configured depend on the Sysmon platform and version.

### What Sysmon is not

Sysmon is **not**:

- An antivirus
- An EDR
- A SIEM
- A firewall
- A prevention engine
- A detection engine by itself

Sysmon records observations. Another system can then interpret those observations.

For example:

```
Windows activity
      │
      ▼
    Sysmon
      │
      ▼
Windows Event Log
      │
      ├──────────► Wazuh ──────► Detection / Alerting
      │
      └──────────► ELK ────────► Search / Investigation / Visualization
```

A useful mental model for this lab:

> **Sysmon = telemetry / eyes**  
> **Wazuh = detection and response**  
> **ELK = investigation and visualization**

No single Sysmon event should automatically be treated as proof of malicious activity. The value comes from context and correlation across events.

---

## 2. Windows Sysmon

### 2.1 Windows event channel

Sysmon for Windows writes events to:

<code>Microsoft-Windows-Sysmon/Operational</code>

In Event Viewer:

```
Applications and Services Logs
└── Microsoft
    └── Windows
        └── Sysmon
            └── Operational
```

The underlying Windows Event Log files are stored under:

<code>C:\Windows\System32\Winevt\Logs</code>

From PowerShell:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 20 |
    Select-Object TimeCreated, Id, ProviderName, Message
```

When collecting Sysmon centrally, prefer the structured event fields rather than relying only on the rendered message.

### 2.2 Useful Windows events

This is a practical starting reference, not a complete event catalogue.

| ID | Event | Why it matters |
|---:|---|---|
| 1 | Process Create | Execution, command line, parent process, hashes |
| 2 | File creation time changed | Timestamp manipulation |
| 3 | Network Connect | Process-to-network relationships |
| 5 | Process Terminated | Execution timelines |
| 6 | Driver Load | Kernel driver activity |
| 7 | Image Load | DLL/executable loading; potentially high volume |
| 8 | CreateRemoteThread | Cross-process thread creation |
| 10 | Process Access | Process interaction and memory-access investigations |
| 11 | File Create | Dropped files, staging, persistence artifacts |
| 12–14 | Registry Event | Registry creation, deletion, and value changes |
| 15 | FileCreateStreamHash | Alternate data stream activity |
| 17–18 | Pipe Event | Named-pipe activity |
| 19–21 | WMI Event | WMI persistence and activity |
| 22 | DNS Query | Process-to-DNS relationships |
| 23 / 26 | File Delete | File deletion, with different archival behavior |
| 24 | Clipboard Change | Clipboard activity |
| 25 | Process Tampering | Process image manipulation |
| 27–29 | Executable file events | Executable detection/blocking telemetry |
| 255 | Error | Sysmon internal errors |

For the complete and version-specific event reference, use Microsoft's [Sysmon events documentation](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-events).

### 2.3 Think in event chains

Sysmon becomes more useful when events are correlated into a timeline.

```
User opens document
      │
      ▼
Event 1 — Process Create
      │
      ▼
Event 22 — DNS Query
      │
      ▼
Event 3 — Network Connect
      │
      ▼
Event 11 — File Create
      │
      ▼
Event 12–14 — Registry activity
```

The individual events are observations. The sequence provides the investigative context.

---

## 3. Install and verify Sysmon on Windows

There are currently two Windows deployment models worth knowing:

- **Windows 10:** standalone Sysinternals Sysmon remains the relevant approach.
- **Windows 11:** Sysmon is also available as a built-in optional Windows feature starting in February 2026.

Built-in Sysmon and standalone Sysmon are not meant to coexist on the same Windows installation. See Microsoft's current documentation before changing an existing installation.

### 3.1 Standalone Sysmon — this lab

The lab Windows VM currently uses the standalone Sysinternals package.

Run the following from an elevated PowerShell session:

```powershell
.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
```

The configuration file path should point to the reviewed configuration you actually deploy.

### 3.2 Verify the service

```powershell
Get-Service Sysmon*
```

Then verify the event channel:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 10
```

Also check Event Viewer manually when troubleshooting.

### 3.3 Lab installation note

The Windows 10 lab VM was successfully tested with standalone Sysmon 15.22.

Keep version-specific installation output in lab notes rather than treating one installed version as a permanent requirement for every deployment.

---

## 4. Sysmon configuration

Sysmon configuration is written in XML.

A configuration controls:

- Which event types are collected
- Which fields are filtered
- Which activity is included or excluded
- Hashing and other enrichment
- Event volume and noise

A simplified structure looks like:

```xml
<Sysmon schemaversion="4.90">
    <HashAlgorithms>SHA256</HashAlgorithms>

    <EventFiltering>

        <ProcessCreate onmatch="include">
            <!-- filtering conditions -->
        </ProcessCreate>

        <NetworkConnect onmatch="exclude">
            <!-- exclusions -->
        </NetworkConnect>

    </EventFiltering>
</Sysmon>
```

### 4.1 Include vs exclude

The two basic modes are:

- <code>include</code> — log only matching events
- <code>exclude</code> — log events except those matching the rule

Microsoft's current documentation notes that exclusion rules take precedence over inclusion rules.

Same-field rules and different-field rules also have specific evaluation behavior, so test the actual configuration rather than assuming the XML reads like ordinary boolean logic.

### 4.2 Useful conditions

Common conditions include:

```
is
is not
contains
contains any
contains all
excludes
excludes any
excludes all
begin with
end with
not begin with
not end with
less than
more than
image
```

The <code>image</code> condition is useful for image/path fields such as <code>Image</code>, <code>ParentImage</code>, <code>SourceImage</code>, and <code>TargetImage</code>.

Example:

```xml
<ProcessCreate onmatch="include">
    <Image condition="image">powershell.exe</Image>
</ProcessCreate>
```

This is intended for image-path matching, rather than arbitrary text fields such as <code>CommandLine</code>.

### 4.3 Configuration engineering

Do not start by enabling everything blindly.

A practical workflow is:

```
Start with high-value telemetry
        │
        ▼
Measure event volume
        │
        ▼
Inspect the events
        │
        ▼
Identify useful signal / noise
        │
        ▼
Tune filters
        │
        ▼
Test again
        │
        ▼
Document the configuration
```

For this homelab, every configuration should record:

- Sysmon version
- Schema version
- Source/reference
- Local modifications
- Date tested
- Host/platform tested on
- Expected event volume
- Rollback procedure

Keep Windows configurations under:

<code>security/telemetry/sysmon/windows/configs/</code>

---

## 5. Example: monitor files created in Downloads

A small experiment is more useful than immediately deploying a huge configuration.

The following rule collects Event ID 11 when a file is created under a path containing <code>\Downloads\</code>:

```xml
<Sysmon schemaversion="4.90">
    <EventFiltering>
        <FileCreate onmatch="include">
            <TargetFilename condition="contains">\\Downloads\\</TargetFilename>
        </FileCreate>
    </EventFiltering>
</Sysmon>
```

Apply the reviewed configuration:

```powershell
.\Sysmon64.exe -c .\sysmonconfig.xml
```

Then create a harmless test file in Downloads and verify Event ID 11:

```powershell
Get-WinEvent -FilterHashtable @{
    LogName = 'Microsoft-Windows-Sysmon/Operational'
    Id      = 11
} -MaxEvents 10
```

The goal of this experiment is to understand:

```
Action
  ↓
Sysmon observes it
  ↓
Event ID 11
  ↓
Windows Event Log
  ↓
Later: Wazuh / ELK
```

This is telemetry validation, not detection by itself.

---

## 6. Sysmon for Linux

Sysmon also has a Linux implementation maintained by Microsoft.

The Linux implementation is **not simply Windows Sysmon copied onto Linux**. Keep the platform-specific behavior and event model separate.

Refer to the [Sysmon for Linux project](https://github.com/microsoft/SysmonForLinux) for:

- Supported distributions
- Installation
- Package requirements
- Configuration syntax
- Service management
- Event output
- Kernel/eBPF requirements

Sysmon for Linux uses the SysinternalsEBPF component and Linux kernel eBPF capabilities. Events are written through the Linux system logging infrastructure rather than the Windows Event Viewer.

The exact log destination depends on the distribution and logging setup.

For this homelab, Linux configurations belong under:

<code>security/telemetry/sysmon/linux/configs/</code>

Record the tested Linux distribution, kernel, Sysmon version, configuration revision, and log destination with each experiment.

> **Important:** Do not reuse Windows Event ID assumptions for Linux. Treat Windows and Linux telemetry as separate platform implementations.

---

## 7. Configuration and operational practices

### Keep configurations versioned

Store reviewed configurations in the repository:

```
security/
└── telemetry/
    └── sysmon/
        ├── windows/
        │   └── configs/
        └── linux/
            └── configs/
```

Do not commit:

- Credentials
- Host-specific secrets
- Private identifiers
- Raw collected logs
- Large generated event dumps

### Before changing a deployed configuration

1. Save the current configuration.
2. Record the version/revision being tested.
3. Apply the change.
4. Verify Sysmon is running.
5. Generate safe test activity.
6. Verify the expected event locally.
7. Measure event volume.
8. Document the result.
9. Keep a rollback path.

### Useful management commands

```powershell
# Install
Sysmon64.exe -accepteula -i sysmonconfig.xml

# Update configuration
Sysmon64.exe -c sysmonconfig.xml

# Dump current configuration
Sysmon64.exe -c

# Reset configuration
Sysmon64.exe -c --

# Show configuration schema
Sysmon64.exe -s

# Show all schema versions
Sysmon64.exe -s all

# Uninstall
Sysmon64.exe -u
```

Always check the command syntax supported by the installed Sysmon version.

---

## 8. Sysmon and Wazuh

Wazuh is a downstream consumer of Sysmon telemetry.

The division of responsibility is:

```
Sysmon
  │
  │ records activity
  ▼
Windows Event Log
  │
  │ collected by Wazuh Agent
  ▼
Wazuh
  │
  ├── decoding
  ├── rules
  ├── correlation
  ├── severity
  └── alerts
```

Keep the detailed Wazuh walkthrough in the separate [Wazuh-SIEM](https://github.com/azizyahyaoui/Wazuh-SIEM) repository.

The HomeLab-stacks repository should contain the **actual lab integration and deployment details**, while Wazuh-SIEM remains the Wazuh-focused learning material.

Current Wazuh/Sysmon reference material:

- [Wazuh-SIEM — Advanced Windows monitoring with Sysmon](https://github.com/azizyahyaoui/Wazuh-SIEM/blob/master/course/WazuhSIEM.md#use-sysmon-for-advanced-windows-monitoring)
- [Local Wazuh Sysmon configuration](./Wazuh/wazuh_sysmonconf.xml)
- [Wazuh Sysmon configuration resource](https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml)

> Detailed Wazuh collection, decoder, rule, and alert configuration should eventually live under <code>security/integrations/wazuh/</code>, not inside this core Sysmon document.

---

## 9. Sysmon and ELK

ELK is another downstream consumer of telemetry.

The intended lab flow is:

```
Windows VM
   │
   └── Sysmon
        │
        ▼
   Event collection
        │
        ▼
   Elasticsearch
        │
        ▼
      Kibana
```

Detailed ELK ingestion, parsing, index design, and dashboards should live under the ELK/integration documentation rather than becoming part of the core Sysmon reference.

**Status:** Integration not yet configured.

---

## 10. Homelab workflow

The long-term goal is:

```
┌─────────────────┐
│   Windows VM    │
│  test activity  │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│      Sysmon     │
│    telemetry    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Windows Event   │
│      Log        │
└────────┬────────┘
         │
     ┌───┴────┐
     ▼        ▼
  Wazuh       ELK
     │        │
     ▼        ▼
Detection   Hunting /
& alerts    visualization
```

The practical learning loop is:

```
Activity
   ↓
Telemetry
   ↓
Detection
   ↓
Investigation
   ↓
Tuning
   ↓
Repeat
```

This keeps Sysmon in its proper role: **collect useful host telemetry so the rest of the security stack has something meaningful to analyze.**

---

## 11. Next experiments

Planned Sysmon experiments for the homelab:

- [ ] Process creation — Event ID 1
- [ ] Network connection — Event ID 3
- [ ] DNS query — Event ID 22
- [ ] File creation — Event ID 11
- [ ] Registry activity — Event IDs 12–14
- [ ] Process access — Event ID 10
- [ ] Process tampering — Event ID 25
- [ ] Named pipes — Event IDs 17–18
- [ ] WMI activity — Event IDs 19–21
- [ ] Linux Sysmon telemetry
- [ ] Sysmon → Wazuh collection
- [ ] Sysmon → ELK ingestion
- [ ] Attack → Telemetry → Detection → Visualization

---

## References

- [Microsoft Sysmon](https://learn.microsoft.com/sysinternals/downloads/sysmon)
- [Sysmon Overview](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/overview)
- [Enable and configure Sysmon](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/how-to-enable-sysmon)
- [Sysmon configuration files](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-configuration-files)
- [Sysmon events](https://learn.microsoft.com/en-us/windows/security/operating-system-security/sysmon/sysmon-events)
- [Sysmon for Linux](https://github.com/microsoft/SysmonForLinux)
