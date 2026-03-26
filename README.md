# Hybrid AD / Entra ID Home Lab
 
A personal home lab project built to develop hands-on experience with enterprise Microsoft identity, endpoint management, and hybrid cloud integration. The lab evolved from a basic Active Directory environment into a realistic hybrid enterprise setup covering on-premises identity, cloud identity sync, device management, and endpoint security.
 
---
 
## What's Running in the Lab
 
- Windows Server 2019 domain controller (`DC01`) with AD DS and DNS
- Windows 10 client (`WIN10-01`) — domain-joined and Intune managed
- VMware Workstation with NAT networking (VMnet8)
- Microsoft Entra ID tenant with Azure AD Connect sync (Password Hash Sync)
- Microsoft 365 E5 trial licensing
- Hybrid Microsoft Entra joined device
- Conditional Access and MFA for admin accounts
- Intune compliance policy, BitLocker encryption, and Defender policy
 
---
 
## Lab Phases
 
### Phase 1 — Lab Foundation and Core Infrastructure
 
**Built:**
- VMware Workstation as the hypervisor platform using NAT networking (VMnet8)
- `DC01` — Windows Server 2019 promoted to domain controller for `lab.local`
- `WIN10-01` — Windows 10 client configured to use `DC01` for DNS
- Static IP design for the DC; DNS hosted on the domain controller
 
**Key concept learned:**
 
Active Directory depends heavily on DNS. The DC is not just an authentication server — it is also the DNS server. Clients use DNS to find the DC, Kerberos and domain logons rely on DNS SRV and A records, and Group Policy relies on DNS and SYSVOL access. Hybrid sync later required DNS forwarders for external resolution.
 
---
 
### Phase 2 — OU Structure, Users, Groups, and Role Design
 
**Built:**
- Top-level `LAB` OU with sub-OUs: `Users`, `Admins`, `Groups`, `Computers`
- Groups: `LAB-Users`, `LAB-IT-Admins`
- Users: `test.user` (standard), `lab.admin` (named admin)
- Group-based RBAC model — avoided relying on the built-in Administrator account
 
**Key concept learned:**
 
> OUs are for policy targeting and administrative organization. Groups are for permissions and access control.
 
This distinction is fundamental to enterprise AD design.
 
---
 
### Phase 3 — DNS Troubleshooting, Domain Join, and Identity Validation
 
**Built:**
- Successfully joined `WIN10-01` to `lab.local`
- Validated domain authentication using `whoami`, `whoami /groups`, `echo %logonserver%`, `nslookup`, `ipconfig /registerdns`
 
**Major issue resolved — DNS failures:**
 
| Symptom | Root Cause | Fix |
|---|---|---|
| `nslookup lab.local` timed out | DC had no DNS forwarders configured | Added forwarders (8.8.8.8 / 1.1.1.1) |
| Client/DC on different subnets | VMware NAT addressing mismatch | Aligned static IPs to VMnet8 subnet |
| Inconsistent name resolution | Client DNS not pointed solely to DC | Corrected client DNS settings |
 
> The client could reach the DC by IP but name resolution failed because the DC had no DNS forwarders. After setting the default gateway and forwarders, the environment resolved both internal and external names correctly.
 
---
 
### Phase 4 — Group Policy Foundations
 
**Built:**
- `LAB-User-Baseline` GPO: blocked Control Panel, command prompt, and registry tools
- `LAB-Computer-Baseline` GPO for computer-level settings
- Verified GPO application using `gpupdate /force`, `gpresult /r`, `rsop.msc`
 
**Key concept learned:**
 
User policies apply based on the **user object's OU**. Computer policies apply based on the **computer object's OU**. Unlinking a GPO does **not** delete the GPO object — the object must be deleted separately from the Group Policy Objects container.
 
---
 
### Phase 5 — Transition Toward Hybrid Identity
 
**Built:**
- Installed and configured Azure AD Connect provisioning agent
- Scoped sync to `LAB\Users` OU
- Created a native Entra admin account (Microsoft-account-backed identity was insufficient for sync configuration)
- Later replaced Cloud Sync with **Azure AD Connect / Entra Connect** to support full hybrid device join scenarios
 
**Architectural decision:**
 
Cloud Sync is sufficient for user/group sync. Azure AD Connect is required for a realistic hybrid enterprise model including hybrid device join. This is an important distinction for enterprise identity architecture discussions.
 
---
 
### Phase 6 — Microsoft Entra Tenant, Sync, UPNs, MFA, and Conditional Access
 
**Built:**
- Entra tenant created and validated
- Azure AD Connect configured with Password Hash Sync and OU scoping
- Users synced with `On-premises sync enabled = Yes` and correct cloud UPNs
- Conditional Access and MFA configured for admin accounts
- Break-glass account created and excluded from Conditional Access
- Security Defaults disabled in favor of enterprise-style Conditional Access
 
**Key issues resolved:**
 
- `.local` domains are not internet-routable and cannot be used as verified Entra custom domains — UPNs needed to be mapped to a routable suffix
- Security Defaults enforce MFA-like behavior independently of Conditional Access — disabling Security Defaults is required for enterprise-style CA-only control
- Excluding an account from Conditional Access does not exempt it from tenant-level admin baseline protections
 
---
 
### Phase 7 — Hybrid Device Identity and Intune Enrollment
 
**Built:**
- Hybrid Microsoft Entra join configured via Azure AD Connect device options
- `WIN10-01` confirmed hybrid joined: `AzureAdJoined: YES`, `DomainJoined: YES`, `AzureAdPrt: YES`
- Intune auto-enrollment enabled and verified
 
**Major issue resolved — device hybrid joined but not appearing in Intune:**
 
Troubleshooting steps included:
- Confirmed MDM user scope and license assignment
- Verified GPO application with `gpresult /scope computer /r`
- Checked enrollment scheduled tasks and policy registry keys
- Manually triggered enrollment using `deviceenroller.exe`
 
> Hybrid join and Intune enrollment are separate processes. GPO-based MDM auto-enrollment is a computer policy — scope, license, and scheduled task execution all matter.
 
---
 
### Phase 8 — Intune Compliance, BitLocker, and Defender
 
**Built:**
- Windows compliance policy evaluating BitLocker, Secure Boot, Firewall/Defender, and minimum OS version
- `WIN10-01` status: Compliant | Managed by Intune | Ownership: Corporate
- Microsoft Defender Antivirus policy via Intune (validated with `Get-MpComputerStatus`, `Get-MpPreference`)
 
**Major issue resolved — BitLocker in a VM without TPM:**
 
| Problem | Root Cause | Fix |
|---|---|---|
| BitLocker did not auto-enable | VMware VM has no TPM by default | Enabled "Allow BitLocker without TPM" in Group Policy |
| No recovery key in Entra | Encryption never started | Added startup password protector, then recovery password protector manually |
| Recovery key not escrowed | Manual escrow required | Backed up recovery key to Entra using Azure backup mechanism |
 
Final state: 100% encrypted | XTS-AES 256 | Protection On | Recovery key visible in Entra/Intune
 
> This is an important distinction: `-adbackup` targets on-premises AD DS, while Entra/Azure key escrow is a separate mechanism. Understanding both is relevant in modern hybrid environments.
 
---
 
## Current Lab State
 
| Area | Details |
|---|---|
| Infrastructure | DC01 (Windows Server 2019), WIN10-01 (Windows 10), VMware NAT |
| On-Prem Identity | `lab.local` AD with OUs, users, groups, GPOs |
| Cloud Identity | Entra tenant, Azure AD Connect sync, hybrid UPNs |
| Security | Conditional Access, MFA, break-glass account, Security Defaults disabled |
| Endpoint Management | Hybrid Entra joined, Intune managed, compliant, BitLocker encrypted, Defender policy applied |
 
---
 
## Planned Next Steps
 
- Add a second domain controller (`DC02`) to practice AD replication, DNS redundancy, and failover
- Simulate a full employee onboarding flow with a new device (`WIN10-02`) — user creation, license assignment, hybrid join, and Intune enrollment
- Build structured troubleshooting scenarios: password sync issues, Conditional Access blocks, BitLocker recovery, DNS failures, GPO application issues, and DC outage/replication problems
 
---
 
## Technologies Used
 
`Windows Server 2019` `Active Directory` `DNS` `Group Policy` `VMware Workstation` `Microsoft Entra ID` `Azure AD Connect` `Entra Connect` `Microsoft 365 E5` `Conditional Access` `MFA` `Microsoft Intune` `BitLocker` `Microsoft Defender` `Hybrid Azure AD Join` `PowerShell`
 
