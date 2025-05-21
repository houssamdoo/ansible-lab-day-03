# winrm setup
Before Ansible can connect using WinRM, the Windows host must have a WinRM listener configured. This listener will listen on the configured port and accept incoming WinRM requests.  

run the following PowerShell snippet to setup the HTTP listener with the defaults:
```powershell
# Enables the WinRM service and sets up the HTTP listener
Enable-PSRemoting -Force

# Opens port 5985 for all profiles
$firewallParams = @{
    Action      = 'Allow'
    Description = 'Inbound rule for Windows Remote Management via WS-Management. [TCP 5985]'
    Direction   = 'Inbound'
    DisplayName = 'Windows Remote Management (HTTP-In)'
    LocalPort   = 5985
    Profile     = 'Any'
    Protocol    = 'TCP'
}
New-NetFirewallRule @firewallParams

# Allows local user accounts to be used with WinRM
# This can be ignored if using domain accounts
$tokenFilterParams = @{
    Path         = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System'
    Name         = 'LocalAccountTokenFilterPolicy'
    Value        = 1
    PropertyType = 'DWORD'
    Force        = $true
}
New-ItemProperty @tokenFilterParams
```
To also add a HTTPS listener with a self signed certificate we can run the following:
```powershell
# Create self signed certificate
$certParams = @{
    CertStoreLocation = 'Cert:\LocalMachine\My'
    DnsName           = $env:COMPUTERNAME
    NotAfter          = (Get-Date).AddYears(1)
    Provider          = 'Microsoft Software Key Storage Provider'
    Subject           = "CN=$env:COMPUTERNAME"
}
$cert = New-SelfSignedCertificate @certParams

# Create HTTPS listener
$httpsParams = @{
    ResourceURI = 'winrm/config/listener'
    SelectorSet = @{
        Transport = "HTTPS"
        Address   = "*"
    }
    ValueSet = @{
        CertificateThumbprint = $cert.Thumbprint
        Enabled               = $true
    }
}
New-WSManInstance @httpsParams

# Opens port 5986 for all profiles
$firewallParams = @{
    Action      = 'Allow'
    Description = 'Inbound rule for Windows Remote Management via WS-Management. [TCP 5986]'
    Direction   = 'Inbound'
    DisplayName = 'Windows Remote Management (HTTPS-In)'
    LocalPort   = 5986
    Profile     = 'Any'
    Protocol    = 'TCP'
}
New-NetFirewallRule @firewallParams
```

## Enumerate Listeners
To view the current listeners that are running on the WinRM service:
```powershell
winrm enumerate winrm/config/Listener
```

## Setup User on Windows
```powershell
# Create the user 'ansible' with password 'Solution.2025'
$Password = ConvertTo-SecureString "Solution.2025" -AsPlainText -Force
New-LocalUser -Name "ansible" -Password $Password -FullName "Ansible User" -Description "Ansible Automation Account" -ErrorAction SilentlyContinue

# Add the user to the Administrators group
Add-LocalGroupMember -Group "Administrators" -Member "ansible" -ErrorAction SilentlyContinue

# Verify the user and group membership
Get-LocalUser -Name "ansible"
Get-LocalGroupMember -Group "Administrators" | Where-Object Name -like "*ansible*"
```
## Verify PowerShell, .NET and set up WinRM
### 1. verify PowerShell version
```powershell
Get-Host | Select-Object Version
```
### 2. verify .NET version

```powershell
Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse | Get-ItemProperty -Name version -EA 0 | Where { $_.PSChildName -Match '^(?!S)\p{L}'} | Select PSChildName, version
```

### 3. Verify WinRM not-configured
```powershell
winrm get winrm/config/Service
```

### 4. Setup windRM
```powershell
PS C:\Users\vagrant> [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
>> $url = "https://raw.githubusercontent.com/ansible/ansible/devel/examples/scripts/ConfigureRemotingForAnsible.ps1"
>> $file = "$env:temp\ConfigureRemotingForAnsible.ps1"
>>
>> (New-Object -TypeName System.Net.WebClient).DownloadFile($url, $file)
>>
>> powershell.exe -ExecutionPolicy ByPass -File $file
PS C:\Users\vagrant>
```
---

# Ansible CONTROLLER
`inventory` 
```bash
[windows]
windows10 ansible_host=15.15.15.13
[windows:vars]
ansible_user=ansible
ansible_password=Solution.2025
ansible_port=5985
ansible_connection=winrm
ansible_winrm_transport=basic
ansible_winrm_server_cert_validation=ignore
```

## Configure Ansible Controller
Install pywinrm:
```bash
pip install pywinrm
```

## install collection on Controller
```bash
ansible-galaxy collection install ansible.windows
```

## upgrade your ansible
```bash
sudo dnf list ansible-core --showduplicates
sudo dnf upgrade ansible-core
sudo dnf install python3-pip
sudo python3 -m pip install --upgrade ansible
```
## Basic authentication is not enabled by default on a Windows host but can be enabled by running the following in PowerShell:
```powershell
Set-Item -Path WSMan:\localhost\Service\Auth\Basic -Value $true

# 1. Check authentication settings
winrm get winrm/config/service/auth

# Should show:
# Basic = true
# Kerberos = true
# Negotiate = true

# 2. Enable Basic auth if needed
winrm set winrm/config/service/auth '@{Basic="true"}'

# 3. Allow unencrypted traffic
winrm set winrm/config/service '@{AllowUnencrypted="true"}'

# 4. Verify listeners
winrm enumerate winrm/config/listener

# Should show HTTP listener on 5985


```

