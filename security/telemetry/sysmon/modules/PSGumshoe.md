# PowerShell module PSGumshoe

[TrustedSec](https://www.youtube.com/watch?v=2JHjRR2Wt4g&list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7&index=12)

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

Then install the module:

```powershell
Install-Module -Name PSGumshoe
```

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

### 1 Review Sysmon configuration changes

`Get-SysmonConfigChange` retrieves Sysmon Event ID 16, which records configuration updates and other configuration-related changes.

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
ConfigurationFileHash : SHA256
```

What this means:

- `EventId 16` means Sysmon configuration changed.
- `Configuration` shows the config source or command used to apply the change.
- `ConfigurationFileHash` helps confirm the exact file or content being loaded.
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

These are not necessarily malicious by themselves, but they are important security events that should be reviewed in context.

---

### 2 Review process creation events

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

## Summary

PSGumshoe is a useful PowerShell companion for Sysmon investigations. It makes it easier to query configuration events and process creation telemetry in a faster, more readable format.

The most important commands to remember are:

```powershell
Get-SysmonConfigChange
Get-SysmonProcessCreateEvent
```
---

## Network Connection

```xml
<Sysmon schemaversion="4.83">
    <HashAlgorithms>sha256</HashAlgorithms>
    <DnsLookup>true</DnsLookup>
    <CheckRevocation/>
    <EventFiltering>
        <RuleGroup name=""></RuleGroup>
            <NetworkConnect onmatch="include">
                <!-- Web, DNS and Time Protocols -->
                <DestinationPort name="C2 Channels" condition="contains any">80;443; 53; 123</DestinationPort>
                <!-- LDAP, LDAPS,GS, GC over SSL and Kerberos -->
                <DestinationPort name="Directory Ports" condition="contains any">389; 636; 3268; 3269; 88</DestinationPort>
                <!-- RDP,WinRM, WinRMS, FTP, SSH, Telnet, SMB and RPC -->
                <DestinationPort name="Management Ports" condition="contains any">3389; 5985; 5986; 21; 22; 23; 445; 135; 138; 139</DestinationPort>
            </NetworkConnect>
        </RuleGroup>
    </EventFiltering>
</Sysmon>
```

```powershell
PS C:\Users\administrator> Get-SysmonNetworkConnect | Out-GridView
PS C: \Users\administrator> Get-SysmonNetworkConnect | select image, destinationport -unique
PS C: \Users\administrator> Get-SysmonNetworkConnect | select image, destinationport -unique | ConvertTo-SysmonRule
```

> [!Note]
> When you utilize `ConvertTo-SysmonRule` change any app-version or user to table using ';' and 'contain all'.



##  Tracking When Drivers Are Loaded 
[Tracking When Drivers Are Loaded (Video 9)](https://www.youtube.com/watch?v=Fs7x7PywdzU&list=PLk-dPXV5k8SG26OTeiiF3EIEoK4ignai7&index=9)
```powershell
PS C: Windows system32> Get-SysmonDriverLoadEvent | select -First 1

```
