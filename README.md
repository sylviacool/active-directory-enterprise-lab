# Active Directory Enterprise Lab

A hands-on Windows Server infrastructure lab simulating a small company network: Active Directory, DNS, DHCP, Group Policy, file sharing, and PowerShell automation, built and documented from the ground up in a virtualized environment.

## Overview

This project simulates the core infrastructure of a small business network. A Windows Server 2022 machine was promoted to a Domain Controller and configured with DNS and DHCP, an Organizational Unit structure was designed for three departments, domain users and security groups were created, a Windows 11 client was joined to the domain, file sharing with NTFS permissions was configured, and Group Policy was used to enforce selected domain settings. A custom PowerShell script was written to automate user account creation.

The goal was not just to follow steps, but to understand *why* each piece works the way it does, and to practice real troubleshooting when things didn't go as expected — several of which are documented below, since that process is as much a part of real sysadmin work as the configuration itself.

## Environment

- **Domain Controller:** Windows Server 2022 (hostname: `DC01`)
- **Client:** Windows 11 Enterprise (hostname: `WS01`)
- **Domain:** `LAB.local`
- **Virtualization:** Oracle VirtualBox, NAT Network mode (isolated internal lab network)
- **Network:** `10.0.2.0/24`

## Technologies and Concepts Covered

- Active Directory Domain Services (AD DS) — installation and promotion
- DNS Server role and domain zone configuration
- DHCP Server role, scope configuration, and client leasing
- Static IP configuration for the Domain Controller and DHCP-based client addressing
- Organizational Unit (OU) design
- Domain user and security group management, including cross-OU group membership
- Domain join (client-side)
- SMB file sharing and NTFS permissions
- Group Policy Objects (GPO): desktop settings, restrictions, and account security policy
- GPO troubleshooting: linking scope, Security Filtering, UNC paths, and policy application
- PowerShell automation for Active Directory (custom script)
- Password management: reset and forced change at next logon

---

## 1. Windows Server & Active Directory Setup

The lab began with a clean Windows Server 2022 installation, followed by installing the AD DS role and promoting the server to a Domain Controller for a new domain, `LAB.local`. Promoting the server automatically installed and configured the DNS role alongside it, since Active Directory depends on DNS for clients to locate the Domain Controller and domain services.

| Screenshot | Description |
|---|---|
| `01-server-manager-baseline.png` | Fresh Server Manager, no roles installed |
| `02-adds-role-installed.png` | AD DS role successfully installed |
| `03-domain-promotion.png` | Server promoted to Domain Controller for LAB.local |
| `04-dns-zone.png` | DNS Manager showing the automatically created LAB.local zone |

## 2. Network Configuration

The Domain Controller was configured with a static IP address so that important infrastructure services such as Active Directory and DNS remain reachable at a consistent address. The Windows 11 client was later configured to obtain its network configuration through DHCP.

| Screenshot | Description |
|---|---|
| `05-static-ip-config.png` | Static IP configured on the Domain Controller (`10.0.2.15`) |
| `06-static-ip-confirmed.png` | `ipconfig` confirming the Domain Controller's static IP configuration |

## 3. DHCP

A DHCP Server role was added to the same Domain Controller to automatically assign IP addresses and other network configuration to client machines, rather than configuring every client manually.

**Troubleshooting note:** the initial DHCP scope was accidentally created with a subnet mask of `255.0.0.0` (a /8, covering the entire 10.x.x.x range) instead of the intended `255.255.255.0` (/24) matching the lab's actual network design. This was caught by noticing that the scope displayed an unexpected network ID in DHCP Manager. The incorrect scope was deleted and recreated with the correct subnet mask.

The DHCP scope was configured with an address range of `10.0.2.50`–`10.0.2.100`, keeping the Domain Controller's static address (`10.0.2.15`) outside the DHCP pool and therefore avoiding an IP address conflict.

| Screenshot | Description |
|---|---|
| `07-DHCP-role-installed.png` | DHCP Server role successfully installed |
| `08-dhcp-scope-config.png` | DHCP scope showing the configured address range |
| `09-dhcp-scope-active.png` | DHCP console showing the `LAB-Client-Scope` as Active |
| `10-dhcp-client-lease.png` | Windows 11 client successfully obtaining `10.0.2.50` from the DHCP server, with the expected subnet mask, gateway, and DNS configuration |

## 4. Active Directory Structure & Users

Three Organizational Units were created to reflect a small company's departmental structure: Engineering, Management, and IT. Domain users were created within their respective OUs rather than being left in AD's default Users container, providing a structured foundation for applying department-specific policies and permissions.

| Screenshot | Description |
|---|---|
| `11-ou-structure.png` | ADUC showing the Engineering, Management, and IT OUs |
| `12-domain-users.png` | ADUC showing domain users inside a custom OU |
| `13-Computer-object-ws01.png` | ADUC showing the WS01 computer object in the domain |

## 5. Domain Join & Authentication

The Windows 11 client was joined to the `LAB.local` domain, and successful domain authentication was verified using a domain user account rather than a local account.

| Screenshot | Description |
|---|---|
| `14-Domain-join.png` | Windows 11 domain-join process showing the `LAB` domain during authentication |
| `15-Domain-login.png` | Successful domain login, verified with `whoami` returning `lab\rmyle` |

## 6. Security Groups & Cross-OU Membership

A security group, `EngineeringShare`, was created to manage access to a shared folder. The group includes users from two different OUs — Nelson Bighetti and Richard Myle from Engineering, and Jared Dunn from Management. This demonstrates that security group membership is independent of the OU in which a user account is stored.

| Screenshot | Description |
|---|---|
| `16-engineeringshare-membership.png` | EngineeringShare group showing members from both Engineering and Management OUs |

## 7. File Sharing & Permissions

An SMB file share, `EngineeringShare`, was created on the Domain Controller. Permissions were configured for the `EngineeringShare` security group, allowing the selected users to access the shared resources while restricting access for users outside the group.

**Troubleshooting note:** after configuring the share, a user outside the `EngineeringShare` group received an "Access Denied" error when attempting to access the share. This provided a practical verification that the configured access restrictions were being enforced.

| Screenshot | Description |
|---|---|
| `17-fileshare-creation.png` | SMB share creation wizard completed successfully |
| `18-server-shares-list.png` | Server Manager showing EngineeringShare alongside the default NETLOGON and SYSVOL shares |
| `19-client-share-access.png` | Windows 11 client successfully accessing EngineeringShare over the network |
| `20-permission-denied-troubleshooting.png` | Access-denied error for a user outside the permitted group |

## 8. Group Policy (GPO)

### Desktop Wallpaper Deployment

A Group Policy Object was created to deploy a desktop wallpaper to specific users. The policy used a wallpaper stored in the domain's NETLOGON share.

The implementation required several rounds of troubleshooting:

1. **File path:** the wallpaper policy required a network-accessible path so that the client could retrieve the image from the server.
2. **File format:** the wallpaper image format was tested and adjusted during troubleshooting to meet the requirements of the Desktop Background policy.
3. **GPO scope:** the GPO was initially linked to the Engineering OU. Because `EngineeringShare` also contained Jared Dunn from the Management OU, the policy scope required additional consideration. The GPO was subsequently linked at the domain level and Security Filtering was configured using the `EngineeringShare` group.
4. **Verification:** the wallpaper was successfully applied on the Windows 11 client for a domain user. Further investigation of the group-based targeting behavior was left for later troubleshooting.

This was a useful practical example of the difference between **where a GPO is linked** and **which users or computers are targeted through Security Filtering**.

| Screenshot | Description |
|---|---|
| `21-gpo-wallpaper-setting.png` | Group Policy Editor showing the Desktop Background configuration |
| `22-wallpaper-applied-client.png` | Windows 11 client showing the wallpaper successfully applied |

### Account Lockout Policy

A domain-wide Account Lockout Policy GPO was configured and linked at the domain level. During verification, the policy was initially checked in the wrong section of `gpresult /r`. Account Lockout Policy is a **Computer Configuration** setting, so the correct section of the report had to be checked to verify its application.

| Screenshot | Description |
|---|---|
| `23-gpo-lockout-scope.png` | Group Policy Management showing AccountLockoutPolicy linked at `LAB.local` with its security filtering configuration |
| `24-gpresult-lockout-applied.png` | `gpresult /r` confirming AccountLockoutPolicy was applied under Computer Configuration |

## 9. PowerShell Automation

A parameterized PowerShell script, `Create-ADUser.ps1`, was written to automate new user creation — accepting first name, last name, username, OU, and domain as input, generating a random password, and creating the account via `New-ADUser`.

**Troubleshooting note:** the script initially failed to create users due to an incorrect Distinguished Name format in the `-Path` parameter. Spaces had been added around the `=` signs (for example, `"OU = $OU, DC = LAB"` instead of `"OU=$OU,DC=LAB"`), resulting in an invalid Active Directory path. After correcting the syntax, and after confirming that the corrected script had actually been saved, the script ran successfully.

| Screenshot | Description |
|---|---|
| `25-powershell-script-code.png` | Full script showing parameter definitions, password generation, and the `New-ADUser` call |
| `26-powershell-execution-verified.png` | Script executed successfully, with `Get-ADUser` confirming that the new account exists in the correct OU |

## 10. Password Management

Standard password administration tasks were performed and verified using both the GUI (ADUC) and PowerShell. This included resetting a user's password and configuring the account so that the user is required to change the password at the next logon.

| Screenshot | Description |
|---|---|
| `27-password-reset-gui.png` | Password reset confirmation via ADUC |
| `28-password-reset-powershell.png` | Password reset using `Set-ADAccountPassword`, with the account then verified using `Get-ADUser` |
| `29-change-password-at-logon.png` | Account password state showing `PasswordExpired: True`, indicating that the account requires a password change |

---

## Key Takeaways

- Built and configured a Windows Server 2022 Active Directory environment including AD DS, DNS, DHCP, OUs, domain users, security groups, file sharing, and Group Policy.

- Practiced troubleshooting across multiple layers, including network configuration, DHCP scope settings, GPO paths, file formats, policy linking, and Security Filtering.

- Learned the distinction between GPO linking and Security Filtering, and how policy scope affects which users and computers receive a policy.

- Used `gpresult /r`, `ipconfig`, Active Directory tools, and PowerShell to verify configurations and troubleshoot issues rather than relying only on the initial configuration.

- Automated Active Directory user creation with a parameterized PowerShell script and performed common password-management tasks using both GUI and PowerShell.
