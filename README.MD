# 🖥️ Active Directory Home Lab — Windows Server 2022 + Windows 11

Link to AD Lab post with all screenshots of work: https://medium.com/@karimsaminur123/active-directory-lab-d237e24b1214

> **A hands-on home lab I built to learn enterprise IT administration, Identity & Access Management (IAM), Group Policy, and PowerShell automation from scratch.**

---

## 📌 Table of Contents

- [Overview](#overview)
- [Lab Environment & VM Setup](#lab-environment--vm-setup)
- [Windows Server 2022 — Domain Controller Setup](#windows-server-2022--domain-controller-setup)
- [Promoting the Server to a Domain Controller](#promoting-the-server-to-a-domain-controller)
- [Connecting Windows 11 to the Domain](#connecting-windows-11-to-the-domain)
- [Network Troubleshooting](#network-troubleshooting)
- [DHCP & DNS Configuration](#dhcp--dns-configuration)
- [Creating Domain Users & Organizational Units](#creating-domain-users--organizational-units)
- [Shared Folder Permissions](#shared-folder-permissions)
- [Group Policy Objects (GPOs)](#group-policy-objects-gpos)
- [PowerShell Scripting for AD](#powershell-scripting-for-ad)
- [Password Reset & Account Lockout Policy](#password-reset--account-lockout-policy)
- [What I Learned](#what-i-learned)
- [What's Next](#whats-next)

---

## Overview

This is a home lab I built as a sophomore studying IT/Cybersecurity. I wanted real hands-on experience with the tools that enterprise IT and security teams actually use day-to-day — not just textbook definitions. Active Directory is literally everywhere in corporate environments, so I figured building a full AD lab from scratch would give me something tangible to talk about and learn from.

The lab simulates a small company network with a domain controller, department-level organizational units, user accounts, shared resources, and enforced group policies — basically a mini corporate environment running on my own machine.

**Technologies Used:**
- Windows Server 2022 (Domain Controller)
- Windows 11 Enterprise (Client Machine)
- UTM (Virtualization on macOS)
- Active Directory Domain Services (AD DS)
- DNS, DHCP, AD Certificate Services
- Group Policy Management
- PowerShell

---

## Lab Environment & VM Setup

I downloaded both ISOs directly from the **Microsoft Evaluation Center** — they offer free 180-day evaluation versions which is perfect for a lab.

### Windows Server 2022 VM Settings

| Setting | Value |
|---|---|
| Machine Type | Intel ICH9-based PC (x86_64) |
| RAM | 6144 MB |
| CPU Cores | Default |
| Storage | 64 GB |
| Network | Shared Network (virtio-net-pci) |

### Windows 11 Enterprise VM Settings

| Setting | Value |
|---|---|
| RAM | 4096 MB |
| CPU Cores | 4 |
| Storage | 64 GB |
| Network | Host Only (virtio-net-pci) |

> **Why Host Only for Windows 11?** I needed both VMs to communicate with each other in an isolated network. Host Only mode means the client VM can only talk to the host machine and other VMs — not the internet directly. This is a closer simulation of a real internal corporate network.

### Guest Tools Setup

After installing both OSes, I installed **UTM guest tools (SPICE tools)** on both VMs. This fixed screen resolution scaling and clipboard sharing between host and VM. Without this, the display was tiny and unusable.

Both machines were renamed for clarity:
- Server → `BigDC`
- Client → `DesktopPC`

---

## Windows Server 2022 — Domain Controller Setup

Once the server was up and running, I opened **Server Manager** and began adding roles.

**Path:** `Server Manager > Manage > Add Roles and Features`

I selected **Role-based or feature-based installation** and chose `BigDC` as the target server. I added the following roles:

- **Active Directory Domain Services (AD DS)** — the core service that manages users, computers, and policies across the domain
- **DHCP Server** — automatically assigns IP addresses to devices that join the network
- **DNS Server** — translates domain names (like `LAB.local`) into IP addresses so devices can find each other

I checked the box for **"Restart if necessary"** and let the install run. After it finished, a notification appeared in Server Manager to **promote the server to a domain controller** — which is the step that actually turns it into the brain of the network.

---

## Promoting the Server to a Domain Controller

After the roles were installed, I clicked the notification flag in Server Manager and selected **"Promote this server to a domain controller."**

In the wizard I chose:
- **Add a new forest** (since this is a brand new domain, not joining an existing one)
- **Root domain name:** `LAB.local`

I set the Directory Services Restore Mode (DSRM) password, kept all other settings at default, and let it run. The server automatically restarted after promotion.

After reboot, I went back and added one more role — **Active Directory Certificate Services (AD CS)** — using all default settings. AD CS allows the domain controller to issue digital certificates, which is useful for securing communications and is something you'd see in real enterprise environments.

At this point, `BigDC` was officially a fully functioning domain controller for `LAB.local`.

---

## Connecting Windows 11 to the Domain

For the Windows 11 client to join the domain, both machines needed to be able to communicate with each other over the network. This required assigning **static IP addresses** to both adapters.

**Path:** `Right-click network icon > Open Network Settings > Ethernet > Change adapter options > Adapter Properties > IPv4 (TCP/IPv4) > Use the following IP address`

### IP Address Scheme

| Machine | IP Address | Subnet Mask | Default Gateway | DNS Server |
|---|---|---|---|---|
| BigDC (Server) | 192.168.64.10 | 255.255.255.0 | 192.168.64.1 | 127.0.0.1 (loopback) |
| DesktopPC (Client) | 192.168.64.20 | 255.255.255.0 | 192.168.64.1 | 192.168.64.10 (server) |

> **Why point the client's DNS at the server?** When the client tries to join `LAB.local`, it needs to resolve that domain name to an IP address. The server running DNS knows about `LAB.local` — so setting the server as the DNS server for the client means it can find the domain correctly.

### Joining the Domain

Once the IPs were set and connectivity confirmed:

`Settings > Access work or school > Add account > Join this device to a local Active Directory domain`

I typed in `LAB.local`, entered the domain admin credentials from the server, and restarted when prompted. After reboot, I could log in using domain accounts.

---

## Network Troubleshooting

This section took me the longest and taught me the most. Here's what went wrong and how I fixed it.

### Problem 1 — Adapter Not Appearing on Windows 11

When I went into network adapter settings on the Windows 11 VM, nothing was showing up at all. After some research I realized the adapter type was wrong.

**Fix:** Changed the network adapter type from `e1000` to `virtio-net-pci` in UTM settings, and switched the network mode from **Shared Network** to **Host Only**. After this, the adapter appeared and I could configure the static IP.

### Problem 2 — Server Could Ping Itself But Not the Client

Even after setting static IPs correctly, pings from the server to the client were timing out. The connection was one-directional.

**Fix:** Windows Firewall on the client was blocking inbound ICMP (ping) traffic by default. I opened PowerShell on the client and ran:

```powershell
netsh advfirewall firewall add rule name="Allow ICMPv4" protocol=icmpv4:8,any dir=in action=allow
```

This created an inbound firewall rule that allows ICMPv4 echo requests (pings). After running this, both machines could ping each other successfully.

> **Why this matters:** In real IT environments, firewall misconfiguration is one of the most common reasons connectivity issues happen. Knowing how to diagnose and fix it via the command line — not just clicking through a GUI — is a skill that comes up constantly.

---

## DHCP & DNS Configuration

### DHCP

**Path:** `Server Manager > Tools > DHCP > Expand server > IPv4 > Right-click > Authorize`

I created a **scope** — a defined pool of IP addresses that the DHCP server can hand out automatically. I configured a range of 50 usable addresses based on the `192.168.64.x` subnet, using the server's IP as the gateway.

This means any new device that joins the network can get an IP address automatically without me having to manually assign one — which is how it works in real enterprise environments with hundreds of devices.

### DNS Verification

To confirm DNS was working, I ran an `nslookup` from the server:

```
nslookup BigDC
```

**Path to verify DNS records:** `Server Manager > Tools > DNS > Forward Lookup Zones > LAB.local`

I confirmed there was an **A record** for `BigDC` pointing to `192.168.64.10` — exactly what it should be. A forward lookup zone translates a hostname to an IP, which is the foundational job of DNS.

---

## Creating Domain Users & Organizational Units

### Organizational Units (OUs)

OUs are like folders inside Active Directory that let you organize users, computers, and groups by department or function. They also let you apply different Group Policies to different groups of people.

**Path:** `Server Manager > Tools > Active Directory Users and Computers > Right-click LAB.local > New > Organizational Unit`

I created the following OUs:

```
LAB.local
├── Engineering
│   └── Test User 3, Test User 4
├── Management
│   └── Test User 1, Test User 2
├── IT
│   ├── Administrator
│   └── Admin (sub-OU)
└── Groups
```

I also cleaned up the default `Users` container — removed everything except `Administrator` and `Guest` — and created 4 test users with simple credentials. I set **"Password never expires"** on all accounts so I could test account lockout and password reset workflows without interference.

---

## Shared Folder Permissions

To simulate real-world access control, I created a shared folder that only specific users could access.

### Step 1 — Create the Group

Inside the `Engineering` OU, I created a new security group called `EngineeringSHARE` and added **Test Users 1, 3, and 4** as members.

### Step 2 — Create the Share

**Path:** `Server Manager > File and Storage Services > Shares > Tasks > New Share`

I named the folder `EngineeringSHARE` and used default settings until I reached the **Permissions** page.

### Step 3 — Configure Permissions

This is where it got interesting. I:
1. Clicked **Disable Inheritance** → **Convert inherited permissions into explicit permissions**
2. Removed permissions for users/groups that shouldn't have access
3. Added the `EngineeringSHARE` security group with **Read/Write** permissions

### Step 4 — Test It

I logged into the Windows 11 client as **Test User 2** (who is NOT in the `EngineeringSHARE` group) and tried to access the share. Access was denied — exactly as expected. This confirmed that the permission model was working correctly and only group members could access the folder.

> This directly mirrors how access control works in enterprise IT — least privilege, role-based access, and explicit group membership governing what resources users can reach.

---

## Group Policy Objects (GPOs)

GPOs let administrators push settings, restrictions, and configurations to users or computers across the domain automatically. This is one of the most powerful tools in any AD environment.

### GPO 1 — Engineering Department Wallpaper

I wanted to enforce a department-specific desktop wallpaper for all Engineering users.

**Steps:**
1. Created a wallpaper image using Paint and saved it
2. Copied the image to `NETLOGON` share: `Server Manager > File and Storage Services > Shares > NETLOGON > Open Share`
3. Copied the full network path of the wallpaper file
4. Created a new GPO: `Server Manager > Tools > Group Policy Management > Forest: LAB.local > Engineering > Right-click > Create a GPO in this domain and Link it here` → Named it `SetEngineeringWallpaper`
5. Edited the GPO: `User Configuration > Policies > Administrative Templates > Desktop > Desktop > Desktop Wallpaper`
6. Enabled the setting, pasted in the wallpaper network path, and clicked OK

**Result:** When I logged in as an Engineering department user, the wallpaper was automatically applied — no manual configuration on the client side at all.

### GPO 2 — Account Lockout Policy

I created a domain-wide account lockout policy to simulate how companies protect against brute force login attempts.

**Path:** `Server Manager > Tools > Group Policy Management > Right-click LAB.local > Create a GPO > Named AccountLockoutPolicy`

**Edit path:** `Computer Configuration > Policies > Windows Settings > Security Settings > Account Policies > Account Lockout Policy`

| Setting | Value | Reason |
|---|---|---|
| Account Lockout Threshold | 3 invalid attempts | Locks account after 3 wrong passwords |
| Account Lockout Duration | 0 minutes | Only an admin can unlock it (not auto-reset) |
| Reset Lockout Counter After | 60 minutes | Counter resets after 60 min if account isn't locked |

I right-clicked the policy and selected **Enforce** to make sure it couldn't be overridden by other GPOs.

**Test:** Logged into the Windows 11 client as Test User 1 and entered the wrong password 3 times. The account locked immediately — confirmed the policy was working.

---

## PowerShell Scripting for AD

PowerShell is how real sysadmins and IT pros automate repetitive tasks at scale. Instead of clicking through the GUI to create every user one by one, I wrote a script to do it automatically.

### Automating New User Account Creation

```powershell
# Define user details
$FirstName = "John"
$LastName = "Doe"
$Username = "jdoe"
$OU = "OU=Engineering,DC=LAB,DC=local"

# Generate a random password
$Password = [System.Web.Security.Membership]::GeneratePassword(12, 2)
$SecurePassword = ConvertTo-SecureString $Password -AsPlainText -Force

# Create the user
New-ADUser `
    -GivenName $FirstName `
    -Surname $LastName `
    -SamAccountName $Username `
    -UserPrincipalName "$Username@LAB.local" `
    -Path $OU `
    -AccountPassword $SecurePassword `
    -Enabled $true `
    -PasswordNeverExpires $false `
    -ChangePasswordAtLogon $true

Write-Host "User $Username created. Temp password: $Password"
```

This script creates a new AD user in the correct OU, auto-generates a secure password, and forces them to reset it on first login. In an enterprise environment this kind of script gets run constantly during onboarding — creating this manually for every new hire isn't realistic at scale.

---

## Password Reset & Account Lockout Policy

After locking out Test User 1 through the GPO test above, I simulated a real helpdesk password reset ticket.

### GUI Method (Active Directory Users and Computers)

**Path:** `Server Manager > Tools > Active Directory Users and Computers > Management OU > Find Objects in AD Domain Services`

1. Searched for `Test User 1`
2. Right-clicked the user > **Reset Password**
3. Set a new temporary password
4. Checked **"User must change password at next logon"**
5. Checked **"Unlock the user's account"**
6. Clicked OK

Logged back in as Test User 1, was prompted to create a new password, did so successfully — full end-to-end ticket resolved.

### PowerShell Method (Much Faster)

```powershell
# Unlock the account
Unlock-ADAccount -Identity "tuser1"

# Reset the password
Set-ADAccountPassword -Identity "tuser1" `
    -NewPassword (ConvertTo-SecureString "TempPass123!" -AsPlainText -Force) `
    -Reset

# Force password change at next login
Set-ADUser -Identity "tuser1" -ChangePasswordAtLogon $true
```

Three commands and the ticket is done. The PowerShell method is way faster once you know the syntax, and it's what you'd actually use if you had 50 tickets in a queue.

---

## What I Learned

- **Active Directory fundamentals** — How AD DS works, what a domain controller does, and how DNS ties everything together in a Windows domain environment
- **Network configuration** — Assigning static IPs, configuring DHCP scopes, troubleshooting connectivity between VMs, and understanding why DNS pointing matters
- **Firewall management** — Creating inbound/outbound rules via `netsh` from the command line, not just through GUI
- **Organizational structure** — Using OUs to mirror real company department structures and applying policy at the OU level
- **Group Policy Objects (GPOs)** — Creating and linking GPOs, editing policy settings for both user and computer configuration, enforcing policies across the domain
- **Access control & permissions** — Configuring shared folder permissions, disabling inheritance, and applying the principle of least privilege
- **PowerShell for AD administration** — Writing scripts to automate user creation, reset passwords, and unlock accounts — skills that directly translate to IT/sysadmin work
- **Troubleshooting methodology** — Researching issues, isolating the problem (firewall vs. network vs. config), and applying targeted fixes rather than guessing

---

## What's Next

This lab gave me a solid foundation in Windows Server administration and IAM basics. Here's what I'm planning to add next:

- [ ] Deploy **Peppermint** (open-source ticketing system) to simulate a real helpdesk environment
- [ ] Integrate more IAM tools into the domain
- [ ] Set up a **SIEM** (like Splunk or Wazuh) to monitor AD login events and GPO changes
- [ ] Configure **Active Directory auditing** and log failed login attempts
- [ ] Explore **Azure AD (Entra ID)** hybrid configurations
- [ ] Automate User lifecycle with **Microsoft (Entra ID)**
- [ ] Set up Okta SSO app integration
- [ ] Entra ID Conditional Access Policy

---

*Built by an upcoming junior IT/Cybersecurity student trying to learn by doing. Feedback and suggestions always welcome.*
