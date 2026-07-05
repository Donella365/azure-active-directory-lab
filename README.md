# 01 Active Directory & Identity Management Lab

---
## [▶️ Lab Walkthrough Video](https://www.loom.com/share/324f9012625e45d0af3748a1a2f56ef0)

## What this lab covers

Active Directory is the identity backbone of every Windows enterprise — it controls who can log in, what they can access, and what rules apply to their machine. In this lab I deployed Windows Server 2025 in Azure and built a functional Active Directory domain.

In this lab I:

- Promoted a Windows Server 2025 VM to a Domain Controller and created a new Active Directory forest (`lab.local`)
- Designed and built an Organisational Unit structure reflecting a real enterprise department layout
- Created security groups and user accounts with role-based access control
- Deployed and configured a Group Policy Object enforcing password complexity, screen lock, and USB restrictions
- Performed core help desk operations — password resets, account unlocks, offboarding, and audit queries

---

## Architecture 

```
┌─────────────────────────────────────────────────────────┐
│                   lab.local (Forest)                    │
│                                                         │
│         ┌──────────────────────────────┐                │
│         │      Domain Controller       │                │
│         │  Windows Server 2025 · DNS   │                │
│         └───────────────┬──────────────┘                │
│                         │                               │
│        ┌────────────────┼──────────────────┐            │
│        ▼                ▼                  ▼            │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐       │
│  │  OU=IT   │    │OU=Finance│    │   OU=HR      │       │
│  │alice.chen│    │bob.patel │    │ carol.jones  │       │
│  │IT_Admins │    │Finance_  │    │  HR_Users    │       │
│  │  (group) │    │  Users   │    │   (group)    │       │
│  └────┬─────┘    └──────────┘    └──────────────┘       │
│       │                                                  │
│  ┌────▼──────────────────┐   ┌───────────────────┐      │
│  │  IT Security Policy   │   │    OU=Computers    │      │
│  │  (GPO linked to IT)   │   │  Domain-joined VMs │      │
│  │  · 12-char passwords  │   └───────────────────┘      │
│  │  · 15-min screen lock │                              │
│  │  · USB block          │                              │
│  └───────────────────────┘                              │
└─────────────────────────────────────────────────────────┘
```

**Domain:** `lab.local`
**Forest root DC:** Windows Server 2025 Datacenter
**Deployment:** Azure (Standard_B2s)

---

## Prerequisites

| Requirement | Details |
|---|---|
| Azure free account | $200 credit — [azure.microsoft.com/free](https://azure.microsoft.com/free) |
| RDP client | Native Remote Desktop app (not browser-based) |
| Time | 3–5 hours across multiple sessions |
| Cost | $0 — fully covered by free tier and evaluation licences |

---

## Lab Details

| Variable | Value |
|---|---|
| Domain Name | `lab.local` |
| VM Size | Standard_B2s (2 vCPU, 4GB RAM) |
| OS | Windows Server 2025 Datacenter Gen2 |
| Region | East US |
| OUs Created | IT · Finance · HR · Sales · Computers |
| Security Groups | IT_Admins · Finance_Users · HR_Users · Sales_Users |
| Test Users | alice.chen · bob.patel · carol.jones · david.smith |
| GPO Name | IT Security Policy |
| Portfolio Repo | azure-active-directory-lab |

---

## Lab walkthrough

### Step 1: Deploy the Azure VM

1. Go to [portal.azure.com](https://portal.azure.com) and create a new Virtual Machine
2. Use these settings:

| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter — Gen2 |
| Size | Standard_B2s (2 vCPU, 4GB RAM) |
| Authentication | Password |
| Inbound ports | RDP (3389) |
| OS disk | Standard SSD |

3. Download the `.rdp` file from the Azure portal and open it with the native Remote Desktop app
4. Before connecting: RDP client → Show Options → Local Resources → check **Clipboard**

> **Cost tip:** Stop the VM at the end of every session. A B2s runs ~$0.05/hr — stopping (not deleting) pauses compute billing.

**Expected result:**
Connected to the Windows Server 2025 desktop via RDP. Server Manager opens automatically on login.

---

### Step 2: Install Active Directory Domain Services

**Via PowerShell (run as Administrator):**

```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```

> Install GPMC in the same session — it's required for Phase 4 and won't appear in the Tools menu until installed.

**Expected result:**
AD DS and GPMC roles installed. No restart required yet.

---

### Step 3: Promote the Server to a Domain Controller

**Via PowerShell:**

```powershell
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName 'lab.local' `
  -DomainNetBiosName 'LAB' `
  -InstallDns:$true `
  -SafeModeAdministratorPassword (ConvertTo-SecureString 'YourDSRMPassword!' -AsPlainText -Force) `
  -Force:$true
```

**Via GUI:**
Server Manager → yellow warning flag → Promote this server to a domain controller → Add a new forest → Root domain name: `lab.local` → set DSRM password → Install

> The server restarts automatically after promotion. Log back in as `LAB\Administrator`.

**Expected result:**
Server has restarted and is now the Domain Controller for `lab.local`. DNS is running on the same machine.

---

### Step 4: Build the Organisational Structure

Open **Active Directory Users and Computers (ADUC)** from the Tools menu in Server Manager.

**Create Organisational Units:**

```powershell
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
```

**Create Security Groups:**

```powershell
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

**Create User Accounts and assign group membership:**

> Run this entire block together — not line by line. The `$password` variable must be defined before the `New-ADUser` commands run.

```powershell
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

New-ADUser -Name "alice.chen" -GivenName "Alice" -Surname "Chen" `
  -SamAccountName "alice.chen" -UserPrincipalName "alice.chen@lab.local" `
  -Path "OU=IT,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "bob.patel" -GivenName "Bob" -Surname "Patel" `
  -SamAccountName "bob.patel" -UserPrincipalName "bob.patel@lab.local" `
  -Path "OU=Finance,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "carol.jones" -GivenName "Carol" -Surname "Jones" `
  -SamAccountName "carol.jones" -UserPrincipalName "carol.jones@lab.local" `
  -Path "OU=HR,DC=lab,DC=local" -AccountPassword $password -Enabled $true

New-ADUser -Name "david.smith" -GivenName "David" -Surname "Smith" `
  -SamAccountName "david.smith" -UserPrincipalName "david.smith@lab.local" `
  -Path "OU=Sales,DC=lab,DC=local" -AccountPassword $password -Enabled $true

Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

**Expected result:**
Five OUs visible in ADUC. Four users created and placed in their respective OUs. Each user is a member of their department security group.

---

### Step 5: Configure Group Policy

Open **Group Policy Management** from the Tools menu in Server Manager.

1. Expand Forest: lab.local → Domains → lab.local
2. Right-click the **IT** OU → Create a GPO in this domain and link it here
3. Name it: `IT Security Policy`
4. Right-click → Edit and configure the following settings:

| Policy path | Setting | Value |
|---|---|---|
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Minimum password length | 12 |
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Password must meet complexity requirements | Enabled |
| Computer Config → Windows Settings → Security → Local Policies → Security Options | Interactive logon: Machine inactivity limit | 900 seconds |
| Computer Config → Administrative Templates → System → Removable Storage Access | All removable storage classes: Deny all access | Enabled |

**Expected result:**
IT Security Policy GPO is linked to the IT OU. Settings are enforced automatically for all users and computers inside that OU.

---

### Step 6: Help Desk Operations

**Reset a password:**

```powershell
Set-ADAccountPassword -Identity "bob.patel" -Reset `
  -NewPassword (ConvertTo-SecureString "NewPass@2026!" -AsPlainText -Force)
Set-ADUser -Identity "bob.patel" -ChangePasswordAtLogon $true
```

**Unlock a locked account:**

```powershell
Unlock-ADAccount -Identity "carol.jones"
```

**Disable an account (offboarding):**

```powershell
Disable-ADAccount -Identity "david.smith"
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
```

**Audit — accounts inactive for 90+ days:**

```powershell
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, LastLogonDate
```

**Check group membership:**

```powershell
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

**Expected result:**
Password reset, account unlock, disable, and audit queries all execute without errors and return expected output.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PowerShell prompts for `Name:` when creating users | The `$password` variable wasn't defined first. Copy and run the entire block, not individual lines. |
| Can't copy/paste into the VM | RDP client → Show Options → Local Resources → check Clipboard. Download the `.rdp` file from Azure portal and open with the native Remote Desktop app. |
| Promotion fails with DNS conflict | Set the NIC's preferred DNS to `127.0.0.1` before promoting. |
| Can't RDP after domain join | Log in as `LAB\Administrator` not just `Administrator`. |
| GPO not applying | Run `gpupdate /force` on the target machine, then `gpresult /r` to see applied policies. |
| User can't log in after creation | Confirm the account is Enabled and `ChangePasswordAtLogon` is not blocking the login. |
| AD Users and Computers not showing | Run `dsa.msc` from the Run dialog. |

---

## Verification

| Check | Command | Expected result |
|---|---|---|
| DC is running | `Get-ADDomainController` | Returns DC info including forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs |
| Users exist and are enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Lists all 4 test accounts |
| Group membership correct | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO is linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows `IT Security Policy` as linked |

---

## Clean Up

- Stop the Azure VM when not in use to pause compute billing
- Do not delete the VM if continuing to Lab 03 — the Windows Server VM is reused as a log forwarder for the Splunk SIEM lab

---

## Key Concepts

| Concept | What It Does |
|---|---|
| Domain Controller | The server that runs Active Directory and answers every authentication request on the network |
| Organisational Unit (OU) | A folder inside AD used to organise users and computers — GPOs linked to an OU apply automatically to everything inside it |
| Security Group | A container for user accounts — permissions are granted to groups, not individuals, enabling role-based access control at scale |
| Group Policy Object (GPO) | A collection of settings enforced automatically across every user and computer in a linked OU |
| DSRM Password | Directory Services Restore Mode password — only needed for disaster recovery, store it securely |
| RBAC | Role-Based Access Control — access follows the role (group membership), not the individual account |

---

## Tools & Services Used

Windows Server, Azure, Powershell, Active Directory

---


