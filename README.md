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
# <ticketID>_<YYYY-mm-dd>_release_<description>  <= Mainline feature branch
# <ticketID>_<YYYY-mm-dd>_brand_<description>    <= Brand feature branch
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
git checkout -b <ticketID>_<YYYY-mm-dd>_release_<description>
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
git checkout -b <ticketID>_<YYYY-mm-dd>_brand_<description>

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
# `brand_<last-8-hash>_release_<last-8-hash>_<YYYY-mm-dd>.patch`
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

## Appendix C: Minimal Daemon Setup

## 1 Purpose
We are considering packaging the tcc daemon only without CPU, webcam, or display
brightness controls. This will allow users to run a quieter fan curve without
the complexity of a full control center implementation. This will also allow
us to migrate to a rebranded Tuxedo Control Center in steps and ensure that we
can work together in harmony without requiring immediate or significant
workflow changes for our customers.

## 2 Research
We do not need Electron or the Angular app. The goal is to keep the service
app with the least amount of changes, and simply feed the parameter to DISABLE
CPU and screen controls. This reverse engineering is done from the TCC release
branch on 2022-04-11.

### 2.1 Systemd Configuration
The daemon is controlled by Systemd. This requires the following:

1. Add systemd service files
2. Add dbus config file
3. Start the daemon
4. Stop the daemon
5. Remove daemon

See bin/deployctl to see much of this in action. More detail is included below
as part of configuration consideration.

#### 2.1.1. Add systemd service files
Copy the service files:
```
sudo cp "${_baseDir}/src/dist-data/tccd.service" \
  "${_baseDir}/src/dist-data/tccd-sleep.service" \
  /etc/systemd/system/ || _abortFn;
```

The main service is tccd.service. It launches the executable, which is a
wrapped nodejs app an npm package called `pkg`.

```
[Unit]
Description=TUXEDO Control Center Service

[Service]
Type=simple
ExecStart=/.../dist/tuxedo-control-center/data/service/tccd --start
ExecStop=/../dist/tuxedo-control-center/data/service/tccd --stop

[Install]
WantedBy=multi-user.target
```
We change the path of the executable AND the shared library file which is next
to it.

The tccd-sleep.service file provides suspend and resume rules:
```
[Unit]
Description=TUXEDO Control Center Service (sleep/resume)
Before=sleep.target
StopWhenUnneeded=yes

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c "systemctl stop tccd"
ExecStop=/bin/bash -c "systemctl start tccd"

[Install]
WantedBy=sleep.target
```

#### 2.1.2. Add dbus config file
```
sudo cp "${_baseDir}/src/dist-data/com.tuxedocomputers.tccd.conf" \
  /usr/share/dbus-1/system.d/
```
The content of com.tuxedocomputers.tccd.conf is below:
```
<!DOCTYPE busconfig PUBLIC "-//freedesktop//DTD D-Bus Bus Configuration 1.0//EN"
    "http://www.freedesktop.org/standards/dbus/1.0/busconfig.dtd">

<busconfig>
  <type>system</type>
  <policy user="root">
    <allow own="com.tuxedocomputers.tccd" />
    <allow send_destination="com.tuxedocomputers.tccd" />
    <allow receive_sender="com.tuxedocomputers.tccd" />
  </policy>
  <policy context="default">
    <allow send_destination="com.tuxedocomputers.tccd" />
    <allow receive_sender="com.tuxedocomputers.tccd" />
  </policy>
</busconfig>
```

#### 2.1.3. Start the daemon
```
sudo systemctl start tccd
sudo systemctl enable tccd tccd-sleep
sudo systemctl daemon-reload
```

#### 2.1.4. Stop the daemon
```
sudo systemctl stop tccd               || _abortFn;
sudo systemctl disable tccd tccd-sleep || _abortFn;
```

#### 2.1.5. Remove daemon
```
sudo rm /etc/systemd/system/tccd*
sudo rm /usr/share/dbus-1/system.d/com.tuxedocomputers.tccd.conf
```

### 2.2 Building the Daemon
It appears that just the service can be build using `npm build-service`.
However, testing this did not do the trick. After reviewing the build steps,
I tried the following and it worked:

```
binr/deployctrl stop # Fans should spin here
rm -rf node_modules
npm run clean
npm install
npm run build-service
# build-native       # This was NOT needed...
# npm run copy-files # The subset, copy-native below, was sufficient
npm run copy-native
binr/deployctrl start
```

### 2.3 Adjusting the Daemon
Our initial goal is to deploy the balanced fan curves only. All other features
are to be disabled.

1. Config paths and files
2. Set the fan curve
3. DO NOT set CPU parameters
4. DO NOT set display brightness
5. DO NOT adjust Webcam on or off

#### 2.3.1. Config paths and files
I found this file src/common/classes/TccPaths.ts which enumerates the
paths used by the tccd. However, this appears to be set by the app, not read
on startup. Or, if it is, it doesn't clean up the files. I found the files
there to reference a non-existent custom profile.  When I deleted the entire
directory and restarted the daemon, /etc/tcc was recreated, but the stored
JSON values in 'autosave', 'profiles', and 'settings' appeared to have been
reset to the defaults.

```
export class TccPaths {
    static readonly PID_FILE: string = '/var/run/tccd.pid';
    static readonly TCCD_EXEC_FILE: string = '/opt/tuxedo-control-center/resources/dist/tuxedo-control-center/data/service/tccd';
    static readonly SETTINGS_FILE: string = '/etc/tcc/settings';
    static readonly PROFILES_FILE: string = '/etc/tcc/profiles';
    static readonly AUTOSAVE_FILE: string = '/etc/tcc/autosave';
    static readonly FANTABLES_FILE: string = '/etc/tcc/fantables';
    static readonly TCCD_LOG_FILE: string = '/var/log/tccd/log';
}
```

IMPORTANT: The `TCCD_EXE_FILE` path needs to be updated to someplace simpler.

#### 2.3.2 Set the fan curve
I found I could adjust the fan curves by modifying
`src/common/models/TccFanTables.ts` using the 'Balanced' profile. The build
process from section 2.2 had to be repeated.

#### 2.3.3. DO NOT set CPU parameters
See src/common/models/TccSettings.ts. I have modified it as below to disable
CPU settings.

```
 32 export const defaultSettings: ITccSettings = {
 33     stateMap: {
 34         power_ac: 'Default',
 35         power_bat: 'Default'
 36     },
 37     shutdownTime: null,
 38     cpuSettingsEnabled: false,
 39     fanControlEnabled: true,
 40     ycbcr420Workaround: []
 41 };
```

Again, a rebuild was necessary to restart this.

#### 2.3.4. DO NOT set display brightness
#### 2.3.5. DO NOT adjust Webcam on or off
See src/common/models/TccProfile.ts for both of these requirements.
These adjustements were made and the daemon rebuilt and redeployed per section
2.2.

```
 76 export const defaultProfiles: ITccProfile[] = [
 77     {
 78         name: 'Default',
 79         display: {
 80             brightness: 100,
 81             useBrightness: false
 82         },
 83         cpu: {
 84             onlineCores: undefined,
 85             useMaxPerfGov: false,
 86             scalingMinFrequency: undefined,
 87             scalingMaxFrequency: undefined,
 88             governor: 'powersave', // unused: see CpuWorker.ts->applyCpuProfile(...)
 89             energyPerformancePreference: 'balance_performance',
 90             noTurbo: false
 91         },
 92         webcam: {
 93             status: true,
 94             useStatus: false
 95         },
 96         fan: {
 97             useControl: true,
 98             fanProfile: 'Balanced',
 99             minimumFanspeed: 0,
100             offsetFanspeed: 0
101         },
102         odmProfile: { name: undefined }
103     }
104 ];
```

[L0010]:https://www.npmjs.com/package/electron-context-menu
[L0020]:https://stackoverflow.com/questions/59019091/electron-run-in-production-mode
[L0030]:https://stackoverflow.com/questions/42695555

