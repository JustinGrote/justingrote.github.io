---
title: Group Based Exchange Online Administration
classes: wide
toc: true
---

## The Old Way
In Exchange Online using Azure AD, the typical practice was that you had to directly assign users to the 
`Exchange Administrator` Azure AD administrative role. What happens behind the scenes is that a special Exchange role 
group named `ExchangeServiceAdmins_(randomid)` is created, and is then nested into `Organization Management` role.

## The Problem with the Fancy New Way
Recently a preview was made available to [allow cloud groups to be assigned roles](https://docs.microsoft.com/en-us/azure/active-directory/roles/groups-concept) so that you don't have to do this on an individual user level. This is great for a lot of Azure AD roles, but as of March 2021 it is broken for Exchange Online. If you enabled it and receive 503 service unavailable or 403 access denied errors with the Exchange Control Panel, you're in the right place.

## The Workaround
Thankfully Exchange already has Group Based administration built into its own separate role based framework, so you can use this in lieu of this new feature.

### Create a Mail-Enabled Universal Group
First you need to create a group (if creating the group in your onprem AD and syncing it to M365, it is VERY IMPORTANT that you both make it a universal security group and mail-enable it, otherwise it will be visible but not work). Wait for it to sync and confirm the group shows grouptype as `Universal, SecurityEnabled` when you run `Get-Group 'MyGroupName'`

### Assign the Group to the Organization Management Role
Next, you can add this group to the Organization Management role using the `Add-RoleGroupMember -Identity 'Organization Management' -Member 'NameOfMyGroup'` Powershell command. One completed, members of this group will have equivalent access you would have gained assigning the Azure AD `Exchange Administrator` role.

### Remove Existing Cloud Group Role Assignments
Finally be sure that you have removed all Azure AD cloud group role assignments from users who will be managing exchange. This assignment will override the native Exchange assignment and the users will still be broken.