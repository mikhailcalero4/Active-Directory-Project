# Phase 1: Active Directory Foundation

Part of a multi-phase home lab project for IAM (Identity and Access Management) skill-building. This phase establishes the core Active Directory environment that later phases build on (RBAC/Group Policy, MFA/federated identity, and centralized logging/SIEM).

## Objective

Stand up a working Active Directory domain in an isolated virtual lab, including a Domain Controller, a domain-joined client, an Organizational Unit (OU) structure, and initial users/groups mapped to job functions.

## Lab Architecture

- **Hypervisor:** VirtualBox
- **Domain Controller (DC01):** Windows Server 2022 (Desktop Experience)
  - Static IP: `192.168.56.10`
  - Roles: AD DS, DNS
- **Client (CLIENT01):** Windows 10/11
  - Static IP: `192.168.56.20`
  - Domain-joined to `lab.local`
- **Network:** VirtualBox Host-Only Adapter (isolated from the internet by design)
- **Domain:** `lab.local` / NetBIOS `LAB`

## Organizational Unit Structure

Four OUs were created under the domain root, each representing a department:

| OU | Purpose | Example user | Security group |
|---|---|---|---|
| IT | IT administrators | jdoe | IT-Admins |
| HR | HR staff | msmith | HR-Staff |
| Finance | Finance staff | bwilson | Finance-Staff |
| Helpdesk | Tier 1 support | tlee | Helpdesk-T1 |

This structure mirrors how real organizations delegate administrative control and apply Group Policy at a departmental level rather than to the whole domain at once — a foundational IAM concept covered further in Phase 2.

## Build Steps

### 1. VM provisioning
Created `DC01` (3 GB RAM, 2 vCPU, 60 GB disk) and `CLIENT01` (2 GB RAM, 2 vCPU, 50 GB disk) in VirtualBox, both attached to a Host-Only network to keep the lab isolated.

### 2. Static IP and DNS configuration
Active Directory depends entirely on DNS for domain functions (locating domain controllers, SRV records, etc.), so the DC was configured to use itself as its DNS server before promotion:

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.56.10 -PrefixLength 24 -DefaultGateway 192.168.56.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 127.0.0.1
```

### 3. Promotion to Domain Controller
Installed the AD DS role and promoted the server to create a new forest:

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Install-ADDSForest -DomainName "lab.local" -DomainNetBiosName "LAB" -InstallDns:$true -SafeModeAdministratorPassword (ConvertTo-SecureString "potentialpassword`" -AsPlainText -Force) -Force:$true
```

### 4. OU structure

```powershell
$base = "DC=lab,DC=local"

New-ADOrganizationalUnit -Name "IT"       -Path $base
New-ADOrganizationalUnit -Name "HR"       -Path $base
New-ADOrganizationalUnit -Name "Finance"  -Path $base
New-ADOrganizationalUnit -Name "Helpdesk" -Path $base
```

### 5. Security groups and users

```powershell
New-ADGroup -Name "IT-Admins"     -GroupScope Global -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "HR-Staff"      -GroupScope Global -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Finance-Staff" -GroupScope Global -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "Helpdesk-T1"   -GroupScope Global -Path "OU=Helpdesk,DC=lab,DC=local"

$pw = (ConvertTo-SecureString "potential password" -AsPlainText -Force)

New-ADUser -Name "your-name"   -SamAccountName your-name    -UserPrincipalName your-name@lab.local    -Path "OU=IT,DC=lab,DC=local"       -AccountPassword $pw -Enabled $true
New-ADUser -Name "your-name" -SamAccountName your-name -UserPrincipalName your-name@lab.local  -Path "OU=HR,DC=lab,DC=local"       -AccountPassword $pw -Enabled $true
New-ADUser -Name "your-name" -SamAccountName your-name -UserPrincipalName your-name@lab.local -Path "OU=Finance,DC=lab,DC=local"  -AccountPassword $pw -Enabled $true
New-ADUser -Name "your-name"    -SamAccountName your-name   -UserPrincipalName your-name@lab.local    -Path "OU=Helpdesk,DC=lab,DC=local" -AccountPassword $pw -Enabled $true

Add-ADGroupMember -Identity "IT-Admins"     -Members your-name
Add-ADGroupMember -Identity "HR-Staff"      -Members your-name
Add-ADGroupMember -Identity "Finance-Staff" -Members your-name
Add-ADGroupMember -Identity "Helpdesk-T1"   -Members your-name
```

### 6. Domain-joining the client

```powershell
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 192.168.56.20 -PrefixLength 24 -DefaultGateway 192.168.56.1
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 192.168.56.10

Add-Computer -DomainName "lab.local" -Credential (Get-Credential) -Restart
```

### 7. Validation

```powershell
Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName
Get-ADUser -Filter * -Properties MemberOf | Select-Object Name, SamAccountName
Get-ADGroup -Filter * | Select-Object Name
```

On the client, confirmed domain membership via `whoami`, which correctly returned `lab\<username>` after logging in with a domain account.

## Troubleshooting Log

Documenting real issues encountered (and how they were resolved) is part of the value of this lab — these are mistakes any AD administrator will eventually hit in production too.

**Issue: `New-ADOrganizationalUnit : The object name has bad syntax`**

Root cause: a typo in the `$base` variable assignment — `DC-local` (hyphen) was typed instead of `DC=local` (equals sign). Active Directory distinguished names use `DC=` as the attribute/value pair for each domain component, so a hyphen breaks LDAP path parsing entirely.

Diagnosis approach: rather than re-running the failing command repeatedly, isolated the problem by printing the `$base` variable back out (`$base` on its own line) to confirm its actual stored value before using it in any further commands. This revealed the typo immediately.

Fix: re-ran the assignment with correct syntax (`$base = "DC=lab,DC=local"`) and verified output before retrying the OU creation command.

**Issue: `An attempt was made to add an object to the directory with a name that is already in use`**

Root cause: an earlier (corrected) run of the OU creation command had already succeeded for one or more OUs before this error was hit on a retry.

Diagnosis approach: ran `Get-ADOrganizationalUnit -Filter * | Select-Object Name, DistinguishedName` to get ground truth on what already existed in the directory, rather than assuming based on console scrollback.

Fix: skipped re-creating OUs that already existed and only ran the creation command for the remaining ones.

**Lesson learned:** When working with AD via PowerShell, verify object state with `Get-AD*` cmdlets before and after changes rather than trusting assumptions about what succeeded. Idempotent verification is a habit worth carrying into every later phase of this lab (and into real IAM administration work).

## Key IAM Concepts Demonstrated

- **Identity foundation:** Domain Controller as the centralized identity store (AD DS)
- **Namespace/resolution:** Why AD-integrated DNS is a hard dependency, not optional
- **Logical segmentation:** OUs as administrative/policy boundaries vs. security groups as permission boundaries (these are commonly confused — OUs control *where GPOs apply*, groups control *what access is granted*)
- **Least-privilege groundwork:** Department-based group structure that Phase 2 will map to actual resource permissions

## Next Phase

[Phase 2: RBAC and Group Policy](./PHASE2-RBAC-Group-Policy.md) — applying least-privilege access control and Group Policy enforcement on top of this foundation.
