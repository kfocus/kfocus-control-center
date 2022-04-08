# Kubuntu Focus Control Center

## Overview
This is a fork of the TUXEDO Control Center which has been created by the team
at TUXEDO Computers GmbH. This repository is the source for patching the
upstream, and is not intended for new feature development. Instead we intend
to contribute PRs there to minimize feature divergence and to give back to all
those who contributed to the TUXEDO Control Center.

PLease see the [TUXEDO Control Center Repository][L0010] for further details.

## WorkFlows
### Workflow: Prepare Git Branches
```
# Desired Branches as shown below
# <ticketID>_<date>-release-<description>  <= Mainline feature branch
# <ticketID>_<date>-brand-<description>    <= Brand feature branch
# brand   <= origin/brand      Our branded release branch
# master  <= upstream/master   Upstream master  - do NOT edit
# release <= upstream/release  Upstream release - do NOT edit

# Setup, default branch is 'brand'
git clone git@github.com:kfocus/kfocus-control-center.git
git fetch -a --prune --all

# Add master and release branches from upstream
git checkout -b master
git branch --set-upstream-to=upsteam/master
git pull

git checkout -b release
git branch --set-upstream-to=upsteam/master
git pull
```

### Workflow: Develop New Feature for Upstream

New features are based off the release or master branch. At the time of this
writing (2022-04-09), the release branch was ahead of the master branch, and
so we use this as the source.

1. Pull upstream release branch as source of new feature branch.

```
git fetch -a --prune --all
git checkout release && git pull
git difftool upstream/release # Should show nothing
git checkout -b <ticketID>_<date>-release-<description>
```

2. Work on branch (Appendix A)
3. Update translations (Appendix B)
4. Create PR with upstream on Github.

### Workflow: Rebrand Upstream for Deployment

The 'brand' branch is based off the release branch. The steps below are
required to rebrand the product successfully.

1. Pull upstream release branch as a basis to create a new feature branch.
```
git fetch -a --prune --all
git checkout release && git pull
git difftool upstream/release # Should show nothing
```

2. Create copy of brand branch to merge release into
```
git fetch -a --prune --all
git checkout brand && git pull
git checkout -b <ticketID>_<date>-brand-<description>

# Merge release changes into this branch
git merge release

# Compare to existing brand to see changes
git difftool origin/brand
```

3. Work on branch (Appendix A)
4. Update translations (Appendix B)
5. Merge into a test branch before merging into brand
```
git fetch -a --prune --all
git checkout brand && git pull
git checkout -b test
git merge <brand-update-branch>

# Compare to origin brand to see changes
git difftool origin/brand

# Test function here
```

6. If test works, delete test branch and merge 
   <brand-update-branch> into 'brand'
```
git fetch -a --prune --all
git checkout brand && git pull
git brand -D test
git merge <brand-update-branch>

# Compare to existing brand to see changes
git difftool origin/brand

# Test function here, and then push
git push
```

7. Create patch
See [this link][L0030]:
```'
# See this file for exact steps employed
bin/make-brand-release-patch

# This will output a file named 
# `brand-<last-8-hash>_release-<last-8-hash>_<YYYY-mm-dd>.patch`
```

8. Test apply patch
```
git fetch -a --prune --all;
git checkout release && git pull;
git checkout -b 
git apply <patch-file-from-above>
```

9. Deliver the patch to packaging for deployment

## Appendix A: Work on Branch

Useful commands to run development.
```
npm install
npm run build
bin/deployctl stop
bin/deployctl start
npm run start
```

[Add context menu and inspect for development][L0010]
Set [electron to dev mode][L0020]: `set ELECTRON_IS_DEV=1 && electron ...`

## Appendix B: Create Translation
```
# Setup
npm run lang-gen # Generates lang.xlf file, below
cd src/ng-app/assets/locale
cp lang.en.xlf lang.tgt.xlf

# Copy over new strings and IDs, add or del new targets as needed
meld lang.xlf lang.tgt.xlf 

# Test
cp lang.tgt.xlf lang.en.xlf
cd ../../..
npm run build
npm run start

# If product does not work as expected, revert
# git checkout src/ng-app/assets/locale/lang.en.xlf

# Repeat with German if needed
```

[L0010]:https://www.npmjs.com/package/electron-context-menu
[L0020]:https://stackoverflow.com/questions/59019091/electron-run-in-production-mode
[L0030]:https://stackoverflow.com/questions/42695555

