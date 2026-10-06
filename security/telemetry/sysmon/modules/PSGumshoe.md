# PowerShell module PSGumshoe

**References:** [TrustedSec video](https://www.youtube.com/watch?v=2JHjRR2Wt4g&list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7&index=12) · [Sysmon Community Guide](https://github.com/trustedsec/SysmonCommunityGuide/tree/master) · [Microsoft Sysmon documentation](https://learn.microsoft.com/en-us/sysinternals/downloads/sysmon)

PSGumshoe is a PowerShell module designed to simplify interaction with Sysmon telemetry. It helps security analysts and homelab operators query Sysmon events quickly, especially process creation and configuration change events, without manually parsing Windows Event Log data.

## Summary

Sysmon collects host activity; PSGumshoe provides PowerShell commands for reading and shaping the events Sysmon has already recorded. It does not enable collection, replace a SIEM, or guarantee that an event exists: results depend on the host's Sysmon version, configuration, event-log retention, and the PSGumshoe version installed.

Start an investigation by checking for unexpected configuration changes, then review process activity and pivot into the event types relevant to the question:

```powershell
Get-SysmonConfigChange
Get-SysmonProcessCreateEvent | Select-Object -First 20
Get-SysmonNetworkConnect | Select-Object -First 20
```

Use the event-specific examples below as starting points, not production-ready detection rules. Validate event support and XML schema against the installed Sysmon version, test filters on representative hosts, and review volume and exclusions before deployment. File and clipboard archiving can retain sensitive content and needs explicit access, storage, and retention controls.

This guide covers:

- Reviewing Sysmon configuration updates
- Investigating process execution
- Correlating parent-child process relationships
- Exploring network, driver, process-access, raw-access, pipe, DNS, registry, WMI, and file events
- Using sample filters carefully and correlating findings with other telemetry

---

## Installing PSGumshoe

PSGumshoe is installed from the PowerShell Gallery. Start by checking the current execution policy rather than changing it automatically:

```powershell
Get-ExecutionPolicy -List
```

Install the module for your user:

```powershell
Install-Module -Name PSGumshoe -Repository PSGallery -Scope CurrentUser
```

See the [PSGumshoe package on PowerShell Gallery](https://www.powershellgallery.com/packages/PSGumshoe/) for available versions and module details. If you need a specific release, use its actual version number with `-RequiredVersion`.

### If PSGumshoe will not run

On a lab machine, you may encounter an execution-policy error when PowerShell tries to load module scripts. If that happens, change the policy for **your current user only**:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

Then retry importing or using the module.

Verify the effective policy afterward:

```powershell
Get-ExecutionPolicy -List
```

`RemoteSigned` allows locally created scripts and requires downloaded scripts to be signed unless they have been unblocked. Execution policy is not a security boundary; avoid changing the `LocalMachine` policy just to get PSGumshoe working.

---

## Useful commands

### Review Sysmon configuration changes

`Get-SysmonConfigChange` retrieves Sysmon Event ID 16, which records Sysmon configuration changes.

```powershell
Get-SysmonConfigChange | more
```

Example output:

```powershell
EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 14126
UtcTime               : 2026-10-03 08:53:20.771
Configuration         : .\config\sysmonconfig.xml
ConfigurationFileHash : SHA256=<hash>
```

The exact fields and values depend on the Sysmon version and how the configuration was applied. For example:

- `EventId 16` means Sysmon configuration changed.
- `Configuration` contains configuration-related context; interpret its value using the full event and the installed Sysmon version.
- `ConfigurationFileHash`, when present, can help identify the configuration content.
- This is useful when validating whether a config change was intentional or suspicious.

The examples below show configuration-related values returned by the command:

```powershell
Get-SysmonConfigChange | more
```

Example output:

```powershell
EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 14548
UtcTime               : 2026-10-03 12:58:29.333
Configuration         : "C:\Users\User\workspace\Sysmon\Sysmon.exe" -c -k lsass.exe
ConfigurationFileHash : -

EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 14481
UtcTime               : 2026-10-03 12:25:22.115
Configuration         : "C:\Users\User\workspace\Sysmon\Sysmon.exe" -c -n malicious.exe
ConfigurationFileHash : -

EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 14340
UtcTime               : 2026-10-03 11:05:35.865
Configuration         : "C:\Users\User\workspace\Sysmon\Sysmon.exe" -c -h *
ConfigurationFileHash : -

EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 14333
UtcTime               : 2026-10-03 11:05:08.239
Configuration         : "C:\Users\User\workspace\Sysmon\Sysmon.exe" -c -l malicious.exe
ConfigurationFileHash : -

EventId               : 16
EventType             : ConfigChange
Computer              : Node2
EventRecordID         : 13141
UtcTime               : 2026-10-03 08:53:20.771
Configuration         : .\config\sysmonconfig.xml
ConfigurationFileHash : SHA256=4516404FA30EE87CEA558567820CDC78863CC4AB07889519E49EAC3CCA92E0D2
```

Treat these command lines as investigation leads, not self-explanatory verdicts. Verify the full command and switch meanings against the documentation for the installed Sysmon version, then compare the resulting configuration with the host's approved baseline. A configuration change is not necessarily malicious.

---

### Review process creation events

`Get-SysmonProcessCreateEvent` pulls Event ID 1 from Sysmon, which contains detailed information about a newly created process.

```powershell
Get-SysmonProcessCreateEvent | more
```

Example output:

```powershell
EventId           : 1
EventType         : ProcessCreate
Computer          : Node2
EventRecordID     : 14210
RuleName          : technique_id=T1204,technique_name=User Execution
UtcTime           : 2026-10-03 10:44:02.889
ProcessGuid       : {f75b2b9f-dc72-6ac0-4f07-000000001200}
ProcessId         : 804
Image             : C:\Windows\System32\notepad.exe
FileVersion       : 10.0.19041.5794 (WinBuild.160101.0800)
Description       : Notepad
Product           : Microsoft® Windows® Operating System
Company           : Microsoft Corporation
OriginalFileName  : NOTEPAD.EXE
CommandLine       : "C:\Windows\system32\notepad.exe"
CurrentDirectory  : C:\Users\User\
User              : Node2\User
LogonGuid         : {f75b2b9f-a8b6-6ac0-83c6-100000000000}
LogonId           : 0x10c683
TerminalSessionId : 1
IntegrityLevel    : Medium
Hashes            : SHA1=..., MD5=..., SHA256=..., IMPHASH=...
ParentProcessGuid : {f75b2b9f-a8da-6ac0-0801-000000001200}
ParentProcessId   : 2612
ParentImage       : C:\Windows\explorer.exe
ParentCommandLine : C:\Windows\Explorer.EXE
ParentUser        : Node2\User
```

Why this matters:

- `Image` identifies the executable that launched.
- `CommandLine` shows how it was started.
- `ParentImage` and `ParentCommandLine` help establish the execution chain.
- `User` and `IntegrityLevel` show who ran it and how privileged it was.
- `Hashes` assist with malware triage and file identification.
- `RuleName` may contain ATT&CK references or custom rule labels.

This is one of the main Sysmon events used to reconstruct execution flow.

---

## Reviewing process chains

A common investigative pattern is to select the fields needed for triage and remove duplicate rows:

```powershell
$processCreation = Get-SysmonProcessCreateEvent |
    Select-Object User, ParentCommandLine, CommandLine -Unique

$processCreation | Out-GridView
$processCreation | ConvertTo-SysmonRule
```

This approach helps answer key questions:

- Who launched a process?
- Which parent process created it?
- What command line was used?
- Is the process execution pattern normal or unusual?

It is useful when you want to inspect many events without dumping the full raw telemetry for every record.

---

## Exporting process data

You can also export the processed results to CSV for analysis or later review:

```powershell
$processCreation = Get-SysmonProcessCreateEvent |
    Select-Object User, ParentCommandLine, CommandLine -Unique

$processCreation | Export-Csv .\processdata.csv -NoTypeInformation
```

This is helpful when:

- hunting through a large dataset,
- preserving evidence for incident review,
- or passing data into other tools for correlation.

---

## Sysmon rule and filter logic

Sysmon rules and filters determine which telemetry is captured. The same process may be visible or absent depending on how the configuration is written.

A high-value operational principle is:

- Collect enough telemetry to answer the investigation question
- Avoid over-collecting noisy events
- Validate the filter behavior on a test system before production use

In practice, this means checking:

- what processes are included or excluded,
- whether network or image load rules are configured,
- whether hashes are being logged,
- and whether configuration changes match a reviewed baseline.

---

## Event-specific examples

### Network connections

```xml
<Sysmon schemaversion="4.91">
  <EventFiltering>
    <RuleGroup name="Selected destination ports" groupRelation="or">
      <NetworkConnect onmatch="include">
        <DestinationPort condition="is">80</DestinationPort>
        <DestinationPort condition="is">443</DestinationPort>
        <DestinationPort condition="is">53</DestinationPort>
        <DestinationPort condition="is">123</DestinationPort>
        <DestinationPort condition="is">389</DestinationPort>
        <DestinationPort condition="is">636</DestinationPort>
        <DestinationPort condition="is">3268</DestinationPort>
        <DestinationPort condition="is">3269</DestinationPort>
        <DestinationPort condition="is">88</DestinationPort>
        <DestinationPort condition="is">3389</DestinationPort>
        <DestinationPort condition="is">5985</DestinationPort>
        <DestinationPort condition="is">5986</DestinationPort>
        <DestinationPort condition="is">21</DestinationPort>
        <DestinationPort condition="is">22</DestinationPort>
        <DestinationPort condition="is">23</DestinationPort>
        <DestinationPort condition="is">445</DestinationPort>
        <DestinationPort condition="is">135</DestinationPort>
        <DestinationPort condition="is">138</DestinationPort>
        <DestinationPort condition="is">139</DestinationPort>
      </NetworkConnect>
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```

This example includes connections to selected common ports; a port alone does not identify the protocol in use or indicate command-and-control activity. An `include` filter limits Event ID 3 logging to matching events, so adapt and test the list against your monitoring needs before applying it.

```powershell
Get-SysmonNetworkConnect | Out-GridView
Get-SysmonNetworkConnect |
    Select-Object Image, DestinationPort -Unique
Get-SysmonNetworkConnect |
    Select-Object Image, DestinationPort -Unique |
    ConvertTo-SysmonRule
```

> [!Note]
> Review generated rules before using them. If a value contains multiple items (such as application versions or users), format the values as a semicolon-separated list and use the appropriate Sysmon condition, such as `contains all`, where applicable.


### Tracking driver loads
[Tracking When Drivers Are Loaded (Video 9)](https://www.youtube.com/watch?v=Fs7x7PywdzU&list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7&index=9)
```powershell
Get-SysmonDriverLoadEvent | Select-Object -First 1
```

### Process access

```powershell
$processAccess = Get-SysmonProcessAccess |
    Select-Object SourceImage, TargetImage, GrantedAccess, SourceUser, TargetUser -Unique

$processAccess | Out-GridView
$processAccess | ConvertTo-SysmonRule
```

`GrantedAccess` is an access mask, not a verdict. Interpret it alongside the source and target process, user, and surrounding events.

```powershell
# Example: decode an access mask
Get-SysmonAccessMask -AccessMask 0x1fffff
# Example output includes rights such as:
PROCESS_QUERY_LIMITED_INFORMATION
PROCESS_SET_LIMITED_INFORMATION
PROCESS_DUP_HANDLE
PROCESS_TERMINATE
PROCESS_CREATE_PROCESS
PROCESS_SET_INFORMATION
PROCESS_QUERY_INFORMATION
PROCESS_VM_OPERATION
PROCESS_CREATE_THREAD
SYNCHRONIZE
PROCESS_SUSPEND_RESUME
PROCESS_VM_WRITE
PROCESS_VM_READ
PROCESS_SET_QUOTA

Get-SysmonAccessMask -AccessMask 0x0800
# Example output:
PROCESS_SUSPEND_RESUME

Get-SysmonAccessMask -AccessRight PROCESS_QUERY_INFORMATION, PROCESS_SUSPEND_RESUME
# Example output:
0xc00
```


### Raw-access reads

```PowerShell
Get-SysmonRawAccessRead | Out-GridView
Get-SysmonRawAccessRead | Select-Object -First 1
Get-SysmonRawAccessRead |
    Select-Object Image, Device, User -Unique |
    ConvertTo-SysmonRule
```

The following is a minimal Event ID 9 include-filter example. Replace the example image with a narrowly scoped, reviewed value; an include rule changes which raw-access events are logged.

```xml
<Sysmon schemaversion="4.91">
  <EventFiltering>
    <RuleGroup name="Example raw-access read" groupRelation="or">
      <RawAccessRead onmatch="include">
        <Image condition="end with">\example.exe</Image>
      </RawAccessRead>
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```


### Named pipes

The following names are commonly associated with Cobalt Strike defaults, but pipe names alone are not proof of Cobalt Strike or malicious activity. See the [Sysmon Modular pipe-event example](https://github.com/olafhartong/sysmon-modular/blob/master/17_18_pipe_event/include_cobaltstrike.xml).

```xml
<Sysmon schemaversion="4.91">
  <EventFiltering>
    <RuleGroup name="Common Cobalt Strike pipe names" groupRelation="or">
      <PipeEvent onmatch="include">
        <Rule groupRelation="and">
          <PipeName condition="begin with">\msse-</PipeName>
          <PipeName condition="end with">-server</PipeName>
        </Rule>
        <PipeName condition="begin with">\msagent_</PipeName>
        <PipeName condition="begin with">\postex_</PipeName>
        <PipeName condition="begin with">\postex_ssh_</PipeName>
        <PipeName condition="begin with">\status_</PipeName>
      </PipeEvent>
    </RuleGroup>
  </EventFiltering>
</Sysmon>
```

### Tracking CreateRemoteThread

```powershell
Get-SysmonCreateRemoteThreadEvent |
    Select-Object SourceImage, TargetImage -Unique |
    ConvertTo-SysmonRule
```

### File Create Stream Hash Event ID 15

- **Video Title:** [Learning Sysmon - File Create Stream Hash Event (Video 15)](https://youtu.be/rrAGRdxf154?list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7)


NTFS supports named alternate data streams (ADS) in addition to a file's default `::$DATA` stream. ADS content is not reflected in the file size shown by Explorer. Attackers can use streams to store scripts or other payloads, while browsers and mail clients commonly create the legitimate `Zone.Identifier` stream, also known as Mark of the Web (MotW).

### Inspecting streams

```powershell
# List a file's streams
Get-Item -Path .\sysmon.zip -Stream *

# Read Mark of the Web metadata
Get-Content -Path .\sysmon.zip -Stream Zone.Identifier

# Create and remove a test stream
$script = Get-Content -Path .\wpe.ps1 -Raw
Set-Content -Path .\sysmon.zip -Stream PSscript -Value $script
Remove-Item -Path .\sysmon.zip -Stream PSscript
```

`Zone.Identifier` may contain download-origin details such as `ZoneId`, `HostUrl`, and `ReferrerUrl`. A `ZoneId` of `3` indicates the Internet zone and can help trace a downloaded file during phishing or initial-access investigations. The stream is useful evidence, but its absence does not prove a file was created locally.

### Interpreting Event 15

Creating a named stream can generate Sysmon Event ID 15 (`FileCreateStreamHash`). The event records the stream path and can include hashes for both the host file's unnamed stream and the named stream:

| Field | Meaning |
|---|---|
| `TargetFilename` | Host file and stream, such as `C:\path\sysmon.zip:PSscript` |
| `Image` | Process that created the stream |
| `Hashes` | Hashes for the host file's unnamed stream and the named stream, when present |
| `RuleName` | Name of the matching Sysmon rule, when configured |

EID 15 records stream creation, not execution. Code in an ADS still needs a process to invoke it; correlate stream events with EID 1 process-creation telemetry and investigate relevant command lines. Utilities such as `rundll32.exe`, `wmic.exe`, `cscript.exe`, `mshta.exe`, and `control.exe` may be involved, but their presence alone is not proof of malicious activity.

### Collection and hunting

`Zone.Identifier` streams are created frequently by browsers and mail clients. Collecting every EID 15 event provides fuller download forensics but increases log volume. If volume is a concern, exclude only verified browser or mail-client **full image paths**, and only for `:Zone.Identifier`; avoid filename-only exclusions, which could also suppress suspicious streams. Check the image paths in your own telemetry before applying exclusions.

In Kibana, a starting query for streams other than MotW is:

```text
event.code : 15 and winlog.channel : "Microsoft-Windows-Sysmon/Operational"
  and not winlog.event_data.TargetFilename : *Zone.Identifier
```

Field names and wildcard behavior depend on the ingest pipeline, so verify them in Discover. Treat this as a hunt query, not a complete detection: pair suspicious stream creation with process and command-line telemetry.

ADS is an NTFS feature. Copying files to filesystems such as FAT/exFAT or uploading them through services that do not preserve streams can remove ADS, including `Zone.Identifier`. Some archive and file-handling paths also fail to preserve MotW.

---

### Tracking and Blocking File Creation (Event IDs 11, 27, and 29)

Event ID 11 (`FileCreate`) records file creation when the configured filters match. Event ID 27 (`FileBlockExecutable`) can block the creation of PE-format executable files by selected processes; it requires a compatible Sysmon version (Sysmon 14 or later) and is a preventive control, not just a log event. Event ID 29 (`FileExecutableDetected`) reports detected PE file creation without blocking it.

The custom rule in [`TrackingBlockingFC_ID11-27.xml`](../windows/custom/TrackingBlockingFC_ID11-27.xml) combines Event ID 11 file-creation filters with Event ID 27 rules for selected applications:

Its Event ID 11 filters cover selected executable and script extensions, macro-enabled Office files, ClickOnce artifacts, and other named targets; they do not provide an inventory of every file created.

- `excel.exe`
- `winword.exe`
- `powerpnt.exe`
- `outlook.exe`
- `msaccess.exe`
- `mspub.exe`
- `onenote.exe`
- `pwsh.exe`
- `powershell.exe`
- `mshta.exe`

When one of those processes tries to create a PE-format executable, Sysmon can block the creation and emit EID 27. This targets a common attack chain where a macro or PowerShell command downloads a payload and writes it to disk. The `name=` attribute in the rule labels the event with the configured ATT&CK technique; it does not establish that the activity is malicious.

This is a behavioral control, not an AV signature check. It may block legitimate activity such as installer workflows as well as malicious writes, so test it before broad deployment. It does not block script files (`.ps1`, `.vbs`, `.js`) or other non-PE content; executable detection is based on file content, not the filename extension.

**MZ header note:** Windows PE files typically begin with the DOS `MZ` signature (`4D 5A`) and contain a DOS stub before the PE header. `MZ` refers to the signature, not a file extension; a `.mz` suffix does not identify the file format, and the signature alone does not prove the file is a valid PE executable.

For safer rollout, collect Event ID 29 first where supported, baseline expected activity, and add only validated, narrowly scoped exclusions before enabling active blocking.

```xml
<!-- filepath: sysmon/windows/custom/TrackingBlockingFC_ID11-27.xml -->
<FileBlockExecutable onmatch="include">
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">excel.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">winword.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">powerpnt.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">outlook.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">msaccess.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">mspub.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">onenote.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">pwsh.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">powershell.exe</Image>
    <Image name="technique_id=T1105,technique_name=Ingress Tool Transfer" condition="image">mshta.exe</Image>
</FileBlockExecutable>
```

Useful operational guidance:

- Validate the Event ID 29 and Event ID 27 behavior on a pilot set of hosts.
- Add exclusions only for validated good cases, preferably by full path or `TargetFilename`.
- Watch `Image`, `TargetFilename`, hash, and user fields in ELK/Splunk to distinguish malicious payload drops from legitimate installer activity.
- Keep a rollback path by reapplying the base Sysmon config with `Sysmon64.exe -c base.xml`.

This is a focused preventive control, not a complete defense. Processes outside the configured rules are not covered; pair it with other telemetry and controls.

```powershell
Get-SysmonFileBlockExecutable
```
---

### Tracking File Deletion and Blocking Shredding

[`FileDeleteDetected.xml`](../windows/custom/FileDeleteDetected.xml) combines deletion auditing, optional file archiving, and shredding prevention. These are separate controls with different storage and operational impacts.

#### File deletion tracking

**Tier 1: File archiving (Event ID 23, `FileDelete`)**

The `FileDelete` rules select files in specified staging and system locations for archiving when deleted. The paths include Outlook's `INetCache\Content.Outlook`, user `Downloads` and `AppData\Local\Temp`, selected `ProgramData` directories, and selected Windows directories. The filename filters cover executable and script formats, Office-related files, installers, archives, and shortcuts; the exact values vary by path rule. Several use `contains any`, which matches substrings rather than validating a file extension. These are configured paths, not a guarantee that every listed directory is broadly writable.

Archiving can aid investigation, but it consumes disk space and may retain sensitive files. Monitor archive growth and access, and scope the rules to your retention and privacy requirements.

**Tier 2: Audit without archiving (Event ID 26, `FileDeleteDetected`)**

The `FileDeleteDetected` rules log deletions from `Downloads` and `AppData\Local\Temp` for `.exe`, `.dll`, `.msi`, `.7z`, and `.zip` files without saving a copy. This is a lower-storage way to establish a deletion baseline, but it does not provide file recovery.

`<ArchiveDirectory>Archive</ArchiveDirectory>` specifies `Archive` as the archive directory name for Event ID 23. The actual location depends on the Sysmon installation and configuration; verify it on the deployed host and ensure it has adequate capacity and appropriate access controls.

#### File shredding prevention (Event ID 28, `FileBlockShredding`)

`FileBlockShredding` is configured in two rule groups:

* The `onmatch="exclude"` group exempts selected processes and users, including Windows services, Defender, and listed user applications. Exclusions reduce false positives but also create blind spots; validate the image paths and identities in your environment.
* The `onmatch="include"` group targets selected extensions, including executables, scripts, web-related files, Office macro formats, ticket artifacts, and delivery containers. Sysmon blocks matching shredding activity; this is not a general-purpose backup or protection against ordinary deletion.

The XML wraps both `FileBlockShredding` rules in `RuleGroup` elements and declares schema version `4.91`. Confirm that the installed Sysmon version supports the configured events and schema before deployment.

Test the configuration on a pilot group, review event volume and exclusions, and consider tabletop exercises to adapt it to your environment. Avoid archiving sensitive data unintentionally and document a rollback plan.

```powershell
Get-SysmonFileDeleteDetectedEvent
```

```powershell
Get-SysmonFileBlockShredding
```

### Tracking Clipboard Changes (Event ID 24)

#### Overview and detection value

Sysmon Event ID 24 (`ClipboardChange`) records changes to the Windows clipboard. It can support investigations into sensitive-data handling and commands copied between systems. It is associated with MITRE ATT&CK T1115 (Clipboard Data); use RDP-related mappings such as T1021.001 only when the surrounding activity supports them. Clipboard monitoring is not itself keylogging (T1056.001).

#### Security and operational considerations

Clipboard data can contain passwords, API keys, personal information, or other sensitive content. Restrict collection to systems and users where there is a documented need—such as jump servers, privileged workstations, or incident-response investigations—and notify users as required by policy and law. Assess privacy, compliance, event volume, and retention before enabling it.

Clipboard data may be archived separately from the Windows Event Log. Configure and secure the Sysmon archive directory, verify its actual location and access controls on the deployed host, and restrict access to authorized personnel. Do not assume clipboard monitoring captures images, files, or other binary formats. Confirm the feature and configuration syntax are supported by the installed Sysmon version. Clipboard capture requires the `<CaptureClipboard/>` setting in the `<Sysmon>` configuration.

#### Event ID 24 fields

| Field | Description |
| --- | --- |
| `RuleName` | Name of the rule that matched, if configured. |
| `UtcTime` | Event timestamp in UTC. |
| `ProcessGuid` / `ProcessId` | Identifiers for the process associated with the clipboard change. |
| `Image` | Image path of the associated process. |
| `Session` | Session identifier. |
| `ClientInfo` | Client details when available, such as for remote sessions. |
| `Hashes` | Hash information associated with archived clipboard data, when available. |
| `Archived` | Indicates whether clipboard data was archived. |

#### Archive retention

If clipboard data is archived, define and enforce a retention period that meets your security and compliance requirements. Test cleanup procedures carefully and ensure they target only the intended archive directory. For example, this removes files older than 30 days; run it under an appropriately restricted account and verify the path before use:

```powershell
$archivePath = "C:\SecureClipboardArchive"
$retentionDays = 30
$cutoff = (Get-Date).AddDays(-$retentionDays)

Get-ChildItem -Path $archivePath -File |
  Where-Object { $_.LastWriteTime -lt $cutoff } |
  Remove-Item -Force
```

```powershell
Get-SysmonClipboardChange
```
---

### Tracking DNS Queries (Event ID 22)

#### Overview and detection value

Sysmon Event ID 22 (`DNSEvent`) records DNS queries made by a process, including queries that fail or are answered from cache. It can support investigations into command-and-control activity, DNS tunneling, and network discovery, but a query alone does not establish malicious activity. This event is available on Windows 8.1 and later; it can generate substantial log volume.

* **Command and control (C2):** Hunt for suspicious domain-generation or fast-flux patterns and queries to newly registered or known malicious infrastructure.
* **Data exfiltration:** Look for unusually long query names or repeated queries to one domain that may indicate DNS tunneling.
* **Reconnaissance:** Review unexpected internal-domain enumeration or connectivity checks.
* **MITRE ATT&CK:** Relevant techniques may include T1071.004 (Application Layer Protocol: DNS), T1568 (Dynamic Resolution), T1048.003 (Exfiltration Over Unencrypted Non-C2 Protocol), and T1590 (Gather Victim Network Information); map only when supported by context.

#### Coverage and volume considerations

* Sysmon DNS events are not a complete record of all DNS traffic; custom resolvers and nonstandard resolution paths may not be represented. Correlate with DNS server or network telemetry where coverage matters.
* Normal hosts generate substantial DNS activity from applications, CDNs, and background services. Measure volume and retention impact before enabling broad collection.
* Enterprise DNS logging can provide complementary coverage and centralized visibility; do not rely on endpoint events alone.

#### Example XML configuration

The repository's [`DNSQueries_EID22.xml`](../windows/custom/DNSQueries_EID22.xml) uses exclusions to reduce volume. Excluded domains will not be available for investigation, and benign-looking domains can be abused. Review and tailor every exclusion to your environment before applying the configuration.

#### Event ID 22 fields

| Field Name | Description |
| --- | --- |
| `RuleName` | The Sysmon rule name that triggered the event, if configured. |
| `UtcTime` | The UTC timestamp when the event was created. |
| `ProcessGuid` | The unique process GUID of the application making the DNS query. |
| `ProcessId` | The process ID (PID) of the application making the DNS query. |
| `QueryName` | The DNS domain name that was requested. |
| `QueryStatus` | The result status code of the query (for example, `0` indicates success). |
| `QueryResults` | The resolved data or IP addresses returned from the query. |
| `Image` | The file path of the executable that initiated the DNS request. |
| `User` | The identity of the user who initiated the request, when available. |

#### Key Investigation Triggers

When analyzing Event ID 22 logs, prioritize hunting for these patterns:

* **DGA-like or newly registered domains:** Use reputation and registration-age data as context; numeric patterns or a recent registration are leads, not proof.
* **Unusual top-level domains:** Treat unfamiliar TLDs as context for triage, not a verdict.
* **Possible DNS tunneling:** Investigate unusually long `QueryName` values or high query volume to one domain; length thresholds are environment-dependent heuristics.
* **Unexpected process activity:** Review system utilities, Office applications, or scripting engines making DNS queries outside their normal role.

```powershell
Get-SysmonDNSQuery
```

---

### Tracking WMI Permanent Events

#### Overview and detection value

Windows Management Instrumentation (WMI) permanent event subscriptions can be abused for persistence. Their filters, consumers, and bindings are stored in the WMI repository and can survive reboots until removed; the consumer's payload may still be a file on disk, and execution context depends on the consumer and system configuration.

Sysmon records these artifacts through three event types:

* **Event ID 19**: `WmiEventFilter` – the WQL trigger that watches for a condition.
* **Event ID 20**: `WmiEventConsumer` – the action to execute when the trigger fires.
* **Event ID 21**: `WmiEventConsumerToFilter` – the binding that links the filter to the consumer.

These events are often low-volume and valuable to investigate. The three events may be created at different times, and missing events do not prove that a subscription is absent.

* **Repository-backed persistence:** Subscription objects are stored in the WMI repository; the action they invoke may use scripts or executables on disk.
* **Execution context:** Review the consumer type, command, and account context rather than assuming a fixed identity.
* **Persistence across reboots:** A permanent subscription remains until it is removed.
* **MITRE ATT&CK:** Relevant techniques include `T1546.003` (Event Triggered Execution: WMI Event Subscription) and `T1047` (Windows Management Instrumentation).

#### Components of a WMI Permanent Event

A permanent WMI event subscription has three core components:

* **Event Filter (Event ID 19)**: Defines the WQL query that triggers the subscription. Typical conditions include startup, user logon, file changes, or process creation.
* **Event Consumer (Event ID 20)**: Defines the action to take when the trigger occurs. In attacker activity this is commonly a command line, encoded PowerShell, or a script execution engine such as `wscript.exe` or `cscript.exe`.
* **Event Binder (Event ID 21)**: Registers the filter and consumer together so the consumer executes when the condition is met.

#### What to Investigate

When reviewing WMI event logs, investigate subscriptions that are not attributable to a verified enterprise management tool.

* **Correlate the components:** Event IDs 19, 20, and 21 describe filters, consumers, and their bindings. They may not appear together or in quick succession.
* **Review the Consumer Destination**: The `Destination` field in Event ID 20 is usually the most important artifact. Look for:
  - Encoded or obfuscated PowerShell commands.
  - `cmd.exe` or `powershell.exe` with suspicious arguments.
  - `cscript.exe` or `wscript.exe` launching scripts from temp or remote locations.
  - Download-and-execute behaviors or execution of credentials tools.
* **Inspect the Filter Query**: Event ID 19 shows the WQL condition. Adversary examples include triggers on system startup, user logon, or selected process creation. Time-based or periodic triggers are also common.
* **Check the User Context**: WMI subscriptions normally require elevated permissions. Investigate subscriptions created by non-admin accounts, service accounts, or recently compromised identities.
* **Correlate With Related Activity**: A WMI subscription created shortly after suspicious PowerShell, script execution, or privilege escalation is highly relevant.

#### Event Field Breakdown

Field availability varies by event type and Sysmon version; use the event XML and rendered fields available on the host for triage:

| Field Name | Description |
| --- | --- |
| `RuleName` | The Sysmon rule that matched the event, if configured. |
| `EventType` | The event category, such as `WmiEventFilter`, `WmiEventConsumer`, or `WmiEventConsumerToFilter`. |
| `UtcTime` | The UTC timestamp recorded for the event. |
| `Operation` | The operation recorded for the object, when present. |
| `User` | The user account that created or modified the WMI object. |
| `EventNamespace` | The WMI namespace in which the object was created. |
| `Name` | The name assigned to the filter or consumer. |
| `Query` | The WQL query associated with the filter. |
| `Type` | The type of consumer object, such as `CommandLineEventConsumer` or `ActiveScriptEventConsumer`. |
| `Destination` | The command or script that will execute when the event is triggered. |
| `Consumer` | The path to the consumer object in the WMI repository. |
| `Filter` | The path to the filter object in the WMI repository. |

#### Detection Gaps and Complementary Logging

Sysmon event telemetry should not be treated as a complete inventory of WMI activity. Correlate it with native WMI activity logging—especially Microsoft-Windows-WMI-Activity/Operational Event ID 5861—and validate coverage on the Windows and Sysmon versions in use.

* **Complementary log:** Microsoft-Windows-WMI-Activity/Operational, especially Event ID 5861.
* **Operational validation:** Confirm which namespace and subscription operations are represented in each source in your environment.

#### Example XML configuration

Given the investigative value of WMI events, consider collecting broadly and filtering in the SIEM rather than excluding broadly. Validate the example against the installed Sysmon schema and review any known management subscriptions.

```xml
<Sysmon schemaversion="4.91">
   <EventFiltering>
      <WmiEvent onmatch="exclude">
         <!-- No exclusions: collect WMI events for review. -->
      </WmiEvent>
   </EventFiltering>
</Sysmon>
```

If exclusions are necessary, prefer filtering on a validated consumer name or operation rather than broad suppression. For example:

```xml
<WmiEvent onmatch="exclude">
   <Rule groupRelation="and">
      <Operation condition="is">Created</Operation>
      <Name condition="contains">SCNotification</Name>
   </Rule>
</WmiEvent>
```

#### Practical Guidance

The most important analyst behavior is simple: investigate every WMI permanent event that appears outside of approved enterprise management tools. In most environments, a single WMI subscription is unusual enough to warrant immediate review, especially when the consumer points to PowerShell, script execution, or remote payloads.


```powershell

Get-SysmonWmiFilter
Get-WinEvent -LogName 'Microsoft-Windows-WMI-Activity/Operational' -MaxEvents 50

```

---

### Detecting Process Tampering

Sysmon **Event ID 25 (ProcessTampering)** records detected changes to a process image. It can help identify process hollowing and process herpaderping, but it is not a complete detector for process injection. Hollowing detection can vary by implementation and sample; herpaderping may also be detected when a file is altered after its image has been mapped. Validate observed behavior in your environment rather than treating the event as guaranteed coverage.

#### What to investigate

Process hollowing commonly involves creating a legitimate process in a suspended state, replacing or modifying its in-memory image, and then resuming it. Herpaderping creates a mismatch between the image mapped for execution and the file later seen on disk. In either case, correlate Event ID 25 with process creation (Event ID 1), process access (Event ID 10), image loads (Event ID 7), and network connections (Event ID 3), when available.

Review the event's **Type**, **Image**, **ProcessGuid**, and **ProcessId** fields. Treat an image-replacement event as a high-priority lead; an image-lock event can have legitimate causes and needs context. Examine the parent process, command line, signer, file path, and related activity. Unexpected tampering of critical processes or on production servers merits prompt triage.

#### Configuration and tuning

Enable ProcessTampering collection and initially record events without exclusions. Baseline activity, verify the cause of recurring benign events, and then add only narrowly scoped exclusions. Prefer exact paths and revalidate them after application updates; broad exclusions can hide malicious activity using a familiar process name.

```xml
<ProcessTampering onmatch="exclude">
  <!-- Example only: exclude a verified application by exact path. -->
  <Image condition="is">C:\Program Files\Example\app.exe</Image>
</ProcessTampering>
```

Place this rule within the `<EventFiltering>` section of the Sysmon configuration. Event ID 25 should complement, not replace, other process and endpoint telemetry.

---


### Tracking Registry Actions

Sysmon registry events provide visibility into persistence, privilege escalation, defense evasion, and credential access:

- **Event ID 12 (Object Added/Deleted):** Records registry key creation or deletion.
- **Event ID 13 (Value Set):** Records registry value modifications.
- **Event ID 14 (Object Renamed):** Records registry key or value renames, but should not be treated as reliable coverage.

#### Volume and coverage considerations

Registry activity is exceptionally noisy: normal Windows operations can generate thousands of events per minute. Use targeted includes for a small set of high-value paths; collecting all registry activity can increase host and SIEM load while burying useful signals. Start with roughly 20–30 paths, test in a lab, and baseline event volume before production deployment. Avoid whole-hive monitoring and broad wildcards.

Event ID 14 may be unreliable on some Sysmon versions. In addition, operations such as PowerShell `Rename-Item` can be implemented as creating a new object and deleting the old one, producing Events 12 and 13 rather than Event 14. Correlate all three event types when investigating changes. Sysmon telemetry is not a complete registry snapshot: do not assume it captures full value payloads, especially large or binary data; use appropriate forensic collection when the contents matter.

#### Configuration

Registry paths for `HKCU` commonly appear beneath `HKU\<SID>\...`, with a different SID for each user. Use operators such as `contains` or `end with` rather than exact matches for user-specific paths. Sysmon appends a value name to its key path with a backslash, so `contains` is often useful for monitoring values beneath a target key.

Use strict includes for known high-value locations. The repository's [registry event filter](../windows/custom/RegistryGhanges.xml) covers common autorun and startup locations, service configuration, Winlogon, Defender and UAC settings, WDigest, and accessibility debugger paths. It declares schema version `4.50`; confirm compatibility with the installed Sysmon version before applying it.

#### What to investigate

- **Accessibility debugger keys:** Creation or modification of `sethc.exe` or `utilman.exe` debugger settings can enable a SYSTEM-level shell at the logon screen and warrants urgent investigation.
- **WDigest:** A change to `UseLogonCredential`, particularly a value of `1`, can weaken credential protections and facilitate credential theft.
- **Defender and UAC:** Investigate changes to Defender exclusions, `DisableAntiSpyware`, or `EnableLUA`, especially when initiated by an unexpected process.
- **Persistence:** Review Run-key, startup, service `ImagePath` or `ServiceDll`, and Winlogon changes. Script engines, command shells, or unverified executables running from Temp or AppData are suspicious initiating processes.

Correlate registry events with process creation and access telemetry, and verify the actor, process command line, target path, user SID, and surrounding activity. Validate path coverage and event volume after configuration changes.

---

## Wrap-up

PSGumshoe makes Sysmon events easier to query, filter, and review, but the quality of an investigation still depends on what Sysmon collected and how the events are interpreted. Start with a clear question, check that the relevant event types are enabled, and correlate suspicious activity across process, file, network, registry, and other available telemetry.

Treat the commands and XML examples in this guide as starting points. Confirm compatibility with the installed Sysmon and PSGumshoe versions, test configuration changes before deployment, and balance visibility against event volume, storage, privacy, and operational impact. Use findings as investigative leads, validate them against host context and approved baselines, and send relevant telemetry to centralized monitoring when broader correlation or retention is needed.
