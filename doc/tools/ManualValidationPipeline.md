# Validation Pipeline

- Unified data file
- Addition and Removal
- Manual Validation Pipeline
- Validation process
- Approval checks

## Unified data file ManualValidationPipeline.csv

Replaces and adds: 

- Auth.csv
  - The Auth file was originally meant to be a stopgap solution, to restrict some uploads to certain users, until the Authorized Publishers feature is complete. It has since grown to include additional features. 
- Review.csv
  - To identify and block on issues that are otherwise hard to capture, the Review file allows a note to show in the PR Watcher output when the specified package is reviewed.
- Auto-waiver system
  - Filling a gap in the waiver application system, this adds a waiver when a package fails with a specific label, but logs show no error. The waiver is only applied if the error label matches the label listed in waiver.csv. 
- Version Parameter check override.
  - Versions are usually dot-delimited integers. Version parameters are the characters (major, minor, etc) between the dots. 
  - Semantic versions can have up to 4 parameters, while strings can have many more. 
  - For most packages, the number of versions does not change very often. But for some packages, one version might have major, minor, and patch, while the subsequent adds a build parameter, and the following removes both the patch and the build. 
  - Packages in this list skip this check.
- Code 200 override 
  - For removing packages that are still available, hence the "Code 200" is the HTTP response code. 
  - This is for packages that frequently get their manifest removed while the package is still available from the host. 
- Agreement check 
  - For the new Agreements section, where the Agreement was verified. 

### Data structure

The file is a CSV as named, with another slash-delimited list of users nested in the gitHubUserName column. Headers:

 ```
PackageIdentifier, gitHubUserName, authStrictness, authUpdateType, autoWaiverLabel, versionParamOverrideUserName, versionParamOverridePR, code200OverrideUserName, code200OverridePR, AgreementOverridePR, reviewText
```

- Formerly Auth.csv
  - `PackageIdentifier` matches to `PackageIdentifier`s from PRs.
  - `gitHubUserName` matches to a GitHub account name.
  - `authStrictness` can be either `should` or `must`. 
  - `authUpdateType` is new, to control if the new automatic update feature should update this row.
- Auto-waiver system
  - `autoWaiverLabel` holds the label allowed for auto-waiver.
- Version Parameter check override.
  - `versionParamOverrideUserName` User who requested override. Might merge with `gitHubUserName` above.
  - `versionParamOverridePR` PR where override was requested.
- Code 200 override - for removing packages that are still available.
  - `code200OverrideUserName` User who requested override. Might merge with `gitHubUserName` above.
  - `code200OverridePR` PR where override was requested.
- Agreement check - for the new Agreements section.
  - `AgreementOverridePR` PR where Agreement was verified.
- Formerly Review.csv (pending)
  - `reviewText` holds warnings or alerts about packages.
  - `reviewPR` PR where review check was requested.

Being listed in the `gitHubUserName`, with either value for `authStrictness`, brings a few benefits:

- Control who uploads manifests for your software packages, for better control over releases. 
  - Block uploads simply by setting `Account` to your GitHub account and `strictness` to `must`, then simply not uploading.
- Skip some Approval checks, such as the version parameter check and no blocking on package removal.

## Word Filter List

The Word Filter List began as a way to catch EULA and other license agreement installer switches, and has expanded as a way to prevent scripted applications and specific hosts. 

### Data structure

The WordFilterList.txt file is a plain text file, interpreted as an array of strings. (To-do - currently a variable in the PS1 file.)

## Version Parameter check


### Data structure

The VersionParameterCheckSkipList.csv file is CSV as named, with 3 columns: PackageIdentifier, UserName who authorized, and PR number where authorized.

## Addition and Removal

We have a policy of one manifest change per PR. This means that adding a new manifest version and removing the previous usually requires 2 PRs, an addition PR and a removal PR.

- Most PRs are "addition PRs", where they add a manifest version to the repository. 
- "Removal PRs" are where a PR is removed, and usually these have "Remove", "Delete", or "Automatic deletion" in the title. 
- Some PRs are a special type of addition-removal. This doesn't violate the one manifest change per PR rule.
- Move PRs are uncommon except on moving day, when they're prevalent. Usually these come in pairs - an addition PR and a removal PR. 

## Manual Validation Pipeline

Sometimes, the pipeline has an internal error. If the error happens during a later pipeline stage, it receives the label `Internal-Error-Dynamic-Scan`. If this occurs, then a team member can perform a Manual Validation, by installing the manifest locally in a VM and performing a Defender scan. If the application installs and launches correctly, and passes a Defender scan, then the team member adds the label `Validation-Complete`. This has been largely automated and is referred to as the Manual Validation Pipeline.

## Validation process

Validation usually proceeds in this way:

1. PR opened.
1. Automatic Validation Pipeline performs analysis of manifest and URLs, then automated install of package in a VM. Any errors in this step will add an error label to the PR. If no errors, proceed to step 7.
1. Error label applied to PR. 
1. An hourly scripted process will extract the error from the validation run logs and add to the PR comments.
  - If no errors, and the package is on the auto-waiver list (described below) then the PR gets a waiver.
  - For Defender errors, it will retry after an 18-hour break. This should be long enough to allow Defender teams to update signatures and clear false positives, and short enough to minimize blocking.
  - Some other specific types of PR have special handling.
1. Remediation of errant PRs.
  - Manifest errors either need a manifest update or sometimes a manifest path update. The error should be available in the comments.
  - URL and download errors can sometimes be fixed through removing the URL from the manifest, if it's not an `InstallerUrl`. If it is, then another source for the package would be needed for the PR to proceed.
  - Installer errors can sometimes be fixed by updating installer switches, or adding a dependency if the error states that one is missing.
  - Application launch and run errors can also sometimes be fixed through adding a dependency, again only if one is called for.
  - Other kinds of errors can't easily be advanced past and might block a PR for some time.
  - If no errors, then install the manifest in a local VM - essentially the Manual Validation Pipeline again - and examine. if still no errors, then the PR gets a waiver. (Some packages, most frequently CLI and tray-based, require manual review.)
1. After remediation, return to step 2.
  - If the issue couldn't be remediated, then the PR doesn't have a clear path and might linger for some time.
1. `Validation-Complete` label applied.
1. Moderator reviews, performs Validation Checks below, and approves if they pass.
1. Publish pipeline reviews and publishes repository into WinGet source data file.
1. CDN flushes to repopulate with the most recent WinGet source data file version.
1. New manifest data is available to WinGet users.

## Approval checks

The following checks are performed on each PR before approval:

- User has a CLA on file.
- Has passed through the Validation process described above and received the label `Validation-Complete` but not any error labels. (`Azure-Pipeline-Passed` label optional.)
- All Conversations are Resolved. (This blocks merging.)
- Any concerns raised are settled and any discussions have reached a conclusion.
- Manual package review, using the Manual Validation Pipeline yet again. 
  - This check is skipped for new versions of packages with manifests already in the repository. 
- PR Version is valid, and either semantic or string. 
  - Note: "latest" is a reserved version number and will need verification. 
- `A` - Auth.csv review.
  - Packages with "must" in Auth.csv have to be submitted by a GitHub user account listed in the `Accounts` column. If not, then an automated message will ask these users to review the PR.
- `R` - Review.csv entry may block approval.
- `W` - Word Filter review doesn't find any listed words.
- `F` - Apps & features or "ARP" (Add & Remove Programs) entries mismatch. Checks against the previous manifest version. This check is skipped for new packages.
  - Previous no, current no - output 0 and continue.
  - Previous yes, current yes - output 1 and continue.
  - Previous no, current yes - automated response and continue.
  - Previous yes, current no - automated response and block.
- `I` - InstallerUrl contains PackageVersion. This check doesn't block but is informative. In the future it might check for other manifest versions, to ensure the installer isn't for another version of the software package.
- `D` -  Directory Listing - If the PR has more than 2 files, then this checks the previous version to see if the files are the same. Blocks if files are different, otherwise output the number of files in the directory. Only checks on addition PRs and skips new packages.
- `V` -  Versions remaining - If the PR is the highest version remaining in the repo, automated reply, and block if the user isn't in Auth.csv. Otherwise output the number of versions in the version directory. Only checks on removal PRs.
- Previous manifest version. 
  - If the previous version is lower than the current PR version, then it's a newer version, output the previous version. 
  - If the previous version is equal to the current PR version, then they're the same (it's an update) so output "=". 
  - If the previous version is higher than the current PR version, then it's a newer version, output the previous version, but in the "caution" color for the selected chromatic set. 

(The letters are meant to not spell any word. If this spells a word, please let me know.)

