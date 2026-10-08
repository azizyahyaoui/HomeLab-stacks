# Windows Logging for SOC

[Room link](https://tryhackme.com/room/windowsloggingforsoc)


## Task 1: Introduction
SOC analysts spend most of their time triaging alerts and hunting threats - using the logs in SIEM. To tell good from bad, analysts have to know the logs well: how they look, how to interpret them, and what malicious action they indicate. This room begins your long journey into Windows logging - a key skill for any SOC analyst or DFIR professional.

Learning Objectives
- Understand how to find and interpret important Windows event logs
- Learn invaluable for monitoring log sources like Sysmon and PowerShell
- Prepare for using the mentioned logs in SOC-SIM and the following rooms
- Practice your log analysis skills on multiple event log datasets

Recommended Rooms
- Remind yourself the Logs Fundamentals
- Learn and practice Sysmon
- Learn how to query Event Logs
- Know Core Windows Processes

---
---

## Task 2: What Is Logged

**Logging Overview**

Whenever you start a program, create a file, or just log in to your laptop, the event is processed by your OS. Then, the OS can log the event, meaning it'll append a line to some journal, stating the time, action details, and the user behind the action. Every recorded event is called a log, and proper logging ensures that all user and system activity is recorded, thus helping SOC with the following activities:

- **Incident Response**: Logs can show when and how the attack occurred.
- **Threat Hunting**: Logs allow you to search for signs of malicious activity.
- **Alerting and Triage**: Logs are a building block of any alert or detection rule.

**Anatomy of a Log Entry**

Windows is an interesting OS as it has very powerful logging capabilities but requires a lot of knowledge to read and understand the logs. Your first challenge may be to just open the logs, as they are stored in a binary format inside the `C:\Windows\System32\winevt\Logs` folder:






            IMAGE FROM THE BOX




List of **EVTX** files on the left and the content of one of the files on the right that is unreadable since the logs are stored in a binary format

Every EVTX file corresponds to a specific log category. For example, Application Logs contain events logged by user-mode applications like the IIS web server or MS SQL database, and Security Logs capture events like logon attempts, process activity, and user management.

**Reading Event Logs**

We will use Event Viewer for this room, a built-in tool that allows you to view and manage event logs. To open Event Viewer, search for "Event Viewer" using Windows Search or press Win + R, type `eventvwr`, and press Enter. Once the tool is loaded, you may see all system logs parsed, grouped, and ready for analysis:

1. **Log Sources**: Every EVTX file corresponds to a single item on the left panel.
2. **Log List**: Each row you see is a single event that contains a few properties you can sort by:
    - **Keywords**: For some events, indicates if the action was successful or not.
    - **Date and Time**: The timestamp when the event occurred (system time, not UTC!).
    - **Event ID**: A unique number for the event name (e.g. a failed login is always *4625*).
3. **Log Details**: The actual content of the log, in a plaintext or XML format ("Details" tab).
4. **Filters Menu**: Use the "*Filter Current Log*" and "*Find*" buttons to filter the logs.
5. Event Viewer window with four highlighted panels - log sources, list of logs, log details, and filters menu.

![alt text](Screenshots/THMEVUI.png)

**What Is Logged**

There are [over 500](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/) event IDs just for the Security logs and many thousands of various event IDs in total! Still, not all events are logged by default and not all events are properly documented, so in this room we will explore the most helpful logs for daily SOC routines.

---

**Answer the questions below**

- Looking at the last screenshot, which event ID describes a successful login?(Answer format: LogSource / ID, e.g. Application / 8194)

---
---

## Task 3: Security Log: Authentication

**Overview**

As a SOC analyst, you can't know in advance which attack you will be handling tomorrow and which logs you will need to triage. However, out of all Windows logs enabled by default, the Security event log is the one that brings you the most value. Let's start our journey from the two most important Security logs: Successful Logon (*4624*) and Failed Logon (*4625*).

| Event ID | Purpose | Logging | Limitations |
| --- | --- | --- | --- |
| 4624<br>(Successful Logon) | Detect suspicious RDP/network logins and identify the attack starting point | Logged on the lab machine, the one you are trying to access | Noisy. You will see hundreds of logon events per minute on loaded servers |
| 4625<br>(Failed Logon) | Detect brute force, password spraying, or vulnerability scanning | Logged on the lab machine, the one you are trying to access | Inconsistent. The logs have lots of caveats that may trick you into the wrong understanding of the event |

**Structure of *4624***

A typical Windows server can generate tens of login events per minute, and every login event is often comprised of many different fields. Still, you can cover most L1/L2 cases just by checking a few core event fields in the image below. You can also read more about other fields and logon types in the [Event ID Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/event.aspx?eventid=4624).

![alt text](Screenshots/Structureof4624.png)

A list of key fields to review in the 4624 event:  Logon ID and logon type, username, source IP and hostname

**Usage of *4624*/*4625***

Even experienced IT admins often rely on security experts to distinguish bad from good events, so don't worry if the workbooks below seem complex at first. Take your time and treat this task as fundamental knowledge that you will use in practice for many upcoming rooms!

- **Detect RDP Brute Force:**
    1. Open Security logs and filter for 4625 event ID (Failed login attempts)
    2. Look for events with Logon Type 3 and 10 (Network and RDP logins)
        - For most *modern systems*, the logon type will be *3* (since *NLA* is enabled by default)
        - For *older or misconfigured systems*, the logon type will be *10* (since NLA is not used)
    3. Every event is now worth your attention, but the main red flags are:
        - **Many attempted** users like admin, helpdesk,  and cctv (Indicates password spraying)
        - **Many login failures** on a single account, usually Administrator (Indicates brute force)
        - **Workstation Name** does not match a corporate pattern (e.g. kali instead of THM-PC-06)
        - **Source IP** is not expected (e.g. your printer trying to connect to your Windows Server)

- **Analyse RDP Logons:**
    1. Open Security logs and filter for *4624* event ID (*Successful* logins)
    2. Look for events with Logon Type *10* (*RDP* logins)
        - If *NLA is enabled*, every RDP logon event is preceded by another 4624 with logon type *3*
        - To get a real Workstation Name, you need to check the preceding logon type 3 event
    3. Your red flags are either a preceding brute force or a suspicious source IP / hostname
    4. If you assume that the login was indeed malicious, find out what happened next:
        - Windows assigns a *Logon ID* to every successful login (e.g. *0x5D6AC*)
        - Logon ID is a *unique* session identifier. Save it for future analysis!

---

- Answer the questions below
Open the "Practice-Security.evtx" file on the VM's Desktop.
    - Which IP performed a brute force of the THM-PC?

    - Which user has been breached as a result of the attack?


    - What was the Logon ID of the malicious RDP login?
        Note: The login you are looking for has a Logon Type 10.

---
---

## Task 4: Security Log: User Management

**Overview**

- Hey Michael, is "svc_sysrestore" your account? Never saw it in the user list before
- No, but it's likely some Windows stuff; better not touch it to avoid any problems in future

Above is a typical IT department discussion that often leads to ransomware attacks. However, with some knowledge of authentication and user management events, it is trivial to find out the whole history of any user account. Below is a breakdown of the common event IDs you can use:

| Event ID | Description | Malicious Usage |
| --- | --- | --- |
| 4720 / 4722 / 4738 | A user account was created / enabled / changed | Attackers might create a backdoor account or even enable an old one to avoid detection |
| 4725 / 4726 | A user account was disabled / deleted | In some advanced cases, threat actors may disable privileged SOC accounts to slow down their actions |
| 4723 / 4724 | A user changed their password / User's password was reset | Given enough permissions, threat actors might reset the password and then access the required user |
| 4732 / 4733 | A user was added to / removed from a security group | Attackers often add their backdoor accounts to privileged groups like "Administrators" |

**Structure of User Management Events**

All user management events have a similar structure and can be split into three parts: *who did the action (Subject)*, *who was the target (Object)*, and *which exact changes were made (Details)*:

- **Subject**: The account doing the action. Note the Logon ID field - you can use it to correlate this event with the preceding 4624 login event!
- **Object**: This can be named differently depending on an event ID (e.g. New Account or Member), but it always means the same - the target of the action.
- **Details**: A target group for 4732 and 4733 events, or new user's attributes like full name or password expiration settings for the 4720 event.


![alt text](Screenshots/structureof4720-4732-4724logs.png)


**Usage of User Management Events**

Many real breaches involved at least some user manipulation events, for example, [these ransomware actors](https://thedfirreport.com/2023/04/03/malicious-iso-file-leads-to-domain-wide-ransomware/#:~:text=bat%0A2.bat-,The%20script%20pass.bat,-proceeded%20to%20reset) reset all user accounts to a single password to slow down the recovery, and these attackers created a new admin account for persistence. Refer to the workbooks below to learn how to hunt for similar attacks:

- **Hunt for Backdoored Users**:
    1. Open Security logs and filter for 4720 / 4732 event IDs
    2. Manually review every event; your red flags are:
        - No one from your IT department can confirm the action
        - Changes were made during non-working hours or on weekends
        - The subject user's name is unknown or unexpected to you
        (e.g. "adm.old.2008" creating new Windows users)
        - The target user's name does not follow a usual naming pattern
        (e.g. "backup" instead of "thm_svc_backup")
    3. If you confirmed that the action was malicious, find out the login details:
        - Copy the Logon ID field from your 4720 / 4732 event
        - Find the corresponding login event with the same Logon ID
        - Refer to the workbooks from the previous task for further analysis

---

- Answer the questions below
    Continue with the "Practice-Security.evtx" file on the VM's Desktop.

    - Which user was created by the attacker soon after the RDP login?

    - Which two privileged groups was the backdoor user added to?
    (Answer in alphabetical order, e.g. "Administrators, Power Users")

    - Does the Logon ID field match what you saw in the previous task (Yea/Nay)?


---
---

## Task 5:


---
---

## Task 6:










---
---

## Task 7:










---
---

## Task 8:










---
---


## NOTES:


> **NLA**
> Network Level Authentication. An RDP pre-authentication layer using CredSSP that validates credentials before the destination allocates session resources. Generates Event 1149 on the destination when the network handshake completes. NLA is the network leg, not credential validation; pair with Security 4624 Type 10 to confirm a session was actually created.

---

> **CredSSP**
> Is a Windows authentication protocol that delegates a user's credentials securely from a client computer to a remote target server over an encrypted TLS channel.

## How CredSSP Works

* Core Function: Facilitates single-sign-on (SSO) and remote authentication, often used in and PowerShell remote management. 
* Underlying Layers: Combines a Transport Layer Security (TLS) encrypted transport channel with the Simple and Protected Negotiate protocol running either Kerberos or NTLM.
* Security Risk: Delegating credentials exposes systems to risks if the remote server is compromised, allowing malicious actors to reuse those credentials across the network.

## Common Errors and Fixes (Encryption Oracle Remediation)

* The Error: RDP connections can fail with "Authentication error. The function requested is not supported" due to a patch mismatch regarding CVE-2018-0886 (Encryption Oracle vulnerability).
* Group Policy Fix: Navigate to Computer Configuration > Administrative Templates > System > Credentials Delegation > Encryption Oracle Remediation, set it to Enabled, and choose Vulnerable protection level.
* Registry Fix: Run Command Prompt as administrator and execute:
reg add "HKLM\Software\Microsoft\Windows\CurrentVersion\Policies\System\CredSSP\Parameters" /f /v AllowEncryptionOracle /t REG_DWORD /d 2