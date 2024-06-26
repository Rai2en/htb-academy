# Windows Fundamentals

## Introduction to Windows

many versions of windows are now deemed legacy and are no longer supported   
orgs often find themselves running older OS to support critical apps due to operational or budget concerns   
a pen tester will need to understand the differences between versions and the various misconfigs and vulnerabilities in each one 

major windows OS and version numbers: 

![](Images/Pasted%20image%2020240416191535.png)

we can use `Get-WmiObject` cmdlet to find info about the OS   
can be used to get instances of WMI classes or info about available WMI classes   
the `win32_OperatingSystem` class shows info like version and build number: 

`Get-WmiObject -Class win32_OperatingSystem | select Version,BuildNumber`

![](Images/Pasted%20image%2020240417160145.png)

from this we can see that the build number is 19041 and the windows NT version is Windows 10

some other useful classes that can be used with `Get-WmiObject` are:   
- `Win32_Process` to get a process listing 
- `Win32_Service` to get a listing of services 
- `Win32_Bios` to get BIOS info 
- `ComputerName` parameter to get info about remote computers 

`Get-WmiObject` can be used to start and stop services on local and remote pcs 

### Accessing windows 

local access = most common way of accessing a computer; physical access to device   
input through keyboard or trackpad/mouse   
output coming from the screen  

remote access = accessing pc over a network   
local access is needed before someone can access another pc remotely   
industries like MSPs and MSSPs are primarily dependent on managing their client's pc remotely   

some of the most common remote access tech: 
- virtual private network VPN 
- secure shell SSH 
- file transfer protocol FTP 
- virtual network computing VNC 
- windows remote management (or powershell remoting) WinRM
- remote desktop protocol RDP 

### Remote desktop protocol 

rdp uses a client/server architecture where client-side app is used to specify a computer's target IP address or hostname over a network where RDP is enabled   
target pc with RDP enabled is considered the server   
listens by default on logical port 3389    

IP address is used as a logical identifier for a pc on a network  
logical port is an identifier assigned to an application   

request reaches destination computer via its IP address then it is directed to an app hosted on the computer based on the port specified 

simply, every pc has an IP address assigned to communicate over a network and apps hosted on target computers listen on specific logical ports 

if we are connecting to a windows target from windows we can use the built in RDP client `Remote Desktop Connection` (mstsc.exe): 

![](Images/Pasted%20image%2020240417162958.png)

for this to work, remote access must be allowed on the target windows system  
it is not enabled by default   

note that RDP also allows us to save connection profiles, which is a common habit among IT admins because it makes connections to remote systems easier   
we can benefit from this by looking for saved `Remote Desktop Files` (.rdp) 

there are other tools like Remote Desktop clients 

from a linux based attack host we can use a tool called `xfreerdp` to remotely access windows targets: 

```shell
xfreerdp /v:<targetIp> /u:htb-student /p:Password
```

![](Images/Pasted%20image%2020240417163245.png)

there are other tools like remmina and rdesktop

## Operating Systems Structure 

in windows the root directory is `<drive letter>:\` and it is commonly in the `C` drive   
also known as the boot partition, and is where the OS is installed 

there are other physical and virtual drives that are assigned other letters like `Data (E:)` 

the directory structure of the boot partition is as follows: 

![](Images/Pasted%20image%2020240417171500.png)

we can explore directories using the `dir` command: 

![](Images/Pasted%20image%2020240417172820.png)

![](Images/Pasted%20image%2020240417172921.png)

the `tree` utility will graphically display the directory structure of a path or disk: 

![](Images/Pasted%20image%2020240417173912.png)

you can alternatively view all of this info one page at a time with: 

```
tree c:\ /f | more
```

can use dir and cat to find flag file: 

![](Images/Pasted%20image%2020240417174154.png)

## File System 

there are 5 types of windows file systems:   
- FAT12 
- FAT16
- FAT32
- NTFS 
- exFAT 

FAT 12 and 16 are no longer used on modern windows OS   

FAT32 (file allocation table) is widely used across many types of storage devices like USB and SD cards but can also be used to format hard drives   
32 = 32 bits of data for identifying data clusters on a storage device 

pros of FAT32:   
- device compatibility - pcs, digital cameras, gaming consoles, smartphones, tablets, etc. 
- OS cross-compatibility - works on all windows OS from 95 and is also supported by mac and linux 

cons of FAT32:  
- can only be used with files that are less than 4gb 
- no build in data protection or file compression 
- must use 3rd part tools for encryption 

NTFS (new technology file system) is the default windows file system since windows NT 3.1   
makes up for FAT32 shortcomings and has better support for metadata and better performance 

pros of NTFS:   
- reliable and can restore consistency of the file system in the even of system failure or power loss 
- provides security by setting granular permissions on both files and folders 
- supports very large-sized partitions 
- has journaling built in meaning that file modifications are logged 

cons of NTFS: 
- most mobile devices do not support NTFS natively 
- older media devices like TVs and digital cameras do not offer support for NTFS storage devices 

some of the key permission types of NTFS are: 

![](Images/Pasted%20image%2020240417180620.png)

files and folders inherit the NTFS permissions of their parent folder for ease of admin   
if permissions dont need to be set explicitly, an admin can disable permissions inheritance 

### Integrity control access control list (icacls)

NTFS permissions on files and folders can be managed through the file explorer GUI under the security tab (properties)
for a finer level of granularity over NTFS we can use the `icacls` utility 

we can list out the NTFS permissions of a directory by running `icacls` from within the working directory or `icacls C:\Windows` for a directory we are not in: 

![](Images/Pasted%20image%2020240417182121.png)

the resource access level is listed after each user in the output  
possible inheritance settings are: 
- `(CI)`: container inherit 
- `(OI)`: object inherit 
- `(IO)`: inherit only 
- `(NP)`: do not propagate inherit 
- `(I)`: permission inherited from parent container 

basic access permissions are:   
- `F`: full access
- `D`: delete access
- `N`: no access
- `M`: modify access
- `RX`: read and execute access
- `R`: read-only access
- `W`: write-only access 

we can add and remove permissions via the command line with `icacls`   

here we are executing `icacls` in the context of a local admin account: 

![](Images/Pasted%20image%2020240417182626.png)

our example user "joe" doesn't have any permissions 

using `icacls c:\users /grant joe:f` we can grant joe full control over the directory, but since `(oi)` and `(ci)` were not included the user joe will only have rights over the `c:\users` folder but not over the user subdirectories and files contained within them: 

![](Images/Pasted%20image%2020240417182826.png)

these permissions could be revoked using `icacls c:\users /remove joe` 

`icacls` is very powerful and can give users specific permissions over a file or folder, explicitly deny access, enable or disable inheritance permissions, and change directory/file ownership 

we can see that the user "bob.smith" has full access control over the users directory: 

![](Images/Pasted%20image%2020240417183047.png)

## NTFS vs. Share Permissions

windows is around 70% of global market share of desktop OS, so malware authors choose this to write exploits for   

many variants of malware for windows can be spread over the network via network shares with lenient permissions applied   

server message block SMB is used in windows to connect shared resources like files and printers: 

![](Images/Pasted%20image%2020240420131503.png)

note that NTFS permissions and share permissions are not the same 

share permissions: 
- full control = all actions given by Change and Read permissions as well as change permissions for NTFS files and subfolders 
- Change = read, edit, delete and add files and subfolders 
- Read = view file and subfolder contents 

NTFS basic permissions: 
- full control = add, edit, move, delete files and folders and change NTFS permissions that apply to all allowed folders 
- modify = permitted or denied permissions to view and modify files and folders, includes adding or deleting files 
- read and execute = permitted or denied permissions to read the contents of files and execute programs 
- list folder contents = permitted or denied permissions to view a  listing of files and subfolders
- read = permitted or denied permissions to read contents of files 
- write = permitted or denied permissions to write changes to a file and add new files to a folder
- special permissions = variety of advanced permissions options 

NTFS special permissions: 
- full control 
- traverse folder / execute file = access subfolder within directory even if user is denied access to contents at parent folder level; also execute programs 
- list folder/read data = view files and folders contained in the parent folder; also open and view files 
- read attributes = view basic attributes of a file or folder (system, archive, read-only, hidden)
- read extended attributes = extended attributes of file or folder 
- create files/write data = create files within a folder and make changes to a file 
- create folder/append data = create subfolders within a folder, data can be added to files but pre-existing content cant be overwritten 
- write attributes = change file attributes, does not grant access to creating files or folders 
- write extended attributes = change extended attributes of file or folder 
- delete subfolders and files = delete subfolders and files, parent folders will not be deleted
- delete = delete parent folders, subfolders, or files 
- read permissions = read permissions of a folder
- change permissions = change permissions of a file or folder
- take ownership = take ownership of a file or folder, owner of a file has full permissions to change any permissions 

NTFS permissions apply to the system where the folder and files are hosted  
folders created in NTFS inherit permissions from parent folders by default, but it is possible to set custom permissions on parent and subfolders   

the share permissions apply when the folder is being accessed through SMB, typically from a different system over the network   
this means that someone logged in locally or via RDP can access the shared folder and files by navigating to them and only need to consider NTFS permissions 

### Creating a network share 

keep in mind that in most orgs, shares are created on a storage area network SAN, network attached storage NAS, or a separate partition on drives accessed via a server OS like windows server 

if we ever see shares on a desktop OS it will probably be a small business or it could be a beachhead system used by a pen tester or malicious attacker to gather and exfiltrate data 

create a new folder: 

![](Images/Pasted%20image%2020240420141751.png)

enable advanced sharing: 

![](Images/Pasted%20image%2020240420141827.png)

![](Images/Pasted%20image%2020240420141849.png)

note that the share name defaults to the name of the folder   
we can also limit the number of users connected to that share at the same time 

similar to NTFS permissions there is an ACL for shared resources   
with shared resources, both SMB and NTFS permissions apply to every resource that gets shared in windows   
ACEs are made up of users and groups (also called security principles) 

we can see the current ACE and permissions settings: 

![](Images/Pasted%20image%2020240420150315.png)

we can test the permissions by using `smbclient` where our pwnbox is our client and the windows 10 target is our server: 

```
smbclient -L SERVER_IP -U htb-student
```

![](Images/Pasted%20image%2020240420150557.png)

```shell
smbclient '\\SERVER_IP\Company Data' -U htb-student
```

![](Images/Pasted%20image%2020240420150617.png)

even though we have all our entries correct and our permissions list has Everyone group with at least read permissions, what could stop us from accessing the share? 

### Windows defender firewall configuration 

the firewall could potentially block us from accessing the SMB share   
in our example we are connecting from a linux system that is not joined to the same `workgroup`   

when a windows system is part of a workgroup, all `netlogon` requests are authenticated against that system's `SAM` database   
when a windows system is joined to a windows domain environment, all netlogon requests are authenticated through active directory   

windows defender firewall profiles: 
- public 
- private
- domain 

firewall rules on desktop systems can be centrally managed when joined to a windows domain environment through the use of Group Policy   

with the proper inbound firewall rules are enabled we can connect to the share  
we can only connect because our user account `htb-student` is in the Everyone group   
once a connection is set with a share, we can create a mount point from our pwnbox to the windows 10 target box's file system 

keep in mind that the NTFS permissions apply to the share: 

![](Images/Pasted%20image%2020240420154731.png)

gray checkmarks mean they are inherited from the parent folder   
by default all NTFS permissions are inherited from the parent directory which is often the `C:\` drive unless an admin has disabled inheritance inside a newly created folder's advanced security settings 

lets give the Everyone group Full Control at the share level and testing a mount point: 

```shell-session
sudo mount -t cifs -o username=htb-student,password=Academy_WinFun! //ipaddoftarget/"Company Data" /home/user/Desktop/
```

```shell-session
sudo apt-get install cifs-utils
```

the `net share` command allows us to view all the shared folders on the system: 

![](Images/Pasted%20image%2020240420155935.png)

Computer Management is another tool we can use to identify and monitor shared resources on a windows system: 

![](Images/Pasted%20image%2020240420160108.png)

this is a good place to check when a breach related to SMB has occurred 

Event Viewer can alo be used to view share access logs: 

![](Images/Pasted%20image%2020240420160254.png)

## Windows Services & Processes

services allow for the creation and management of long-running processes   
they can be started auto at system boot without user intervention   
they can continue to run in the background even after the user logs out of their account  

applications can also be created to install a service like a network monitoring app installed on the server  

services are managed via the service control manager SCM system, accessible via the `services.msc` MMC add-in   
this add-in provides a GUI interface for interacting with and managing services 

can query and manage services via the command line using `sc.exe` using powershell cmdlets like `Get-Service`: 

`Get-Service | ? {$_.Status -eq "Running"} | select -First 2 |fl`

![](Images/Pasted%20image%2020240424154437.png)

service statuses can be running, stopped, or paused and they can be start manually, auto, or on a delay at system boot   
they can also be shown in the state of starting or stopping if an action has triggered them 

three categories of services: 
- local services
- network services
- system services 

windows has some critical system services that can't be stopped or restarted without a system restart   
if we update any file or resource in use by one of these resources then we need to restart: 

![](Images/Pasted%20image%2020240424154637.png)

### Processes

processes run in the background on windows systems   
they either run auto or are started by other installed apps 

some processes like those from apps can be terminated with little affect on the system but there are also critical processes that if stopped will stop certain components of the OS from running correctly   

some examples of critical processes are: 
- windows logon application 
- system 
- system idle process
- windows start up application 
- client server runtime 
- windows session manager 
- service host 
- local security authority subsystem services LSASS

### Local security authority subsystem service (LSASS)

`lsass.exe` is the process for enforcing the security policy on windows systems   
when a user attempts to logon, lsass will verify their log on and create access tokens based on the user's permission levels   
also responsible for password changes   

all events associated with this process are logged in the Windows security log   

several tools exist to extract both cleartext and hashed credentials stored in memory by this process 

### Sysinternals tools

the sysinternals tools suite is a set of portable windows apps that can be used to admin windows systems usually without installation   

can either be downloaded from microsoft or by loading them directly from the internet file share with: 

`\\live.sysinternals.com\tools`

![](Images/Pasted%20image%2020240424155652.png)

has tools like process explorer, and enhanced task manager, and process monitor   

these are good tools for pen tests because they can discover processes and possible privilege escalation paths for lateral movement 

### Task manager 

provides info about running processes, system performance, running services, startup programs, logged-in users or logged in user processes, and services   

ways to open: 
- right click taskbar
- ctrl + shift + esc 
- ctrl + alt + delete 
- start menu
- `taskmgr` from cmd or powershell 

![](Images/Pasted%20image%2020240424160015.png)

### Process explorer 

part of the sysinternals suite   
can show which handles and DLL processes are loaded when a program runs   
shows list of currently running processes, and from there we can see what handles the process has selected or the DLLs and memory swapped files that have been loaded   

can also analyze parent-child process relationships which can help troubleshoot issues like an orphaned child from a process that has been terminated

## Service Permissions

services allow for the management of long-running processes and are a critical part of windows OS   
these are often potential threat vectors for things like loading malicious DLLs, execute apps unauthenticated, escalate privileges and even maintain persistence  

first need to understand importance of service permissions and that they exist or be mindful of them  
critical services like DHCP and AD commonly get installed using the admin account   
these services by default are configured to run with the privileges of the user who is currently logged on   
it is highly recommended to create an individual user account to run critical network services, or service accounts 

### Examining services using services.msc 

we can use the Services app to view the service associated with windows update: 

![](Images/Pasted%20image%2020240423162623.png)

the path to the executable is the full path to the program and the command to execute when the service starts  
if the NTFS permissions of the destination directory are configured with weak permissions then an attacker could replace the original executable with one created for malicious processes   

most services run with LocalSystem privileges by default which is the highest level of access allowed on an individual windows OS: 

![](Images/Pasted%20image%2020240423162830.png)

not all apps need local system permissions so it is good to perform research on a case-by-case basis when installing new apps in an windows environment 

notable built-in service accounts in windows:   
- LocalService
- NetworkService
- LocalSystem

we can also create new accounts and use them for the sole purpose of running a service 

the recovery tab shows steps should a service fail: 

![](Images/Pasted%20image%2020240423163020.png)

you can also set a program to run after the first failure, and this is another vector that could be used to run a malicious program 

### Examining services using sc 

sc can also be used to configure and mange services: 

`sc qc wuauserv`

![](Images/Pasted%20image%2020240423163646.png)

`sc qc` is used to query the service and knowing the name of the service comes in handy   

if we wanted to query a service used over the network we could specify the hostname or IP address: 

`sc <hostname or IP>`

we can also use this to start and stop services: 

`sc stop wuauserv`

however we might be denied access to start/stop services but we can run command prompt with elevated privileges: 

`sc config wuauserv binPath=C:\<malicious program path>`

![](Images/Pasted%20image%2020240423164027.png)

if we were investigating a situation with potential malware, `sc` would give us the ability to quickly search and analyze commonly targeted services and newly created services   

we can also examine service permissions with `sdshow`: 

`sc sdshow wuaserv`

![](Images/Pasted%20image%2020240423164210.png)

every named object in windows is a securable object, and even some unnamed objects are securable   
if its securable it will have a security descriptor which identify the object's owner and primary group containing a discretionary access control list DACL and system access control list SACL  

generally a DACL is used to control access to an object and SACL is used to account for and log access attempts   

```
D:(A;;CCLCSWRPLORC;;;AU)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;SY)
```

the above is a format known as security descriptor definition language SDDL   
it isn't very helpful to read from left to right, and instead can be read in this order: 

`D: (A;;CCLCSWRPLORC;;;AU)`

1. `D:` - proceeding characters are DACL permissions
2. `AU:` - security principle authenticated users
3. `A;;` - access is allowed 
4. `CC` - SERVICE_QUERY_CONFIG is the full name, and is a query to the service control manager SCM for the service configuration 
5. `LC` - SERVICE_QUERY_STATUS is the full name, and is a query to the service control manager SCM for the current status of the service 
6. `SW` - SERVICE_ENUMERATE_DEPENDENTS which enumerates a list of dependent services
7. `RP` - SERVICE_START will start the service 
8. `LO` - SERVICE_INTERROGATE will query the service for its current status 
9. `RC` - READ_CONTROL will query the security descriptor of the service 

each set of 2 characters between semi-colons is an action allowed to be performed by a specific user or group   

then the characters after the last set of semi-colons specify the security principal (user and or group) that is permitted to perform these actions: 

`;;;AU`

then after the opening parenthesis and before the first set of semicolons defines whether the actions are allowed or denied: 

`A;;`

in this example, the entire security descriptor for the windows update service has three sets of access control entries because there are three different security principles   

### Examine service permissions using powershell

`Get-Acl` will examine service permissions by targeting the path of a service in the registry: 

`Get-ACL -Path HKLM:\System\CurrentControlSet\Services\wuauserv | Format-List`

![](Images/Pasted%20image%2020240423165419.png)

this will output specific account permissions in easy to read format and in an SDDL   
the SID for each security principle is present in the SDDL   

## Windows Sessions 

### Interactive 

an interactive or local logon session is when a user authenticates to a local or domain system by entering their credentials   
can be initiated by logging on directly into the system, requesting a secondary logon session using `runas` command, or with RDP 

### Non-interactive 

non-interactive accounts do not require login credentials 

3 types of non-interactive accounts: 
- local system account
- local service account 
- network service account

generally used by windows OS to auto start services and apps without user interaction   
no password associated to them and are used to start services when system boots or to run scheduled tasks 

local system account - `NT AUTHORITY\SYSTEM` account; most powerful account in windows systems. Used for variety of OS tasks like starting services. More powerful than accounts in the local admin group 

local service account - `NT AUTHORITY/LocalService`, less privileged version of the SYSTEM account and has similar privileges to a local user account. Granted limited functionality and can start some services 

network service account - `NT AUTHORITY\NetworkService`, similar to standard domain user account. Has similar privileges to local service account on the local machine. Can establish authenticated sessions for certain network services 

## Interacting with the Windows Operating System 

### GUI

sys admins can use GUI for admin active directory, config IIS, or interacting with databases 

### RDP 

proprietary microsoft protocol that allows users to connect to a remote system over a network connection and use a GUI   

connect using RDP client software to a target system running RDP server software   

uses port 3389

can also be used by users connecting to work pc while traveling after connecting to a VPN 

### Windows command line 

used for automation 

cmd and powershell 

### CMD

enter and execute commands 

start menu, `cmd` in run dialogue box, or `C:\Windows\system32\cmd.exe`

![](Images/Pasted%20image%2020240429174727.png)

can do `help <command name>`: 

![](Images/Pasted%20image%2020240429174809.png)

some commands have their own help menus which can be used with `<command> /?` like: 

![](Images/Pasted%20image%2020240429174920.png)

### Powershell 

command shell more for admin 

very powerful for interacting with the OS 

gives us direct access to the file system and we can run most of the same commands that we can run within a cmd shell 

### Cmdlets 

powershell uses `cmdlets` which are small single-function tools built into the shell   
more than 100 cmdlets and many others that have been written and we can also make our own 

in the form of `Verb-Noun`   

`Get-ChildItem` can be used to list current directory: 

![](Images/Pasted%20image%2020240429175203.png)

if we start an argument with `-` like `Get-ChildItem -` we can use tab to cycle through all the available arguments   
for example we can use `-Recurse` to show all contents of subdirectories as well, or we can use `-Path` and provide a path to get the contents of that folder   
we could also combine those two 

### Aliases 

many cmdlets have aliases   
for example aliases for the cmdlet `Set-Location` to change directories is either `cd` or `sl`   
`Get-ChildItem` is `ls` and `gci`   

we can view all available aliases with `Get-Alias`: 

![](Images/Pasted%20image%2020240429175518.png)

we can also set up our own aliases with `New-Alias` and get the alias for any cmdlet with `Get-Alias -Name`: 

`New-Alias -Name "Show-Files" Get-ChildItem` 

`Get-Alias -Name "Show-Files"`

powershell also has a help system for cmdlets, functions, scripts, and concepts; not installed by default  
we can either run `Get-Help <cmdlet-name> -Online` to open the online help in web browser, or we can use `Update-Help` to download and install help files locally 

using a command like `Get-Help Get-AppPackage` will return the partial help unless the help files are installed 

### Running scripts 

the powershell ISE (integrated scripting environment) allows users to write scripts 

we can run in a variety of ways  
if we know the functions we can run the script locally or after loading into memory with a download cradle like: 

`.\PowerView.ps1;Get-LocalGroup |fl`

![](Images/Pasted%20image%2020240429180646.png)

common way to work with a script is to import it so that all functions are available with: 

`Import-Module .\PowerView.ps1`  

we can then either start a command and cycle through the options or use `Get-Module` to list all loaded modules and their associated commands: 

`Get-Module | select Name,ExportedCommands | fl` 

### Execution policy 

execution policy will prevent the execution of malicious scripts 

the possible policies are: 

![](Images/Pasted%20image%2020240429181719.png)

can view the execution policy with `Get-ExecutionPolicy -List`: 

![](Images/Pasted%20image%2020240429182017.png)

the execution policy is not meant to be a security control as it can be easily bypassed by either writing the commands of a script one by one or by changing the execution policy if the user can do so   

we can change the execution policy with `Set-ExecutionPolicy Bypass -Scope Process`

## Windows Management Instrumentation (WMI)

WMI is a subsystem of powershell that gives admin powerful tools for system monitoring   
goal is to consolidate device and app management across corporate networks     
core part of OS 

made up of these components: 

![](Images/Pasted%20image%2020240429182747.png)

some of the uses for WMI are: 
- status info for local/remote systems 
- config security settings on remote machines/apps 
- setting and changing user and group permissions 
- setting/modifying system properties
- code execution 
- scheduling processes
- setting up logging

wmi can be run be run with windows command prompt with `WMIC` to open interactive shell or by using a command like `wmic computersystem get name` to get the hostname   
can also get a list of wmic commands with `WMIC /?` 

`wmic os list brief` will list info about the OS: 

![](Images/Pasted%20image%2020240429184000.png)

`list` will show data, `brief` will provide just the core set of properties 

wmi can be used with powershell using `Get-WmiObject`   
used to get instances of WMI classes or info about available classes   

get info about the OS with: 

`Get-WmiObject -Class Win32_OperatingSystem | select SystemDirectory,BuildNumber,SerialNumber,Version | ft`

we can also use the `Invoke-WmiMethod` module which calls the methods of WMI objects   

can rename a file with: 

`Invoke-WmiMethods -Path "CIM_DataFile.Name='C:\users\public\spns.csv'" -Name Rename -ArgumentList "C:\Users\Public\kerberoasted_users.csv"` 

