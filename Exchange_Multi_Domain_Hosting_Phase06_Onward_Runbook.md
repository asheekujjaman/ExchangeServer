# Exchange Multi-Domain Email Hosting
## Customer Segmentation & Delegated Administration Runbook

**Scope:** Phase 06 onward — Address Book Isolation, ABP Routing, Customer ECP/RBAC delegation, validation and future customer onboarding  
**Environment:** Lab implementation on one Active Directory forest and one Exchange organization  
**Last updated:** 20 November 2026

---

## 1. Purpose and scope

This runbook documents the implementation performed from Phase 06 onward for the shared Exchange hosting lab. It includes the actual configuration, GUI paths, Exchange Management Shell commands, verification steps and remaining work.

The design uses:

- One AD forest: `bpatcl.com`
- One Exchange organization: `mail2.bptcl.com`
- Multiple customer email domains
- A customer OU in the central AD for each hosted customer
- Customer-specific email address policies, address lists, GALs, OABs and ABPs
- Universal Security Groups mapped to OU-scoped Exchange RBAC assignments

This runbook does **not** repeat the earlier implementation of the AD forest, Accepted Domains and initial Email Address Policies except where needed as prerequisites.

## 2. Current environment

| Component | Current value |
|---|---|
| AD forest/domain | `bpatcl.com` |
| Domain controller | `hvdc.bpatcl.com` |
| DC IP | `10.10.10.90` |
| Exchange server | `mail2.bptcl.com` |
| Exchange IP | `10.10.10.41` |
| Exchange platform | Exchange Server 2019 (lab) |
| HM customer email domain | `hmlocal.com` |
| BPATCL customer email domain | `bpatcl.com` |
| HM OU | `OU=hm,OU=ExMail,DC=bpatcl,DC=com` |
| BPATCL OU | `OU=bpatcl,OU=ExMail,DC=bpatcl,DC=com` |
| Parent customer OU | `OU=ExMail,DC=bpatcl,DC=com` |

### 2.1 Existing Exchange prerequisites verified earlier

| Setting | Value / status |
|---|---|
| Accepted Domain `bpatcl.com` | Authoritative; Default = True |
| Accepted Domain `hmlocal.com` | Authoritative; Default = False |
| Default Email Address Policy | `SMTP:@bpatcl.com`; Priority = Lowest |
| HM Email Address Policy | `EAP-HMLocal`; scope = `ExMail/hm`; template = `SMTP:%m@hmlocal.com`; Priority = 1 |
| HM UPN suffix | `hmlocal.com` |

The Default Email Address Policy was left unchanged. The HM-specific policy was created for the HM OU.

### 2.2 Current OU layout

```text
bpatcl.com
└── ExMail
    ├── hm
    │   ├── HM mailbox users
    │   └── sg-hm-admin
    ├── bpatcl
    │   ├── BPATCL mailbox users
    │   └── sg-bpatcl-admin
    └── sg-admin
```

The three security groups are located in the OUs shown above. Customer mailbox users are stored in their own customer OU.

### 2.3 Current mailbox inventory (latest supplied output)

| Customer | Mailbox count | Expected primary SMTP suffix | ABP |
|---|---:|---|---|
| HM | 14 | `@hmlocal.com` | `ABP-HM` |
| BPATCL | 12 | `@bpatcl.com` | `ABP-BPATCL` |
| **Total** | **26** | | |

The counts include temporary/test mailboxes created during ECP testing. Review and remove only confirmed disposable test accounts during a later cleanup; do not delete production/customer accounts based only on their display names.

---

# Phase 06 — Customer Address Book Isolation

## 3. Design

Each mailbox is tagged with `CustomAttribute15` so that Exchange can build customer-specific address lists and GALs.

| Customer | `CustomAttribute15` value | Address List | GAL | OAB | ABP |
|---|---|---|---|---|---|
| HM | `HM` | `AL-HM-All` | `GAL-HM` | `OAB-HM` | `ABP-HM` |
| BPATCL | `BPATCL` | `AL-BPATCL-All` | `GAL-BPATCL` | `OAB-BPATCL` | `ABP-BPATCL` |

Room lists are also created for future use:

- `AL-HM-Rooms`
- `AL-BPATCL-Rooms`

The ABP ties together the customer's Address Lists, GAL, OAB and Room List. Every customer mailbox must be assigned the correct ABP.

### 3.1 Important meaning of CustomAttribute15

`CustomAttribute15` is Exchange recipient metadata. It is not a login name, UPN, email address, password or AD OU. In this implementation it labels each recipient's customer:

- HM mailbox: `CustomAttribute15 = HM`
- BPATCL mailbox: `CustomAttribute15 = BPATCL`

The Address List and GAL filters use this value. A new mailbox can have the correct UPN, SMTP address, Email Address Policy and ABP but still be absent from the customer GAL if `CustomAttribute15` is blank.

**Operational rule:** after creating a mailbox, set Custom Attribute 15 in EAC or Exchange Management Shell and verify it.

## 4. Set customer identifier on mailboxes

### 4.1 GUI — Exchange Admin Center

For each mailbox:

1. Open EAC: `https://mail2.bptcl.com/ecp` (or the lab URL/alias configured for EAC).
2. Go to **Recipients → Mailboxes**.
3. Select the mailbox and choose **Edit**.
4. Open **More options / Custom attributes** (label can vary by Exchange build/EAC view).
5. Set **Custom attribute 15**:
   - HM: `HM`
   - BPATCL: `BPATCL`
6. Save.

For multiple existing mailboxes, select the relevant mailboxes and use **Bulk Edit → Custom attributes → Custom attribute 15**, if available in the EAC view.

### 4.2 CLI — Exchange Management Shell

HM existing mailboxes:

```powershell
$HM = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/hm" -ResultSize Unlimited
$HM | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -CustomAttribute15 "HM"
}
```

BPATCL existing mailboxes:

```powershell
$BP = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/bpatcl" -ResultSize Unlimited
$BP | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -CustomAttribute15 "BPATCL"
}
```

Single mailbox example:

```powershell
Set-Mailbox -Identity "hmuser101" -CustomAttribute15 "HM"
Set-Mailbox -Identity "buser102" -CustomAttribute15 "BPATCL"
```

Verify:

```powershell
Get-Mailbox -ResultSize Unlimited |
Where-Object { $_.CustomAttribute15 -in @("HM","BPATCL") } |
Format-Table Name,PrimarySmtpAddress,CustomAttribute15
```

Expected latest tagged inventory was 11 HM + 11 BPATCL at the earlier check. Three additional HM mailboxes and one additional BPATCL mailbox were subsequently created through ECP. The user confirmed setting the three missing HM Custom Attribute 15 values through ECP, but a final read-back of those three values was not supplied. Re-run the verification command to close that check.

## 5. Create customer Address Lists

The following custom OPATH filters are used:

- HM: `CustomAttribute15 -eq 'HM'`
- BPATCL: `CustomAttribute15 -eq 'BPATCL'`

### 5.1 GUI

EAC supports Address List creation under **Organization → Address lists** for the filter options exposed by that EAC version. For this implementation, the exact `CustomAttribute15` custom recipient filters were created with Exchange Management Shell. Use EMS when the GUI cannot express the required custom filter exactly.

### 5.2 CLI — create lists

```powershell
New-AddressList -Name "AL-HM-All" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'HM'))"

New-AddressList -Name "AL-HM-Rooms" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'HM') -and ((RecipientDisplayType -eq 'ConferenceRoomMailbox') -or (RecipientDisplayType -eq 'SyncedConferenceRoomMailbox')))"

New-AddressList -Name "AL-BPATCL-All" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'BPATCL'))"

New-AddressList -Name "AL-BPATCL-Rooms" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'BPATCL') -and ((RecipientDisplayType -eq 'ConferenceRoomMailbox') -or (RecipientDisplayType -eq 'SyncedConferenceRoomMailbox')))"
```

These objects were created successfully in the lab. If rerunning in an existing environment, first check whether each name already exists; do not create duplicates.

Verify:

```powershell
Get-AddressList |
Where-Object { $_.Name -like "AL-HM-*" -or $_.Name -like "AL-BPATCL-*" } |
Format-Table Name,RecipientFilterApplied
```

Update membership:

```powershell
Update-AddressList -Identity "AL-HM-All"
Update-AddressList -Identity "AL-HM-Rooms"
Update-AddressList -Identity "AL-BPATCL-All"
Update-AddressList -Identity "AL-BPATCL-Rooms"
```

## 6. Create customer Global Address Lists

In Exchange Server, GAL creation and updates are performed using Exchange Management Shell.

### 6.1 CLI

```powershell
New-GlobalAddressList -Name "GAL-HM" -RecipientFilter "CustomAttribute15 -eq 'HM'"
New-GlobalAddressList -Name "GAL-BPATCL" -RecipientFilter "CustomAttribute15 -eq 'BPATCL'"
```

Update:

```powershell
Update-GlobalAddressList -Identity "GAL-HM"
Update-GlobalAddressList -Identity "GAL-BPATCL"
```

Verify GAL objects:

```powershell
Get-GlobalAddressList | Format-Table Name,RecipientFilterApplied
```

Verify membership with recipient preview:

```powershell
$GAL = Get-GlobalAddressList -Identity "GAL-HM"
Get-Recipient -ResultSize Unlimited -RecipientPreviewFilter $GAL.RecipientFilter |
Format-Table Name,PrimarySmtpAddress

$GAL = Get-GlobalAddressList -Identity "GAL-BPATCL"
Get-Recipient -ResultSize Unlimited -RecipientPreviewFilter $GAL.RecipientFilter |
Format-Table Name,PrimarySmtpAddress
```

**Observed result:** 11 HM recipients and 11 BPATCL recipients were shown at the time of the GAL test. Later ECP-created mailboxes increased the OU inventory; ensure each new mailbox has CustomAttribute15 set and refresh the relevant Address List/GAL membership.

## 7. Create customer Offline Address Books

### 7.1 CLI

```powershell
New-OfflineAddressBook -Name "OAB-HM" -AddressLists "GAL-HM" -GlobalWebDistributionEnabled $true -IsDefault $false

New-OfflineAddressBook -Name "OAB-BPATCL" -AddressLists "GAL-BPATCL" -GlobalWebDistributionEnabled $true -IsDefault $false
```

The existing **Default Offline Address Book** remains the default. Do not replace it with a customer-specific OAB.

Verify:

```powershell
Get-OfflineAddressBook |
Format-List Name,AddressLists,GeneratingMailbox,IsDefault,VirtualDirectories,GlobalWebDistributionEnabled
```

Verified lab result:

- `OAB-HM` → `GAL-HM`; `IsDefault=False`; `GlobalWebDistributionEnabled=True`
- `OAB-BPATCL` → `GAL-BPATCL`; `IsDefault=False`; `GlobalWebDistributionEnabled=True`
- Existing Default OAB remains `IsDefault=True`

### 7.2 GUI

The OAB objects in this lab were created using EMS. In EAC, the relevant OAB is selected as part of the mailbox's Address Book Policy; creating and configuring these customer OABs was done by EMS for this exact setup.

## 8. Create customer Address Book Policies

### 8.1 CLI

```powershell
New-AddressBookPolicy -Name "ABP-HM" -AddressLists "AL-HM-All" -GlobalAddressList "GAL-HM" -OfflineAddressBook "OAB-HM" -RoomList "AL-HM-Rooms"

New-AddressBookPolicy -Name "ABP-BPATCL" -AddressLists "AL-BPATCL-All" -GlobalAddressList "GAL-BPATCL" -OfflineAddressBook "OAB-BPATCL" -RoomList "AL-BPATCL-Rooms"
```

Verify:

```powershell
Get-AddressBookPolicy |
Format-List Name,AddressLists,GlobalAddressList,OfflineAddressBook,RoomList
```

### 8.2 Assign ABPs to mailboxes

HM:

```powershell
$HM = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/hm" -ResultSize Unlimited
$HM | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -AddressBookPolicy "ABP-HM"
}
```

BPATCL:

```powershell
$BP = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/bpatcl" -ResultSize Unlimited
$BP | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -AddressBookPolicy "ABP-BPATCL"
}
```

Verify:

```powershell
Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/hm" -ResultSize Unlimited |
Format-Table Name,PrimarySmtpAddress,AddressBookPolicy

Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/bpatcl" -ResultSize Unlimited |
Format-Table Name,PrimarySmtpAddress,AddressBookPolicy
```

### 8.3 GUI — mailbox ABP assignment

For an individual mailbox:

1. EAC → **Recipients → Mailboxes**.
2. Select the mailbox → **Edit**.
3. Open **Mailbox** / **Manage mailbox policies** (wording depends on EAC view).
4. Set Address Book Policy to `ABP-HM` or `ABP-BPATCL`.
5. Save.

For bulk assignment, select the intended mailboxes and use the EAC **Mailbox policies** bulk-edit action if available in the installed Exchange build.

**Important:** New mailboxes are not automatically assigned an ABP unless the creation workflow explicitly selects it or an administrator assigns it afterward. Verify the `AddressBookPolicy` property for every newly created mailbox.

## 9. Install and enable the ABP Routing Agent

The agent is required so recipients from different GALs are resolved as external-style recipients and their contact cards are not exposed across ABPs. ABP routing does not block email delivery.

### 9.1 Check agent DLL

Exchange Management Shell on `mail2.bptcl.com`:

```powershell
$AgentPath = "$env:ExchangeInstallPath\TransportRoles\agents\AddressBookPolicyRoutingAgent\Microsoft.Exchange.Transport.Agent.AddressBookPolicyRoutingAgent.dll"
Test-Path $AgentPath
```

Observed result: `True`.

### 9.2 Check registration

```powershell
Get-TransportAgent -Identity "ABP Routing Agent" -ErrorAction SilentlyContinue |
Format-List Identity,Enabled,Priority,AssemblyPath
```

Initially, the agent was not registered.

### 9.3 Install and enable

```powershell
Install-TransportAgent -Name "ABP Routing Agent" -TransportAgentFactory "Microsoft.Exchange.Transport.Agent.AddressBookPolicyRoutingAgent.AddressBookPolicyRoutingAgentFactory" -AssemblyPath "$env:ExchangeInstallPath\TransportRoles\agents\AddressBookPolicyRoutingAgent\Microsoft.Exchange.Transport.Agent.AddressBookPolicyRoutingAgent.dll"

Enable-TransportAgent -Identity "ABP Routing Agent"
```

The installation and enable commands require an MSExchangeTransport service restart before the agent takes effect.

### 9.4 Check queue, restart transport and enable routing

Before restart:

```powershell
Get-Queue | Format-Table Identity,Status,MessageCount,NextHopDomain
```

Observed lab queue: `mail2\Submission`, `MessageCount=0`.

Restart during a suitable maintenance window (even in a lab, this briefly interrupts SMTP transport processing):

```powershell
Restart-Service MSExchangeTransport
Get-Service MSExchangeTransport
```

Enable organization-wide ABP routing:

```powershell
Set-TransportConfig -AddressBookPolicyRoutingEnabled $true
```

Verify:

```powershell
Get-TransportAgent -Identity "ABP Routing Agent" |
Format-List Identity,Enabled,Priority,AssemblyPath

Get-Service MSExchangeTransport

Get-TransportConfig |
Format-List AddressBookPolicyRoutingEnabled
```

Verified lab state:

- ABP Routing Agent: `Enabled=True`, Priority `10`
- `MSExchangeTransport`: Running
- `AddressBookPolicyRoutingEnabled=True`

Microsoft notes that changing `AddressBookPolicyRoutingEnabled` can take up to 30 minutes to take effect.

## 10. Phase 06 functional tests and observed result

The following tests were performed in OWA:

| Test | Observed result |
|---|---|
| HM user opens People/Directory | HM address lists shown (`AL-HM-All`, `AL-HM-Rooms`) |
| BPATCL user opens People/Directory | BPATCL address lists shown (`AL-BPATCL-All`, `AL-BPATCL-Rooms`) |
| HM user types BPATCL recipient | Recipient resolves by SMTP address; other tenant's display name/contact details are not exposed in the screenshot |
| Cross-customer email delivery | HM and BPATCL mailbox screenshots showed cross-customer messages in inbox/sent items |

Expected behavior with ABP routing enabled: recipients assigned to different GALs appear as external-style recipients to one another; the agent limits contact-card visibility. It does **not** prevent one customer from sending email to another customer.

**Phase 06 status:** Configuration completed and OWA directory segmentation/cross-customer recipient behavior observed successfully. Re-run GAL membership checks after new mailbox creation or attribute changes.

---

# Phase 07 — Customer ECP Administration with Exchange RBAC

## 11. Administration model

Three Universal Security Groups represent the customer administrator roles:

| Security Group | AD location | Members observed | Scope |
|---|---|---|---|
| `sg-hm-admin` | `OU=hm,OU=ExMail,DC=bpatcl,DC=com` | `hmuser01` | HM customer OU |
| `sg-bpatcl-admin` | `OU=bpatcl,OU=ExMail,DC=bpatcl,DC=com` | `buser01` | BPATCL customer OU |
| `sg-admin` | `OU=ExMail,DC=bpatcl,DC=com` | `mailadmin` | All hosted customer OUs under ExMail |

All three groups were verified as:

- `GroupScope = Universal`
- `GroupCategory = Security`
- Mail-enabled in Exchange
- Hidden from address lists (`HiddenFromAddressListsEnabled=True`)

### 11.1 Create groups — GUI

On `hvdc.bpatcl.com`, open **Active Directory Users and Computers** (`dsa.msc`). In the target OU, right-click → **New → Group**:

| Group | OU | Group scope | Group type |
|---|---|---|---|
| sg-hm-admin | ExMail/hm | Universal | Security |
| sg-bpatcl-admin | ExMail/bpatcl | Universal | Security |
| sg-admin | ExMail | Universal | Security |

Add customer user accounts as members using the group's **Members** tab. A customer mailbox user may also be a customer administrator if deliberately added to the appropriate group.

### 11.2 Create groups — AD PowerShell (reference for future deployment)

Run on a DC or a management host with the ActiveDirectory module. Check for existing groups before running to avoid duplicates.

```powershell
New-ADGroup -Name "sg-hm-admin" -SamAccountName "sg-hm-admin" -GroupScope Universal -GroupCategory Security -Path "OU=hm,OU=ExMail,DC=bpatcl,DC=com" -Description "Exchange administration for HM"

New-ADGroup -Name "sg-bpatcl-admin" -SamAccountName "sg-bpatcl-admin" -GroupScope Universal -GroupCategory Security -Path "OU=bpatcl,OU=ExMail,DC=bpatcl,DC=com" -Description "Exchange administration for BPATCL"

New-ADGroup -Name "sg-admin" -SamAccountName "sg-admin" -GroupScope Universal -GroupCategory Security -Path "OU=ExMail,DC=bpatcl,DC=com" -Description "Central administration for hosted customers"
```

Membership examples:

```powershell
Add-ADGroupMember -Identity "sg-hm-admin" -Members "hmuser01"
Add-ADGroupMember -Identity "sg-bpatcl-admin" -Members "buser01"
Add-ADGroupMember -Identity "sg-admin" -Members "mailadmin"
```

To promote or demote a customer admin, add/remove the account from the relevant group. Do not add nested groups; use direct user membership.

### 11.3 Mail-enable existing security groups — Exchange Management Shell

The groups were mail-enabled with `Enable-DistributionGroup` and hidden from address lists:

```powershell
Enable-DistributionGroup -Identity "sg-hm-admin"
Enable-DistributionGroup -Identity "sg-bpatcl-admin"
Enable-DistributionGroup -Identity "sg-admin"

Set-DistributionGroup -Identity "sg-hm-admin" -HiddenFromAddressListsEnabled $true
Set-DistributionGroup -Identity "sg-bpatcl-admin" -HiddenFromAddressListsEnabled $true
Set-DistributionGroup -Identity "sg-admin" -HiddenFromAddressListsEnabled $true
```

Verify group properties:

```powershell
Get-DistributionGroup |
Where-Object { $_.Name -in @("sg-hm-admin","sg-bpatcl-admin","sg-admin") } |
Format-Table Name,PrimarySmtpAddress,GroupType,HiddenFromAddressListsEnabled
```

Verify membership using **Get-DistributionGroupMember** because these are mail-enabled security groups, not Exchange role groups:

```powershell
Get-DistributionGroupMember -Identity "sg-hm-admin" |
Format-Table Name,PrimarySmtpAddress,RecipientType

Get-DistributionGroupMember -Identity "sg-bpatcl-admin" |
Format-Table Name,PrimarySmtpAddress,RecipientType

Get-DistributionGroupMember -Identity "sg-admin" |
Format-Table Name,PrimarySmtpAddress,RecipientType
```

## 12. OU-scoped RBAC assignments

The implementation uses two Exchange management roles for each security group:

- `Mail Recipients`
- `Mail Recipient Creation`

The role assignments are scoped to the relevant OU using `RecipientOrganizationalUnitScope`.

### 12.1 Create the six assignments — Exchange Management Shell

```powershell
# HM
New-ManagementRoleAssignment -Name "HM-Mail-Recipients" -Role "Mail Recipients" -SecurityGroup "sg-hm-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/hm"

New-ManagementRoleAssignment -Name "HM-Mail-Recipient-Creation" -Role "Mail Recipient Creation" -SecurityGroup "sg-hm-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/hm"

# BPATCL
New-ManagementRoleAssignment -Name "BPATCL-Mail-Recipients" -Role "Mail Recipients" -SecurityGroup "sg-bpatcl-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/bpatcl"

New-ManagementRoleAssignment -Name "BPATCL-Mail-Recipient-Creation" -Role "Mail Recipient Creation" -SecurityGroup "sg-bpatcl-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/bpatcl"

# Central administrator: all customer recipients beneath ExMail
New-ManagementRoleAssignment -Name "Central-Mail-Recipients" -Role "Mail Recipients" -SecurityGroup "sg-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail"

New-ManagementRoleAssignment -Name "Central-Mail-Recipient-Creation" -Role "Mail Recipient Creation" -SecurityGroup "sg-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail"
```

These assignments were created and verified. The `CustomRecipientWriteScope` property showed the expected OU paths.

### 12.2 Verify assignments

```powershell
Get-ManagementRoleAssignment |
Where-Object {
    $_.Name -in @(
        "HM-Mail-Recipients",
        "HM-Mail-Recipient-Creation",
        "BPATCL-Mail-Recipients",
        "BPATCL-Mail-Recipient-Creation",
        "Central-Mail-Recipients",
        "Central-Mail-Recipient-Creation"
    )
} |
Format-Table Name,Role,RoleAssigneeName,RecipientWriteScope,CustomRecipientWriteScope -AutoSize
```

### 12.3 GUI equivalents and limitations

- **AD group membership:** manageable through ADUC → group Properties → Members.
- **Mailbox creation and recipient management:** customer admins use ECP at `https://mail2.bptcl.com/ecp` (or the configured lab alias).
- **Custom OU-scoped role assignments:** the exact `RecipientOrganizationalUnitScope` assignments in this runbook were configured through Exchange Management Shell. Do not assume that assigning a group to a built-in broad EAC role group is equivalent; it can grant organization-wide access.

## 13. ECP customer workflow

### 13.1 HM administrator

- Sign in with an account that is a member of `sg-hm-admin`.
- Open **Recipients → Mailboxes**.
- To create a mailbox, choose **+ → User mailbox**.
- In the **Organizational Unit** field, click **Browse** and select `bpatcl.com → ExMail → hm`.
- Use the HM UPN suffix `hmlocal.com`.
- Select `ABP-HM` in the Address Book Policy field if the form exposes it.
- Save the mailbox.
- Edit the new mailbox and set **Custom attribute 15 = HM**.
- Verify the primary SMTP address and policy values after creation.

### 13.2 BPATCL administrator

Use the same ECP workflow, but select:

- OU: `bpatcl.com → ExMail → bpatcl`
- UPN/email suffix: `bpatcl.com`
- Address Book Policy: `ABP-BPATCL`
- Custom attribute 15: `BPATCL`

### 13.3 Central administrator

The `sg-admin` scope is `bpatcl.com/ExMail`. The central admin can select the relevant child customer OU (`hm` or `bpatcl`) when creating a mailbox.

### 13.4 Why the correct OU must be selected

An attempted creation initially failed with an error similar to:

```text
'bpatcl.com/Users/<new user>' isn't within your current write scopes.
Can't perform save operation.
```

This happened because ECP defaulted to the Users container, outside the administrator's OU write scope. Select the customer OU using **Browse** before saving. This is expected RBAC behavior, not an Exchange mail-flow fault.

### 13.5 New mailbox post-creation checklist

For every newly created mailbox, verify all of the following:

| Item | HM | BPATCL |
|---|---|---|
| AD OU | `ExMail/hm` | `ExMail/bpatcl` |
| UPN / sign-in suffix | `@hmlocal.com` | `@bpatcl.com` |
| Primary SMTP | `@hmlocal.com` | `@bpatcl.com` |
| Email Address Policy | `EAP-HMLocal` | Default Policy |
| Address Book Policy | `ABP-HM` | `ABP-BPATCL` |
| CustomAttribute15 | `HM` | `BPATCL` |

CustomAttribute15 must be set explicitly; it is not inherited automatically from the OU.

Verification example:

```powershell
Get-Mailbox -Identity "hmuser101" |
Format-List Name,UserPrincipalName,PrimarySmtpAddress,EmailAddressPolicyEnabled,AddressBookPolicy,CustomAttribute15

Get-Mailbox -Identity "buser102" |
Format-List Name,UserPrincipalName,PrimarySmtpAddress,EmailAddressPolicyEnabled,AddressBookPolicy,CustomAttribute15
```

## 14. ECP/RBAC test results recorded

The user tested the three administrator accounts through ECP:

| Test | Observed result |
|---|---|
| HM Admin ECP login | Successful |
| BPATCL Admin ECP login | Successful |
| Central Admin ECP login | Successful |
| HM Admin create mailbox in HM OU | Successful after selecting the correct OU |
| BPATCL Admin create mailbox in BPATCL OU | Successful after selecting the correct OU |
| HM Admin modify BPATCL recipient | Blocked by write scope |
| BPATCL Admin modify HM recipient | Blocked by write scope |
| Central Admin manage both customer OUs | Successful |
| Other customer recipients visible in ECP | Yes |

### 14.1 Critical limitation: ECP read visibility

The current implementation gives **OU-scoped write/management permissions**, not full tenant isolation.

- Customer admins can see recipients from other customer OUs in the ECP recipient list.
- They cannot modify recipients outside their assigned write scope (verified by the user).
- ABP controls the address book view in Outlook/OWA; it does not filter every ECP administrative list.
- The `Mail Recipients` and `Mail Recipient Creation` roles have read-scope behavior separate from the OU write scope. The current output does not prove customer-only read visibility.

This limitation is accepted in the current shared Exchange organization design. Do not describe the design as complete data-plane or administrative read isolation. If hiding all other-customer recipient records in ECP is an absolute requirement, this single-organization native ECP model does not meet that requirement by itself.

### 14.2 Existing built-in role group audit

Observed results:

- `Organization Management`: member `Administrator`
- `Recipient Management`: no members shown
- `View-Only Organization Management`: no members shown

The three customer security groups are not Exchange role groups. To view their membership use `Get-DistributionGroupMember`, not `Get-RoleGroupMember`.

---

# Phase 08 — Password Change and Final Validation

## 15. Forced password change test

The ECP **Require password change on next logon** option was tested.

Observed behavior:

- The user was presented with the Outlook/OWA password-change page.
- Entering the UPN in the `Domain\user name` field returned a username/password error.
- Using the AD NetBIOS domain plus `sAMAccountName` succeeded.

For this lab, the working forced-change identity format was:

```text
BPATCL\hmuser03
```

The normal OWA sign-in UPN remains:

```text
hmuser03@hmlocal.com
```

For a BPATCL mailbox, use the same pattern with its SAM account name, for example `BPATCL\buser03`.

**Note:** The successful forced-change test proves the user can change an expired/required-change password using the NetBIOS\SAM format. It does not, by itself, prove that the customer administrator's ECP password-reset permission has been tested.

## 16. Final verification commands

### 16.1 Mailbox inventory by customer

```powershell
Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/hm" -ResultSize Unlimited |
Format-Table Name,UserPrincipalName,PrimarySmtpAddress,EmailAddressPolicyEnabled,AddressBookPolicy,CustomAttribute15

Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/bpatcl" -ResultSize Unlimited |
Format-Table Name,UserPrincipalName,PrimarySmtpAddress,EmailAddressPolicyEnabled,AddressBookPolicy,CustomAttribute15
```

### 16.2 Customer tag counts

```powershell
(Get-Mailbox -ResultSize Unlimited -Filter "CustomAttribute15 -eq 'HM'").Count
(Get-Mailbox -ResultSize Unlimited -Filter "CustomAttribute15 -eq 'BPATCL'").Count
```

### 16.3 ABP and routing state

```powershell
Get-AddressBookPolicy |
Format-List Name,AddressLists,GlobalAddressList,OfflineAddressBook,RoomList

Get-TransportAgent -Identity "ABP Routing Agent" |
Format-List Identity,Enabled,Priority,AssemblyPath

Get-TransportConfig |
Format-List AddressBookPolicyRoutingEnabled
```

### 16.4 RBAC assignments

```powershell
Get-ManagementRoleAssignment |
Where-Object { $_.RoleAssigneeName -in @("sg-hm-admin","sg-bpatcl-admin","sg-admin") } |
Format-Table Name,Role,RoleAssigneeName,CustomRecipientWriteScope -AutoSize
```

---

## 17. Remaining work — not yet verified as complete

The following items were not fully tested in the recorded implementation and should remain open until tested:

1. **Customer admin password reset:** confirm that HM Admin can reset an HM user's password and cannot reset a BPATCL user's password; perform the reverse test for BPATCL Admin.
2. **Disable mailbox/user:** test only with disposable accounts, then confirm cross-customer disable is blocked.
3. **Delete mailbox/user:** test only with disposable accounts, then confirm cross-customer deletion is blocked. Do not use the real admin accounts (`mailadmin`, `hmuser01`, `buser01`) for destructive tests.
4. **Admin-group self-membership protection:** verify that a customer admin cannot add/remove members of `sg-hm-admin` or `sg-bpatcl-admin`, or otherwise grant itself additional privileges. Keep control of group membership with the central administrator unless a separate secure delegation is designed.
5. **New mailbox tag automation:** current ECP mailbox creation does not automatically populate CustomAttribute15. Continue setting it through EAC Custom Attribute 15 or automate the onboarding workflow, then verify the GAL includes the new mailbox.
6. **Final read-back of the three extra HM CustomAttribute15 values:** the user applied `HM` in ECP for the previously blank accounts, but a final PowerShell read-back was not supplied.
7. **Public Internet mail flow:** the recorded MX/A checks were lab/internal DNS tests resolving the mail host to a private IP. Public MX, public A record, NAT, SPF, DKIM, DMARC and external delivery have not been verified in this runbook.
8. **Production security and lifecycle:** the environment is a lab. Exchange Server 2019 reached end of support on 14 October 2025. Before production hosting, plan migration to Exchange Server Subscription Edition or another supported platform, and review current security updates and support status.

## 18. New customer onboarding checklist

For each future customer, repeat the following pattern. Replace `<CUSTOMER>` and `<DOMAIN>` with the customer's actual short code and email domain.

1. Confirm the customer's domain is an Exchange Accepted Domain (Authoritative if all recipients for that domain are hosted here).
2. Add `<DOMAIN>` as an AD UPN suffix if customer users must sign in using that email-style UPN.
3. Create `OU=<CUSTOMER>,OU=ExMail,DC=bpatcl,DC=com`.
4. Create a Universal Security Group `sg-<customer>-admin` in that customer OU; add approved customer admin user(s).
5. Mail-enable and hide the security group from address lists.
6. Create an Email Address Policy scoped to that OU, with `SMTP:%m@<DOMAIN>`; leave the existing Default Email Address Policy unchanged.
7. Create customer Address Lists, GAL, OAB and ABP using a unique `CustomAttribute15` value.
8. Assign the ABP to all customer mailboxes.
9. Create the two OU-scoped RBAC assignments for `Mail Recipients` and `Mail Recipient Creation`.
10. Assign `CustomAttribute15` to every new mailbox; verify EAP, UPN, primary SMTP and ABP.
11. Test OWA directory segmentation, recipient resolution and mail delivery.
12. Test ECP login, own-OU create/edit, cross-OU write denial and group membership protection.

Do not reuse the literal value `HM` or `BPATCL` for a new customer. Give every customer a unique, consistent tag, list/GAL/OAB/ABP names and security group.

---

## 19. Architecture summary

```text
                         Single AD Forest
                           bpatcl.com
                        DC 10.10.10.90
                               |
                +--------------+--------------+
                |                             |
          ExMail/hm                    ExMail/bpatcl
          HM users                    BPATCL users
          sg-hm-admin                 sg-bpatcl-admin
                |                             |
       CustomAttribute15=HM        CustomAttribute15=BPATCL
                |                             |
          EAP-HMLocal                 Default EAP
          GAL-HM                      GAL-BPATCL
          OAB-HM                      OAB-BPATCL
          ABP-HM                      ABP-BPATCL
                |                             |
                +--------------+--------------+
                               |
                    One Exchange Organization
                       mail2.bptcl.com
                         10.10.10.41
                               |
                   ABP Routing Agent enabled
                               |
             +-----------------+------------------+
             |                 |                  |
         HM Admin          BPATCL Admin       Central Admin
       sg-hm-admin       sg-bpatcl-admin       sg-admin
       Write scope:       Write scope:         Write scope:
       ExMail/hm          ExMail/bpatcl        ExMail
```

**Security boundary reminder:** This is logical customer segmentation within one Exchange organization. It is not equivalent to separate Exchange organizations or separate infrastructure. ABP segments the address book experience; OU-scoped RBAC limits write operations. Native ECP can still display recipient records outside the administrator's write scope.

## 20. Microsoft references

- [Address book policies in Exchange Server](https://learn.microsoft.com/en-us/exchange/email-addresses-and-address-books/address-book-policies/address-book-policies)
- [New-AddressList (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-addresslist)
- [Address list procedures in Exchange Server](https://learn.microsoft.com/en-us/exchange/email-addresses-and-address-books/address-lists/address-list-procedures)
- [New-GlobalAddressList (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-globaladdresslist)
- [New-OfflineAddressBook (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-offlineaddressbook)
- [New-AddressBookPolicy (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-addressbookpolicy)
- [New-ManagementRoleAssignment (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-managementroleassignment)
- [Set-TransportConfig (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-transportconfig)
- [Enable-DistributionGroup (Exchange PowerShell)](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-distributiongroup)
- [Exchange Server 2019 end-of-support roadmap](https://learn.microsoft.com/en-us/troubleshoot/exchange/administration/exchange-2019-2016-end-of-support)

---

**Document status:** Phase 06 configuration and observed tests recorded; Phase 07 RBAC configuration and observed ECP tests recorded; final security validation items in Section 17 remain open.
