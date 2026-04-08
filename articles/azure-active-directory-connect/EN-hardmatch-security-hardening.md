---
title: Changes to Microsoft Entra Connect Hard Match Behavior
date: 2026-04-08 12:00:00
tags:
  - Microsoft Entra Connect
---

# Changes to Microsoft Entra Connect Hard Match Behavior

It was announced in the public information "Releases and Announcements” and MC1263280 that, for security enhancements, the behavior of hard match in Microsoft Entra Connect (hereafter MEC) will change starting in July 2026.

Public documentation
[General Availability - Microsoft Entra Connect security hardening to prevent user account takeover](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)

In this blog, we introduce the changes to the behavior and the hard match procedure after the change, and we answer frequently asked questions (FAQ) in a Q&A format. Going forward, we will add content as appropriate for frequently asked questions regarding hard match, and we will also provide additional explanations for items that are not described in the public documentation.

## 1. Changes to Microsoft Entra Connect Hard Match

### 1-1. Changes effective on or after July 1, 2026

In MEC, hard match refers to the method used, when synchronizing a new object from on-premises AD, to evaluate whether the on-premises AD account being synchronized matches an account in Microsoft Entra ID, based on the sourceAnchor attribute.

With the change scheduled to take effect on or after July 1, 2026, for security enhancement purposes, validation of the OnPremisesObjectIdentifier attribute will be added in addition to validation of the sourceAnchor attribute when synchronizing new objects from on-premises AD, and the behavior will change so that a hard match succeeds only when all of the following conditions are met.


- a. The ImmutableId of the Microsoft Entra ID account matches the sourceAnchor of the on-premises AD account being synchronized
- b. The tenant property BlockCloudObjectTakeoverThroughHardMatch is set to False
- **c. The OnPremisesObjectIdentifier attribute of the target Entra ID account is empty (New)**

**Previously, a hard match was performed as long as conditions a. and b. were met. However, with this behavior change, condition c. has been added.**

Once a user has been synchronized from on-premises AD even once, the OnPremisesObjectIdentifier attribute is populated with the object ID value of the source user.
Because this behavior change is applied when synchronizing new objects from on-premises AD, a hard match is performed only when the OnPremisesObjectIdentifier attribute is empty.


### 1-2. Hard match procedure after the hard match behavior change

To perform a hard match even after the behavior change, the OnPremisesObjectIdentifier attribute must be empty. Therefore, administrators must clear the OnPremisesObjectIdentifier of synchronized users in advance by using the Microsoft Graph API.
Below, we explain this based on a scenario in which hard match is used.

Note: In this procedure, the sourceAnchor is assumed to be configured as the mS-DS-ConsistencyGuid attribute in MEC. If an attribute other than mS-DS-ConsistencyGuid is used as the sourceAnchor, please read this procedure by replacing it with the attribute specified as the sourceAnchor.

#### Scenario: Linking an existing Entra ID user to a non-synchronized on-premises AD user

Due to reasons such as on-premises AD forest migration or employee secondment, there may be cases where the on-premises user that serves as the synchronization source for an existing Entra ID user needs to be changed. In this scenario, for an existing user A' in Entra ID, the on-premises user serving as the synchronization source is switched from user A in domain X to user B in domain Y.

Before:
Domain X on-premises user A -> Entra ID user A'
Domain Y on-premises user B

After:
Domain X on-premises user A
Domain Y on-premises user B -> Entra ID user A'

Hard match procedure:

1. To prevent unintended synchronization, [disable scheduled synchronization](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sync-feature-scheduler#disable-the-scheduler) in Entra Connect.

PowerShell command to run:
```powershell
Set-ADSyncScheduler -SyncCycleEnabled $false
```

2. For Entra ID user A', set onPremisesObjectIdentifier to null by using the Microsoft Graph API so that a hard match can be performed.

> [!NOTE]
> The Graph API that allows setting onPremisesObjectIdentifier to null will be released first in the Beta version of the API.
> The release timeline for the v1.0 API is currently undetermined. 
> Even for synchronized users such as user A', it is possible to set onPremisesObjectIdentifier to null without changing the Source of Authority (SOA).

Required permissions:
- User-OnPremisesSyncBehavior.ReadWrite.All
- User.ReadWrite.All

Graph API to run:
```http
PATCH https://graph.microsoft.com/beta/users/<UserId>
{ onPremisesObjectIdentifier: null }
```

Example using Microsoft Graph PowerShell :
```powershell
Connect-MgGraph  -Scopes 'User-OnPremisesSyncBehavior.ReadWrite.All','User.ReadWrite.All' -TenantId  contoso.onmicrosoft.com
Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/beta/users/12345678-2468-adcd-dcba-1234567890ab"
Invoke-MgGraphRequest -Method PATCH -Uri "https://graph.microsoft.com/beta/users/12345678-2468-abcd-dcba-1234567890ab" -Body '{"onPremisesObjectIdentifier": null }'
```

3. Place on-premises user B in a non-synchronized OU, and set the mS-DS-ConsistencyGuid attribute to the same value as the mS-DS-ConsistencyGuid attribute of on-premises user A.

Note that the non-synchronized OU must be prepared in advance by using [Domain and OU filtering](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-custom#domain-and-ou-filtering) in the MEC configuration wizard.

4. Move on-premises user A to a non-synchronized OU.

5. Run [Delta synchronization](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-sync-feature-scheduler#delta-sync-cycle) 
twice, and delete MEC and Entra ID user A' from Entra ID. Entra ID user A' will then become a "Deleted user".


PowerShell command to run:
```powershell
Start-ADSyncSyncCycle delta
```

6. Move on-premises user B to a synchronized OU.

7. Run a delta synchronization and synchronize the information of on-premises user B to Entra ID user A'.

PowerShell command to run:
```powershell
Start-ADSyncSyncCycle delta
```

8. A hard match is performed, linking Entra ID user A' with on-premises user B, and from that point onward, changes made to on-premises user B will be synchronized to Entra ID user A'.

9. Re-enable scheduled synchronization in Entra Connect.

PowerShell command to run:
```powershell
Set-ADSyncScheduler -SyncCycleEnabled $true
```

## 2. Frequently Asked Questions and Answers Regarding Changes to Hard Match Behavior (FAQ)

Below is a compilation of frequently asked questions regarding the recent changes to hard match behavior.

---
### <span style="color: blue; ">Q:</span> Is this change effective only for specific versions of MEC, or does it also apply to older versions?

<span style="color: red; ">A:</span> 
This change is implemented on the Microsoft Entra ID service side. Since validation of the OnPremisesObjectIdentifier attribute is performed by the service during hard match operations, it is not dependent on the MEC version. No MEC upgrade is required, and this change applies to all supported MEC versions on or after July 1, 2026.

---
### <span style="color: blue; ">Q:</span> Is it possible to stop the new hard match behavior that validates whether the OnPremisesObjectIdentifier attribute is empty?
<span style="color: red; ">A:</span> 
: No, it is not possible to stop the new behavior that validates hard match.

---
### <span style="color: blue; ">Q:</span> When will this change be applied to each tenant?
 
<span style="color: red; ">A:</span> 
The change will be rolled out to tenants in a phased manner starting on July 1, 2026. The specific rollout date for each tenant has not been publicly disclosed.

---
### <span style="color: blue; ">Q:</span> In the following public announcement, there is a reference to a “recommended flag to disable hard match takeover.” What does this flag refer to?
 
[General Availability - Microsoft Entra Connect security hardening to prevent user account takeover](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)
 
<span style="color: red; ">A:</span> 
It refers to the tenant property BlockCloudObjectTakeoverThroughHardMatch. This is a flag that disables hard match and is an existing setting, which by default is configured to allow hard match (False). For improved security, it is recommended to set this flag to block hard match during normal operation (True). However, when synchronizing a new user by using hard match, this flag must be set to False.


---
### <span style="color: blue; ">Q:</span> How can I check the current setting of BlockCloudObjectTakeoverThroughHardMatch?
 
<span style="color: red; ">A:</span> 
You can check it by using the following command.

```PowerShell
Connect-MgGraph -Scopes "OnPremDirectorySynchronization.Read.All"
(Get-MgBetaDirectoryOnPremiseSynchronization).Features |select BlockCloudObjectTakeoverThroughHardMatchEnabled
```

Note that when changing the BlockCloudObjectTakeoverThroughHardMatch setting, it can be modified by using a command such as the one shown below. (In the example below, BlockCloudObjectTakeoverThroughHardMatch is set to false.)

```powershell
Connect-MgGraph -Scopes "OnPremDirectorySynchronization.ReadWrite.All"
$sync = Get-MgBetaDirectoryOnPremiseSynchronization | Select-Object -First 1
Update-MgBetaDirectoryOnPremiseSynchronization -OnPremisesDirectorySynchronizationId $sync.Id -BodyParameter @{features=@{blockCloudObjectTakeoverThroughHardMatchEnabled=$false}}
```


References：
[Get onPremisesDirectorySynchronization - Microsoft Graph v1.0 | Microsoft Learn](https://learn.microsoft.com/en-us/graph/api/onpremisesdirectorysynchronization-get?view=graph-rest-1.0&tabs=powershell)
 
[Get-EntraDirSyncFeature (Microsoft.Entra.DirectoryManagement) | Microsoft Learn](https://learn.microsoft.com/en-us/powershell/module/microsoft.entra.directorymanagement/get-entradirsyncfeature?view=entra-powershell)

---
### <span style="color: blue; ">Q:</span> Are users for whom the OnPremisesObjectIdentifier attribute is not synchronized affected by this change?
 
<span style="color: red; ">A:</span>
Users for whom the OnPremisesObjectIdentifier attribute is not synchronized are not affected.

---
### <span style="color: blue; ">Q:</span> Are users who were previously synchronized using hard match affected by this behavior change? 

<span style="color: red; ">A:</span>
Since this change is evaluated at the time a hard match is performed, it does not affect users who have already been synchronized using hard match.

---
### <span style="color: blue; ">Q:</span> After the rollout in July 2026, is it still possible to set BlockCloudObjectTakeoverThroughHardMatch to True?
 
<span style="color: red; ">A:</span>
Yes, you can.

---
### <span style="color: blue; ">Q:</span> Is there a way to identify users whose synchronization source was switched as a result of hard match?

<span style="color: red; ">A:</span>
No, there is no way to directly identify such users. However, as an indirect method to help determine whether the synchronization source may have changed due to hard match, it is possible—only in cases where the sourceAnchor is set to mS-DS-ConsistencyGUID—to check the sourceAnchor and onPremisesObjectIdentifier values of a user synchronized to the cloud. When mS-DS-ConsistencyGUID is used as the sourceAnchor, if synchronization is performed while mS-DS-ConsistencyGUID is null, the sourceAnchor is generated based on the on-premises Object GUID, and the generated sourceAnchor is written back to mS-DS-ConsistencyGUID. Therefore, unless synchronization is performed after intentionally setting a different value in mS-DS-ConsistencyGUID, the mS-DS-ConsistencyGUID and onPremisesObjectIdentifier values will match. If they do not match, it can be inferred that hard match may have been used.
 
---
### <span style="color: blue; ">Q:</span> The following MC was announced. This blog does not mention hard match for administrator users—does MC1262584 relate to the content of this blog?

MC1262584
Upcoming change – Microsoft Entra Connect security update to block hard match for users with Microsoft Entra roles
 
<span style="color: red; ">A:</span>
MC1262584 concerns blocking hard match attempts that target cloud users with administrative roles. It is a separate announcement and is not related to the content of this blog.
 
---


## 3. References

[General Availability - Microsoft Entra Connect security hardening to prevent user account takeover](https://learn.microsoft.com/en-us/entra/fundamentals/whats-new#general-availability---microsoft-entra-connect-security-hardening-to-prevent-user-account-takeover)

[Microsoft Entra Connect: When you have an existing tenant](https://learn.microsoft.com/en-us/entra/identity/hybrid/connect/how-to-connect-install-existing-tenant)

[Get onPremisesDirectorySynchronization](https://learn.microsoft.com/ja-jp/graph/api/onpremisesdirectorysynchronization-get?view=graph-rest-1.0&tabs=powershell)
 
[Get-EntraDirSyncFeature](https://learn.microsoft.com/en-us/powershell/module/microsoft.entra.directorymanagement/get-entradirsyncfeature?view=entra-powershell)
