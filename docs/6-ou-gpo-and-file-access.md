# Active Directory OUs, Group Policy, and File Access

I set up department groups, shared folders, and user policies in `denyslab.com`. I used two accounts on `CLIENT-W11-01` to check which policies applied and whether the Finance drive was mapped.

## Requirement

Finance and HR need separate shared folders, with access assigned through AD security groups. I also wanted a common screen saver policy for department users and a mapped drive for Finance.

## Design

The lab has `Finance`, `HR`, `IT`, and `Workstations` OUs. I used `DC01` for the file shares because this lab is running on a host with limited memory. A separate file server is not part of this stage.

The folders are on `DC01` under `C:\SharedFolders`. Each folder has its own domain-local security group:

| Folder on DC01 | Share | Group with NTFS Modify permission |
| --- | --- | --- |
| `C:\SharedFolders\Finance` | `\\DC01\Finance` | `Finance_Local` |
| `C:\SharedFolders\HR` | `\\DC01\HR` | `HR_Local` |

For Finance, the account `jackhanma` belongs to `Finance_Hosts_Global`. That global group is a member of `Finance_Local`. I assigned the folder permission to the domain-local group rather than to Jack Hanma directly.

I configured two user GPOs. `User_ScreenLock` applies the screen saver settings to department users. `Finance-DriveMap` maps the Finance share to `F:` for members of `Finance_Hosts_Global`. HR users access their share by its network path; I have not configured an HR drive-mapping GPO.

## Implementation

### OU and group membership

In Active Directory Users and Computers, the `Finance` OU contains Jack Hanma and the `Finance_Hosts_Global` and `Finance_Local` security groups. The group membership shown below is:

`Jack Hanma → Finance_Hosts_Global → Finance_Local`

![Finance user and group membership](../screenshots/6-ad-group-membership.png)

*Jack Hanma is a member of `Finance_Hosts_Global`, which is nested in `Finance_Local`.*

### Shared folders

I created the Finance and HR folders under `C:\SharedFolders` on `DC01` and assigned Modify permission to different domain-local groups. The Security tabs also show SYSTEM and administrative accounts.

![Finance and HR folder permissions](../screenshots/6-file-ntfs-permissions.png)

*`Finance_Local` has Modify permission on Finance; `HR_Local` has Modify permission on HR.*

### Group Policy

I configured `User_ScreenLock` with these user settings:

- Enable screen saver: Enabled
- Force specific screen saver: `scrnsave.scr`
- Password protect the screen saver: Enabled
- Screen saver timeout: 1200 seconds (20 minutes)

![Screen saver policy settings](../screenshots/6-gpo-screenlock-settings.png)

*`User_ScreenLock` settings in Group Policy Management.*

For `Finance-DriveMap`, I used Group Policy Preferences with the following settings:

| Setting | Value |
| --- | --- |
| Action | Update |
| Location | `\\DC01\Finance` |
| Drive letter | `F:` |
| Reconnect | Enabled |
| Item-level targeting | `DENYSLAB\Finance_Hosts_Global` |

![Finance drive mapping policy](../screenshots/6-gpo-finance-drivemap.png)

*`Finance-DriveMap` maps `F:` to the Finance share for the Finance global group.*

## Validation

### Finance account

I signed in to `CLIENT-W11-01` as `DENYSLAB\jackhanma` and checked the user policy results. `gpresult` listed both `User_ScreenLock` and `Finance-DriveMap` as applied. The `net use` output showed `F:` connected to `\\DC01\Finance`.

![Finance user policy results and mapped drive](../screenshots/6-finance-gpresult-drive.png)

*Both user GPOs are applied, and `F:` is connected to the Finance share.*

### HR account

I also checked the HR user session on the same Windows 11 VM. The `gpresult` output showed `User_ScreenLock` applied. `Finance-DriveMap` was not in the applied list, and `net use` showed no mapped connections.

![HR user policy results](../screenshots/6-hr-gpresult.png)

*The HR session has the screen saver policy but no mapped Finance drive.*

In File Explorer, the HR share opened. An attempt to open `\\DC01\Finance` displayed a network-access error while Windows was still checking for details.

![HR share and Finance access attempt](../screenshots/6-hr-share-access-check.png)

*The HR share is open; Windows reports a problem accessing Finance.*

## Troubleshooting and remaining checks

The Finance path did not open during the HR test, but the screenshot does not show the final error code. I have not confirmed whether the failure was specifically caused by NTFS permissions. That still needs a final error message or an effective-access check.

The `gpresult` checks confirm that `User_ScreenLock` was applied to both test users. I have not recorded a separate test showing the workstation requesting a password after 20 minutes of inactivity. I also have not recorded a create/edit test on the shared folders.

## Next stage

Next I plan to back up the Windows configuration and shared files outside the ESXi host, then perform a small restore test.
