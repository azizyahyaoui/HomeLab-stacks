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


## Summary

PSGumshoe provides PowerShell commands for inspecting Sysmon telemetry without manually parsing Windows Event Log records. Start with configuration changes and process creation, then query other event types as needed:

