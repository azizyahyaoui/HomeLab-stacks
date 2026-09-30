# Sysmon

## Introduction

Sysmon (System Monitor) is a Microsoft host-monitoring utility that records
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

Sysmon can help filter telemetry, reduce storage and SIEM licensing costs, and
limit detection noise. It also enhances SIEM coverage by filling visibility
gaps with detailed host activity.

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
	filename and path if they differ from, in my case is under `.\windows\config\sysmonconfig.xml`.
3. Install Sysmon with a configuration:

	```powershell
	.\Sysmon64.exe -accepteula -i .\sysmonconfig.xml

	System Monitor v15.22 - System activity monitor
	By Mark Russinovich and Thomas Garnier
	Copyright (C) 2014-2026 Microsoft Corporation
	Using libxml2. libxml2 is Copyright (C) 1998-2012 Daniel Veillard. All Rights Reserved.
	Sysinternals - www.sysinternals.com

	Loading configuration file with schema version 4.90
	Sysmon schema version: 4.91
	Configuration file validated.
	Sysmon64 installed.
	SysmonDrv installed.
	Starting SysmonDrv. 
	SysmonDrv started.
	Starting Sysmon64..
	Sysmon64 started.
	```
	> Here sysmon startmon install driver for monitoring windows logs cause is talk direct to the windows kernel to detect any activity in the system especially for LOTL process creation and file modification and network connection and so on.

4. Verify that Sysmon is running and that expected events appear in the
	 Operational channel. A missing event can mean the event type is not enabled
	 or is filtered by the configuration.
5. Apply a reviewed configuration update with:

	 ```powershell
	.\Sysmon64.exe -c .\sysmonconfig.xml
	 ```
6. To apply the default configuration:

	 ```powershell
	.\Sysmon64.exe -c --
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

---

## Custom rules

This section contains examples of custom Sysmon rules. Additional examples can
be found under `sysmon/windows/custom`. Review and test each rule before
deployment; Sysmon records activity but does not prevent it.

> [!NOTE]
> Reminder: In Event Viewer, find it under **Applications and Services Logs** >
> **Microsoft** > **Windows** > **Sysmon** > **Operational**. On Windows hosts,
> use this channel to validate any custom rule and confirm the resulting events.

### Monitor files created in Downloads

The following rule collects Sysmon Event ID 11 (`FileCreate`) when a file is
created in a user's `Downloads` directory. Adjust the path condition if the
environment uses a different download location.

```xml
<Sysmon schemaversion="4.90">
	<EventFiltering>
		<FileCreate onmatch="include">
			<TargetFilename condition="contains">\Downloads\</TargetFilename>
		</FileCreate>
	</EventFiltering>
</Sysmon>
```

Merge this rule into the existing configuration rather than replacing other
production rules, then apply the reviewed configuration from an elevated
PowerShell session:

```powershell
 .\Sysmon64.exe -c .\sysmonconfig.xml
```

Verify the resulting Event ID 11 events in the
`Microsoft-Windows-Sysmon/Operational` channel. A broad `contains` match may
include application data directories; use a user-specific `begin with` path
when narrower collection is required.


## Sysmon integrated with Wazuh

> [!WARNING]
> This section assumes Wazuh is installed and UP&running.

Wazuh can collect and analyze Sysmon events, but it does not replace a
carefully designed Sysmon configuration. Sysmon decides what activity is
recorded; Wazuh then decodes those events and applies correlation, severity,
and alerting. Configure both systems together and validate the entire path from
Windows host to Wazuh manager before relying on an alert.

For a practical walkthrough of using Sysmon for advanced Windows monitoring,
see the [Wazuh-SIEM lab guide](https://github.com/azizyahyaoui/Wazuh-SIEM/blob/master/course/WazuhSIEM.md#use-sysmon-for-advanced-windows-monitoring).
A matching example configuration is available in this repository at the
[Wazuh Sysmon configuration](https://github.com/azizyahyaoui/HomeLab-stacks/blob/master/security/telemetry/sysmon/Wazuh/wazuh_sysmonconf.xml).
Wazuh also maintains an example configuration in its
[Sysmon configuration resource](https://wazuh.com/resources/blog/emulation-of-attack-techniques-and-detection-with-wazuh/sysmonconfig.xml).

#### Deployment guidance

- Treat these configurations as reference material. Review every rule,
	exclude known-good software, and test changes on representative hosts before
	broad deployment.
- Keep Sysmon collection focused on events that support investigation. High-
	volume events, especially image-load and network telemetry, can increase
	storage, processing, and alert noise.
- Confirm that the Wazuh agent is collecting the Sysmon channel and that the
	manager receives structured fields, including event ID, timestamp, host,
	process image, command line, user, and hashes when available.
- Tune Wazuh rules separately from Sysmon filters. A Sysmon exclusion prevents
	the event from being collected; a Wazuh rule exclusion only changes downstream
	analysis.
- Document the configuration version, local changes, deployment scope, and
	rollback procedure. Do not commit credentials or collected event data.

After each change, generate safe test activity, verify the event in the
Windows Sysmon Operational channel, confirm that it reaches Wazuh, and ensure
that the resulting alert contains enough context for investigation.

---

## Sysmon integrated with ELK stack

TODO

---

## Hiding service and driver for sec purpose

[text](https://youtu.be/MlGc44dfFBg?t=900)
[text](https://youtu.be/MlGc44dfFBg?t=1336)

[Sysmon DarkOperator extension](https://marketplace.visualstudio.com/publishers/DarkOperator)