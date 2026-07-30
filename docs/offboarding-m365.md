# Offboarding a Microsoft 365 User Without Leaving Orphans

Deleting a Microsoft 365 user takes about four clicks. Doing it without breaking something downstream takes considerably longer, and the difference usually doesn't surface for weeks.

The problem is that most offboarding guides treat the user account as the only thing being removed. In practice, an account that has been active for a year or two accumulates ownership — of automation, of groups, of sites, of a phone number, of a calendar full of recurring meetings. Delete the account and those things don't fail loudly. They fail quietly, at some later point, in a way that's hard to trace back to a departure nobody remembers.

This is the sequence I use, and more usefully, the reasoning behind why the steps sit in this order.

## The short version

```
[ ] Block sign-in
[ ] Revoke active sessions
[ ] Inventory what the account owns
[ ] Preserve mailbox and OneDrive content
[ ] Reassign ownership of groups, flows, apps, and sites
[ ] Unassign phone number and voice policies
[ ] Remove group memberships
[ ] Delete the account
[ ] Verify license reclamation
[ ] Record the offboarding
```

Everything below is why each line is there.

All examples use the Microsoft Graph PowerShell SDK. The older MSOnline and AzureAD modules are retired, so anything you find online using `Connect-MsolService` should be treated as out of date.

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All","Organization.Read.All"
```

## 1. Block sign-in

First action, always. It stops new authentication immediately.

```powershell
Update-MgUser -UserId jdoe@contoso.com -AccountEnabled:$false
```

For a planned departure this can wait until the final day. For an involuntary one it happens before the conversation, not after. Same command either way; the timing is an HR decision rather than a technical one.

## 2. Revoke active sessions

This is the step most guides omit, and omitting it leaves a window that's measured in hours.

Blocking sign-in prevents the account from acquiring a *new* token. It does nothing to the tokens already issued. An access token typically stays valid for around an hour, and a refresh token far longer than that. Someone with Outlook or Teams already open can keep reading mail after you've disabled their account, which tends to surprise people who assume the toggle is instant.

```powershell
Revoke-MgUserSignInSession -UserId jdoe@contoso.com
```

This invalidates the refresh tokens so clients can't silently renew. It isn't uniformly immediate: workloads that support Continuous Access Evaluation — Exchange Online, SharePoint Online, Teams — react within a few minutes. Others wait out the remaining access token lifetime.

Treat steps 1 and 2 as a single action. Doing either one alone is not offboarding.

## 3. Inventory what the account owns

Before anything is deleted, find out what depends on it. This is the step that separates a clean offboarding from one you'll be repairing later.

Graph will tell you what the account directly owns:

```powershell
Get-MgUserOwnedObject -UserId jdoe@contoso.com -All |
    ForEach-Object { $_.AdditionalProperties['displayName'] }

Get-MgUserMemberOf -UserId jdoe@contoso.com -All |
    ForEach-Object { $_.AdditionalProperties['displayName'] }
```

That covers groups, applications, and service principals. It does not cover everything, and the gaps are where the orphans live.

**Automation.** Power Automate flows owned solely by the departing user stop running once the account is gone. The connections belonged to that account and cannot be recovered — you rebuild them from scratch, assuming you can work out what they were connected to. The same applies to Power Apps, custom connectors, and connection references. Reassign these through the Power Platform admin center, or in bulk:

```powershell
# Microsoft.PowerApps.Administration.PowerShell
Set-AdminFlowOwnerRole -EnvironmentName <env> -FlowName <guid> `
    -RoleName CanEdit -PrincipalObjectId <new-owner-object-id> -PrincipalType User
```

**Groups and Teams.** A Microsoft 365 Group or Team whose only owner is deleted becomes ownerless. Members keep working in it and nobody can administer it — no adding members, no changing settings, no deleting it. The same problem applies to distribution lists with a single manager and SharePoint sites with a single site collection administrator.

**Azure and identity.** RBAC role assignments where the departing user is the only principal with access to a resource group or subscription. App registrations and service principals listing them as sole owner. Privileged Identity Management assignments, including eligible ones that aren't currently active.

**Calendar.** Recurring meetings organized by the account lose their organizer. Attendees keep the series on their calendars and nobody can amend or cancel it.

**Devices.** Intune-enrolled and Entra-registered devices tied to the account.

The practical point about all of this: adding a co-owner to a group takes ten seconds while the account exists. Doing it after deletion is a support exercise. Front-load the work.

## 4. Preserve mailbox and OneDrive content

Both are deleted with the account. Both need a decision before that happens.

**Mailbox.** Converting to a shared mailbox is usually the right call. The content is preserved, no license is required while it stays under 50 GB, and colleagues can be granted access to handle anything still arriving.

```powershell
# Exchange Online PowerShell
Set-Mailbox jdoe@contoso.com -Type Shared
```

If there's a legal or compliance requirement, a litigation hold or retention policy is the better instrument. If the content needs to leave the tenant entirely, export to PST before you touch anything else.

**OneDrive.** Assign a secondary owner while the account still exists:

```powershell
# SharePoint Online PowerShell
Set-SPOSite -Identity https://contoso-my.sharepoint.com/personal/jdoe_contoso_com `
    -Owner manager@contoso.com
```

If you skip this, the OneDrive is retained for 30 days after the account is deleted — by default, since the retention period is configurable in the SharePoint admin center — and then it's gone. That window relies on someone remembering the content exists, which is not a control.

## 5. Unassign the phone number

Relevant to anyone running Teams Phone, and skipped almost universally.

Remove the phone number assignment and voice policies **before** removing the Teams Phone license:

```powershell
# Microsoft Teams PowerShell
Remove-CsPhoneNumberAssignment -Identity jdoe@contoso.com -RemoveAll
```

Strip the license first and the number can remain associated with the account, which keeps it out of your available pool and makes reassignment more work than it should be. Doing it in this order costs nothing; doing it in the wrong order costs a support ticket.

## 6. Remove group memberships, and understand your licensing model

How licenses come back depends on how they were assigned:

- **Direct-assigned licenses** are released when the account is deleted.
- **Group-based licenses** are released when the user leaves the licensing group. Delete the account without removing them from that group first and the numbers in the admin center become harder to reason about than they need to be.

Dynamic groups resolve themselves once the underlying attributes change. Static groups need explicit removal.

Also remove privileged role assignments here, including PIM eligible assignments, which persist independently of active ones.

## 7. Delete the account

```powershell
Remove-MgUser -UserId jdoe@contoso.com
```

This is a soft delete. The account sits in the Entra ID recycle bin for 30 days and is fully restorable during that period, which is your safety net if the inventory in step 3 turned out to be incomplete.

```powershell
Get-MgDirectoryDeletedItemAsUser -All |
    Select-Object DisplayName, UserPrincipalName, DeletedDateTime
```

Hard-delete only when you're certain, because it forecloses recovery entirely. There are legitimate reasons to do it — a stakeholder decision that no data is to be retained, for instance — but it should be a decision rather than a default, and it should be recorded as one.

## 8. Verify license reclamation

Don't assume it worked. The license page in the admin center can lag, so confirm in PowerShell:

```powershell
Get-MgSubscribedSku |
    Select-Object SkuPartNumber, ConsumedUnits,
        @{ N = 'Available'; E = { $_.PrepaidUnits.Enabled - $_.ConsumedUnits } }
```

If the consumed count hasn't moved, something is still holding the license — most often group-based assignment that wasn't removed in step 6.

## The full checklist

```
[ ] Sign-in blocked
[ ] Sessions revoked
[ ] Owned objects inventoried (Get-MgUserOwnedObject)
[ ] Mailbox converted to shared, or placed on hold, or exported
[ ] OneDrive secondary owner assigned
[ ] Sole-owned groups and Teams reassigned
[ ] Sole-owned flows, apps, and connections reassigned
[ ] SharePoint site collection admin reassigned
[ ] Azure RBAC and app registration ownership reassigned
[ ] Recurring meetings identified and handed over
[ ] Phone number unassigned, voice policies removed
[ ] Devices removed from Intune and Entra
[ ] Privileged and PIM role assignments removed
[ ] Group memberships removed
[ ] Account deleted
[ ] License reclamation verified in PowerShell
[ ] Offboarding recorded: date, authorizer, what was retained and where
```

## The last line matters

Record the offboarding somewhere durable: the date, who authorized it, what was preserved, and where it went. Six months later, when somebody asks whether a departed employee's mailbox still exists, the answer needs to be a lookup rather than an investigation.

None of the steps above are difficult in isolation. What makes offboarding go wrong is that it's usually done in a hurry, by whoever is available, on the day someone leaves — which is precisely when nobody has time to work out what the account owned. Writing the sequence down once, and following it every time, is the whole trick.
