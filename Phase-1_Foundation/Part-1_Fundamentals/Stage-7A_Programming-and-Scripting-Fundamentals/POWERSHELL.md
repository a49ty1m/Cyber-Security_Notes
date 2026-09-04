# PowerShell — Windows Automation & Enterprise Administration Practice Questions

Use this as a learning checklist. Solve each question yourself, test edge cases, and keep solutions in clearly named `.ps1` files. For Active Directory, remoting, and security exercises, use only lab environments, virtual machines, or systems you own and are explicitly authorized to administer.

---

## 1. Language Fundamentals

### 1.1 Variables and Data Types

- [ ] Declare variables of the following types and verify with `GetType()`: `[int]`, `[double]`, `[string]`, `[bool]`, `[datetime]`, `[char]`, `[byte]`, and `[array]`.
- [ ] Explain the difference between strongly-typed `[int]$count = 5` and dynamically-typed `$count = 5`; demonstrate what happens when you assign an incompatible value to each.
- [ ] Use string interpolation inside double-quoted strings: `"Hello, $name"` and `"Today is $(Get-Date)"` (sub-expression); explain why single-quoted strings suppress expansion.
- [ ] Use here-strings (`@" ... "@` and `@' ... '@`) to store multi-line text; explain when each quoting style is appropriate.
- [ ] Use `$null` to represent the absence of a value; test for null with `-eq $null`; explain why `$null -eq $variable` is the safer comparison order.
- [ ] Use automatic variables: `$_` (pipeline item), `$?` (last success), `$LASTEXITCODE`, `$PSVersionTable`, `$HOME`, `$env:USERNAME`, `$MyInvocation`; print each and explain its purpose.
- [ ] Cast between types: convert a string `"42"` to `[int]`, a number to `[string]`, and a string date to `[datetime]`; handle `InvalidCastException` with `try/catch`.
- [ ] Use `[System.Collections.ArrayList]` and a typed `[System.Collections.Generic.List[string]]`; compare with a plain `@()` array — explain which supports `.Add()` and `.Remove()` without reassignment.

### 1.2 Operators

- [ ] Use comparison operators: `-eq`, `-ne`, `-lt`, `-le`, `-gt`, `-ge`, `-like` (wildcard), `-match` (regex), `-contains`, `-in`, `-is`; give one example of each.
- [ ] Demonstrate case-insensitive vs case-sensitive operators: `-ceq`, `-clike`, `-cmatch`; explain when case sensitivity matters in Windows contexts.
- [ ] Use logical operators: `-and`, `-or`, `-not`, `-xor`; combine them to build a compound conditional.
- [ ] Use the range operator `..` to generate `1..10`; use it in a `foreach` loop and to slice an array `$arr[2..5]`.
- [ ] Use the null-coalescing operator `??` (PowerShell 7+) and null-conditional operator `?.` to safely access a property that may be `$null`; explain the fallback for PowerShell 5.1.
- [ ] Use the ternary operator `condition ? trueValue : falseValue` (PowerShell 7+); rewrite the same logic using `if/else` for 5.1 compatibility.
- [ ] Use `-replace` (regex) and `-split` (regex) on strings; demonstrate that `-split` without a regex literal is a simple delimiter split.

### 1.3 Control Flow

- [ ] Write `if / elseif / else` to classify a number as positive, negative, or zero; use `-gt`, `-lt`, and `-eq`.
- [ ] Write a `switch` statement that maps HTTP status codes (200, 301, 400, 401, 403, 404, 500) to descriptions; use `default` for unknown codes; demonstrate `switch -Wildcard` and `switch -Regex`.
- [ ] Write a `for` loop, a `foreach` loop over an array, a `while` loop, and a `do/while` loop — give a practical use case for each.
- [ ] Use `ForEach-Object` in a pipeline: `1..10 | ForEach-Object { $_ * 2 }`; compare performance with a `foreach` statement for large collections.
- [ ] Use `break`, `continue`, and labeled loops (`break :outerLoop`) inside nested loops; explain the label syntax.
- [ ] Use `Where-Object` to filter pipeline objects: `Get-Process | Where-Object { $_.CPU -gt 10 }`; rewrite using the simplified syntax `Where-Object CPU -gt 10`.
- [ ] Use `Select-Object` to choose specific properties, add calculated properties with `@{Name='...'; Expression={...}}`, and use `-First`, `-Last`, and `-Skip`.

### 1.4 Functions and Modules

- [ ] Write a function using `function Verb-Noun { param(...) }` following the approved verb convention; list approved verbs with `Get-Verb`.
- [ ] Add parameter attributes: `[Parameter(Mandatory=$true)]`, `[Parameter(ValueFromPipeline=$true)]`, `[ValidateNotNullOrEmpty()]`, `[ValidateRange(1,100)]`, `[ValidateSet('A','B','C')]`; test each validation.
- [ ] Use `[CmdletBinding()]` to make a function an advanced function; add `-Verbose`, `-Debug`, `-WhatIf`, and `-Confirm` support; explain what `ShouldProcess` does.
- [ ] Write a function that accepts pipeline input using `process { }`, `begin { }`, and `end { }` blocks; demonstrate piping objects into it.
- [ ] Explain the difference between `return` (exits function, optionally with a value) and simply placing an expression on a line (PowerShell outputs it automatically).
- [ ] Create a script module (`.psm1`), export specific functions with `Export-ModuleMember`, and import it with `Import-Module`; inspect it with `Get-Module` and `Get-Command -Module`.
- [ ] Write a module manifest (`.psd1`) with `New-ModuleManifest`; specify `ModuleVersion`, `Author`, `RequiredModules`, and `FunctionsToExport`.
- [ ] Explain the difference between a script module (`.psm1`), a binary module (`.dll`), a manifest module (`.psd1`), and a dynamic module (`New-Module`).

### 1.5 Aliases

- [ ] List all built-in aliases with `Get-Alias`; find the full cmdlet name for `ls`, `dir`, `cat`, `echo`, `cd`, `cls`, `rm`, `cp`, `mv`, and `ps`.
- [ ] Create a custom alias with `New-Alias -Name 'gh' -Value 'Get-Help'`; make it persistent by adding `Set-Alias` to your `$PROFILE`.
- [ ] Explain why aliases should not be used in scripts intended for others: they are session-dependent, can be overridden, and reduce readability.
- [ ] Use `Set-Alias` to create an alias that overrides a built-in; then remove it with `Remove-Alias`; explain how to avoid accidental override.

### 1.6 Providers and PSDrives

- [ ] List all available providers with `Get-PSProvider`; list all PSDrives with `Get-PSDrive`; navigate the `Registry:`, `Cert:`, `Env:`, and `Function:` drives using `Set-Location`.
- [ ] Read a registry value: `Get-ItemProperty -Path 'HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion' -Name 'ProductName'`.
- [ ] List items in the `Env:` drive using `Get-ChildItem Env:`; read a specific environment variable with `$env:APPDATA`.
- [ ] Use `New-PSDrive` to map a UNC share as a drive letter within a session; explain how this differs from a persistent mapped drive.
- [ ] Use `Get-Item` and `Get-ChildItem` against the `Cert:` drive to list installed certificates; filter for certificates expiring within 30 days.

### 1.7 Execution Policies

- [ ] List all execution policies with `Get-ExecutionPolicy -List`; explain the scope order: `MachinePolicy > UserPolicy > Process > CurrentUser > LocalMachine`.
- [ ] Explain each policy: `Restricted`, `AllSigned`, `RemoteSigned`, `Unrestricted`, `Bypass`, `Undefined`; state the default on Windows Server vs desktop.
- [ ] Set the execution policy for the current process only: `Set-ExecutionPolicy Bypass -Scope Process`; explain why this is safer than changing the machine policy.
- [ ] Explain how execution policy is not a security boundary — a user can bypass it with `powershell.exe -ExecutionPolicy Bypass -File script.ps1`; describe the actual security mechanisms that matter (script signing, AppLocker, WDAC).
- [ ] Use `Get-AuthenticodeSignature` to check whether a script is signed; use `Set-AuthenticodeSignature` to sign a script with a code-signing certificate from the local cert store.

### 1.8 Error Handling

- [ ] Explain the difference between terminating errors (thrown exceptions) and non-terminating errors (written to the error stream); show how `-ErrorAction Stop` promotes non-terminating errors to terminating.
- [ ] Use `try / catch / finally`; catch a specific exception type: `catch [System.IO.FileNotFoundException]`; use a general `catch` as a fallback; always clean up in `finally`.
- [ ] Use `$ErrorActionPreference = 'Stop'` at the top of a script to make all non-terminating errors terminating; explain the trade-off.
- [ ] Access `$Error[0]` and `$Error[0].Exception.Message`; use `$Error.Clear()` to reset the error buffer; explain the `-ErrorVariable` parameter.
- [ ] Use `Write-Error`, `Write-Warning`, `Write-Verbose`, `Write-Debug`, and `Write-Host` correctly; explain why `Write-Host` bypasses the pipeline and should be avoided in functions.
- [ ] Use `throw "message"` to raise a terminating error from a function; throw a custom exception object: `throw [System.InvalidOperationException]::new("message")`.
- [ ] Demonstrate error handling for `Invoke-WebRequest`: catch `System.Net.WebException`, extract the HTTP status code from the exception, and retry on 429 or 503 with exponential backoff.

### 1.9 Classes and Enums

- [ ] Define a PowerShell class with a constructor, properties (with getters/setters), a static method, and an instance method; instantiate it with `[ClassName]::new()`.
- [ ] Define a class that inherits from another class; override a method; call the base class method with `([BaseClass]$this).Method()`.
- [ ] Define an enum with `enum Status { Active; Inactive; Suspended }`; use it as a parameter type in a function to restrict valid values.
- [ ] Implement a class that implements a custom interface by inheriting from an abstract .NET base class; explain when PowerShell classes are preferable to PSCustomObject.
- [ ] Use `[PSCustomObject]@{ Name='...'; Value=42 }` to create a lightweight object; add a method to it with `Add-Member -MemberType ScriptMethod`; explain when a full class is overkill.

### 1.10 Background Jobs and Runspaces

- [ ] Start a background job with `Start-Job`; monitor it with `Get-Job`; retrieve its output with `Receive-Job`; remove it with `Remove-Job`.
- [ ] Use `Wait-Job` to block until a job completes; use `-Timeout` to impose a maximum wait time; handle jobs that time out.
- [ ] Use `Start-ThreadJob` (from the `ThreadJob` module) and compare it with `Start-Job` in terms of overhead, speed, and variable sharing.
- [ ] Use `ForEach-Object -Parallel` (PowerShell 7+) to process a collection concurrently; use `-ThrottleLimit` to cap concurrency; compare with a sequential `foreach`.
- [ ] Explain the difference between a job (separate process), a thread job (same process, separate runspace), and a runspace (raw concurrency primitive); state when each is appropriate.

---

## 2. PowerShell Core Concepts

### 2.1 The Object Pipeline

- [ ] Explain why PowerShell passes .NET objects through the pipeline, not text; demonstrate by piping `Get-Process` to `Get-Member` and reading the property list.
- [ ] Use `Get-Member` on the output of `Get-Service`, `Get-EventLog`, and `Get-ChildItem` to discover properties and methods; filter by member type with `-MemberType Property`.
- [ ] Use `Select-Object` to project specific properties: `Get-Process | Select-Object Name, CPU, WorkingSet`; add a calculated property that converts `WorkingSet` from bytes to MB.
- [ ] Use `Where-Object` to filter: find services whose status is `Running`; find processes consuming more than 100 MB of RAM; find files modified in the last 24 hours.
- [ ] Use `Sort-Object` to sort by one or more properties; use `-Descending`; use `-Unique` to deduplicate.
- [ ] Use `Group-Object` to group processes by company name; count group members with `-NoElement`; use `Measure-Object` to compute sum, average, min, and max of a numeric property.
- [ ] Use `ForEach-Object` with `$_` to transform each object; explain the performance difference between `ForEach-Object` (streams one at a time) and `foreach` (loads all into memory first).

### 2.2 Output and Formatting

- [ ] Use `Out-File -FilePath report.txt -Encoding UTF8` to save output; explain why `> file.txt` (redirect) and `Out-File` behave differently with encoding.
- [ ] Use `Export-Csv -Path data.csv -NoTypeInformation` to export objects; use `Import-Csv` to read them back as objects; explain the round-trip fidelity.
- [ ] Use `ConvertTo-Json` and `ConvertFrom-Json` to serialize and deserialize objects; use `-Depth` to control nesting level; explain what happens to objects deeper than the limit.
- [ ] Use `ConvertTo-Html` to build an HTML report from a collection of objects; add a CSS style block with `-Head`; open the report in a browser.
- [ ] Use `Format-Table`, `Format-List`, `Format-Wide`, and `Format-Custom` to control terminal display; explain why formatted output should never be piped to further processing commands.
- [ ] Use `Tee-Object` to write pipeline output to a file and simultaneously pass it downstream: `Get-Process | Tee-Object -FilePath procs.txt | Where-Object CPU -gt 5`.

### 2.3 XML Processing

- [ ] Load an XML file with `[xml]$doc = Get-Content file.xml`; navigate the DOM using dot notation: `$doc.root.child.value`.
- [ ] Use `Select-Xml` with an XPath expression to query a complex XML document; explain when XPath is cleaner than dot notation.
- [ ] Create an XML document programmatically using `[System.Xml.XmlDocument]`; add elements, set attributes, and save with `$doc.Save("output.xml")`.
- [ ] Parse the output of `nmap -oX` (XML format) with `[xml]`; extract host, port, state, and service name for all open ports.
- [ ] Explain the security risk of XML External Entity (XXE) injection; demonstrate that `[xml]` in PowerShell resolves external entities by default; show how to use `XmlReaderSettings` with `DtdProcessing = Prohibit` to disable it.

---

## 2.5 Mini Projects (Fundamentals)

> Build these in isolation before moving to Windows Administration. Each script must run clean, handle errors with `try/catch`, and output structured objects — not raw text.

- [ ] Build a **system info reporter**: collect hostname, OS version, uptime (`(Get-Date) - (Get-CimInstance Win32_OperatingSystem).LastBootUpTime`), CPU count, total RAM, and disk usage; output a `[PSCustomObject]` and export it to JSON.
- [ ] Build a **file search tool**: accept `-Path`, `-Pattern` (wildcard), `-MinSizeMB`, and `-ModifiedWithinDays` parameters; return matching files as objects with Name, Size, LastWriteTime, and FullName; support pipeline output.
- [ ] Build a **batch URL checker**: read URLs from a text file one per line; use `Invoke-WebRequest -Method Head -ErrorAction SilentlyContinue`; output one PSCustomObject per URL with URL, StatusCode, and ResponseTime in milliseconds.
- [ ] Build a **CSV processor**: accept an input CSV and a list of required column names; validate all columns are present; deduplicate rows by a key column; output a cleaned CSV and a summary of removed rows.
- [ ] Build a **log parser**: read a plain-text log file; extract lines matching severity patterns (`ERROR`, `WARN`, `INFO`) using `-match`; count each severity level; output a summary object and export matched lines to separate files.
- [ ] Build a **config manager**: read a `config.json` file; validate that required keys are present with correct types; load values into a `$Config` hashtable; expose a `Get-ConfigValue` function; handle a missing file gracefully with a clear error.
- [ ] Build a **retry wrapper** `Invoke-WithRetry`: accept a `ScriptBlock`, `-MaxAttempts`, and `-DelaySeconds`; re-run the block on failure with exponential backoff; log each attempt with `Write-Verbose`; return the last error if all attempts fail.
- [ ] Build a **multi-host ping checker**: accept a list of hostnames; run `Test-Connection -Count 1 -ErrorAction SilentlyContinue` in parallel using `ForEach-Object -Parallel -ThrottleLimit 20`; output a table of hostname, IP, reachable (bool), and latency ms.

---

## 3. Windows Administration

### 3.1 Active Directory Enumeration

- [ ] Import the `ActiveDirectory` module: `Import-Module ActiveDirectory`; verify the domain controller is reachable with `Get-ADDomainController`.
- [ ] Enumerate all users in the domain: `Get-ADUser -Filter *`; retrieve specific properties with `-Properties`; filter with `-Filter {Enabled -eq $true}`.
- [ ] Find all disabled accounts: `Get-ADUser -Filter {Enabled -eq $false} -Properties LastLogonDate | Select-Object Name, LastLogonDate`.
- [ ] Find accounts that have not logged in for 90 days: filter on `LastLogonDate` using a calculated `[datetime]` comparison.
- [ ] Enumerate all groups and their members: `Get-ADGroup -Filter *`; use `Get-ADGroupMember -Recursive` to flatten nested group membership.
- [ ] Enumerate all computers in the domain: `Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonDate`.
- [ ] Enumerate all Organizational Units: `Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName`.
- [ ] Retrieve domain-level information: `Get-ADDomain` and `Get-ADForest`; extract `DomainFunctionalLevel`, `PDCEmulator`, `InfrastructureMaster`, and `SchemaMaster` FSMO roles.

### 3.2 User and Group Management

- [ ] Create a new AD user with `New-ADUser`; set `GivenName`, `Surname`, `SamAccountName`, `UserPrincipalName`, `AccountPassword`, and `-Enabled $true` in one command.
- [ ] Reset a user's password with `Set-ADAccountPassword -Reset`; force a password change at next logon with `Set-ADUser -ChangePasswordAtLogon $true`.
- [ ] Disable and re-enable accounts with `Disable-ADAccount` and `Enable-ADAccount`; unlock a locked-out account with `Unlock-ADAccount`.
- [ ] Add and remove a user from a group: `Add-ADGroupMember` and `Remove-ADGroupMember`; verify membership with `Get-ADGroupMember`.
- [ ] Create a new security group, add members, and move it to a specific OU using `New-ADGroup` and `Move-ADObject`.
- [ ] Bulk-create users from a CSV file: read each row with `Import-Csv`; call `New-ADUser` inside `ForEach-Object`; log successes and failures to separate files.

### 3.3 Registry Operations

- [ ] Read a registry key value: `Get-ItemProperty -Path 'HKLM:\SOFTWARE\...' -Name 'KeyName'`; handle the case where the key does not exist without throwing.
- [ ] Create a new registry key with `New-Item -Path 'HKCU:\Software\MyApp'`; create a value with `New-ItemProperty -Name 'Setting' -Value 'On' -PropertyType String`.
- [ ] Modify an existing registry value with `Set-ItemProperty`; delete a value with `Remove-ItemProperty`; delete an entire key with `Remove-Item -Recurse`.
- [ ] Export a registry key to a `.reg` file using `reg export` via `Invoke-Expression` or `Start-Process`; re-import it with `reg import` for backup/restore.
- [ ] Enumerate all subkeys under `HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run` to identify startup programs; explain the security implication of attacker-controlled entries.
- [ ] Check whether a registry path exists before reading it: `Test-Path -Path 'HKLM:\...'`; wrap in a function that returns a default value when the key is missing.

### 3.4 Service Management

- [ ] Use `Get-Service` to list all services; filter for running services; filter for services with `StartType -eq 'Automatic'` but `Status -eq 'Stopped'` (auto-start failures).
- [ ] Start, stop, and restart a service: `Start-Service`, `Stop-Service`, `Restart-Service`; add `-Force` to stop services with dependents; use `-WhatIf` first.
- [ ] Change a service's start type: `Set-Service -StartupType Disabled`; explain the security use case (disabling unnecessary services).
- [ ] Query service binary paths using WMI: `Get-CimInstance Win32_Service | Select-Object Name, PathName, StartMode, State`; identify services running from unusual paths.
- [ ] Write a function that monitors a list of services, restarts any that are stopped, and logs the restart event with a timestamp to a file.

### 3.5 Process Management

- [ ] List all processes with `Get-Process`; sort by `CPU` descending; display `Name`, `Id`, `CPU`, and `WorkingSet` (in MB as a calculated property).
- [ ] Find all processes owned by a specific user: `Get-Process -IncludeUserName | Where-Object UserName -Like "*administrator*"`.
- [ ] Stop a process by name: `Stop-Process -Name "notepad" -Force`; stop by PID: `Stop-Process -Id 1234`; use `-WhatIf` to preview.
- [ ] Start a new process: `Start-Process notepad.exe`; start with arguments: `Start-Process powershell.exe -ArgumentList '-File script.ps1'`; start hidden: `-WindowStyle Hidden`.
- [ ] Use `Get-CimInstance Win32_Process` to get the command line of each running process — including arguments; identify processes with unusual command lines.

### 3.6 Event Log Collection

- [ ] Use `Get-EventLog -LogName Security -Newest 100` to retrieve the last 100 security events; filter for Event ID 4625 (failed logon) and count per source IP.
- [ ] Use `Get-WinEvent -LogName 'Security' -FilterHashtable @{LogName='Security'; Id=4624; StartTime=(Get-Date).AddHours(-1)}` to retrieve events from the last hour.
- [ ] Export filtered events to a CSV: `Get-WinEvent ... | Select-Object TimeCreated, Id, Message | Export-Csv events.csv -NoTypeInformation`.
- [ ] Use `Get-WinEvent -FilterXml` with an EVTX XML query to perform a structured search; write an XML filter that finds all 4625 events from a specific workstation.
- [ ] Parse the `Message` property of a `System.Diagnostics.Eventing.Reader.EventLogRecord` using `[xml]$e.ToXml()` to extract individual fields like username, source IP, and logon type.
- [ ] Export a Windows event log to an `.evtx` file using `wevtutil epl Security C:\backup\Security.evtx`; re-import it with `Get-WinEvent -Path`.

### 3.7 WMI and CIM

- [ ] Explain the difference between WMI (`Get-WmiObject`) and CIM (`Get-CimInstance`): CIM uses WS-Man (WSMAN) by default, is cross-platform in PowerShell 7, and is the preferred modern approach.
- [ ] Use `Get-CimInstance Win32_OperatingSystem` to get OS version, install date, last boot time, and free physical memory.
- [ ] Use `Get-CimInstance Win32_LogicalDisk` to list drives, total size, and free space; alert when free space falls below 10%.
- [ ] Use `Get-CimInstance Win32_NetworkAdapterConfiguration -Filter "IPEnabled=$true"` to retrieve network adapter IP configuration.
- [ ] Use `Get-CimInstance Win32_UserAccount -Filter "Disabled=$false"` to list local user accounts; identify accounts with `PasswordRequired=$false`.
- [ ] Use `Invoke-CimMethod` to call a WMI method: create a process with `Invoke-CimMethod -ClassName Win32_Process -MethodName Create -Arguments @{CommandLine='notepad.exe'}`.
- [ ] Use `Get-CimClass` to discover all WMI classes in a namespace: `Get-CimClass -Namespace root\CIMV2 -ClassName Win32_*`; use `Get-CimInstance -Query` to run a WQL query.

### 3.8 Scheduled Tasks

- [ ] List all scheduled tasks: `Get-ScheduledTask`; filter for enabled tasks; retrieve the trigger and action for each with `Get-ScheduledTaskInfo`.
- [ ] Create a scheduled task: define an action with `New-ScheduledTaskAction`, a trigger with `New-ScheduledTaskTrigger`, settings with `New-ScheduledTaskSettingsSet`; register with `Register-ScheduledTask`.
- [ ] Enable, disable, and unregister a scheduled task: `Enable-ScheduledTask`, `Disable-ScheduledTask`, `Unregister-ScheduledTask`.
- [ ] Audit scheduled tasks for persistence: list tasks whose actions run binaries from `%TEMP%`, `%APPDATA%`, or non-standard paths; flag them for review.
- [ ] Export all scheduled tasks to XML with `Export-ScheduledTask`; re-import with `Register-ScheduledTask -Xml`.

### 3.9 File and Directory Management

- [ ] Use `Get-ChildItem -Path C:\ -Recurse -Filter *.log -ErrorAction SilentlyContinue` to find all log files recursively; suppress access-denied errors.
- [ ] Use `Get-Item`, `New-Item`, `Copy-Item`, `Move-Item`, `Rename-Item`, and `Remove-Item` with `-WhatIf` and `-Confirm`; explain when `-Force` is necessary.
- [ ] Read a file with `Get-Content`; read it as a single string with `-Raw`; write with `Set-Content`; append with `Add-Content`; use `-Encoding UTF8` explicitly.
- [ ] Use `Test-Path` to check existence before operating on a file; combine with `try/catch` for robust file handling.
- [ ] Use `Get-Acl` to read a file or folder's ACL; use `Set-Acl` to apply an ACL; add a new access rule with `[System.Security.AccessControl.FileSystemAccessRule]`.
- [ ] Compute the SHA-256 hash of a file with `Get-FileHash -Algorithm SHA256`; compare against a known-good hash to verify integrity.
- [ ] Find files modified in the last 24 hours: `Get-ChildItem -Recurse | Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-24) }`.

### 3.10 SMB, Firewall, Defender, and Certificates

- [ ] List all SMB shares with `Get-SmbShare`; view connected sessions with `Get-SmbSession`; view open files with `Get-SmbOpenFile`.
- [ ] Create and remove an SMB share: `New-SmbShare -Name 'Data' -Path 'C:\Data' -FullAccess 'DOMAIN\Admins'`; remove with `Remove-SmbShare`.
- [ ] List Windows Firewall rules with `Get-NetFirewallRule`; filter for enabled inbound rules; add a new rule with `New-NetFirewallRule`; remove with `Remove-NetFirewallRule`.
- [ ] Check Windows Defender status: `Get-MpComputerStatus`; retrieve threat definitions version, real-time protection status, and last scan time.
- [ ] Trigger a Defender quick scan: `Start-MpScan -ScanType QuickScan`; view recent detections with `Get-MpThreatDetection`.
- [ ] List certificates in the local machine store: `Get-ChildItem Cert:\LocalMachine\My`; filter for expired certificates; find certificates issued to a specific subject.
- [ ] Export a certificate to a `.cer` file with `Export-Certificate`; import one with `Import-Certificate`; explain the difference between exporting with and without the private key (`.pfx`).

### 3.11 Local User and Group Management

> These cmdlets manage **local** accounts on a single machine — distinct from Active Directory. Essential for workgroup environments, standalone servers, and enumerating local admin membership on domain-joined machines.

- [ ] List all local users with `Get-LocalUser`; filter for enabled accounts; identify accounts with `PasswordNeverExpires` or `PasswordRequired -eq $false`.
- [ ] Create a local user: `New-LocalUser -Name 'testuser' -Password (Read-Host -AsSecureString) -FullName 'Test' -Description 'Lab account'`; set an expiry with `-AccountExpires`.
- [ ] Enable, disable, and remove a local user: `Enable-LocalUser`, `Disable-LocalUser`, `Remove-LocalUser`; always confirm before removing.
- [ ] List all local groups with `Get-LocalGroup`; list members of a specific group: `Get-LocalGroupMember -Group 'Administrators'`.
- [ ] Add and remove a user from a local group: `Add-LocalGroupMember`, `Remove-LocalGroupMember`; verify membership after each operation.
- [ ] Compare `Get-LocalGroupMember -Group 'Administrators'` output with `net localgroup Administrators`; explain any discrepancies in SID vs name resolution.
- [ ] Write a function that audits local Administrators membership across a list of machines using `Invoke-Command`; flag any member not in an approved baseline list.

---

## 4. Enterprise Automation (PowerShell Remoting)

### 4.1 WinRM and Session Setup

- [ ] Enable PowerShell Remoting on a local lab VM: `Enable-PSRemoting -Force`; verify WinRM is running with `Get-Service WinRM`.
- [ ] Test connectivity to a remote host: `Test-WsMan -ComputerName server01`; use `Test-NetConnection -Port 5985` to verify WinRM accessibility.
- [ ] Explain WinRM listeners: HTTP (port 5985) vs HTTPS (port 5986); configure an HTTPS listener with a self-signed certificate for lab use; explain why HTTPS is mandatory in production.
- [ ] Add a host to the `TrustedHosts` list (for workgroup environments): `Set-Item WSMan:\localhost\Client\TrustedHosts -Value "server01"`; explain why this is not required in a domain.

### 4.2 Invoke-Command and PSSession

- [ ] Run a command on a single remote machine: `Invoke-Command -ComputerName server01 -ScriptBlock { Get-Process }`; use `-Credential` to specify alternate credentials.
- [ ] Run the same command on multiple machines simultaneously: `Invoke-Command -ComputerName server01, server02, server03 -ScriptBlock { Get-Service Spooler }`.
- [ ] Use `$using:` to pass a local variable into a remote `ScriptBlock`: `$threshold = 90; Invoke-Command -ComputerName server01 -ScriptBlock { Get-Process | Where-Object CPU -gt $using:threshold }`.
- [ ] Create a persistent session with `New-PSSession`; reuse it for multiple `Invoke-Command` calls; close it with `Remove-PSSession`; explain why persistent sessions are more efficient than per-command connections.
- [ ] Enter an interactive session with `Enter-PSSession -ComputerName server01`; exit with `Exit-PSSession`; explain when interactive sessions are appropriate vs scripted `Invoke-Command`.
- [ ] Run a local script on a remote machine: `Invoke-Command -ComputerName server01 -FilePath .\script.ps1`; explain that the script file is sent and executed remotely.
- [ ] Use `Invoke-Command` to collect data from 50 machines in parallel; use `$results = Invoke-Command ...` to capture all output as objects; handle unreachable hosts with `-ErrorAction SilentlyContinue`.

### 4.3 CIM Sessions

- [ ] Create a CIM session: `$cs = New-CimSession -ComputerName server01`; use it with `Get-CimInstance -CimSession $cs Win32_OperatingSystem`; close with `Remove-CimSession`.
- [ ] Create a CIM session over WSMAN (default) and over DCOM: `New-CimSessionOption -Protocol Dcom`; explain when DCOM fallback is necessary (legacy Windows without WinRM).
- [ ] Use a CIM session to query multiple machines: `New-CimSession -ComputerName (Get-Content servers.txt)`; use `Get-CimInstance` with the `-CimSession` array.
- [ ] Run a CIM method remotely via session: restart a service with `Invoke-CimMethod -CimSession $cs -ClassName Win32_Service -MethodName StopService -Filter "Name='Spooler'"`.

### 4.4 Remote Event Log Collection

- [ ] Use `Get-WinEvent -ComputerName server01 -LogName Security -MaxEvents 50` to query a remote event log; provide credentials with `-Credential`.
- [ ] Build a script that collects Security Event ID 4625 (failed logons) from a list of servers; deduplicate source IPs across all servers; output a ranked summary.
- [ ] Use `Invoke-Command` to collect events from 10 servers in parallel; merge and sort results by `TimeCreated`; export to a single CSV.

### 4.5 Group Policy and DSC (Awareness)

- [ ] Explain what Group Policy is and how `gpupdate /force` and `gpresult /r` are used to apply and review policy; use `Invoke-Command` to run `gpupdate` on a remote machine.
- [ ] Use the `GroupPolicy` module (requires RSAT): `Get-GPO -All` to list all GPOs; `Get-GPOReport -Name 'Default Domain Policy' -ReportType Html -Path gpo.html` to export a report.
- [ ] Explain Desired State Configuration (DSC) at an awareness level: what a DSC configuration document is, how `Start-DscConfiguration` applies it, and how `Test-DscConfiguration` verifies drift — without writing a full configuration.

---

## 5. Active Directory Automation

### 5.1 User and Group Enumeration

- [ ] Find all users who are members of the `Domain Admins` group (including nested membership): `Get-ADGroupMember -Identity 'Domain Admins' -Recursive`.
- [ ] Find all users whose passwords never expire: `Get-ADUser -Filter {PasswordNeverExpires -eq $true} -Properties PasswordNeverExpires`.
- [ ] Find users with `AdminCount = 1` (protected accounts): `Get-ADUser -Filter {AdminCount -eq 1} -Properties AdminCount, MemberOf`.
- [ ] Find stale computer accounts not logged in for 90 days: `Get-ADComputer -Filter {LastLogonDate -lt $cutoff} -Properties LastLogonDate`.
- [ ] List all service accounts (users with `ServicePrincipalName` set — Kerberoastable accounts): `Get-ADUser -Filter {ServicePrincipalName -ne "$null"} -Properties ServicePrincipalName`.
- [ ] Export full user inventory to CSV: name, SAM account, email, department, manager, enabled status, last logon, password last set, and group memberships.

### 5.2 OU, GPO, ACL, and Trust Enumeration

- [ ] List all OUs and their distinguished names; find OUs with no GPO linked using `Get-GPInheritance`.
- [ ] Enumerate GPO links for each OU: `Get-ADOrganizationalUnit -Filter * | ForEach-Object { (Get-GPInheritance -Target $_.DistinguishedName).GpoLinks }`.
- [ ] Inspect ACLs on the domain object: `Get-Acl "AD:DC=domain,DC=local"` and parse `Access` entries; flag non-standard principals with `GenericAll` or `WriteDacl` rights.
- [ ] Enumerate domain trusts: `Get-ADTrust -Filter *`; identify the trust direction, type (forest/external), and whether `SID Filtering` is enabled.
- [ ] List delegation settings on OUs: find OUs where `ProtectedFromAccidentalDeletion` is `$false`; find OUs with delegated control to non-admin accounts.

---

## 6. Essential Modules & APIs

### 6.1 Microsoft.PowerShell.Management and Utility

- [ ] Use `Get-ComputerInfo` to retrieve a comprehensive system inventory in one cmdlet; select and export the key properties to JSON.
- [ ] Use `Invoke-WebRequest` to send a GET request; inspect `StatusCode`, `Headers`, and `Content`; parse the response as HTML using `$r.Links` and `$r.Forms`.
- [ ] Use `Invoke-RestMethod` to call a REST API; compare it with `Invoke-WebRequest` — explain that `Invoke-RestMethod` automatically deserializes JSON/XML responses.
- [ ] Add request headers (Authorization Bearer, Content-Type, User-Agent) to `Invoke-WebRequest` and `Invoke-RestMethod`; handle `401` and `429` responses explicitly.
- [ ] Use `New-TemporaryFile` and `[System.IO.Path]::GetTempPath()` to create secure temporary files; clean up with `Remove-Item` in a `try/finally`.
- [ ] Use `Start-Sleep` for delays; use `Measure-Command { ... }` to benchmark a script block; use `[System.Diagnostics.Stopwatch]` for high-resolution timing.

### 6.2 NetTCPIP

- [ ] Use `Get-NetIPAddress` to list all IP addresses on all adapters; use `Get-NetAdapter` to list network adapters and their link speed and status.
- [ ] Use `Get-NetTCPConnection` to list all TCP connections and listening ports; filter for `State -eq 'Listen'`; identify the owning process with `-OwningProcess`.
- [ ] Use `Test-NetConnection -ComputerName host -Port 443` to test TCP connectivity; use `-InformationLevel Detailed` for full diagnostic output including route and ping.
- [ ] Use `Resolve-DnsName` to query A, AAAA, MX, NS, and TXT records; compare with `nslookup` output; explain when `Resolve-DnsName` is preferred in scripts.
- [ ] Use `Get-NetRoute` to display the routing table; identify the default gateway; use `New-NetRoute` to add a static route in a lab environment.

### 6.3 ScheduledTasks and CimCmdlets

- [ ] Write a wrapper function around `New-ScheduledTask*` cmdlets that accepts a script path, schedule type, and run-as credential; registers and verifies the task in one call.
- [ ] Use `Get-CimClass -Namespace root\CIMV2` to search for WMI classes by wildcard; use `Get-CimInstance -ClassName` with `-Filter` (WQL WHERE clause) for targeted queries.
- [ ] Use `Register-CimIndicationEvent` to subscribe to a WMI event (e.g., process creation) in a lab VM; use `Wait-Event` to receive the first notification; explain the use case in detection engineering.

### 6.4 PSReadLine and PowerShellGet

- [ ] Configure PSReadLine history: `Set-PSReadLineOption -HistorySavePath "$env:APPDATA\PSReadLine\history.txt" -MaximumHistoryCount 5000`; explain the security concern of history files containing credentials.
- [ ] Use `Set-PSReadLineKeyHandler` to bind a custom key to a scriptblock; use `Set-PSReadLineOption -PredictionSource History` (or `HistoryAndPlugin`) for intelligent completion.
- [ ] Use `Find-Module` to search the PowerShell Gallery; use `Install-Module -Scope CurrentUser -Force`; use `Update-Module`; explain `AllowClobber` and its risk.
- [ ] Inspect a module before installing it: `Find-Module -Name PSScriptAnalyzer | Select-Object -ExpandProperty Dependencies`; view its source on the Gallery; explain supply-chain risk in public repositories.
- [ ] Use `Save-Module` to download a module without installing it; inspect its contents before deployment; publish a module to a private NuGet feed with `Publish-Module`.

---

## 7. Automation Projects

### 7.1 Active Directory Inventory

- [ ] Build an AD inventory script that exports: all users (with department, manager, enabled status, last logon, password age), all groups (with member count), all computers (with OS version, last logon), and all OUs — to separate CSV files.
- [ ] Add error handling for when the `ActiveDirectory` module is unavailable; print a clear installation message (`Install-WindowsFeature RSAT-AD-PowerShell`).
- [ ] Add a `--Credential` parameter so the script can run against a remote domain with alternate credentials.
- [ ] Schedule the inventory script as a daily task using `Register-ScheduledTask`; send the CSV outputs to a shared network path.

### 7.2 User Audit Reports

- [ ] Build a user audit report that flags: accounts inactive for >90 days, accounts with passwords older than 90 days, accounts with `PasswordNeverExpires`, accounts with `AdminCount=1` that are not in the expected admin group, and accounts with no manager set.
- [ ] Format the report as an HTML table using `ConvertTo-Html` with conditional row coloring (red for critical findings); save as `UserAudit_YYYY-MM-DD.html`.
- [ ] Add a summary section at the top of the HTML report showing finding counts per category.

### 7.3 Event Log Collector

- [ ] Build a multi-server event log collector: read a list of servers from a file; collect the last 24 hours of Security events (IDs 4624, 4625, 4648, 4672, 4720, 4726, 4732, 4756) from each server using `Invoke-Command`; merge all events into a single CSV sorted by `TimeCreated`.
- [ ] Add filtering: exclude known-good service accounts from logon events; flag events from IPs in a deny-list.
- [ ] Handle unreachable servers gracefully: log the error with timestamp; continue to next server; include reachability status in the final report.

### 7.4 Service Monitoring Script

- [ ] Build a service monitor that reads a list of `ServerName,ServiceName` pairs from a CSV; checks each service state with `Invoke-Command`; restarts stopped services; logs every restart with timestamp, server, and service name.
- [ ] Add alerting: use `Send-MailMessage` to email an alert when a service is restarted (configure SMTP settings from environment variables, not hardcoded).
- [ ] Add a `--WhatIf` mode that reports which services would be restarted without actually doing it.

### 7.5 Registry Auditing Tool

- [ ] Build a script that audits common persistence registry keys across a list of machines: `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run`, `RunOnce`, `HKCU\...\Run`; use `Invoke-Command` to read remotely; compare against a known-good baseline CSV and flag new or modified entries.
- [ ] Output findings as a structured PSCustomObject with `Server`, `Hive`, `KeyPath`, `ValueName`, `ValueData`, and `ChangeType` (New / Modified / Removed).

### 7.6 Windows Asset Inventory

- [ ] Build an asset inventory script that collects from each machine: OS version, uptime, CPU model, RAM, disk layout, IP addresses, installed software (`Get-CimInstance Win32_Product` — note its performance cost), running services, and local admin group members.
- [ ] Save results as one JSON file per host; merge all JSON files into a single summary CSV using `ConvertFrom-Json` and `Select-Object`.
- [ ] Warn and skip hosts that take longer than 30 seconds to respond, using `Invoke-Command -AsJob` and `Wait-Job -Timeout 30`.

### 7.7 Remote Administration Toolkit

- [ ] Build a toolkit module with functions: `Invoke-RemoteCommand`, `Push-Script`, `Get-RemoteInventory`, `Restart-RemoteService`, and `Get-RemoteEventLog`; each must accept `-ComputerName` as a pipeline-capable parameter.
- [ ] Add verbose logging with `Write-Verbose` throughout; ensure all functions support `-WhatIf` for destructive operations.
- [ ] Write Pester tests for each function using mocked `Invoke-Command` responses.

### 7.8 File Integrity Checker

- [ ] Build a baseline tool: compute `SHA-256` hashes of all files in a directory with `Get-FileHash -Recurse`; save results to a JSON baseline file with path, hash, and timestamp.
- [ ] Build a comparison tool: re-hash the same directory; compare against the baseline; report Added, Removed, and Modified files.
- [ ] Schedule daily checks via `Register-ScheduledTask`; append findings to a cumulative audit log file.

### 7.9 Windows Health Monitoring

- [ ] Build a health check script: collect CPU load average (from `Get-CimInstance Win32_Processor`), memory usage (`Win32_OperatingSystem`), disk free space (`Win32_LogicalDisk`), and top 5 CPU processes; alert via `Write-Warning` and log when any threshold is exceeded.
- [ ] Run the health check against a list of servers with `Invoke-Command`; output a `[PSCustomObject]` per server with `Status` (OK/WARN/CRITICAL) for each metric.

### 7.10 Enterprise Reporting Dashboard

- [ ] Build a script that aggregates outputs from the above projects (user audit, asset inventory, event log, service monitor) into a single HTML dashboard with one section per data source.
- [ ] Use `ConvertTo-Html` with a `<style>` block for a clean, table-based layout; include a generation timestamp in the footer.
- [ ] Parameterize the output path and add a `--OpenInBrowser` switch that opens the report with `Invoke-Item`.

---

## 8. Security & Defensive Automation

### 8.1 Windows System Enumeration

- [ ] Write a system enumeration script for authorized systems: collect OS, patch level (`Get-HotFix`), local users, local groups and members, running services, scheduled tasks, startup items (registry Run keys), and installed applications.
- [ ] Identify misconfigurations: services running from writable paths, unquoted service paths (`Get-CimInstance Win32_Service | Where-Object { $_.PathName -match ' ' -and $_.PathName -notmatch '"' }`), and world-writable directories in `%PATH%`.
- [ ] Collect `AlwaysInstallElevated` registry settings: check `HKLM` and `HKCU` policy keys; flag if both are set to `1` (privilege escalation risk).

### 8.2 Active Directory Auditing

- [ ] Write an AD security audit function: find accounts with Kerberoastable SPNs, AS-REP roastable accounts (`DoesNotRequirePreAuth`), users with unconstrained delegation, computers with unconstrained delegation, and ACEs granting `GenericAll` or `WriteDacl` to non-admin principals.
- [ ] Find shadow admin accounts: users not in `Domain Admins` but with `AdminCount=1` or with direct `WriteDacl`/`GenericAll` ACEs on the domain object.
- [ ] Compare current Domain Admin membership against a known-good baseline list; alert on additions or removals.

### 8.3 Windows Defender and Update Auditing

- [ ] Write a Defender status checker: `Get-MpComputerStatus`; flag machines where `RealTimeProtectionEnabled`, `AntivirusEnabled`, or `AntispywareEnabled` is `$false`.
- [ ] Check Windows Update status: `Get-HotFix | Sort-Object InstalledOn -Descending | Select-Object -First 10`; calculate days since the last update; alert if it exceeds 30 days.
- [ ] Use `Get-MpThreatDetection` to retrieve recent threat detections; export findings with threat name, detection time, and affected file path.

### 8.4 Event Log Analysis

- [ ] Build a logon analysis script: collect Event IDs 4624 (success), 4625 (failure), 4648 (explicit logon), 4672 (special privileges); count failures per account; flag accounts with more than 5 failures in 10 minutes (brute-force indicator).
- [ ] Collect Event ID 4720 (account created), 4726 (account deleted), 4732 (user added to security-enabled local group), 4756 (user added to universal group); alert on changes to privileged groups.
- [ ] Build a PowerShell activity log parser: collect Event ID 4103 (module logging) and 4104 (script block logging) from `Microsoft-Windows-PowerShell/Operational`; flag script blocks containing suspicious keywords (`Invoke-Expression`, `DownloadString`, `EncodedCommand`, `bypass`).

### 8.5 IOC Hunting

- [ ] Write an IOC hunter that reads a list of IP addresses, domains, and file hashes from a JSON IOC file; searches running processes, network connections, and the file system for matches; reports findings with context.
- [ ] Use `Get-NetTCPConnection | Where-Object { $iocIPs -contains $_.RemoteAddress }` to detect active connections to known-bad IPs.
- [ ] Use `Get-FileHash -Algorithm MD5` against files in common user-writable directories; compare against a hash blocklist.

### 8.6 Security Baseline Verification

- [ ] Write a CIS benchmark checker for a subset of Windows Server controls: verify password policy (`net accounts`), audit policy (`auditpol /get /category:*`), SMB signing, NTLMv2 enforcement registry keys, and Remote Desktop NLA setting.
- [ ] Output a structured report: each check with `Control`, `Expected`, `Actual`, and `Status` (Pass/Fail); export as CSV and HTML.

### 8.7 Local Administrator Auditing

- [ ] Write a local admin auditing script: for each machine in a list, run `Invoke-Command { Get-LocalGroupMember -Group 'Administrators' }` and collect results as a PSCustomObject with hostname and member list.
- [ ] Compare the collected local admin list against a known-good baseline CSV (`approved_admins.csv`); flag any account present on a machine that is not in the approved baseline for that machine.
- [ ] Identify machines where a domain user account (not a service account or expected admin) has been added to the local Administrators group; explain the lateral movement risk.
- [ ] Detect built-in Administrator accounts that are enabled and have no password expiry: `Get-LocalUser -Name 'Administrator' | Where-Object { $_.Enabled -and $_.PasswordNeverExpires }`; flag as a finding.
- [ ] Write a remediation function that removes an unauthorized local admin member using `Remove-LocalGroupMember`; log the change with timestamp, machine name, account removed, and operator identity before executing.
- [ ] Schedule the audit daily via `Register-ScheduledTask`; email a delta report (additions since last run) using `Send-MailMessage` with SMTP credentials loaded from environment variables.

### 8.8 Scheduled Task Auditing

- [ ] Enumerate all scheduled tasks across a list of remote machines: `Invoke-Command { Get-ScheduledTask | Get-ScheduledTaskInfo }`; collect task name, action path, trigger, run-as principal, and last run time.
- [ ] Flag tasks whose action executable path is in a suspicious location: `%TEMP%`, `%APPDATA%`, `%PUBLIC%`, or any user-writable directory outside `C:\Windows` and `C:\Program Files`.
- [ ] Flag tasks running as `SYSTEM` or a domain admin that were not present in a known-good baseline; export as `New_Task` findings.
- [ ] Check tasks whose action arguments contain `-EncodedCommand`, `-Enc`, `IEX`, `DownloadString`, or `Invoke-Expression`; these are common persistence indicators used by malware.
- [ ] Compare `LastTaskResult` for all tasks; flag tasks returning a non-zero exit code (not `0x0`) — a silently failing task may indicate a broken or tampered script.
- [ ] Build a task baseline tool: on first run, export all task definitions to a JSON snapshot; on subsequent runs, diff against the snapshot and report Added, Removed, and Modified tasks as `[PSCustomObject]`.

### 8.9 Service Permission Auditing

> Misconfigured service DACLs are a common Windows privilege escalation vector. A non-admin user with `SERVICE_CHANGE_CONFIG` permission can replace a service's binary path and escalate to SYSTEM.

- [ ] Enumerate all services and their binary paths: `Get-CimInstance Win32_Service | Select-Object Name, PathName, StartMode, StartName, State`; identify services running as `LocalSystem` with paths in user-writable directories.
- [ ] Find unquoted service paths: `Get-CimInstance Win32_Service | Where-Object { $_.PathName -match ' ' -and $_.PathName -notmatch '"' -and $_.PathName -notmatch '^[A-Z]:\\Windows' }`; explain how Windows resolves unquoted paths and why each space is a potential hijack point.
- [ ] Use `sc.exe sdshow <ServiceName>` to retrieve the SDDL of a service; parse the SDDL string to identify which SIDs have `CC` (`SERVICE_CHANGE_CONFIG`) or `RP` (`SERVICE_START`) rights.
- [ ] Write a function that translates a SID from a service SDDL into a human-readable account name: `(New-Object System.Security.Principal.SecurityIdentifier('S-1-...')).Translate([System.Security.Principal.NTAccount]).Value`.
- [ ] Audit all service DACLs: loop over all services, retrieve their SDDL, parse it, and flag any service where a non-privileged SID has write permissions; output `ServiceName, PrincipalName, Permissions, Risk` as CSV.
- [ ] Check registry ACLs on service keys: `Get-Acl 'HKLM:\SYSTEM\CurrentControlSet\Services\Spooler' | Select-Object -ExpandProperty Access`; flag entries where non-admin principals have `FullControl` or `SetValue` rights.
- [ ] Explain the difference between service binary hijacking, DLL search-order hijacking, and unquoted service path exploitation; describe which DACL or registry misconfiguration enables each attack.

---

## 9. Debugging & Best Practices

### 9.1 PowerShell ISE and VS Code

- [ ] Set up VS Code with the PowerShell extension; configure the default shell to `pwsh` (PowerShell 7); explain the difference between the integrated terminal and the PowerShell Extension Host.
- [ ] Use the VS Code PowerShell extension to: set breakpoints, step through code (F10/F11), inspect variables in the Variables panel, and use the Debug Console.
- [ ] Use the PowerShell ISE Debugger (Windows PowerShell 5.1): set a breakpoint with `Set-PSBreakpoint -Script script.ps1 -Line 42`; list breakpoints with `Get-PSBreakpoint`; remove with `Remove-PSBreakpoint`.
- [ ] Use `Set-PSBreakpoint -Variable varName -Mode ReadWrite` to break whenever a variable is read or written; explain the use case for tracking unexpected state changes.

### 9.2 PSScriptAnalyzer

- [ ] Install PSScriptAnalyzer: `Install-Module -Name PSScriptAnalyzer`; run it on a script: `Invoke-ScriptAnalyzer -Path script.ps1`; fix every reported warning and explain the rule behind each.
- [ ] Run with a specific rule set: `Invoke-ScriptAnalyzer -Path . -Settings PSGallery`; use `-IncludeRule` and `-ExcludeRule` to customize the analysis.
- [ ] Add a PSScriptAnalyzer check to a VS Code `tasks.json` so it runs on save; add it to a `pre-commit` Git hook that blocks commits with `Error`-severity findings.
- [ ] Explain the most important PSScriptAnalyzer rules: `PSAvoidUsingWriteHost`, `PSUseSingularNouns`, `PSAvoidUsingPlainTextForPassword`, `PSAvoidUsingInvokeExpression`, and `PSUseShouldProcessForStateChangingFunctions`.

### 9.3 Logging

- [ ] Write a `Write-Log` function that accepts `-Level` (INFO, WARN, ERROR), prefixes each line with `[YYYY-MM-DD HH:mm:ss] [LEVEL]`, and writes to both a file and the appropriate stream (`Write-Verbose`, `Write-Warning`, `Write-Error`).
- [ ] Use `Start-Transcript -Path log.txt -Append` to record an entire session to a file; stop with `Stop-Transcript`; explain what is and is not captured.
- [ ] Redact sensitive fields before logging: mask all but the last four characters of a credential string; exclude `SecureString` and `PSCredential` objects from log output.
- [ ] Use `Write-EventLog -LogName Application -Source 'MyScript' -EventId 1001 -EntryType Information -Message '...'` to write custom events to the Windows Event Log (after registering the source).

### 9.4 Script Signing and Secure Credential Handling

- [ ] Generate a self-signed code-signing certificate for lab use: `New-SelfSignedCertificate -Type CodeSigningCert -Subject 'CN=MyScriptSigner' -CertStoreLocation Cert:\CurrentUser\My`.
- [ ] Sign a script with `Set-AuthenticodeSignature -FilePath script.ps1 -Certificate $cert`; verify the signature with `Get-AuthenticodeSignature`; explain what the signature protects (integrity) and what it does not protect (confidentiality).
- [ ] Use `Get-Credential` to prompt for credentials at runtime; store a `SecureString` password with `ConvertTo-SecureString` and `ConvertFrom-SecureString`; save the encrypted blob to a file (note: this is tied to the user and machine that created it).
- [ ] Use the `Microsoft.PowerShell.SecretManagement` module (awareness): install it with `Install-Module`; register a vault extension; use `Get-Secret` and `Set-Secret` to retrieve and store credentials without embedding them in scripts.
- [ ] Never hardcode passwords in scripts; never log `PSCredential` objects; never pass passwords as plain-text command arguments (use `SecureString` piped to `-Password`).

### 9.5 Testing with Pester

> Pester is the standard PowerShell testing framework — the equivalent of pytest (Python) and BATS (Bash). Every reusable function and module you write should have Pester tests.

- [ ] Install Pester: `Install-Module -Name Pester -Force -SkipPublisherCheck`; verify with `Get-Module Pester -ListAvailable`; explain why the version bundled with Windows is outdated and should not be used.
- [ ] Write a `Describe` block with `It` tests: one that verifies a function returns the expected value on valid input, and one that verifies it throws on invalid input: `{ ... } | Should -Throw`.
- [ ] Use `Should` assertions: `-Be`, `-BeExactly`, `-BeNullOrEmpty`, `-BeGreaterThan`, `-Match`, `-Contain`, `-Throw`, `-Not`; give one practical example of each.
- [ ] Use `BeforeAll`, `AfterAll`, `BeforeEach`, and `AfterEach` blocks to set up and tear down test state; create and remove a temporary directory in `BeforeEach`/`AfterEach`.
- [ ] Use `Mock` to stub an external command: `Mock Invoke-RestMethod { return [PSCustomObject]@{ status = 'ok' } }`; verify it was called with `Assert-MockCalled -CommandName Invoke-RestMethod -Times 1`.
- [ ] Use `InModuleScope` to test private (non-exported) functions inside a `.psm1` module; explain when this is necessary and when it is a design smell.
- [ ] Run tests with `Invoke-Pester -Path tests/ -Output Detailed`; use `-CodeCoverage` to measure line coverage; interpret the coverage report and identify untested branches.
- [ ] Use `Invoke-Pester -OutputFile results.xml -OutputFormat NUnitXml` to produce machine-readable output for CI; add the test run to a Git `pre-push` hook that blocks the push if any tests fail.
- [ ] Write a test for a function that reads a CSV: mock `Import-Csv` to return a controlled dataset; verify the function handles an empty file, a missing required column, and valid data all correctly.

### 9.6 Git for Version Control

- [ ] Initialize a Git repository for your scripts; write a `.gitignore` that excludes `*.log`, `*.csv`, `*.xml`, `secrets.json`, `*.pfx`, `*.key`, and `transcript_*.txt`.
- [ ] Use `git-secrets` or `trufflehog` to scan the repository for accidentally committed credentials; remediate with `git filter-repo`.
- [ ] Write a `pre-commit` hook that runs `Invoke-ScriptAnalyzer` on all staged `.ps1` files and blocks the commit on `Error`-severity findings.
- [ ] Use semantic commit prefixes: `feat:`, `fix:`, `chore:`, `sec:`, `docs:`; write a commit message convention guide in `CONTRIBUTING.md` for the scripts repository.

---

## 10. Learning Outcome

- [ ] Write a complete, professional PowerShell script from scratch — `[CmdletBinding()]`, parameter validation, `try/catch/finally`, structured logging, `Write-Verbose`/`Write-Warning`, a `--WhatIf` guard, and a cleanup `finally` block — without referencing templates.
- [ ] Use the PowerShell pipeline fluently: `Get-*`, `Where-Object`, `Select-Object`, `Sort-Object`, `Group-Object`, `Measure-Object`, `Export-Csv`, `ConvertTo-Json` — in meaningful combinations to answer real administrative questions.
- [ ] Build a modular automation suite: a shared library module (`.psm1`), a manifest (`.psd1`), consistent logging, a `--DryRun` switch, and a `--Help` function.
- [ ] Diagnose a broken script using VS Code breakpoints, `PSScriptAnalyzer`, and `$Error[0]`; identify the root cause without guessing.
- [ ] Automate a complete operational workflow (AD audit, event log collection, or service monitoring) that runs reliably as an unattended scheduled task and produces a structured report.
- [ ] Write a security automation script (IOC hunter, privilege enumeration, or baseline checker) that handles errors gracefully, redacts sensitive data from logs, and produces a structured CSV/HTML report.
- [ ] Explain the output of `Get-ADUser`, `Get-CimInstance Win32_Process`, `Get-WinEvent`, `Get-NetTCPConnection`, and `Get-NetFirewallRule` without looking them up.
- [ ] Obtain explicit written authorization before running any enumeration, auditing, or reconnaissance script against systems you do not personally own and administer.
