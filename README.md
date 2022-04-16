# Kubuntu Focus Control Center

## 1. Overview
This is a fork of the TUXEDO Control Center which has been created by the team
at TUXEDO Computers GmbH. This repository is the source for patching the
upstream, and is not intended for new feature development. Instead, we intend
to contribute PRs there to minimize feature divergence and to give back to all
those who contributed to the TUXEDO Control Center.

PLease see the [TUXEDO Control Center Repository][L0005] for further details.

## 2. WorkFlows
### 2.1. Workflow: Prepare GIT branches and upstream

We clone the kfocus-control-center which sets the default branch to `brand`:

  ```
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

The desired branch names are shown below:

  ```
  # Feature branch to submit upstream as PR
  # <ticketID>_<YYYY-mm-dd>_release_<description>

  # Feature branch to apply changes to rebrand of TCC
  # <ticketID>_<YYYY-mm-dd>_brand_<description>
  ```

### 2.2. Workflow: Develop new feature for upstream
New features should be based off the release or master branch. At the time of
this writing (2022-04-09), the release branch was ahead of the master branch,
and so we use this as the source. The steps are summarized below:

- 2.2.1. Pull the upstream release branch
- 2.2.2. Create the feature branch 
- 2.2.3. Work on the feature branch
- 2.2.4. Update translations
- 2.2.5. Create a pull-request

#### 2.2.1. Pull the upstream release branch
This is used as the source of the new feature branch:

  ```
  git fetch -a --prune --all
  git checkout release && git pull
  git difftool upstream/release # Should show nothing
  ```

#### 2.2.2. Create the feature branch 
  ```
  git checkout -b <ticketID>_<YYYY-mm-dd>_release_<description>
  ```

#### 2.2.3. Work on the feature branch
See Appendix A.

#### 2.2.4. Update translations
See Appendix B.

#### 2.2.5. Create a pull-request
This is easiest to accomplish on the GitHub web interface.

### 2.3. Workflow: Rebrand upstream for deployment
The 'brand' branch is based off the release branch. The steps are summarized
below:

- 2.3.1. Pull the upstream release branch
- 2.3.2. Create the feature branch
- 2.3.3. Work on the feature branch
- 2.3.4. Update translations
- 2.3.5. Make and merge into the test branch
- 2.3.6. Confirm and merge into the brand branch
- 2.3.7. Create the patch
- 2.3.8. Test the patch
- 2.3.9. Send the patch to packaging

#### 2.3.1. Pull the upstream release branch
  ```
  git fetch -a --prune --all
  git checkout release && git pull
  git difftool upstream/release # Should show nothing
  ```

#### 2.3.2. Create the feature branch
Create a feature branch from the existing `brand` branch.

  ```
  git fetch -a --prune --all
  git checkout brand && git pull
  git checkout -b <ticketID>_<YYYY-mm-dd>_brand_<description>

  # Inspect and merge release changes into this branch
  git difftool release
  git merge release

  # Compare to origin/brand to see changes
  git difftool origin/brand
  ```

#### 2.3.3. Work on the feature branch
See Appendix A.

#### 2.3.4. Update translations
See Appendix B.

#### 2.3.5. Make and merge into the test branch
Compare to upstream brand

  ```
  git fetch -a --prune --all
  git checkout brand && git pull
  git checkout -b test
  git merge <brand-update-branch>

  # Compare to origin/brand to see changes
  git difftool origin/brand

  # Test function here
  ```

#### 2.3.6. Confirm and merge into the brand branch
If the test branch works, we switch back the `brand` branch, delete the test
branch, and then merge in the changes.

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

#### 2.3.7. Create the patch
See [this link][L0030]:

  ```
  # See this file for exact steps employed
  bin/make-brand-release-patch

  # This will output a file named
  # `brand_<last-8-hash>_release_<last-8-hash>_<YYYY-mm-dd>.patch`
  ```

#### 2.3.8. Test the patch
  ```
  git fetch -a --prune --all;
  git checkout release && git pull;
  git checkout -b
  git apply <patch-file-from-above>
  ```

#### 2.3.9. Send the patch to packaging

## Appendix A: Work on Branch

Useful commands to run development:

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

### C.1. Purpose
We are considering packaging the tcc daemon only without CPU, webcam, or display
brightness controls. This will allow users to run a quieter fan curve without
the complexity of a full control center implementation. This will also allow
us to migrate to a rebranded Tuxedo Control Center in steps and ensure that we
can work together in harmony without requiring immediate or significant
workflow changes for our customers.

### C.2. Research
We do not need Electron or the Angular app. The goal is to keep the service
app with the least amount of changes, and simply feed the parameter to DISABLE
CPU and screen controls. This reverse engineering is done from the TCC release
branch on 2022-04-11. Here are the steps we researched and resolve.
IMPORTANT: the script `bin/deployctl` embodies these steps and is always the
most up-to-date reference. This was last updated 2022-04-15.

- C.2.1. Build the daemon
- C.2.2. Adjust the daemon
- C.2.3. Add systemd service files
- C.2.4. Add dbus config file
- C.2.5. Start the daemon
- C.2.6. Stop the daemon
- C.2.7. Remove daemon

#### C.2.1. Build the daemon
It appears that just the service can be built using `npm build-service`.
However, testing this did not do the trick, and we applied some more research.
Run `bin/deployctl build` to execute these steps.

  ```
  _baseDir='/home/mmikowski/kcc';

  # Clean-up and previous daemons and files
  if sudo systemctl is-active --quiet tccd; then
    sudo systemctl stop tccd;
    sudo systemctl disable tccd tccd-sleep;
    sudo systemctl daemon-reload;
  fi
  rm -rf /etc/systemd/system/tccd*;
  rm -rf /usr/share/dbus-1/system.d/com.tuxedocomputers.tccd.conf;

  # Build service
  cd "${_baseDir}";
  npm run clean;
  rm -rf ./node_modules;
  npm install;

  # npm build-native was NOT needed
  npm run build-service;

  # npm run copy-files was NOT needed;
  # The copy-native subset is enough.
  npm run copy-native;
  ```

#### C.2.2. Adjust the daemon
Our initial goal is to deploy the balanced fan curves only. All other features
are to be disabled.

- C.2.2.1. Config paths and files
- C.2.2.2. Set the fan curve
- C.2.2.3. Disable CPU control
- C.2.2.4. Disable brightness and webcam control

##### C.2.2.1. Config paths and files
I found this file src/common/classes/TccPaths.ts which enumerates the paths used
by `tccd`. When I deleted the entire directory and restarted the daemon,
/etc/tcc was recreated, but the stored JSON values in `autosave`, `profiles`,
and `settings` appeared to have been reset to the defaults.

```
export class TccPaths {
    static readonly TCCD_EXEC_FILE: string
       = '/opt/tuxedo-control-center/resources/dist/'
       + 'tuxedo-control-center/data/service/tccd';
    static readonly PID_FILE:       string = '/var/run/tccd.pid';
    static readonly SETTINGS_FILE:  string = '/etc/tcc/settings';
    static readonly PROFILES_FILE:  string = '/etc/tcc/profiles';
    static readonly AUTOSAVE_FILE:  string = '/etc/tcc/autosave';
    static readonly FANTABLES_FILE: string = '/etc/tcc/fantables';
    static readonly TCCD_LOG_FILE:  string = '/var/log/tccd/log';
}
```

IMPORTANT: The `TCCD_EXE_FILE` path should probably be updated to
`INSTALL_DIR/service` or similar.  However, a survey of files after the build
indicates that this is only used to create the `tccd.service` file, which we
already modify for deployment to use `INSTALL_DIR/service` above.

##### C.2.2.2. Set the fan curve
I found I could adjust the fan curves by modifying
`src/common/models/TccFanTables.ts` using the 'Balanced' profile. The build
process from section 2.2 had to be repeated.

##### C.2.2.3. Disable CPU control
I modified `src/common/models/TccSettings.ts` as below to disable CPU control.

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

##### C.2.2.4. Disable brightness and webcam control
I modified `src/common/models/TccProfile.ts` for this purpose.
The modified parameters were `webcam.useStatus: false` and `display.
useBrightness:false`.

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

See `bin/deployctl` to see much of this in action. More detail is included below
as part of configuration consideration.

#### C.2.3. Add systemd service files
Copy the service files:

  ```
  sudo cp "${_baseDir}/src/dist-data/tccd.service" \
    "${_baseDir}/src/dist-data/tccd-sleep.service" \
    /etc/systemd/system/ || _abortFn;
  ```

The main service file is `tccd.service`. It specified the `tccd` executable, 
which is a wrapped nodejs app created by a npm app called `pkg`.

  ```
  [Unit]
  Description=TUXEDO Control Center Service

  [Service]
  Type=simple
  ExecStart=INSTALL_DIR/service/tccd --start
  ExecStop=INSTALL_DIR/service/tccd --stop

  [Install]
  WantedBy=multi-user.target
  ```

  We change the path of the executable AND the shared library file which is next
  to it.

  The `tccd-sleep.service` file provides suspend and resume rules:

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

#### C.2.4. Add dbus config file
Copy the config file to the share directory.

  ```
  sudo cp "${_baseDir}/src/dist-data/com.tuxedocomputers.tccd.conf" \
    /usr/share/dbus-1/system.d/
  ```

The content of the file is shown below.

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

#### C.2.5. Start the daemon
This is done with systemctl (systemd).

  ```
  sudo systemctl enable tccd tccd-sleep
  sudo systemctl start tccd
  sudo systemctl daemon-reload
  ```

#### C.2.6. Stop the daemon
  ```
  sudo systemctl stop tccd               || _abortFn;
  sudo systemctl disable tccd tccd-sleep || _abortFn;
  ```

#### C.2.7. Remove daemon
  ```
  sudo rm /etc/systemd/system/tccd*
  sudo rm /usr/share/dbus-1/system.d/com.tuxedocomputers.tccd.conf
  ```

### C.3 Installation files

This is a summary of file that must be installed for the minimal `tccd` daemon
to run. The `bin/deployctl` script creates an `INSTALL_DIR` as shown below.

  ```
  INSTALL_DIR
  ├── dist-data
  │   ├── com.tuxedocomputers.tccd.conf
  │   ├── de.tuxedocomputers.tcc.policy
  │   ├── hardwarelist.json
  │   ├── tccd.service
  │   └── tccd-sleep.service
  └── service
      ├── tccd
      └── TuxedoIOAPI.node
  ```

Files are also installed for systemd:

  ```
  /etc/systemd/system
  ├── tccd.service
  └── tccd-sleep.service
  ```

And then a configuration file for dbus, which is likely unused here:

  ```
  /usr/share/dbus-1/system.d/com.tuxedocomputers.tccd.conf
  ```

These files are dynamically generated:

  ```
  /etc/tcc/autosave
  /etc/tcc/fantables
  /etc/tcc/profiles
  /etc/tcc/settings
  /var/log/tccd/log
  /var/run/tccd.pid
  ```

## Appendix D: TUXEDO Control Center README (EXCERPT)

The TUXEDO Control Center (short: TCC) gives TUXEDO laptop users full control over their hardware like CPU cores, fan speed and more. \
To get a more detailed description of features, plans and the ideas behind please check our press release ([english](https://www.tuxedocomputers.com/en/Infos/News/Everything-under-control-with-the-TUXEDO-Control-Center.tuxedo) | [german](https://www.tuxedocomputers.com/de/Infos/News/Alles-unter-Kontrolle-mit-dem-TUXEDO-Control-Center_1.tuxedo)) and info pages ([english](https://www.tuxedocomputers.com/en/TUXEDO-Control-Center.tuxedo#) | [german](https://www.tuxedocomputers.com/de/TUXEDO-Control-Center.tuxedo)).

### D.1. Using it

There are pre-built packages for Ubuntu 16.04/18.04/20.04 as well as 
openSUSE Leap 15.x and Tumbleweed available at our repositories. For details please have a look [over here](https://www.tuxedocomputers.com/en/Infos/Help-and-Support/Instructions/Add-TUXEDO-Computers-software-package-sources.tuxedo#).

Note: TCC depends on the `tuxedo-io` module from the `tuxedo-keyboard` package for some core functionality like fan control.

### D.2. Project structure

```
tuxedo-control-center
|  README.md
|--src
|  |--ng-app            Angular GUI (aka electron renderer)
|  |--e-app             Electron main
|  |--service-app       Daemon part (Node 12)
|  |--common            Common shared sources
|  |  |--classes
|  |  |--models
|  |--dist-data         Data needed for packaging
|--build-src            Source used for building
```

### D.3. Development setup

1. Install git, gcc, g++, make, nodejs, npm and libudev-dev \
   Ex (deb):
   ```
   curl -sL https://deb.nodesource.com/setup_12.x | sudo -E bash -
   sudo apt install -y git gcc g++ make nodejs libudev-dev
   ```
2. Clone & install libraries
    ```
    git clone https://github.com/tuxedocomputers/tuxedo-control-center
    cd tuxedo-control-center
    npm install
    ```
   **Note:** Do ***not*** continue with `npm audit fix`. Known to cause various issues.

3. Install service file that points to development build path (or use installed service from packaged version)

   Manual instructions:
   1. Copy `tccd.service` and `tccd-sleep.service` (from src/dist-data) to `/etc/systemd/system/`
   2. Edit the `tccd.service` (exec start/stop) to point to `<dev path>/dist/tuxedo-control-center/data/service/tccd`.
   3. Copy `com.tuxedocomputers.tccd.conf` to `/usr/share/dbus-1/system.d/`
   4. Start service `systemctl start tccd`. (And enable for autostart `systemctl enable tccd tccd-sleep`)

### D.4 NPM scripts
`npm run <script-name>`

| Script name           | Description                                                     |
|-----------------------|-----------------------------------------------------------------|
| build                 | Build all apps service/electron/angular                         |
| start                 | Normal start of electron app after build                        |
| start-watch           | Start GUI with automatic reload on changes to angular directory |
| test-common           | Test common files (jasmine)                                     |
| gen-lang              | Generate base for translation (`ng-app/assets/locale/lang.xlf`) |
| pack-prod all/deb/rpm | Build and package for chosen target(s)                          |
| inc-version-patch     | Patch version increase (updates package.json files)             |
| inc-version-minor     | Minor version increase (updates package.json files)             |
| inc-version-major     | Major version increase (updates package.json files)             |


### D.5 Debugging
Debugging of electron main and render process is configured for vscode in .vscode/launch.json

[L0005]:https://github.com/tuxedocomputers/tuxedo-control-center
[L0010]:https://www.npmjs.com/package/electron-context-menu
[L0020]:https://stackoverflow.com/questions/59019091/electron-run-in-production-mode
[L0030]:https://stackoverflow.com/questions/42695555
