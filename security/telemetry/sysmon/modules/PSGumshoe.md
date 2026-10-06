# PowerShell module PSGumshoe

[TrustedSec](https://www.youtube.com/watch?v=2JHjRR2Wt4g&list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7&index=12)
[SysmonCommunityGuide](https://github.com/trustedsec/SysmonCommunityGuide/tree/master)

PSGumshoe is a PowerShell module designed to simplify interaction with Sysmon telemetry. It helps security analysts and homelab operators query Sysmon events quickly, especially process creation and configuration change events, without manually parsing Windows Event Log data.

It is particularly useful for:

- Reviewing Sysmon configuration updates
- Investigating process execution
- Correlating parent-child process relationships
- Examining command lines and user context
- Quickly validating whether a Sysmon change was expected or suspicious

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

`RemoteSigned` is a reasonable lab setting because it allows locally created scripts while requiring downloaded scripts to be signed, unless they have been unblocked. Avoid changing the `LocalMachine` policy just to get PSGumshoe working.

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
- `Configuration` can show the configuration source or command used to apply the change.
- `ConfigurationFileHash`, when present, can help identify the configuration content.
- This is useful when validating whether a config change was intentional or suspicious.

A real-world example may show command-line-driven changes, such as:

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

This is useful for understanding changing Sysmon behavior:

- `-l malicious.exe` configures image load logging for a suspicious process
- `-n malicious.exe` modifies network logging filters
- `-k lsass.exe` can be used to monitor credential access related to LSASS
- `-h *` enables hashing for all files or processes, which can increase telemetry and impact performance

The command lines are examples of values that may appear in the event; their exact meaning can vary by Sysmon version. Verify any switches against the documentation for the installed version. A configuration change is not necessarily malicious, but should be checked against the host's intended configuration.

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

A common investigative pattern is to review process creation events and then sort by user, parent process, or command line.

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

$processCreation | Export-Csv .\processdata.csv
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

## Key operational idea

PSGumshoe is not a replacement for Sysmon itself; it is a convenience layer that helps you query the data Sysmon has already collected.

Think of it like this:

- Sysmon records the host telemetry
- PSGumshoe helps present that telemetry in a cleaner, analyst-friendly way
- Wazuh, ELK, or a SIEM later use that data for detection, correlation, and alerting

---

## Practical workflow

A typical workflow may look like this:

```powershell
Get-SysmonConfigChange
Get-SysmonProcessCreateEvent | more
Get-SysmonProcessCreateEvent |
    Select-Object User, ParentImage, ParentCommandLine, Image, CommandLine |
    Out-GridView
```

This lets you:

1. confirm configuration changes,
2. inspect process execution,
3. understand the parent-child process relationship,
4. and identify suspicious behavior quickly.

---

## Additional event investigations

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
Get-SysmonCreateRemoteThreadEvent | select sourceimage,targetimage -Unique | ConvertTo-SysmonRule
```

### File Create Stream Hash Event ID 15

- **Video Title:** [Learning Sysmon - File Create Stream Hash Event (Video 15)](https://youtu.be/rrAGRdxf154?list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7)


NTFS supports named alternate data streams (ADS) in addition to a file's default `::$DATA` stream. ADS content is not reflected in the file size shown by Explorer. Attackers can use streams to hide scripts or other payloads, while browsers and mail clients commonly create the legitimate `Zone.Identifier` stream, also known as Mark of the Web (MotW).

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

Creating a named stream can generate Sysmon Event ID 15 (`FileCreateStreamHash`). The event records the stream path and a hash of the stream contents, not the host file:

| Field | Meaning |
|---|---|
| `TargetFilename` | Host file and stream, such as `C:\path\sysmon.zip:PSscript` |
| `Image` | Process that created the stream |
| `Hash` | Hash of the stream contents |
| `RuleName` | Name of the matching Sysmon rule, when configured |

EID 15 records stream creation, not execution. Code in an ADS still needs a process to invoke it; correlate stream events with EID 1 process-creation telemetry and investigate relevant command lines. LOLBins such as `rundll32.exe`, `wmic.exe`, `cscript.exe`, `mshta.exe`, and `control.exe` may be involved, but their presence alone is not proof of malicious activity.

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

### Tracking and Blocking File Creation ID 11/27

Sysmon Event ID 27 (`FileBlockExecutable`) is the first Sysmon event type that does more than log: it actively blocks the creation of executable files on disk. This requires Sysmon 14 or later and is mapped to ATT&CK T1105 (Ingress Tool Transfer).

The custom rule in `sysmon/windows/custom/TrackingBlockingFC_ID11-27.xml` watches for executable writes by a short list of high-risk processes:

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

When one of those processes tries to write a PE file such as `.exe`, `.dll`, or `.sys`, Sysmon blocks the write and emits EID 27. This targets a common attack chain where a macro or PowerShell one-liner downloads a payload and drops it to disk. The `name=` attribute in the rule tags the event with the corresponding ATT&CK technique so the `RuleName` field shows the mapped behavior.

This is a behavioral control, not an AV signature check. It blocks any executable created by those processes, not just malicious files. That means it can also block legitimate admin activity such as `Invoke-WebRequest -OutFile` or packaged installer workflows, so it should be tested before broad deployment. It does not block script files (`.ps1`, `.vbs`, `.js`) or non-PE content; the detection is based on the PE header, not the extension.

**MZ header note:** Windows PE files typically begin with the DOS `MZ` signature (`4D 5A`) and contain a DOS stub before the PE header. The signature alone does not prove a file is a valid PE executable, and `.mz` is not the header itself; executable detection is based on file content rather than its filename extension.

For safer rollout, use audit mode first. Sysmon 15+ adds Event ID 29 (`FileExecutableDetected`), which logs executable creation without blocking. Review the hits for a week, then add exclusions for known-good paths or file names before enabling active blocking.

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

- Audit first with EID 29, then enable EID 27 on a pilot set of hosts.
- Add exclusions only for validated good cases, preferably by full path or `TargetFilename`.
- Watch `Image`, `TargetFilename`, hash, and user fields in ELK/Splunk to distinguish malicious payload drops from legitimate installer activity.
- Keep a rollback path by reapplying the base Sysmon config with `Sysmon64.exe -c base.xml`.

This is a focused preventive control. It does not cover browsers, `cmd.exe`, `curl`, `certutil`, `bitsadmin`, or Python, so it should be paired with other telemetry and controls rather than treated as a complete defense.

```powershell
Get-SysmonFileBlockExecutable
```
---

### Tracking File Deletion and Blocking Shredding

The attached `FileDeleteDetected.xml` combines deletion auditing, optional file archiving, and shredding prevention. These are separate controls with different storage and operational impacts.

#### File deletion tracking

**Tier 1: File archiving (Event ID 23, `FileDelete`)**

The `FileDelete` rules select files in specified staging and system locations for archiving when deleted. The paths include Outlook's `INetCache\Content.Outlook`, user `Downloads` and `AppData\Local\Temp`, selected `ProgramData` directories, and selected Windows directories. The extension filters cover executable and script formats, Office-related files, installers, archives, and shortcuts; the exact extensions vary by path rule. These are configured paths, not a guarantee that every listed directory is world-writable.

Archiving can aid investigation, but it consumes disk space and may retain sensitive files. Monitor archive growth and access, and scope the rules to your retention and privacy requirements.

**Tier 2: Audit without archiving (Event ID 26, `FileDeleteDetected`)**

The `FileDeleteDetected` rules log deletions from `Downloads` and `AppData\Local\Temp` for `.exe`, `.dll`, `.msi`, `.7z`, and `.zip` files without saving a copy. This is a lower-storage way to establish a deletion baseline, but it does not provide file recovery.

`<ArchiveDirectory>Archive</ArchiveDirectory>` specifies `Archive` as the archive directory name for Event ID 23. The actual location depends on the Sysmon installation and configuration; verify it on the deployed host and ensure it has adequate capacity and appropriate access controls.

#### File shredding prevention (Event ID 28, `FileBlockShredding`)

`FileBlockShredding` is configured in two rule groups:

* The `onmatch="exclude"` group exempts selected processes and users, including Windows services, Defender, and listed user applications. Exclusions reduce false positives but also create blind spots; validate the image paths and identities in your environment.
* The `onmatch="include"` group targets selected extensions, including executables, scripts, web-related files, Office macro formats, ticket artifacts, and delivery containers. Sysmon blocks matching shredding activity; this is not a general-purpose backup or protection against ordinary deletion.

The attached XML already wraps both `FileBlockShredding` rules in `RuleGroup` elements and declares schema version `4.91`. Its `FileDeleteDetected` extension list uses `.dll` (with the dot), so the alleged `dll` typo does not apply to this file. Confirm that the installed Sysmon version supports the configured events and schema before deployment.

Test the configuration on a pilot group, review event volume and exclusions, and consider tabletop exercises to adapt it to your environment. Avoid archiving sensitive data unintentionally and document a rollback plan.

```powershell
Get-SysmonFileDeleteDetectedEvent
```

```powershell
Get-SysmonFileBlockShredding
```

### Tracking Clipboard Change EID 24

#### Overview and detection value

Sysmon Event ID 24 (`ClipboardChange`) records changes to text in the Windows clipboard. It can help investigate credential theft, sensitive-data staging, and commands copied between systems. It is most directly associated with MITRE ATT&CK T1115 (Clipboard Data); use RDP-related mappings such as T1021.001 only when the surrounding activity supports them. Clipboard monitoring is not itself keylogging (T1056.001).

#### Security and operational considerations

Clipboard data can contain passwords, API keys, personal information, or other sensitive text. Restrict collection to systems and users where there is a documented need—such as jump servers, privileged workstations, or incident-response investigations—and notify users as required by policy and law. Assess privacy, compliance, event volume, and retention before enabling it.

Clipboard payloads may be archived separately from the Windows Event Log. Configure and secure the Sysmon archive directory, verify its actual location and access controls on the deployed host, and restrict access to authorized personnel. Sysmon clipboard monitoring is for text; do not assume it captures images, files, or other binary clipboard formats. Confirm the feature and configuration syntax are supported by the installed Sysmon version. Clipboard capture requires the `<CaptureClipboard/>` setting in the `<Sysmon>` configuration.

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

### Tracking DNS Queries EID 22

#### Overview & Detection Value

Sysmon **Event ID 22** logs DNS queries executed through the Windows `DnsQuery_*` API calls within `dnsapi.dll`. While this capability generates moderate to high log volumes, it provides visibility into command and control (C2) communications, DNS tunneling, and network discovery activities.

* **Command and Control (C2)**: Detects domains generated by Domain Generation Algorithms (DGAs), fast-flux DNS activity, and connections to newly registered or known malicious infrastructure.
* **Data Exfiltration**: Identifies DNS tunneling techniques where attackers encode data within unusually long DNS queries or generate a high volume of queries to a single domain.
* **Reconnaissance**: Exposes attacker discovery phases, such as internal domain enumeration or internet connectivity checks.
* **MITRE ATT&CK Mapping**: Correlates with T1071.004 (C2 over DNS), T1568 (Dynamic Resolution), T1048.003 (Exfiltration over DNS), and T1590 (Gather Victim Network Information).

#### Critical Technical Limitations

* **API Dependency & Blind Spots**: Sysmon captures queries routed through standard Windows APIs via Event Tracing for Windows (ETW) on Windows 8.1 and above. It cannot track DNS traffic if an attacker uses custom DNS resolution stacks, direct raw sockets, or alternative DNS libraries.
* **Performance & Scale Constraints**: Standard hosts constantly generate substantial DNS traffic from CDNs, telemetry, and background services. Logging all queries on client machines can consume significant resources; it is better suited to server environments or controlled scenarios.
* **Enterprise Strategy**: For large organizations, dedicated enterprise DNS solutions are preferable to relying solely on Sysmon for client-side DNS logging, balancing visibility against storage and performance constraints.

#### Production XML Configuration

Because of the volume of benign DNS traffic, an **exclusion-based strategy** can reduce noise. This schema 4.91 configuration logs DNS queries except for the listed domains; review and tailor exclusions to your environment, since excluded queries will not be available for investigation. For a ready-to-use example, refer to the Sysmon XML file in `/opt/stacks/security/telemetry/sysmon/windows/custom/` and adjust the DNS query exclusions to fit your environment.

#### Event ID 22 Fields Breakdown

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

* **DGA-Like and New Domains**: Look for algorithmically generated domains featuring numeric patterns or long strings of consonants, as well as domains registered within the last 30 to 90 days.
* **Unusual Top-Level Domains (TLDs)**: Flag queries to uncommon TLDs frequently abused by attackers, such as `.tk`, `.pw`, or `.cc`.
* **DNS Tunneling Indicators**: Investigate `QueryName` values exceeding 50 to 100 characters, which may indicate data encoded as a subdomain, or a high volume of queries directed at a single domain.
* **Suspicious Processes**: Monitor standard system utilities, Office applications, or scripting engines making unexpected DNS queries.

```powershell
Get-SysmonDNSQuery
```

---

### Tracking WMI Permanent Events

#### Overview & Detection Value

Windows Management Instrumentation (WMI) permanent event subscriptions are a classic attacker persistence technique that remains highly valuable for defenders. When an attacker creates a permanent subscription, the trigger, consumer, and binding are stored in the WMI repository (CIM database), which makes the persistence fileless, privileged, and resilient across reboots.

Sysmon records these artifacts through three event types:

* **Event ID 19**: `WmiEventFilter` – the WQL trigger that watches for a condition.
* **Event ID 20**: `WmiEventConsumer` – the action to execute when the trigger fires.
* **Event ID 21**: `WmiEventConsumerToFilter` – the binding that links the filter to the consumer.

These are low-volume, high-fidelity events that should be treated as high priority whenever they appear. A complete subscription chain usually consists of all three event IDs occurring in quick succession.

* **Fileless Persistence**: Stored in the WMI repository instead of a file on disk.
* **SYSTEM-Level Execution**: Permanent subscriptions run as `SYSTEM` by default.
* **Persistence Across Reboots**: Survives system restarts unless explicitly removed.
* **High Detection Value**: Legitimate enterprise use exists, but WMI subscriptions are uncommon in most Windows environments.
* **MITRE ATT&CK Mapping**: `T1546.003` (Event Triggered Execution: WMI Event Subscription) and `T1047` (Windows Management Instrumentation).

#### Components of a WMI Permanent Event

A permanent WMI event subscription has three core components:

* **Event Filter (Event ID 19)**: Defines the WQL query that triggers the subscription. Typical conditions include startup, user logon, file changes, or process creation.
* **Event Consumer (Event ID 20)**: Defines the action to take when the trigger occurs. In attacker activity this is commonly a command line, encoded PowerShell, or a script execution engine such as `wscript.exe` or `cscript.exe`.
* **Event Binder (Event ID 21)**: Registers the filter and consumer together so the consumer executes when the condition is met.

#### What to Investigate

When reviewing WMI event logs, treat every occurrence as suspicious unless it is tied to a verified enterprise management tool.

* **Look for the Full Chain**: Event IDs 19, 20, and 21 often appear close together in time. A complete attack chain will contain all three.
* **Review the Consumer Destination**: The `Destination` field in Event ID 20 is usually the most important artifact. Look for:
  - Encoded or obfuscated PowerShell commands.
  - `cmd.exe` or `powershell.exe` with suspicious arguments.
  - `cscript.exe` or `wscript.exe` launching scripts from temp or remote locations.
  - Download-and-execute behaviors or execution of credentials tools.
* **Inspect the Filter Query**: Event ID 19 shows the WQL condition. Adversary examples include triggers on system startup, user logon, or selected process creation. Time-based or periodic triggers are also common.
* **Check the User Context**: WMI subscriptions normally require elevated permissions. Investigate subscriptions created by non-admin accounts, service accounts, or recently compromised identities.
* **Correlate With Related Activity**: A WMI subscription created shortly after suspicious PowerShell, script execution, or privilege escalation is highly relevant.

#### Event Field Breakdown

The following fields are typically present and useful for triage:

| Field Name | Description |
| --- | --- |
| `RuleName` | The Sysmon rule that matched the event, if configured. |
| `EventType` | The event category, such as `WmiFilterEvent`, `WmiConsumerEvent`, or `WmiBindingEvent`. |
| `UtcTime` | The UTC timestamp when the subscription object was created, modified, or removed. |
| `Operation` | Whether the object was created, changed, or deleted. |
| `User` | The user account that created or modified the WMI object. |
| `EventNamespace` | The WMI namespace in which the object was created. |
| `Name` | The name assigned to the filter or consumer. |
| `Query` | The WQL query associated with the filter. |
| `Type` | The type of consumer object, such as `CommandLineEventConsumer` or `ActiveScriptEventConsumer`. |
| `Destination` | The command or script that will execute when the event is triggered. |
| `Consumer` | The path to the consumer object in the WMI repository. |
| `Filter` | The path to the filter object in the WMI repository. |

#### Detection Gaps and Complementary Logging

Sysmon captures WMI event subscriptions created under the standard `Root\Subscription` namespace, but it does not log objects created in the `Root` namespace. Attackers aware of this limitation may bypass Sysmon by placing their subscriptions in `Root` instead. Because of this, native Windows WMI activity logging is essential for complete coverage.

* **Sysmon Gap**: Excludes `Root` namespace items.
* **Recommended Complementary Log**: Microsoft-Windows-WMI-Activity/Operational, especially **Event ID 5861**.
* **Operational Visibility**: Event ID 5861 includes query and consumer information and can expose subscriptions that Sysmon misses.

#### Production XML Configuration

Given the low volume and high value of WMI event data, the recommended approach is usually to log everything and filter in the SIEM instead of excluding broadly. This schema 4.91 example enables WMI logging for all event subscriptions and is suitable for most environments where enterprise tooling has been reviewed.

```xml
<Sysmon schemaversion="4.22">
   <HashAlgorithms>*</HashAlgorithms>
   <CheckRevocation/>
   <EventFiltering>
      <RuleGroup name="" groupRelation="or">
         <WmiEvent onmatch="exclude">
            <!-- Log all WMI events by default; volume is typically very low -->
            <!-- Exclude only known-good management subscriptions after validation -->
         </WmiEvent>
      </RuleGroup>
   </EventFiltering>
</Sysmon>
```

If exclusions are necessary, prefer filtering on a validated consumer name or operation rather than broad suppression. For example:

```xml
<WmiEvent onmatch="exclude">
   <Operation condition="is">Created</Operation>
   <Consumer condition="contains">SCNotification</Consumer>
</WmiEvent>
```

#### Practical Guidance

The most important analyst behavior is simple: investigate every WMI permanent event that appears outside of approved enterprise management tools. In most environments, a single WMI subscription is unusual enough to warrant immediate review, especially when the consumer points to PowerShell, script execution, or remote payloads.


```powershell

Get-SysmonWmiFilter
Get-WinEvent -LogName 'Microsoft-Windows-WMI-Activity/Operational' -MaxEvents 50

```

---

###  Detecting Process Tampering

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


---

