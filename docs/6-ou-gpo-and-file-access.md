# Active Directory Groups, GPOs, and File Shares

I configured department groups, file permissions, and two user policies in `denyslab.com`. I tested the results with Finance and HR accounts on `CLIENT-W11-01`.

## Requirement

Finance and HR need separate shared folders. Access should be controlled through Active Directory groups, not assigned to users one by one. I also wanted a common screen saver policy and an automatically mapped drive for Finance.

## Design

I used the `Finance`, `HR`, `IT`, and `Workstations` OUs. For this lab, `DC01` also hosts the shared folders. This keeps the number of VMs down on the 16 GB ESXi host.

The two shares are:

- `\\DC01\Finance` — `C:\SharedFolders\Finance`, with Modify permission for `Finance_Local`.
- `\\DC01\HR` — `C:\SharedFolders\HR`, with Modify permission for `HR_Local`.

For Finance, I used a global group for user membership and a domain-local group for folder permissions. The membership is `Jack Hanma → Finance_Hosts_Global → Finance_Local`. This keeps folder permissions attached to a group rather than to an individual account.

## Implementation

### Users and groups

The `Finance` OU contains Jack Hanma and the `Finance_Hosts_Global` and `Finance_Local` security groups. Jack Hanma is a member of `Finance_Hosts_Global`, which is nested in `Finance_Local`.

![Finance group membership](../screenshots/6-ad-group-membership.png)

*Finance user and group membership in Active Directory.*

### File permissions

I created the Finance and HR folders under `C:\SharedFolders`. Their NTFS permissions use separate domain-local groups. `Finance_Local` has Modify permission on Finance; `HR_Local` has Modify permission on HR. SYSTEM and administrator entries remain in place.

![Finance and HR NTFS permissions](../screenshots/6-file-ntfs-permissions.png)

*NTFS permissions on both department folders.*

### Group Policy

I configured two user GPOs.

**`User_ScreenLock`** enables the screen saver, uses `scrnsave.scr`, requires a password when resuming, and sets the timeout to **1200 seconds (20 minutes)**.

![Screen saver GPO settings](../screenshots/6-gpo-screenlock-settings.png)

*`User_ScreenLock` settings in Group Policy Management.*

**`Finance-DriveMap`** maps `\\DC01\Finance` to drive `F:` using Group Policy Preferences. The action is `Update`, `Reconnect` is enabled, and item-level targeting uses `DENYSLAB\Finance_Hosts_Global`.

![Finance drive mapping GPO settings](../screenshots/6-gpo-finance-drivemap.png)

*Drive mapping and Finance group targeting.*

There is no drive-mapping GPO for HR. HR users access their share through `\\DC01\HR`.

## Validation

I checked the user policies on `CLIENT-W11-01` using `gpresult /r /scope:user` and checked mapped drives with `net use`.

### Finance

Under `DENYSLAB\jackhanma`, `gpresult` listed both `User_ScreenLock` and `Finance-DriveMap`. `net use` showed `F:` connected to `\\DC01\Finance`.

![Finance policy results and mapped drive](../screenshots/6-finance-gpresult-drive.png)

*Both GPOs applied to Finance; drive `F:` is connected.*

### HR

Under the HR test account `hr_test`, `gpresult` listed `User_ScreenLock`. `Finance-DriveMap` was not applied, and `net use` showed no mapped drives.

![HR policy results](../screenshots/6-hr-gpresult.png)

*HR received the screen saver GPO without the Finance drive mapping.*

I opened `\\DC01\HR` in File Explorer under the HR account. An attempt to open `\\DC01\Finance` brought up a Windows network-access error.

![HR file-share access check](../screenshots/6-hr-share-access-check.png)

*HR share opened; the Finance path returned a network-access error.*

## Next stage

Next I will back up the Windows configuration and shared folder data outside the ESXi host and test restoring a small file.
