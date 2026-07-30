---
description: A sequenced Microsoft 365 offboarding runbook covering session revocation, ownership handover, and license reclamation verification.
---

# Offboarding a Microsoft 365 User Without Leaving Orphans

Deleting a Microsoft 365 user takes about four clicks. Doing it properly takes considerably longer, and the gap between the two usually doesn't show up for weeks.

Here's the problem with most offboarding guides. They treat the user account as the only thing being removed. But an account that's been active for a year or two has accumulated ownership of things: flows, groups, a SharePoint site, a phone number, a calendar full of recurring meetings that only it can amend. Delete the account and none of that fails loudly. It fails quietly, weeks later, and by then nobody connects the broken thing to a departure they've forgotten about.

So this is the sequence, and more usefully, why the steps sit in this order.

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

Everything below explains why each line earns its place.

All examples use the Microsoft Graph PowerShell SDK. If you find a guide using `Connect-MsolService`, it's out of date — the MSOnline and AzureAD modules are retired.

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Directory.ReadWrite.All","Organization.Read.All"
```

## 1. Block sign-in

First action, always.

```powershell
Update-MgUser -UserId jdoe@contoso.com -AccountEnabled:$false
```

New authentication attempts stop immediately. For a planned departure you can run this on the last day. For an involuntary one it goes before the conversation, not after. Same command; the timing is an HR call rather than a technical one.

## 2. Revoke active sessions

Most guides skip this, and skipping it leaves a gap you can measure in hours.

Blocking sign-in stops the account getting a *new* token. It does nothing about the ones already issued. Access tokens usually live around an hour and refresh tokens live far longer, which means someone with Outlook already open keeps reading mail after you've disabled them. People tend to assume that toggle is instant. It isn't.

```powershell
Revoke-MgUserSignInSession -UserId jdoe@contoso.com
```

That invalidates the refresh tokens so clients can't quietly renew. It's still not uniform: Exchange Online, SharePoint Online and Teams support Continuous Access Evaluation and react within a few minutes, while everything else waits out whatever's left of the access token lifetime.

Treat these first two steps as one action. Either alone isn't offboarding.

## 3. Inventory what the account owns

This is where a clean offboarding diverges from one you'll spend a week repairing.

Graph will tell you what the account directly owns:

```powershell
Get-MgUserOwnedObject -UserId jdoe@contoso.com -All |
    ForEach-Object { $_.AdditionalProperties['displayName'] }

Get-MgUserMemberOf -UserId jdoe@contoso.com -All |
    ForEach-Object { $_.AdditionalProperties['displayName'] }
```

That gets you groups, applications and service principals. It doesn't get you everything, and the gaps are exactly where the orphans live.

**Automation.** Any Power Automate flow whose only owner is the departing user stops running once the account goes. The connections belonged to that account and there's no recovering them, so you rebuild from scratch, assuming you can work out what they were connected to in the first place. Power Apps, custom connectors and connection references have the same problem. Reassign through the Power Platform admin center, or in bulk:

```powershell
# Microsoft.PowerApps.Administration.PowerShell
Set-AdminFlowOwnerRole -EnvironmentName <env> -FlowName <guid> `
    -RoleName CanEdit -PrincipalObjectId <new-owner-object-id> -PrincipalType User
```

**Groups and Teams.** A Microsoft 365 Group or Team with one owner becomes ownerless when that owner is deleted. Members carry on working in it quite happily and nobody can administer it: no adding members, no settings changes, no deleting it. Distribution lists with a single manager and SharePoint sites with a single site collection admin fail the same way.

**Azure and identity.** RBAC assignments where the departing user is the only principal with access to a resource group. App registrations and service principals listing them as sole owner. PIM assignments, including eligible ones that aren't currently active and are therefore easy to miss.

**Calendar.** Recurring meetings they organized lose their organizer. Attendees keep the series and nobody can change or cancel it.

**Devices.** Anything Intune-enrolled or Entra-registered under that account.

One practical note on all of this. Adding a co-owner to a group takes about ten seconds while the account exists. Doing the same thing after deletion is a support ticket. Front-load the work.

## 4. Preserve mailbox and OneDrive content

Both go with the account, so both need a decision first.

**Mailbox.** Converting to a shared mailbox is usually right: content survives, no license needed under 50 GB, and colleagues can be given access to handle whatever's still arriving.

```powershell
# Exchange Online PowerShell
Set-Mailbox jdoe@contoso.com -Type Shared
```

If there's a legal or compliance angle, use a litigation hold or retention policy instead. If the content has to leave the tenant, export to PST before touching anything else.

**OneDrive.** Assign a secondary owner while the account is still there:

```powershell
# SharePoint Online PowerShell
Set-SPOSite -Identity https://contoso-my.sharepoint.com/personal/jdoe_contoso_com `
    -Owner manager@contoso.com
```

Skip it and the OneDrive sits in a 30-day retention window after deletion, by default at least, since the period is configurable in the SharePoint admin center. Then it's gone. Relying on that window means relying on someone remembering the content exists, which isn't a control.

## 5. Unassign the phone number

If you're running Teams Phone this matters, and almost nobody mentions it.

Remove the number assignment and voice policies **before** you take the Teams Phone license away:

```powershell
# Microsoft Teams PowerShell
Remove-CsPhoneNumberAssignment -Identity jdoe@contoso.com -RemoveAll
```

Do it the other way round and the number can stay stuck to the account, out of your available pool, and getting it back is more work than it should be. Doing it in the right order costs nothing.

## 6. Remove group memberships and check your licensing model

How licenses come back depends entirely on how they went out.

Direct-assigned licenses are released when you delete the account. Group-based licenses are released when the user leaves the licensing group — so if you delete the account first and leave them in the group, the numbers in the admin center get harder to reason about than they need to be.

Dynamic groups sort themselves out once the underlying attributes change. Static groups need you to do it.

Strip privileged role assignments here too, PIM eligible ones included.

## 7. Delete the account

```powershell
Remove-MgUser -UserId jdoe@contoso.com
```

That's a soft delete. The account sits in the Entra ID recycle bin for 30 days, fully restorable, which is your safety net if step 3 turned out to be incomplete.

```powershell
Get-MgDirectoryDeletedItemAsUser -All |
    Select-Object DisplayName, UserPrincipalName, DeletedDateTime
```

Hard-delete only when you're sure, because it closes off recovery entirely. There are good reasons to do it — a stakeholder deciding no data is to be retained, for one — but it should be a decision someone made and recorded, not a default you fell into.

## 8. Verify license reclamation

Don't assume it worked. The admin center license page lags, so check in PowerShell:

```powershell
Get-MgSubscribedSku |
    Select-Object SkuPartNumber, ConsumedUnits,
        @{ N = 'Available'; E = { $_.PrepaidUnits.Enabled - $_.ConsumedUnits } }
```

If the consumed count hasn't moved, something's still holding that license. Nine times out of ten it's group-based assignment you didn't remove in step 6.

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

## Write down that you did it

Record the offboarding somewhere durable. The date, who authorized it, what was kept and where it went.

Six months later someone will ask whether a departed employee's mailbox still exists. You want that to be a lookup, not an investigation.

None of these steps is difficult on its own. Offboarding goes wrong because it happens in a hurry, on the day someone leaves, handled by whoever's free — which is precisely the moment nobody has time to work out what the account owned. Writing the sequence down once and following it every time is the whole trick.
