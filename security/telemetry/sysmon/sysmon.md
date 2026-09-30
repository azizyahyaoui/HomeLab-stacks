# Sysmon

## Introduction

So Sysmon (System Monitor) is a Microsoft host-monitoring utility that records
detailed system activity for security investigation and detection. This guide
covers Sysmon for Windows and Sysmon for Linux; their installation, event
formats, and log destinations are platform-specific.

## Official documentation

- [Sysmon for Windows](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)
- [Sysmon for Linux](https://github.com/microsoft/SysmonForLinux)

## What Sysmon does

Sysmon installs a host service and, on Windows, a driver that observe selected
system activity and write events for other tools to collect. Depending on the
platform and configuration, events can describe process creation, network
connections, image loads, file activity, registry activity, and DNS queries.
The configuration controls which event types are collected and filtered.

Sysmon is a telemetry source, not an antivirus, endpoint detection and response
(EDR) product, or prevention control. It does not decide whether activity is
malicious, block processes, or replace operating-system auditing. Detection
depends on the events collected, their context, and the analysis performed by
another tool.

## Windows event reference

These commonly useful Windows event IDs are a starting point, not a complete
coverage checklist. Confirm the event definitions against Microsoft's current
documentation and the deployed Sysmon version. Whether an event is generated
depends on the active configuration.

| Event ID | Event | Typical investigative value |
| --- | --- | --- |
| 1 | Process creation | Process command line, parent process, hashes, and user context. |
| 3 | Network connection | Network connections associated with a process. |
| 6 | Driver loaded | Driver image and signature information. |
| 7 | Image loaded | DLL and executable image loads; can be high volume. |
| 10 | Process access | One process opening or accessing another process. |
| 11 | File created | File creation activity. |
| 12-14 | Registry events | Registry key/value creation, deletion, or value changes. |
| 22 | DNS query | DNS queries associated with a process. |

### Windows Event Log

Sysmon for Windows writes to the `Microsoft-Windows-Sysmon/Operational`
channel. In Event Viewer, find it under **Applications and Services Logs** >
**Microsoft** > **Windows** > **Sysmon** > **Operational**. On Windows hosts,
the underlying event log files live in the standard Windows Event Log directory:
`C:\Windows\System32\Winevt\Logs`.

The Sysmon operational log is typically stored in the EVTX file associated with
that channel, and it can be reviewed directly or collected by a log forwarder.

From PowerShell, inspect recent events with:

```powershell
Get-WinEvent -LogName 'Microsoft-Windows-Sysmon/Operational' -MaxEvents 20 |
	Select-Object TimeCreated, Id, ProviderName, Message
```

The message is useful for manual inspection; collectors should preserve the
structured event fields as well as the rendered message.

## Install and manage on Windows

1. Download Sysmon from the official Microsoft page above and review the
	 included license and usage information.
2. Review the XML configuration before installing. Run the commands from an
	elevated PowerShell session, substituting the reviewed configuration's
	filename and path if they differ from `sysmonconfig.xml`.
3. Install Sysmon with a configuration:

	 ```powershell
	.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml
	 ```

4. Verify that Sysmon is running and that expected events appear in the
	 Operational channel. A missing event can mean the event type is not enabled
	 or is filtered by the configuration.
5. Apply a reviewed configuration update with:

	 ```powershell
	.\Sysmon64.exe -c .\sysmonconfig.xml
	 ```

Consult Microsoft's documentation for architecture-specific executable names,
command options, upgrades, and removal. Test installation and configuration
changes on a non-critical host before wider deployment.

## Linux

Use the [Sysmon for Linux project documentation](https://github.com/microsoft/SysmonForLinux)
for supported distributions, installation, configuration syntax, service
management, and event output. Do not assume that Windows event IDs, the Windows
Operational channel, or Windows XML settings map directly to Linux.

Keep Linux-specific configurations under `linux/configs/` and document the
tested distribution, Sysmon version, and log destination alongside each
configuration. Validate events locally before configuring a log forwarder.

## Configuration and operations

- Keep platform configurations separate under `windows/configs/` and
	`linux/configs/`. The directories are currently empty; no baseline is deployed
	by this repository yet.
- Record the configuration source, version or revision, and any local changes.
	Review upstream updates before adopting them.
- Prefer focused collection and explicit filters. Measure event volume and
	review privacy implications before enabling high-volume event types.
- Back up the currently deployed configuration and document a rollback path
	before applying changes.
- Verify host clock synchronization, event retention, and access controls. Do
	not commit host-specific data, credentials, or collected event logs here.

## Forwarding and analysis

Sysmon events are intended to complement other security telemetry and may be
integrated with solutions such as Wazuh and the repository's ELK Stack. This
repository does not currently configure Sysmon deployment or event forwarding.
Before relying on an integration, verify that events arrive with timestamps and
structured fields intact, that parsing is correct, and that retention and access
controls meet the needs of the environment.

## Custom Rules
TODO

### Wazuh custom rules

> [!NOTE]
> **THIS PART NEED WAZUH UP AND RUNNING!**

Wazuh can collect and analyze Sysmon events, but it does not replace a
carefully designed Sysmon configuration. Sysmon determines which activity is
recorded; Wazuh decodes the resulting events and applies rules, severity, and
alerting. Configure both components together and validate the complete path
from the Windows host to the Wazuh manager before relying on an alert.

For a practical walkthrough of using Sysmon for advanced Windows monitoring,
see the [Wazuh-SIEM lab guide](https://github.com/azizyahyaoui/Wazuh-SIEM/blob/master/course/WazuhSIEM.md#use-sysmon-for-advanced-windows-monitoring).
The corresponding example configuration is available as the
[Wazuh Sysmon configuration in this repository](https://github.com/azizyahyaoui/HomeLab-stacks/blob/master/security/telemetry/sysmon/Wazuh/wazuh_sysmonconf.xml).
Wazuh also publishes a maintained example in its
[Sysmon configuration resource](https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml).

#### Deployment guidance

- Treat these configurations as reference material. Review every rule,
	exclude known-good software, and test changes on representative hosts before
	deploying them broadly.
- Keep Sysmon collection focused on events that support an investigation.
	High-volume events, especially image-load and network telemetry, can increase
	storage, processing, and alert noise.
- Confirm that the Wazuh agent collects the Sysmon channel and that the manager
	receives structured event fields, including the event ID, timestamp, host,
	process image, command line, user, and hashes where available.
- Tune Wazuh rules separately from Sysmon filters. A Sysmon exclusion prevents
	the event from being collected, while a Wazuh rule exclusion only changes
	downstream analysis.
- Document the configuration version, local modifications, deployment scope,
	and rollback procedure. Do not commit credentials or collected event data.

After each change, generate safe test activity, verify the event in the
Windows Sysmon Operational channel, confirm its arrival in Wazuh, and check
that the resulting alert contains enough context for investigation.