# SSIL Customer Onboarding Runbook
## Add `ssil.com` to the Existing Multi-Domain Exchange Hosting Environment

**Document type:** Implementation runbook  
**Target customer:** SSIL  
**Email domain:** `ssil.com`  
**Environment:** Existing single Active Directory forest + single on-premises Exchange organization  
**Prepared:** 20 November 2026  
**Approach:** Reuse the proven HM/BPATCL pattern; do not rebuild or alter existing customer configuration.

---

## 1. Purpose

This runbook adds SSIL as a new hosted customer in the existing Exchange environment. SSIL users will have accounts in the central `bpatcl.com` Active Directory and mailboxes on the existing Exchange server. SSIL will not need its own AD domain, forest, domain controller, or Exchange server.

The implementation creates a separate customer OU, email address policy, Address List, GAL, OAB, ABP, admin security group, and OU-scoped RBAC assignments. Existing HM and BPATCL configuration remains unchanged.

> **Assumption:** SSIL wants the entire `ssil.com` SMTP namespace hosted on this Exchange organization. If only some `ssil.com` recipients are hosted here while others remain at another mail provider, stop before creating an **Authoritative** Accepted Domain and design an Internal Relay / split-domain mail-flow plan instead.

## 2. Existing environment — do not change

| Component | Existing value |
|---|---|
| AD forest/domain | `bpatcl.com` |
| Domain Controller | `hvdc.bpatcl.com` |
| DC IP | `10.10.10.90` |
| Exchange server | `mail2.bptcl.com` |
| Exchange IP | `10.10.10.41` |
| Exchange | Exchange Server 2019 (lab) |
| Parent customer OU | `OU=ExMail,DC=bpatcl,DC=com` |
| Existing HM email domain | `hmlocal.com` |
| Existing BPATCL email domain | `bpatcl.com` |
| Existing HM OU | `OU=hm,OU=ExMail,DC=bpatcl,DC=com` |
| Existing BPATCL OU | `OU=bpatcl,OU=ExMail,DC=bpatcl,DC=com` |
| Existing HM policy / ABP | `EAP-HMLocal` / `ABP-HM` |
| Existing BPATCL ABP | `ABP-BPATCL` |
| Default email address policy | `Default Policy` → `@bpatcl.com` (leave unchanged) |
| ABP Routing Agent | Installed and enabled; priority 10 (verified) |
| ABP routing | `AddressBookPolicyRoutingEnabled = True` (verified) |

Current customer structure:

```text
bpatcl.com
└── ExMail
    ├── hm
    │   └── sg-hm-admin
    ├── bpatcl
    │   └── sg-bpatcl-admin
    └── sg-admin
```

## 3. SSIL target design

### 3.1 Customer object naming standard

| Object | SSIL value |
|---|---|
| Customer short code | `SSIL` |
| Email domain | `ssil.com` |
| AD OU | `OU=ssil,OU=ExMail,DC=bpatcl,DC=com` |
| UPN suffix | `ssil.com` |
| Exchange Accepted Domain | `ssil.com` (Authoritative, if full domain is hosted) |
| Email Address Policy | `EAP-SSIL` |
| Address List | `AL-SSIL-All` |
| Room Address List | `AL-SSIL-Rooms` |
| Global Address List | `GAL-SSIL` |
| Offline Address Book | `OAB-SSIL` |
| Address Book Policy | `ABP-SSIL` |
| Customer admin group | `sg-ssil-admin` |
| Customer tag | `CustomAttribute15 = SSIL` |
| RBAC assignment names | `SSIL-Mail-Recipients`, `SSIL-Mail-Recipient-Creation` |
| RBAC recipient write scope | `bpatcl.com/ExMail/ssil` |

### 3.2 Logical architecture

```mermaid
flowchart TB
    subgraph AD[Single AD forest: bpatcl.com]
      EX[ExMail OU]
      EX --> HM[hm OU<br/>HM users + sg-hm-admin]
      EX --> BP[bpatcl OU<br/>BPATCL users + sg-bpatcl-admin]
      EX --> SS[ssil OU<br/>SSIL users + sg-ssil-admin]
      EX --> CA[sg-admin<br/>Central admin]
    end
    subgraph EXCH[One Exchange organization — mail2.bptcl.com / 10.10.10.41]
      EAP[EAP-SSIL<br/>@ssil.com]
      GAL[GAL-SSIL]
      OAB[OAB-SSIL]
      ABP[ABP-SSIL]
      AL[AL-SSIL-All + AL-SSIL-Rooms]
      AL --> GAL --> OAB --> ABP
      EAP --> SS
    end
    SS -->|SSIL mailboxes| EAP
    SS -->|CustomAttribute15 = SSIL| AL
    ABP -->|Assigned to all SSIL mailboxes| SS
    SG[sg-ssil-admin] -->|RBAC write scope: ExMail/ssil| SS
    CA -->|Existing scope: ExMail| SS
```

### 3.3 User experience

| Item | SSIL result |
|---|---|
| Sign-in UPN | `user@ssil.com` |
| Primary SMTP | `user@ssil.com` |
| Mailbox location | Existing Exchange database on `mail2.bptcl.com` |
| OWA/ECP URL | Existing shared Exchange client-access URL; it does not need to be `ssil.com` |
| Address book | `ABP-SSIL` / `GAL-SSIL` |
| Customer admin | Member of `sg-ssil-admin` |

**Important:** `ssil.com` is the customer email/login suffix. It is not an AD domain or a separate forest. Do not create a new AD domain or trust for SSIL.

---

## 4. Pre-change checks

Run the commands in Exchange Management Shell unless the section says **Domain Controller**. Capture the output before changing anything.

### 4.1 Check existing objects and avoid duplicates

```powershell
Get-AcceptedDomain | Format-Table Name,DomainName,DomainType,Default
Get-EmailAddressPolicy | Format-Table Name,Priority,RecipientContainer,EnabledEmailAddressTemplates
Get-AddressList | Format-Table Name
Get-GlobalAddressList | Format-Table Name
Get-OfflineAddressBook | Format-Table Name,IsDefault
Get-AddressBookPolicy | Format-Table Name
Get-DistributionGroup -Identity "sg-ssil-admin" -ErrorAction SilentlyContinue
Get-ManagementRoleAssignment | Where-Object {$_.Name -like "SSIL-*"} | Format-Table Name,Role,RoleAssigneeName
```

If an SSIL object already exists, inspect it and reuse/fix it. Do not blindly rerun `New-*` commands with duplicate names.

### 4.2 Confirm the current Exchange and AD values

On the Domain Controller:

```powershell
Import-Module ActiveDirectory
Get-ADForest | Format-List RootDomain,Domains,UPNSuffixes
Get-ADDomain | Format-List DNSRoot,NetBIOSName,DistinguishedName
```

On Exchange:

```powershell
Get-ExchangeServer | Format-List Name,Fqdn,AdminDisplayVersion
Get-MailboxDatabase | Format-Table Name,Server,Mounted
Get-TransportConfig | Format-List AddressBookPolicyRoutingEnabled
Get-TransportAgent -Identity "ABP Routing Agent" | Format-List Identity,Enabled,Priority
```

Expected based on the existing lab: AD root `bpatcl.com`, `hmlocal.com` already in the UPN suffix list, Exchange `mail2.bptcl.com`, ABP routing enabled, and ABP Routing Agent enabled.

### 4.3 Backup and change safety

Before onboarding:

- Confirm a recent AD System State backup and Exchange database backup exist (or record that this is a disposable lab).
- Export current Accepted Domains, Email Address Policies, Address Lists, GALs, OABs, ABPs, and RBAC assignments if this is anything beyond a disposable lab.
- Do not change the Default Accepted Domain or Default Email Address Policy.
- Do not alter existing HM/BPATCL address lists, ABPs, role assignments, send connectors, or receive connectors.

---

# Phase A — AD preparation

## 5. Add `ssil.com` as an AD UPN suffix

This lets an SSIL user sign in with `user@ssil.com` while the account remains inside `bpatcl.com`.

### GUI

On `hvdc.bpatcl.com`:

1. Open **Active Directory Domains and Trusts** (`domain.msc`).
2. Right-click **Active Directory Domains and Trusts** at the top of the tree → **Properties**.
3. Under **UPN Suffixes**, add `ssil.com`.
4. Click **Add** → **Apply** → **OK**.
5. Re-open the Properties page and confirm `ssil.com` is listed.

### CLI — Domain Controller

This preserves existing suffixes and adds only `ssil.com`:

```powershell
Import-Module ActiveDirectory
$Forest = Get-ADForest
$CurrentSuffixes = @($Forest.UPNSuffixes)
if ($CurrentSuffixes -notcontains "ssil.com") {
    $UpdatedSuffixes = @($CurrentSuffixes + "ssil.com" | Sort-Object -Unique)
    Set-ADForest -Identity $Forest.Name -UPNSuffixes $UpdatedSuffixes
}

Get-ADForest | Select-Object -ExpandProperty UPNSuffixes
```

Expected: existing suffixes remain, and `ssil.com` is added. Do not replace the current list with only `ssil.com`.

## 6. Create the SSIL OU

### GUI — Active Directory Users and Computers

1. Open **Active Directory Users and Computers** (`dsa.msc`).
2. Expand `bpatcl.com` → `ExMail`.
3. Right-click **ExMail** → **New → Organizational Unit**.
4. Name it `ssil`.
5. Keep **Protect container from accidental deletion** enabled.

Expected:

```text
bpatcl.com
└── ExMail
    ├── hm
    ├── bpatcl
    ├── ssil       <-- new
    └── sg-admin
```

### CLI — Domain Controller

```powershell
Import-Module ActiveDirectory
$ParentOU = "OU=ExMail,DC=bpatcl,DC=com"
$ExistingOU = Get-ADOrganizationalUnit -LDAPFilter "(ou=ssil)" -SearchBase $ParentOU -ErrorAction SilentlyContinue
if (-not $ExistingOU) {
    New-ADOrganizationalUnit -Name "ssil" -Path $ParentOU -ProtectedFromAccidentalDeletion $true
}

Get-ADOrganizationalUnit -LDAPFilter "(ou=ssil)" -SearchBase $ParentOU |
Format-List Name,DistinguishedName,ProtectedFromAccidentalDeletion
```

Expected DN:

```text
OU=ssil,OU=ExMail,DC=bpatcl,DC=com
```

## 7. Create SSIL admin Universal Security Group

This group identifies who receives SSIL ECP administration. As in the current model, an existing SSIL mailbox user can be made an admin by adding that user to this group.

### GUI

In ADUC, right-click `ExMail/ssil` → **New → Group**:

- Group name: `sg-ssil-admin`
- Group scope: **Universal**
- Group type: **Security**

Add the approved SSIL administrator account through the group's **Members** tab. If the admin mailbox user will be created later, add that user after mailbox creation.

### CLI — Domain Controller

```powershell
Import-Module ActiveDirectory
$Group = Get-ADGroup -Identity "sg-ssil-admin" -ErrorAction SilentlyContinue
if (-not $Group) {
    New-ADGroup -Name "sg-ssil-admin" -SamAccountName "sg-ssil-admin" `
        -GroupScope Universal -GroupCategory Security `
        -Path "OU=ssil,OU=ExMail,DC=bpatcl,DC=com" `
        -Description "Exchange administration for SSIL customer"
}

Get-ADGroup -Identity "sg-ssil-admin" -Properties GroupScope,GroupCategory |
Format-List Name,GroupScope,GroupCategory,DistinguishedName
```

Add the approved admin user when known:

```powershell
Add-ADGroupMember -Identity "sg-ssil-admin" -Members "<ssil-admin-samAccountName>"
Get-ADGroupMember -Identity "sg-ssil-admin"
```

Replace the placeholder with a real user in `ExMail/ssil`. Do not add a nested group. Group membership changes require a fresh sign-in/logon to refresh the user's security token.

---

# Phase B — Exchange domain and email policy

## 8. Add `ssil.com` as an Accepted Domain

Use **Authoritative** only if all `@ssil.com` recipients are intended to be hosted by this Exchange organization.

### GUI — EAC

1. Open the existing Exchange Admin Center.
2. Go to **Mail flow → Accepted domains** (classic EAC may show the Accepted Domains tab under Mail Flow).
3. Click **+**.
4. Name: `SSIL`.
5. Accepted domain: `ssil.com`.
6. Select **Authoritative domain**.
7. Save.
8. Confirm `bpatcl.com` remains the default domain.

### CLI — Exchange Management Shell

```powershell
$ExistingAccepted = Get-AcceptedDomain -Identity "ssil.com" -ErrorAction SilentlyContinue
if (-not $ExistingAccepted) {
    New-AcceptedDomain -Name "SSIL" -DomainName "ssil.com" -DomainType Authoritative
}

Get-AcceptedDomain | Format-Table Name,DomainName,DomainType,Default
```

Expected: `ssil.com` appears as Authoritative and `Default=False`. Do not run `Set-AcceptedDomain -Default` and do not change the existing default `bpatcl.com`.

## 9. Create the SSIL Email Address Policy

The policy must be scoped to the SSIL OU so it cannot assign `@ssil.com` addresses to HM/BPATCL recipients.

### GUI — EAC (if the installed build exposes the required OU filter)

1. Go to **Mail flow → Email address policies**.
2. Click **+** and create `EAP-SSIL`.
3. Set the recipient scope to mailbox users in `bpatcl.com/ExMail/ssil`.
4. Add the SMTP template `SMTP:%m@ssil.com`.
5. Set its priority after `EAP-HMLocal` but before `Default Policy`.
6. Save and apply the policy.

If the EAC wizard does not let you specify the exact recipient container/OU, use the EMS method below. Do not create a broad policy that applies to all recipients.

### CLI — Exchange Management Shell

First inspect priorities:

```powershell
Get-EmailAddressPolicy | Format-Table Name,Priority,RecipientContainer,EnabledEmailAddressTemplates
```

In the currently documented lab state, `EAP-HMLocal` is priority 1 and `Default Policy` is `Lowest`, so use priority 2 for SSIL. If another non-default policy has priority 2, select the next unused numeric priority before `Lowest`.

```powershell
New-EmailAddressPolicy -Name "EAP-SSIL" `
    -RecipientContainer "bpatcl.com/ExMail/ssil" `
    -IncludedRecipients MailboxUsers `
    -EnabledEmailAddressTemplates "SMTP:%m@ssil.com" `
    -Priority 2
```

Apply it to its scoped recipients:

```powershell
Update-EmailAddressPolicy -Identity "EAP-SSIL"
```

Verify:

```powershell
Get-EmailAddressPolicy -Identity "EAP-SSIL" |
Format-List Name,Priority,RecipientContainer,EnabledEmailAddressTemplates,IncludedRecipients,RecipientFilterApplied
```

Expected:

- Name: `EAP-SSIL`
- RecipientContainer: `bpatcl.com/ExMail/ssil`
- Template: `SMTP:%m@ssil.com`
- IncludedRecipients: `MailboxUsers`
- Priority: 2 (if still the next free priority)

Leave `Default Policy` unchanged.

---

# Phase C — SSIL Address Book segmentation

## 10. Customer tag convention

Every SSIL mailbox must have:

```text
CustomAttribute15 = SSIL
```

This tag is used by SSIL's Address Lists and GAL filters. It is not inherited from the OU and does not get populated automatically just because the mailbox is in `ExMail/ssil`.

### GUI — set the tag on each mailbox

1. EAC → **Recipients → Mailboxes**.
2. Select the SSIL mailbox → **Edit**.
3. Open **Custom attributes**.
4. Set **Custom attribute 15** to `SSIL`.
5. Save.

For multiple existing SSIL mailboxes, use **Bulk Edit → Custom attributes → Custom attribute 15** if the installed EAC view exposes it.

### CLI — set the tag on all current SSIL mailboxes

```powershell
$SSIL = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/ssil" -ResultSize Unlimited
$SSIL | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -CustomAttribute15 "SSIL"
}
```

For a single newly created mailbox:

```powershell
Set-Mailbox -Identity "ssiluser01" -CustomAttribute15 "SSIL"
```

## 11. Create SSIL Address Lists

### GUI

Use EAC **Organization → Address lists** if the installed interface supports the exact custom attribute filter. For the exact OPATH filters below, EMS is the reliable method.

### CLI

```powershell
New-AddressList -Name "AL-SSIL-All" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'SSIL'))"

New-AddressList -Name "AL-SSIL-Rooms" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'SSIL') -and ((RecipientDisplayType -eq 'ConferenceRoomMailbox') -or (RecipientDisplayType -eq 'SyncedConferenceRoomMailbox')))"
```

Update membership:

```powershell
Update-AddressList -Identity "AL-SSIL-All"
Update-AddressList -Identity "AL-SSIL-Rooms"
```

Verify:

```powershell
Get-AddressList | Where-Object {$_.Name -like "AL-SSIL-*"} |
Format-Table Name,RecipientFilterApplied
```

## 12. Create SSIL Global Address List

```powershell
New-GlobalAddressList -Name "GAL-SSIL" -RecipientFilter "((Alias -ne `$null) -and (CustomAttribute15 -eq 'SSIL'))"
Update-GlobalAddressList -Identity "GAL-SSIL"
```

Verify membership:

```powershell
$GAL = Get-GlobalAddressList -Identity "GAL-SSIL"
Get-Recipient -ResultSize Unlimited -RecipientPreviewFilter $GAL.RecipientFilter |
Format-Table Name,PrimarySmtpAddress,CustomAttribute15
```

Only recipients tagged `SSIL` should appear. The number of results will match however many SSIL recipients have been created and tagged.

## 13. Create SSIL Offline Address Book

```powershell
New-OfflineAddressBook -Name "OAB-SSIL" -AddressLists "GAL-SSIL" -GlobalWebDistributionEnabled $true -IsDefault $false
```

Verify:

```powershell
Get-OfflineAddressBook -Identity "OAB-SSIL" |
Format-List Name,AddressLists,GeneratingMailbox,IsDefault,VirtualDirectories,GlobalWebDistributionEnabled
```

Expected: `AddressLists={\GAL-SSIL}`, `IsDefault=False`, `GlobalWebDistributionEnabled=True`. Do not change the existing Default Offline Address Book.

## 14. Create SSIL Address Book Policy

```powershell
New-AddressBookPolicy -Name "ABP-SSIL" -AddressLists "AL-SSIL-All" -GlobalAddressList "GAL-SSIL" -OfflineAddressBook "OAB-SSIL" -RoomList "AL-SSIL-Rooms"
```

Verify:

```powershell
Get-AddressBookPolicy -Identity "ABP-SSIL" |
Format-List Name,AddressLists,GlobalAddressList,OfflineAddressBook,RoomList
```

For any SSIL mailbox already created, assign the policy:

```powershell
$SSIL = Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/ssil" -ResultSize Unlimited
$SSIL | ForEach-Object {
    Set-Mailbox -Identity $_.Identity -AddressBookPolicy "ABP-SSIL"
}
```

New mailboxes must also be assigned `ABP-SSIL`, either in the ECP mailbox-creation form if the field is available or afterward with `Set-Mailbox`.

---

# Phase D — SSIL administrator and RBAC

## 15. Mail-enable and hide `sg-ssil-admin`

The group must remain a **Universal Security** group. Mail-enable the existing AD group; do not create a separate Distribution Group.

### CLI — Exchange Management Shell

```powershell
Get-DistributionGroup -Identity "sg-ssil-admin" -ErrorAction SilentlyContinue
```

If it is not already mail-enabled:

```powershell
Enable-DistributionGroup -Identity "sg-ssil-admin"
```

Hide the group from customer address books:

```powershell
Set-DistributionGroup -Identity "sg-ssil-admin" -HiddenFromAddressListsEnabled $true
```

Verify:

```powershell
Get-DistributionGroup -Identity "sg-ssil-admin" |
Format-List Name,PrimarySmtpAddress,GroupType,HiddenFromAddressListsEnabled
```

Expected GroupType includes `Universal, Security`; HiddenFromAddressListsEnabled is `True`.

### GUI

- Create and manage membership in **ADUC** → `ExMail/ssil` → `sg-ssil-admin` → Properties → Members.
- Use EMS for `Enable-DistributionGroup` and hidden-from-address-list configuration if the EAC does not expose the conversion for an existing non-mail-enabled security group.

Add an approved SSIL admin mailbox user directly to the group (not a nested group):

```powershell
Add-ADGroupMember -Identity "sg-ssil-admin" -Members "<ssil-admin-samAccountName>"
Get-ADGroupMember -Identity "sg-ssil-admin"
```

The customer admin should not control membership of `sg-ssil-admin` unless that delegation is explicitly designed and tested. Keep group membership administration with the central administrator.

## 16. Create SSIL OU-scoped Exchange RBAC assignments

Use the existing roles and the same proven scope pattern as HM/BPATCL.

```powershell
New-ManagementRoleAssignment -Name "SSIL-Mail-Recipients" -Role "Mail Recipients" -SecurityGroup "sg-ssil-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/ssil"

New-ManagementRoleAssignment -Name "SSIL-Mail-Recipient-Creation" -Role "Mail Recipient Creation" -SecurityGroup "sg-ssil-admin" -RecipientOrganizationalUnitScope "bpatcl.com/ExMail/ssil"
```

Verify:

```powershell
Get-ManagementRoleAssignment |
Where-Object {$_.Name -in @("SSIL-Mail-Recipients","SSIL-Mail-Recipient-Creation")} |
Format-Table Name,Role,RoleAssigneeName,RecipientWriteScope,CustomRecipientWriteScope -AutoSize
```

Expected `CustomRecipientWriteScope` is `bpatcl.com/ExMail/ssil` for both assignments.

### Existing central administrator

The existing `sg-admin` assignments are scoped to `bpatcl.com/ExMail`, which is the parent OU of `ExMail/ssil`. Therefore SSIL is already inside the central administrator's write scope. Do **not** create another central assignment unless the existing parent scope has been changed; verify it instead:

```powershell
Get-ManagementRoleAssignment |
Where-Object {$_.Name -in @("Central-Mail-Recipients","Central-Mail-Recipient-Creation")} |
Format-Table Name,Role,RoleAssigneeName,CustomRecipientWriteScope -AutoSize
```

### Security limitation — read visibility vs write scope

These assignments limit recipient write/management operations to the SSIL OU. They do **not** guarantee that SSIL admins cannot see other customers' recipient rows in Native ECP. In the existing lab, HM and BPATCL admins could see other customer rows but could not edit them. ABP controls OWA/Outlook address-book views; it does not remove out-of-scope rows from all ECP admin lists. This same limitation applies to SSIL.

---

# Phase E — Create SSIL mailboxes

## 17. ECP GUI — preferred customer workflow

Sign in to ECP with an authorized member of `sg-ssil-admin`:

1. Open the existing shared ECP URL, for example `https://mail2.bptcl.com/ecp` (use the actual working URL and certificate namespace configured in the lab).
2. Go to **Recipients → Mailboxes**.
3. Click **+ → User mailbox**.
4. Choose **New user**.
5. In **Organizational Unit**, click **Browse** and select `bpatcl.com → ExMail → ssil`.
   - This selection is mandatory. If ECP defaults to `bpatcl.com/Users`, saving will fail because that container is outside the SSIL admin's write scope.
6. Enter the user's name and logon name (example `ssiluser01`).
7. Select UPN suffix `ssil.com` so the sign-in UPN is `ssiluser01@ssil.com`.
8. Set the initial password and password-change option according to SSIL's onboarding policy.
9. Choose the existing mounted mailbox database shown in the ECP wizard.
10. If the Address Book Policy field is available, select `ABP-SSIL`.
11. Save.
12. Edit the new mailbox → **Custom attributes** → set **Custom attribute 15** to `SSIL` → Save.
13. If ABP was not selected during creation, assign `ABP-SSIL` afterward using the command below.

### Post-create ECP actions

For each mailbox:

- Ensure Primary SMTP is `username@ssil.com`.
- Ensure Custom Attribute 15 is `SSIL`.
- Ensure Address Book Policy is `ABP-SSIL`.
- Ensure EmailAddressPolicyEnabled is `True`.

## 18. EMS — create one SSIL user and mailbox together

Run in Exchange Management Shell. Replace the username/display name as needed.

```powershell
$Password = Read-Host "Enter initial SSIL password" -AsSecureString

New-Mailbox -Name "SSIL User 01" `
    -DisplayName "SSIL User 01" `
    -Alias "ssiluser01" `
    -SamAccountName "ssiluser01" `
    -UserPrincipalName "ssiluser01@ssil.com" `
    -OrganizationalUnit "OU=ssil,OU=ExMail,DC=bpatcl,DC=com" `
    -Password $Password `
    -ResetPasswordOnNextLogon $false
```

Then apply the SSIL address policy and customer metadata:

```powershell
Update-EmailAddressPolicy -Identity "EAP-SSIL"
Set-Mailbox -Identity "ssiluser01" -CustomAttribute15 "SSIL" -AddressBookPolicy "ABP-SSIL"
```

Do not set `-PrimarySmtpAddress` on the mailbox creation command if you want the Email Address Policy to manage the primary address. Verify the resulting address instead.

## 19. Bulk-create SSIL mailboxes (optional)

Use only if SSIL has a confirmed user list. Choose unique `SamAccountName`/Alias values; do not reuse HM/BPATCL names.

```powershell
$OU = "OU=ssil,OU=ExMail,DC=bpatcl,DC=com"
$Password = Read-Host "Enter initial SSIL password" -AsSecureString

1..10 | ForEach-Object {
    $Number = $_
    $Username = "ssiluser{0:D2}" -f $Number
    $UPN = "$Username@ssil.com"

    $Existing = Get-Mailbox -Identity $Username -ErrorAction SilentlyContinue
    if ($Existing) {
        Write-Warning "$Username already has a mailbox. Skipping."
        return
    }

    $ExistingADUser = Get-User -Identity $Username -ErrorAction SilentlyContinue
    if ($ExistingADUser) {
        Write-Warning "$Username already exists in AD without the expected mailbox. Review manually; skipping."
        return
    }

    New-Mailbox -Name "SSIL User $Number" `
        -DisplayName "SSIL User $Number" `
        -Alias $Username `
        -SamAccountName $Username `
        -UserPrincipalName $UPN `
        -OrganizationalUnit $OU `
        -Password $Password `
        -ResetPasswordOnNextLogon $false
}

Update-EmailAddressPolicy -Identity "EAP-SSIL"

Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/ssil" -ResultSize Unlimited |
ForEach-Object {
    Set-Mailbox -Identity $_.Identity -CustomAttribute15 "SSIL" -AddressBookPolicy "ABP-SSIL"
}
```

**Password note:** a shared password for many users is suitable only for a disposable lab test. For real customer accounts use unique initial passwords or an approved provisioning process. Avoid storing passwords in scripts or logs.

---

# Phase F — DNS, client access and mail flow

## 20. Internal DNS and shared OWA/ECP URL

SSIL users use the existing Exchange client-access namespace. They do not need a separate Exchange URL just because their email address is `@ssil.com`.

The existing internal DNS should continue resolving the Exchange hostname to the Exchange private IP:

```text
mail2.bptcl.com  →  10.10.10.41
```

Verify from an SSIL client or the relevant internal DNS server:

```powershell
nslookup mail2.bptcl.com
```

Use the existing working URL, such as `https://mail2.bptcl.com/owa`, only if its certificate and DNS configuration are valid for that host. The user's email domain and the Exchange HTTPS service hostname are separate namespaces.

## 21. Public DNS — only if SSIL will send/receive internet email

The customer's authoritative public DNS provider must publish records consistent with the existing shared Exchange internet-facing namespace.

| Record | Purpose | Required planning |
|---|---|---|
| MX for `ssil.com` | Inbound mail route | Point to the existing public mail gateway/hostname used by this Exchange service |
| A/AAAA for mail hostname | Resolve the mail gateway | Public routable IP, not `10.10.10.41` |
| SPF TXT | Authorize outbound sender IP/service | Include the actual public outbound mail IP or relay provider |
| DKIM TXT | Publish signing public key | Configure a DKIM signing gateway/solution; do not assume Exchange Server 2019 signs DKIM natively |
| DMARC TXT | Domain spoofing policy/reporting | Start with a monitored policy such as `p=none`, then harden after validation |
| Autodiscover | Outlook profile discovery | Follow the existing supported HTTPS/certificate namespace design; do not publish an Autodiscover record until the certificate and endpoint are confirmed |

**Critical:** Never publish an MX hostname whose public A record resolves to the private address `10.10.10.41`. The public MX must point to a publicly reachable mail gateway. The existing `mail2.bptcl.com` hostname can be used as the MX target only if its public DNS, NAT/firewall, TCP 25 reachability, certificate and inbound connector design are already correct.

Before changing SSIL's public MX, confirm that the domain is authorized for onboarding and agree on the mail cutover time. If SSIL currently receives mail at another provider, changing MX will move new inbound delivery; preserve and migrate existing mail separately.

## 22. Internal and cross-customer mail-flow tests

After mailbox creation:

1. SSIL → SSIL: send between two SSIL test mailboxes.
2. SSIL → HM: send to `hmuser03@hmlocal.com`.
3. HM → SSIL: reply to the SSIL sender.
4. SSIL → BPATCL: send to `buser02@bpatcl.com`.
5. BPATCL → SSIL: reply.
6. Confirm messages arrive in Inbox and Sent Items, and review NDRs/queues if delivery fails.

Check the queue if needed:

```powershell
Get-Queue | Format-Table Identity,Status,MessageCount,NextHopDomain
```

ABP routing should make cross-ABP recipients appear as external-style recipients without exposing the other customer's contact card. It does not block cross-customer email delivery.

---

# Phase G — Verification and acceptance

## 23. SSIL mailbox verification

```powershell
Get-Mailbox -OrganizationalUnit "bpatcl.com/ExMail/ssil" -ResultSize Unlimited |
Format-Table Name,UserPrincipalName,PrimarySmtpAddress,EmailAddressPolicyEnabled,AddressBookPolicy,CustomAttribute15
```

Expected for each SSIL mailbox:

| Property | Expected |
|---|---|
| OU | `bpatcl.com/ExMail/ssil` |
| UPN | `username@ssil.com` |
| Primary SMTP | `username@ssil.com` |
| EmailAddressPolicyEnabled | `True` |
| AddressBookPolicy | `ABP-SSIL` |
| CustomAttribute15 | `SSIL` |

## 24. SSIL GAL membership verification

```powershell
$GAL = Get-GlobalAddressList -Identity "GAL-SSIL"
Get-Recipient -ResultSize Unlimited -RecipientPreviewFilter $GAL.RecipientFilter |
Format-Table Name,PrimarySmtpAddress,CustomAttribute15
```

All returned recipients should have `CustomAttribute15 = SSIL`. If a new mailbox is missing, check its CustomAttribute15 first, then update the Address List/GAL:

```powershell
Update-AddressList -Identity "AL-SSIL-All"
Update-AddressList -Identity "AL-SSIL-Rooms"
Update-GlobalAddressList -Identity "GAL-SSIL"
```

## 25. SSIL RBAC verification

```powershell
Get-ManagementRoleAssignment |
Where-Object {$_.Name -in @("SSIL-Mail-Recipients","SSIL-Mail-Recipient-Creation")} |
Format-Table Name,Role,RoleAssigneeName,CustomRecipientWriteScope -AutoSize
```

Expected scope for both assignments: `bpatcl.com/ExMail/ssil`.

Confirm the existing central admin scope still covers the parent `ExMail` OU:

```powershell
Get-ManagementRoleAssignment |
Where-Object {$_.Name -in @("Central-Mail-Recipients","Central-Mail-Recipient-Creation")} |
Format-Table Name,Role,RoleAssigneeName,CustomRecipientWriteScope -AutoSize
```

## 26. ECP and OWA acceptance tests

Use fresh/private browser sessions for each admin account.

| Test account | Test | Expected |
|---|---|---|
| SSIL admin (member of `sg-ssil-admin`) | Sign in to ECP | Allowed |
| SSIL admin | Create mailbox in `ExMail/ssil` | Allowed |
| SSIL admin | Create mailbox in `ExMail/hm` or `ExMail/bpatcl` | Denied by write scope |
| HM admin | Modify SSIL mailbox | Denied by write scope |
| BPATCL admin | Modify SSIL mailbox | Denied by write scope |
| Central admin | Manage SSIL mailbox | Allowed via existing parent `ExMail` scope |
| SSIL user | OWA People/Directory | SSIL directory entries only (`AL-SSIL-All`, `AL-SSIL-Rooms`) |
| SSIL user | Type HM/BPATCL full email address | Other ABP's display/contact details not exposed; delivery remains possible |
| SSIL user | Send to HM and BPATCL | Delivered if mail routing is healthy |

Native ECP limitation: SSIL admin may still see other customer recipient rows in ECP, even though the admin cannot modify those recipients. This is the known shared-organization read-visibility limitation already observed with HM/BPATCL. Do not report complete ECP read isolation as achieved.

## 27. Password-change testing

Normal OWA sign-in uses the user's UPN, for example `ssiluser01@ssil.com`.

If ECP sets **Require password change on next logon** and the forced-change page asks for `Domain\\user name`, the tested lab format is:

```text
BPATCL\ssiluser01
```

Use the actual NetBIOS domain returned by:

```powershell
Get-ADDomain | Format-List NetBIOSName
```

The NetBIOS domain name plus the user's `sAMAccountName` was successful in the existing lab. The UPN remains the normal OWA login identity.

## 28. SSIL onboarding acceptance checklist

- [ ] SSIL domain ownership/hosting authorization confirmed.
- [ ] `ssil.com` UPN suffix added without removing existing suffixes.
- [ ] `ExMail/ssil` OU created and protected from accidental deletion.
- [ ] `sg-ssil-admin` Universal Security Group created in the SSIL OU.
- [ ] Approved SSIL admin added directly to `sg-ssil-admin`.
- [ ] SSIL Accepted Domain added as Authoritative (only if full domain is hosted).
- [ ] `EAP-SSIL` created with correct OU scope and `SMTP:%m@ssil.com`.
- [ ] SSIL Address Lists, GAL, OAB and ABP created.
- [ ] `ABP-SSIL` assigned to every SSIL mailbox.
- [ ] `CustomAttribute15 = SSIL` assigned to every SSIL recipient.
- [ ] `sg-ssil-admin` mail-enabled, hidden from address lists and scoped to `ExMail/ssil` via RBAC.
- [ ] Existing central `sg-admin` scope verified to include SSIL.
- [ ] ECP own-OU create/edit test passed.
- [ ] Cross-customer modification tests denied.
- [ ] OWA People/Directory shows SSIL address book only.
- [ ] Cross-customer email tests pass.
- [ ] Public DNS/MX/SPF/DKIM/DMARC cutover verified if internet mail is required.

---

# Phase H — Safe rollback for SSIL only

Do not run rollback commands as a general cleanup. Use them only if the SSIL onboarding must be fully reverted and the SSIL customer has confirmed that its mailboxes/data can be removed.

Recommended rollback order:

1. Stop new SSIL mailbox provisioning and record/export any required SSIL mailbox data.
2. Remove or migrate SSIL mailboxes only after explicit approval and verified backup/export.
3. Remove the two SSIL RBAC assignments.
4. Remove SSIL ABP, OAB, GAL and Address Lists after no mailbox references them.
5. Remove `EAP-SSIL` after confirming no recipients still depend on it.
6. Remove Accepted Domain `ssil.com` only after confirming no SSIL addresses remain and mail flow has been cut back safely.
7. Disable mail-enabling on `sg-ssil-admin` if the AD group must be retained, or remove the group only after checking membership and dependencies.
8. Remove the `ssil.com` UPN suffix only if no AD user still uses it; preserve all other suffixes.
9. Remove `ExMail/ssil` only after it is empty and after verifying no service/security objects remain.

RBAC removal commands (only for the SSIL-specific assignments):

```powershell
Remove-ManagementRoleAssignment -Identity "SSIL-Mail-Recipients" -Confirm:$true
Remove-ManagementRoleAssignment -Identity "SSIL-Mail-Recipient-Creation" -Confirm:$true
```

Before deleting any other object, inspect references with `Get-*` and remove dependencies in the order above. Do not remove the existing Default Policy, Default GAL, Default OAB, HM/BPATCL objects, or the existing ABP Routing Agent.

---

## 29. Operational rules for future customer onboarding

Use the same reusable pattern for every additional customer:

| Component | Naming rule | SSIL example |
|---|---|---|
| OU | `ExMail/<shortcode>` | `ExMail/ssil` |
| UPN suffix | `<domain>` | `ssil.com` |
| Accepted Domain | `<domain>` | `ssil.com` |
| Email Address Policy | `EAP-<CODE>` | `EAP-SSIL` |
| Address List | `AL-<CODE>-All` | `AL-SSIL-All` |
| Room List | `AL-<CODE>-Rooms` | `AL-SSIL-Rooms` |
| GAL | `GAL-<CODE>` | `GAL-SSIL` |
| OAB | `OAB-<CODE>` | `OAB-SSIL` |
| ABP | `ABP-<CODE>` | `ABP-SSIL` |
| Customer admin group | `sg-<code>-admin` | `sg-ssil-admin` |
| CustomAttribute15 | unique customer tag | `SSIL` |
| RBAC scope | `bpatcl.com/ExMail/<code>` | `bpatcl.com/ExMail/ssil` |

**Every new customer must have a unique `CustomAttribute15` value.** Never reuse `HM` or `BPATCL` for SSIL or another customer.

For every new mailbox created through ECP, complete the post-create steps: verify UPN and primary SMTP, set CustomAttribute15, assign the correct ABP, and verify the mailbox is in the customer OU. The OU alone does not automatically populate CustomAttribute15 or assign an ABP.

---

## 30. Important platform lifecycle note

This environment is documented as a lab using Exchange Server 2019. Microsoft states that Exchange Server 2019 reached end of support on **14 October 2025**. It can continue running, but it no longer receives normal product support or security updates. Do not treat this lab configuration as production-ready for a real customer without a supported-platform plan. Plan a migration to Exchange Server Subscription Edition (SE) or another supported mail platform before production hosting.

Official Microsoft reference: [Exchange Server 2019 end-of-support roadmap](https://learn.microsoft.com/en-us/troubleshoot/exchange/administration/exchange-2019-2016-end-of-support).

## 31. Microsoft technical references

- [New-AcceptedDomain](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-accepteddomain)
- [New-EmailAddressPolicy](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-emailaddresspolicy)
- [Exchange recipient-filter properties](https://learn.microsoft.com/en-us/powershell/exchange/recipientfilter-properties)
- [New-AddressList](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-addresslist)
- [New-GlobalAddressList](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-globaladdresslist)
- [New-OfflineAddressBook](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-offlineaddressbook)
- [New-AddressBookPolicy](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-addressbookpolicy)
- [Address Book Policies in Exchange Server](https://learn.microsoft.com/en-us/exchange/email-addresses-and-address-books/address-book-policies/address-book-policies)
- [New-ManagementRoleAssignment](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-managementroleassignment)
- [Enable-DistributionGroup](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/enable-distributiongroup)
- [Exchange Server supportability matrix](https://learn.microsoft.com/en-us/exchange/plan-and-deploy/supportability-matrix)

---

**Runbook status:** Prepared for SSIL onboarding. Commands are scoped to new SSIL objects and the SSIL OU. Execute sequentially, verify each phase, and do not change existing HM/BPATCL/default Exchange objects.
