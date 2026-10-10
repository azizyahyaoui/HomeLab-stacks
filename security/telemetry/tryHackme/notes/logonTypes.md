There are 9 commonly referenced active Windows logon types (numbers 2 through 11, with a few values omitted), plus a small set of legacy or special system logons. Windows Security Event Logs—such as Event ID 4624 for successful logons—use specific numeric codes to identify how a user or service authenticated to a system.

## Common Windows Logon Types

* Type 2 - Interactive: A user logs on physically at the computer console using a keyboard, mouse, or screen.

* Type 3 - Network: A user or computer accesses shared resources, printers, or mapped drives over a network.

* Type 4 - Batch: Logons generated automatically by the operating system to run scheduled tasks or batch jobs.

* Type 5 - Service: Logons created when a Windows service starts under a specific user or system account.

* Type 7 - Unlock: A user unlocks a workstation that was previously locked by a password-protected screen saver or lock screen.

* Type 8 - NetworkCleartext: A network logon in which the password was transmitted in cleartext, typically associated with legacy IIS authentication.

* Type 9 - NewCredentials: A session started with alternate credentials, such as when using RunAs with specific network parameters.

* Type 10 - RemoteInteractive: A remote session established through Remote Desktop Protocol (RDP) or Remote Assistance.

* Type 11 - CachedInteractive: A domain logon that uses locally cached credentials when a domain controller is unavailable offline.

(Note: Type 0 represents a system logon, while higher specialized cached variants such as Types 12 and 13 exist for advanced remote unlock scenarios.)

If you are investigating a security event, let me know which Event ID or logon type number you found in your logs so I can help you analyze whether it is normal behavior or a potential risk.