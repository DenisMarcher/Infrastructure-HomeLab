# OUs, Group Policy, and File Access

I set up department groups, two shared folders, and two user GPOs in `denyslab.com`. I tested the policies and file access on `CLIENT-W11-01` using Finance and HR accounts.

## Requirement

Finance and HR needed separate shared folders. I wanted folder permissions assigned to AD groups rather than individual users. I also needed a screen saver policy for both departments and an automatically mapped drive for Finance.

## Design

The domain has `Finance`, `HR`, `IT`, and `Workstations` OUs. The Finance and HR file shares run on `DC01`.

For Finance, group membership and file permissions are kept separate:

`jackhanma` → `Finance_Hosts_Global` → `Finance_Local` → `\\DC01\Finance`

`Finance_Hosts_Global` contains the user; `Finance_Local` has Modify permission on the folder. This follows AGDLP (Account → Global group → Domain Local group → Permission).

The HR folder uses a separate domain-local group, `HR_Local`, with Modify permission.

Despite its name, `Finance_Hosts_Global` contains users, not computer accounts.

The folders are stored at `C:\SharedFolders\Finance` and `C:\SharedFolders\HR`. They are shared as `\\DC01\Finance` and `\\DC01\HR`.

I used one GPO for screen saver settings and another for the Finance drive mapping. HR can open its share directly; I did not configure an HR drive mapping.

## Implementation

### Active Directory groups

In the `Finance` OU, I added Jack Hanma to the global security group `Finance_Hosts_Global` and added that group to the domain-local `Finance_Local` group.

![Finance user and group membership](../screenshots/6-ad-group-membership.png)

*Finance group membership in Active Directory Users and Computers.*

### File permissions

On `DC01`, I configured separate NTFS permissions for the Finance and HR folders. `Finance_Local` has Modify permission on Finance, and `HR_Local` has Modify permission on HR. The SYSTEM and administrator entries are also present.

![Finance and HR folder permissions](../screenshots/6-file-ntfs-permissions.png)

*Separate NTFS permissions for `Finance` and `HR`.*

The screenshots show NTFS permissions and user access tests, not the share-level ACLs.

### Group Policy

`User_ScreenLock` enables the screen saver, requires a password on resume, and sets a **1200-second (20-minute)** timeout.

![Screen saver policy settings](../screenshots/6-gpo-screenlock-settings.png)

*Screen saver and password-on-resume settings in `User_ScreenLock`.*

`Finance-DriveMap` maps `\\DC01\Finance` to drive `F:`. The action is `Update`, `Reconnect` is enabled, and item-level targeting checks membership in `DENYSLAB\Finance_Hosts_Global`.

![Finance drive mapping settings](../screenshots/6-gpo-finance-drivemap.png)

*Drive `F:` and the Finance group filter in Group Policy Preferences.*

The GPMC Scope views show `User_ScreenLock` linked to the `Finance`, `HR`, and `IT` OUs, and `Finance-DriveMap` linked to the `Finance` OU. All four links shown in the screenshot are enabled.

![GPO links](../screenshots/6-gpo-links.png)

*Link locations of both GPOs in Group Policy Management.*

## Validation

### Finance

I signed in to `CLIENT-W11-01` as `DENYSLAB\jackhanma` and ran `gpresult /r /scope:user` and `net use`. Both `User_ScreenLock` and `Finance-DriveMap` appeared in the applied GPO list, and `F:` was connected to `\\DC01\Finance`.

![Finance GPO results and mapped drive](../screenshots/6-finance-gpresult-drive.png?v=575c4ee)

*The Finance user received both GPOs. The mapped drive was checked separately with `net use`.*

I created `finance-test.txt` on drive `F:` as `jackhanma` and found the same file under `C:\SharedFolders\Finance` on `DC01`. I also checked its contents on the server.

![Finance write test](../screenshots/6-finance-write-test.png)

*A file created through the mapped drive appears in the Finance folder on `DC01`.*

### HR

I repeated the checks as `hr_test`. `gpresult` listed `User_ScreenLock`, while `net use` showed no mapped drives. The Finance mapping was not applied to this account.

![HR GPO results](../screenshots/6-hr-gpresult.png)

*The HR user received `User_ScreenLock` without the Finance drive mapping.*

I opened `\\DC01\HR` in File Explorer under the HR account. In the same session, `dir \\DC01\Finance` returned `Access is denied.`

![HR and Finance share access checks](../screenshots/6-hr-share-access-check.png?v=575c4ee)

*The HR share opened; the Finance share returned `Access is denied`.*

## Scope of checks

The screenshots cover NTFS permissions, GPO application, and file access under Finance and HR accounts. They do not cover share-level ACLs or the complete HR group chain. The last captured computer-object location was the default `Computers` container. Windows Server is an evaluation build.

## Next stage

[Linux and Docker](7-linux-and-docker.md) covers Debian administration, storage, and containers.
