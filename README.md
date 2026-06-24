# Lab 01 — Active Directory & Identity Management
**Windows Server 2025 · Azure Free Tier · lab.local domain**

> Building and managing an enterprise identity system from scratch — users, groups, OUs, GPOs, and the help desk tasks that come with them every day.

---

## What this lab covers

Active Directory is the identity backbone of every Windows enterprise environment. This lab walks through standing one up from a bare server, structuring it to reflect a real organisation, and performing the day-to-day operations that support and security teams actually deal with.

| Area | Skills demonstrated |
|---|---|
| Infrastructure | Deploying Windows Server 2025, promoting a Domain Controller, configuring DNS |
| Identity design | Organisational Units, security groups, role-based access control |
| User management | Account creation, group membership, password policies |
| Policy enforcement | Group Policy Objects, password complexity, screen lock, USB restrictions |
| Operations | Password resets, account unlocks, offboarding, audit queries |

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
│        ┌────────────────┼─────────────────┐             │
│        ▼                ▼                 ▼             │
│  ┌──────────┐    ┌──────────┐    ┌──────────────┐       │
│  │  OU=IT   │    │OU=Finance│    │   OU=HR      │       │
│  │alice.chen│    │bob.patel │    │ carol.jones  │       │
│  │IT_Admins │    │Finance_  │    │  HR_Users    │       │
│  │  (group) │    │  Users   │    │   (group)    │       │
│  └────┬─────┘    └──────────┘    └──────────────┘       │
│       │                                                  │
│  ┌────▼─────────────────┐   ┌───────────────────┐       │
│  │  IT Security Policy  │   │    OU=Computers    │       │
│  │  (GPO linked to IT)  │   │  Domain-joined VMs │       │
│  │  · 12-char passwords │   └───────────────────┘       │
│  │  · 15-min screen lock│                               │
│  │  · USB block         │                               │
│  └──────────────────────┘                               │
└─────────────────────────────────────────────────────────┘
```

**Domain:** `lab.local`  
**Forest root DC:** Windows Server 2025 Datacenter  
**Deployment target:** Azure (Standard_B2s) or VirtualBox (local)

---

## Prerequisites

| Requirement | Details |
|---|---|
| Azure free account | $200 credit, 30 days — [azure.microsoft.com/free](https://azure.microsoft.com/free) |
| OR local machine | 8GB+ RAM, 60GB disk, virtualisation enabled in BIOS |
| RDP client | Native Remote Desktop app (not browser-based) |
| Time | 3–5 hours across multiple sessions |
| Cost | $0 — fully covered by free tier and evaluation licences |

---

## Environment setup

### Option A — Azure (recommended)

No local hardware requirements. The VM runs in Azure and you connect via RDP.

1. Create a free account at [azure.microsoft.com/free](https://azure.microsoft.com/free)
2. Sign into [portal.azure.com](https://portal.azure.com)
3. Create a Virtual Machine with these settings:

| Setting | Value |
|---|---|
| Region | East US |
| Image | Windows Server 2025 Datacenter — Gen2 |
| Size | Standard_B2s (2 vCPU, 4GB RAM) |
| Authentication | Password |
| Inbound ports | RDP (3389) |
| OS disk | Standard SSD |

> **Cost tip:** Stop the VM at the end of every session. A B2s runs ~$0.05/hr — stopping (not deleting) pauses compute billing and stretches your free credits.

**Fix clipboard before connecting:**  
Open the Remote Desktop app → enter the VM's public IP → Show Options → Local Resources tab → check Clipboard. Download the `.rdp` file from the Azure portal rather than using the browser console for best results.

### Option B — VirtualBox (local)

1. Download [VirtualBox](https://virtualbox.org) (free)
2. Download the [Windows Server 2025 Evaluation ISO](https://www.microsoft.com/en-us/evalcenter/evaluate-windows-server-2025)
3. Create a VM: 4GB RAM minimum, 60GB disk, Windows Server 2022 type
4. Mount the ISO, boot, and select **Windows Server 2025 Datacenter with Desktop Experience**

---

## Lab walkthrough

### Step 1 — Install Active Directory Domain Services

RDP into the VM. Server Manager opens automatically on login.

**Via GUI:**  
Server Manager → Manage → Add Roles and Features → Server Roles → check **Active Directory Domain Services** → Add Features → Install

**Via PowerShell:**
```powershell
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Install-WindowsFeature -Name GPMC
```

> Install GPMC in the same session — it's required for Step 4 and won't appear in the Tools menu until it's installed.

---

### Step 2 — Promote the server to a Domain Controller

Installing the AD DS role doesn't create a domain. Promotion does.

**Via GUI:**  
Server Manager → yellow warning flag → Promote this server to a domain controller → Add a new forest → Root domain name: `lab.local` → set DSRM password → Install (server restarts automatically)

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

After the restart, this server is the authoritative DNS and identity source for everything that joins `lab.local`.

---

### Step 3 — Build the organisational structure

Open **Active Directory Users and Computers (ADUC)** from the Tools menu.

#### Create Organisational Units

```powershell
New-ADOrganizationalUnit -Name "IT"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Finance"   -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "HR"        -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Sales"     -Path "DC=lab,DC=local"
New-ADOrganizationalUnit -Name "Computers" -Path "DC=lab,DC=local"
```

#### Create Security Groups

```powershell
New-ADGroup -Name "IT_Admins"     -GroupScope Global -GroupCategory Security -Path "OU=IT,DC=lab,DC=local"
New-ADGroup -Name "Finance_Users" -GroupScope Global -GroupCategory Security -Path "OU=Finance,DC=lab,DC=local"
New-ADGroup -Name "HR_Users"      -GroupScope Global -GroupCategory Security -Path "OU=HR,DC=lab,DC=local"
New-ADGroup -Name "Sales_Users"   -GroupScope Global -GroupCategory Security -Path "OU=Sales,DC=lab,DC=local"
```

#### Create Users and assign group membership

> **Important:** Run this entire block together — not line by line. The `$password` variable must be defined before the `New-ADUser` commands execute.

```powershell
# Define password first
$password = ConvertTo-SecureString "Welcome@2026!" -AsPlainText -Force

# Create users
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

# Assign group memberships
Add-ADGroupMember -Identity "IT_Admins"     -Members "alice.chen"
Add-ADGroupMember -Identity "Finance_Users" -Members "bob.patel"
Add-ADGroupMember -Identity "HR_Users"      -Members "carol.jones"
Add-ADGroupMember -Identity "Sales_Users"   -Members "david.smith"
```

---

### Step 4 — Configure Group Policy

Open **Group Policy Management** from the Tools menu in Server Manager.

1. Expand Forest: lab.local → Domains → lab.local
2. Right-click the **IT** OU → Create a GPO in this domain and link it here
3. Name it: `IT Security Policy`
4. Right-click → Edit, then configure the following:

| Policy path | Setting | Value |
|---|---|---|
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Minimum password length | 12 |
| Computer Config → Windows Settings → Security → Account Policies → Password Policy | Password must meet complexity requirements | Enabled |
| Computer Config → Windows Settings → Security → Local Policies → Security Options | Interactive logon: Machine inactivity limit | 900 seconds |
| Computer Config → Administrative Templates → System → Removable Storage Access | All removable storage classes: Deny all access | Enabled |

**Test the GPO:**  
Join a second VM to `lab.local`, move its computer account into the IT OU, run `gpupdate /force`, log in as `alice.chen`, and verify the screen lock engages after 15 minutes of inactivity.

---

### Step 5 — Help desk operations

These are the tasks every IT support role expects on day one.

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
# Disable — preserves account history and group memberships for audit
Disable-ADAccount -Identity "david.smith"

# Find all disabled accounts
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName
```

**Audit: accounts inactive for 90+ days:**
```powershell
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $cutoff -and Enabled -eq $true} `
  -Properties LastLogonDate | Select-Object Name, LastLogonDate
```

**Check group membership:**
```powershell
Get-ADPrincipalGroupMembership -Identity "alice.chen" | Select-Object Name
```

---

## Verification

Run these checks to confirm everything is wired up correctly.

| Check | Command | Expected result |
|---|---|---|
| DC is running | `Get-ADDomainController` | Returns DC info including forest `lab.local` |
| OUs exist | `Get-ADOrganizationalUnit -Filter *` | Lists all 5 OUs |
| Users exist and are enabled | `Get-ADUser -Filter {Enabled -eq $true}` | Lists all 4 test accounts |
| Group membership correct | `Get-ADGroupMember -Identity IT_Admins` | Returns `alice.chen` |
| GPO is linked | `Get-GPInheritance -Target 'OU=IT,DC=lab,DC=local'` | Shows `IT Security Policy` as linked |

---

## Troubleshooting

| Problem | Fix |
|---|---|
| PowerShell prompts for `Name:` when creating users | The `$password` variable wasn't defined first. Copy and run the entire block from Step 3, not individual lines. |
| Can't copy/paste into the VM | RDP client → Show Options → Local Resources → check Clipboard. Or download the `.rdp` file from Azure portal and open with the native Remote Desktop app. |
| Promotion fails with DNS conflict | Set the NIC's preferred DNS to `127.0.0.1` before promoting. |
| Can't RDP after domain join | Log in as `LAB\Administrator` (not just `Administrator`). |
| GPO not applying | Run `gpupdate /force` on the target machine, then `gpresult /r` to see applied policies. |
| User can't log in after creation | Confirm the account is Enabled and `ChangePasswordAtLogon` is not blocking the login. |
| AD Users and Computers not showing | Run `dsa.msc` from the Run dialog, or install: `Add-WindowsFeature RSAT-ADDS` |

---

## Key concepts

**Domain Controller** — The server that runs Active Directory. It answers every authentication request on the network. When a user logs into any domain-joined machine, their credentials are validated against the DC.

**Organisational Unit (OU)** — A folder inside Active Directory used to organise users, computers, and groups by department or function. The real power: GPOs can be linked to an OU, automatically applying settings to everything inside it.

**Security Group** — A container for user accounts. Permissions are granted to groups, not individuals. When someone changes roles, you update their group membership — every downstream access right changes automatically.

**Group Policy Object (GPO)** — A collection of settings applied automatically to every user or computer in a linked OU. Password policies, screen lock timers, USB restrictions — all enforced centrally without touching individual machines.

**DSRM password** — Directory Services Restore Mode password. Only needed for disaster recovery. Write it down and store it securely.

---

## Certification alignment

| Certification | Topics this lab covers |
|---|---|
| CompTIA Security+ | Identity and access management, least privilege, account lifecycle, policy enforcement |
| CompTIA Network+ | DNS configuration, domain architecture, network authentication |
| AZ-104 Azure Administrator | Entra ID uses the same concepts: users, groups, roles, conditional access — on-prem AD knowledge transfers directly |

---

## Cloud relevance

Microsoft Entra ID (formerly Azure AD) is the cloud equivalent of on-premises Active Directory. Users, groups, security policies, and RBAC work on the same principles — just hosted in Azure rather than on a local server. Hybrid environments sync on-premises AD to Entra ID, meaning the same identities appear in both places. Understanding how to build and manage an AD environment is foundational for any Azure cloud or security engineering role.

---

*Lab 01 of an ongoing cloud and identity engineering series.*# azure-active-directory-lab
Deploying and administering Active Directory Domain Services in Microsoft Azure using Windows Server 2025.
