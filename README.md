# 🔒 Group Policy Objects (GPO) Implementation on Windows Server 2022

Home Lab Project — Part 03

![Windows Server](https://img.shields.io/badge/Windows%20Server-2022-blue?style=flat-square&logo=windows)
![Active Directory](https://img.shields.io/badge/Active%20Directory-Configured-green?style=flat-square)
![GPO](https://img.shields.io/badge/GPO-Implemented-green?style=flat-square)
![PowerShell](https://img.shields.io/badge/PowerShell-Automation-blue?style=flat-square&logo=powershell)
![Status](https://img.shields.io/badge/Status-Complete-brightgreen?style=flat-square)

> **Home Lab Project — Part 3**
> Implementing Group Policy Objects (GPOs) on Windows Server 2022 to enforce security policies, desktop restrictions, software restrictions, and network drive mapping across a domain environment — simulating real enterprise IT management.

---

## 📑 Table of Contents

- [Lab Environment](#lab-environment)
- [Objectives](#objectives)
- [What is a GPO?](#what-is-a-gpo)
- [Organizational Units](#organizational-units)
- [GPO 1 — Password & Lockout Policy](#gpo-1--password--lockout-policy)
- [GPO 2 — Login Banner](#gpo-2--login-banner)
- [GPO 3 — Desktop Restrictions](#gpo-3--desktop-restrictions)
- [GPO 4 — Software Restrictions](#gpo-4--software-restrictions)
- [GPO 5 — Network Drive Mapping](#gpo-5--network-drive-mapping)
- [GPO Summary](#gpo-summary)
- [Temporary Elevation](#temporary-elevation)
- [Architecture](#architecture)
- [Issues & Fixes](#issues--fixes)
- [Skills Demonstrated](#skills-demonstrated)
- [Related Projects](#related-projects)

---

## 🏗️ Lab Environment

| Component | Details |
|---|---|
| **Hypervisor** | Oracle VirtualBox |
| **Domain Controller** | WIN-6ARV79FMB48 — Windows Server 2022 |
| **Client Machine** | CPC-NSimon-001 — Windows 10 Pro |
| **Domain** | mylab.local |
| **Server IP** | 192.168.1.200 |
| **Client IP** | 192.168.1.10 |

---

## 🎯 Objectives

- [x] Create Organizational Units (OUs) for user and computer management
- [x] Move users and computers into appropriate OUs
- [x] Configure Fine-Grained Password & Lockout Policy
- [x] Deploy Login Banner GPO
- [x] Deploy Desktop Restrictions GPO (Control Panel, Task Manager)
- [x] Deploy Software Restrictions GPO (Registry Editor, Run dialog)
- [x] Create and share a network folder (CompanyShare)
- [x] Map network drive via GPO
- [x] Verify all GPOs applied on Windows 10 client
- [x] Document temporary elevation procedures

---

## 📖 What is a GPO?

A **Group Policy Object (GPO)** is a collection of settings that control the working environment of user accounts and computer accounts in Active Directory. GPOs allow administrators to centrally manage and configure operating systems, applications, and user settings across an entire domain.

**How GPOs work:**
```
Domain Controller (WinServer2022)
        │
        ├── Creates & stores GPOs
        │
        ▼
Active Directory (mylab.local)
        │
        ├── Links GPOs to OUs
        │
        ▼
Windows 10 Client (CPC-NSimon-001)
        │
        ├── Downloads & applies GPOs at login
        │
        ▼
User (nsimon) gets restricted/configured environment
```

**Real-world use:** Every company using Windows uses GPOs to enforce security policies, deploy software, map drives, set wallpapers, restrict access, and much more — all from one central server.

---

## 🗂️ Organizational Units

### OUs Created

```powershell
New-ADOrganizationalUnit -Name "VMAdmin" -Path "DC=mylab,DC=local"
New-ADOrganizationalUnit -Name "VMUsers" -Path "DC=mylab,DC=local"
New-ADOrganizationalUnit -Name "VirtualWorkstation" -Path "DC=mylab,DC=local"
```

### Objects Moved to OUs

```powershell
# Move nsimon to VMUsers
Move-ADObject -Identity "CN=Neil Marvin Simon,CN=Users,DC=mylab,DC=local" `
  -TargetPath "OU=VMUsers,DC=mylab,DC=local"

# Move Administrator to VMAdmin
Move-ADObject -Identity "CN=Administrator,CN=Users,DC=mylab,DC=local" `
  -TargetPath "OU=VMAdmin,DC=mylab,DC=local"

# Move Win10 computer to VirtualWorkstation
Move-ADObject -Identity "CN=CPC-NSimon-001,CN=Computers,DC=mylab,DC=local" `
  -TargetPath "OU=VirtualWorkstation,DC=mylab,DC=local"
```

### OU Structure

| OU | Members | Purpose |
|---|---|---|
| **VMAdmin** | Administrator | Admin accounts — no GPO restrictions |
| **VMUsers** | nsimon | Standard users — all GPO restrictions applied |
| **VirtualWorkstation** | CPC-NSimon-001 | Computer objects |

---

## 🔐 GPO 1 — Password & Lockout Policy

### Fine-Grained Password Policy applied to nsimon

```powershell
New-ADFineGrainedPasswordPolicy `
  -Name "VMUsers-Password-Policy" `
  -Precedence 10 `
  -MinPasswordLength 10 `
  -PasswordHistoryCount 5 `
  -MaxPasswordAge "90.00:00:00" `
  -MinPasswordAge "1.00:00:00" `
  -ComplexityEnabled $true `
  -ReversibleEncryptionEnabled $false `
  -LockoutThreshold 5 `
  -LockoutDuration "00:30:00" `
  -LockoutObservationWindow "00:30:00"

# Apply to nsimon
Add-ADFineGrainedPasswordPolicySubject `
  -Identity "VMUsers-Password-Policy" `
  -Subjects "nsimon"
```

### Policy Settings

| Setting | Value |
|---|---|
| **Minimum Password Length** | 10 characters |
| **Password Complexity** | Enabled |
| **Password History** | 5 passwords |
| **Maximum Password Age** | 90 days |
| **Minimum Password Age** | 1 day |
| **Lockout Threshold** | 5 failed attempts |
| **Lockout Duration** | 30 minutes |
| **Observation Window** | 30 minutes |

---

## 🔔 GPO 2 — Login Banner

Displays a legal warning message before any user logs in — standard in enterprise environments.

```powershell
New-GPO -Name "Security Policy - Login Banner" `
  -Comment "Displays legal warning before login"

New-GPLink -Name "Security Policy - Login Banner" `
  -Target "DC=mylab,DC=local"

# Set banner title
Set-GPRegistryValue -Name "Security Policy - Login Banner" `
  -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "legalnoticecaption" `
  -Type String -Value "AUTHORIZED ACCESS ONLY"

# Set banner message
Set-GPRegistryValue -Name "Security Policy - Login Banner" `
  -Key "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "legalnoticetext" `
  -Type String -Value "This system is for authorized users only. All activities are monitored and logged. Unauthorized access is strictly prohibited."
```

### Result ✅
The login banner appeared on the Windows 10 client showing:
> **AUTHORIZED ACCESS ONLY**
> This system is for authorized users only. All activities are monitored and logged. Unauthorized access is strictly prohibited.

---

## 🖥️ GPO 3 — Desktop Restrictions

Applied to **VMUsers OU** — restricts what standard users can access.

```powershell
New-GPO -Name "Desktop Policy - Restrictions" `
  -Comment "Restricts desktop settings for standard users"

New-GPLink -Name "Desktop Policy - Restrictions" `
  -Target "OU=VMUsers,DC=mylab,DC=local"

# Disable Control Panel
Set-GPRegistryValue -Name "Desktop Policy - Restrictions" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
  -ValueName "NoControlPanel" -Type DWord -Value 1

# Disable Task Manager
Set-GPRegistryValue -Name "Desktop Policy - Restrictions" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "DisableTaskMgr" -Type DWord -Value 1
```

### Restrictions Applied

| Restriction | Status |
|---|---|
| **Control Panel** | ❌ Blocked |
| **Task Manager** | ❌ Blocked |

---

## 🚫 GPO 4 — Software Restrictions

Blocks unauthorized tools for standard users.

```powershell
New-GPO -Name "Software Policy - App Restrictions" `
  -Comment "Restricts unauthorized applications for standard users"

New-GPLink -Name "Software Policy - App Restrictions" `
  -Target "OU=VMUsers,DC=mylab,DC=local"

# Block Registry Editor
Set-GPRegistryValue -Name "Software Policy - App Restrictions" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\System" `
  -ValueName "DisableRegistryTools" -Type DWord -Value 1

# Block Run dialog
Set-GPRegistryValue -Name "Software Policy - App Restrictions" `
  -Key "HKCU\Software\Microsoft\Windows\CurrentVersion\Policies\Explorer" `
  -ValueName "NoRun" -Type DWord -Value 1
```

### Restrictions Applied

| Application | Status |
|---|---|
| **Registry Editor (regedit)** | ❌ Blocked |
| **Run Dialog (Win+R)** | ❌ Blocked |

---

## 🗄️ GPO 5 — Network Drive Mapping

### Step 1 — Create Shared Folder on Server

```powershell
# Create folder structure
New-Item -Path "C:\CompanyShare" -ItemType Directory
New-Item -Path "C:\CompanyShare\IT" -ItemType Directory
New-Item -Path "C:\CompanyShare\Users" -ItemType Directory
New-Item -Path "C:\CompanyShare\General" -ItemType Directory

# Create SMB share
New-SmbShare -Name "CompanyShare" `
  -Path "C:\CompanyShare" `
  -FullAccess "Domain Admins" `
  -ReadAccess "Domain Users"

# Set permissions
Grant-SmbShareAccess -Name "CompanyShare" `
  -AccountName "Domain Admins" -AccessRight Full -Force

Grant-SmbShareAccess -Name "CompanyShare" `
  -AccountName "Domain Users" -AccessRight Read -Force
```

### Share Structure

```
C:\CompanyShare\
├── IT\          ← IT department files
├── Users\       ← User files
└── General\     ← General company files
```

### Step 2 — Map Drive via GPO

```powershell
New-GPO -Name "Drive Map Policy - CompanyShare" `
  -Comment "Automatically maps CompanyShare as Z: drive for all domain users"

New-GPLink -Name "Drive Map Policy - CompanyShare" `
  -Target "OU=VMUsers,DC=mylab,DC=local"

# Map Z: drive via registry
Set-GPRegistryValue -Name "Drive Map Policy - CompanyShare" `
  -Key "HKCU\Network\Z" `
  -ValueName "RemotePath" `
  -Type String -Value "\\192.168.1.200\CompanyShare"
```

### Manual Mapping (Verified Working)

```cmd
net use Z: \\192.168.1.200\CompanyShare /persistent:yes
```

### Result ✅
- Z: drive mapped to `\\192.168.1.200\CompanyShare`
- IT, Users, and General folders visible from Win10 client
- Domain Admins have Full Access
- Domain Users have Read Access

---

## 📊 GPO Summary

| GPO Name | Linked To | Purpose | Status |
|---|---|---|---|
| **Security Policy - Password and Lockout** | DC=mylab,DC=local | Password rules | ✅ Active |
| **Security Policy - Login Banner** | DC=mylab,DC=local | Login warning | ✅ Active |
| **Desktop Policy - Restrictions** | OU=VMUsers | Block Control Panel & Task Mgr | ✅ Active |
| **Software Policy - App Restrictions** | OU=VMUsers | Block Registry & Run | ✅ Active |
| **Drive Map Policy - CompanyShare** | OU=VMUsers | Map Z: network drive | ✅ Active |

---

## 🔑 Temporary Elevation

When IT needs to access restricted settings on a user's computer without logging them out:

### Option 1: Run as Different User
```cmd
runas /user:mylab\Administrator "control"
```

### Option 2: Temporarily Disable GPO
```powershell
# Disable restriction
Set-GPLink -Name "Desktop Policy - Restrictions" `
  -Target "OU=VMUsers,DC=mylab,DC=local" `
  -LinkEnabled No

# Re-enable after work is done
Set-GPLink -Name "Desktop Policy - Restrictions" `
  -Target "OU=VMUsers,DC=mylab,DC=local" `
  -LinkEnabled Yes
```

### Option 3: Remove Specific Restriction
```powershell
Remove-GPRegistryValue -Name "Desktop Policy - Restrictions" `
  -Key "HKCU\Software\Policies\Microsoft\Windows\System" `
  -ValueName "DisableCMD"
```

> **Best Practice:** Always use a separate admin account for IT tasks. Never give standard users admin rights. Use "Run as different user" to maintain the user's session while performing admin tasks.

---

## 📐 Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      mylab.local Domain                      │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Active Directory Structure              │   │
│  │                                                     │   │
│  │  OU=VMAdmin          OU=VMUsers      OU=VirtualWS   │   │
│  │  ├─ Administrator    ├─ nsimon       ├─ CPC-NSimon  │   │
│  │  └─ (No GPO)         └─ (All GPOs)   └─ (Computer)  │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌──────────────────────┐   ┌──────────────────────────┐   │
│  │   WinServer2022      │   │   CPC-NSimon-001         │   │
│  │   DC + GPO Server    │──►│   Windows 10 Pro         │   │
│  │                      │   │                          │   │
│  │   GPOs Applied:      │   │   nsimon sees:           │   │
│  │   ✅ Password Policy │   │   🔒 Login Banner        │   │
│  │   ✅ Login Banner    │   │   ❌ No Control Panel    │   │
│  │   ✅ Desktop Restr.  │   │   ❌ No Task Manager     │   │
│  │   ✅ App Restr.      │   │   ❌ No Registry Editor  │   │
│  │   ✅ Drive Mapping   │   │   💾 Z: Drive Mapped     │   │
│  │                      │   │                          │   │
│  │   IP: 192.168.1.200  │   │   IP: 192.168.1.10      │   │
│  └──────────────────────┘   └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 🔴 Issues & Fixes

### Issue 1: DisableCMD Not Removing After GPO Change
| | |
|---|---|
| **Cause** | DisableCMD was set in two different GPOs (Desktop Policy and Software Policy) |
| **Fix** | Used `Get-GPRegistryValue` to identify which GPO had the active setting, then used `Remove-GPRegistryValue` on the correct GPO |

### Issue 2: Invoke-GPUpdate Remote Timeout
| | |
|---|---|
| **Cause** | Windows 10 firewall blocking Remote Scheduled Tasks Management |
| **Fix** | Ran `gpupdate /force` directly on the Win10 client instead |

### Issue 3: Administrator Login After Moving to VMAdmin OU
| | |
|---|---|
| **Cause** | Password cache conflict after moving account to new OU |
| **Fix** | Reset password using `net user Administrator` and re-enabled the account |

### Issue 4: Z: Drive Not Auto-Mapping via GPO
| | |
|---|---|
| **Cause** | Registry-based drive mapping requires specific conditions to trigger |
| **Fix** | Manually mapped using `net use Z: \\192.168.1.200\CompanyShare /persistent:yes` as workaround |

---

## 📚 Skills Demonstrated

- Active Directory Organizational Unit (OU) Management
- Group Policy Object (GPO) Creation & Linking
- Fine-Grained Password Policy Configuration
- Login Banner / Legal Notice Deployment
- Desktop & Application Restriction via GPO
- SMB File Share Creation & Permission Management
- Network Drive Mapping
- GPO Troubleshooting & Registry Analysis
- Temporary Privilege Elevation Techniques
- PowerShell Automation for AD & GPO Management

---

## 🚀 Future Improvements

- [ ] Configure GPO logon script for automatic drive mapping
- [ ] Set company wallpaper via GPO
- [ ] Deploy software via GPO (e.g., 7-Zip, Chrome)
- [ ] Configure Windows Defender settings via GPO
- [ ] Implement audit policies for login tracking
- [ ] Create department-specific OUs with separate GPOs
- [ ] Configure folder redirection for user documents

---

## 🔗 Related Projects

| Project | Description |
|---|---|
| **Part 1** | [DNS & DHCP Server Setup](https://yuzuki007.github.io/DNS-DHCP-Server-Setup-on-Windows-Server-2022/) |
| **Part 2** | [Windows 10 Client & AD Integration](https://yuzuki007.github.io/Windows-10-Client-Setup-Active-Directory-Integration/) |
| **Part 3** | Group Policy Objects (GPOs) ← You are here |

---

## 👤 Author

**Neil Marvin Simon**
Home Lab Project — Part 3 | Windows Server 2022 | Group Policy Objects | Active Directory
*Built for IT Portfolio purposes*
