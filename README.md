# Kubuntu Focus Control Center

## Overview
This is a fork of the TUXEDO Control Center which has been created by the team
at TUXEDO Computers GmbH. This repository is the source for patching the
upstream, and is not intended for new feature development. Instead we intend
to contribute PRs there to minimize feature divergence and to give back to all
those who contributed to the TUXEDO Control Center.

PLease see the [TUXEDO Control Center Repository][L0010] for further details.

## WorkFlows

### Primary Branches
master  <= upstream/master
release <= upstream/release
rebrand

### Upstream Feature Development Workflow

New features are based off the master branch.

1. Merge upstream changes into master and create a new feature branch.

```
git checkout master
git fetch -a --prune
git pull
git merge upstream/master
git difftool upstream/master # Should show nothing
git checkout -b <ticketID>_<date>-master-<description>
```
2. Work on branch (Appendix A)
3. Update translations (Appendix B)
4. Create PR with upstream on Github.

### Update 'brand' Branch

The 'brand' branch is based off the release branch.

1. Update release branch from upstream
```
git checkout release
git fetch -a --prune
git pull
git merge upstream/release
git difftool upstream/release # Should show nothing
```

2. Create a copy of existing brand branch for development
```
git checkout brand
git fetch -a --prune
git pull
git checkout -b <ticketID>_<date>-brand-<description>

# Compare to existing brand to see changes
git difftool brand
```

3. Work on branch (Appendix A)
4. Update translations (Appendix B)
5. Merge into brand and check
```
git checkout brand
git fetch -a --prune
git pull
git merge <brand-update-branch>
git difftool origin/brand # Confirm
```

6. Create patch
See [this link][L0030]:
```
git checkout release && git fetch -a --prune && git pull;
git checkout brand   && git fetch -a --prune && git pull;
git diff brand..release > brand-<last-8-hash>_release-<last-8-hash>_<YYYY-mm-dd>.patch
```

7. Test apply patch
```
git checkout release  && git fetch -a --prune && git pull;
patch < brand-<date>.patch
```

8. Deliver the patch to packaging for deployment

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

