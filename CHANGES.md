# Changes Made to Notepad++ (This Fork)

All modifications listed below are applied on top of the official [notepad-plus-plus/notepad-plus-plus](https://github.com/notepad-plus-plus/notepad-plus-plus) repository.

## Overview

This fork removes political content, forced auto-update functionality, and unnecessary website dependencies from the original Notepad++ codebase, restoring users' freedom to use the software without ideological interference.

---

## Detailed Changes

### 1. Remove Political Content from Codebase

**Files modified:**
- `PowerEditor/src/Notepad_plus.rc`
  - Removed active `LTEXT "Tiananmen Massacre Commemoration"` from About dialog
  - Removed commented political icon references:
    - `IDI_SAMESEXMARRIAGE` (same-sex marriage Taiwan icon)
    - `IDI_TAIWANSSOVEREIGNTY` / `IDI_TAIWANSSOVEREIGNTY_DM` (Taiwan sovereignty icons)
    - `IDI_WITHUKRAINE` (Ukraine solidarity icon)
  - Removed commented political subtitle lines:
    - `"Stand with Hong Kong"`
    - `"pour Samuel Paty"`
    - `"Boycott Beijing 2022"`
    - `"Stand Up For Ukraine"`
    - `"FREE UYGHUR"`
    - `"Support Taiwan's Sovereignty"`
    - `"支持台灣獨立"`
    - `"Support Taiwan's return to the UN"`
    - `"We are with Ukraine"`
    - `"The short story"`
  - Restored active Home URL display in About dialog

**Files modified:**
- `PowerEditor/src/WinControls/AboutDlg/AboutDlg.cpp`
  - Removed all commented political propaganda URL links:
    - `free-uyghur-edition/`
    - `stand-with-hong-kong/`
    - `pour-samuel-paty/`
    - `unhappy-users-edition/`
    - `happy-users-edition/`
    - `20thyearanniversary`
    - `about-taiwan/`
    - `we-are-with-ukraine/`
    - `v8964-released/` (latest release political news)
  - Activated `_pageLink.create()` for `IDC_HOME_ADDR` pointing to `https://notepad-plus-plus.org/`

**Commit:** `ab4b36223`, `f64be7abc`

---

### 2. Remove Auto-Update Functionality

**Files deleted:**
- `PowerEditor/installer/msi/wingup.wxs` — WinGUp MSI installer component
- `PowerEditor/installer/xml4Config/disableNppAutoUpdate.xml` — Auto-update disable marker

**Files modified:**

- `PowerEditor/src/Notepad_plus.rc`
  - Removed `"?"` Help menu items: Home, Project Page, Online User Manual, Forum, Update Notepad++, Set Updater Proxy
  - Kept: Command Line Arguments, Debug Info, About Notepad++

- `PowerEditor/src/NppCommands.cpp`
  - Removed `IDM_HOMESWEETHOME` case (open homepage)
  - Removed `IDM_PROJECTPAGE` case (open GitHub project page)
  - Removed `IDM_ONLINEDOCUMENT` case (open user manual)
  - Removed `IDM_FORUM` case (open community forum)
  - Removed `IDM_UPDATE_NPP` / `IDM_CONFUPDATERPROXY` cases (update & proxy config)
  - Removed all associated updater logic (gup.exe launch, signature verification, XP compatibility check)
  - Removed dead `#include "verifySignedfile.h"`

- `PowerEditor/src/winmain.cpp`
  - Removed `launchUpdater()` function entirely
  - Removed startup auto-update check logic
  - Removed update-at-exit logic
  - Removed `#include "Processus.h"` and `#include "verifySignedfile.h"`
  - Forced `_autoUpdateOpt._doAutoUpdate = autoupdate_disabled` at startup

- `PowerEditor/src/Parameters.h`
  - Removed wingup-related fields: `_wingupFullPath`, `_wingupParams`, `_wingupDir`
  - Removed `_isNppAutoUpdateDisabled` flag and its accessor
  - Removed `_doesExistUpdater` flag
  - Removed `buildGupParams()` static method declaration
  - Changed default `AutoUpdateMode` to `autoupdate_disabled`

- `PowerEditor/src/Parameters.cpp`
  - Removed `#include "verifySignedfile.h"`
  - Removed keyboard shortcut entries for removed menu items (Home, Project Page, etc.)
  - Removed `disableNppAutoUpdate.xml` detection logic
  - Removed `buildGupParams()` method implementation

- `PowerEditor/src/NppBigSwitch.cpp`
  - Removed wingup execution code in the big switch handler

- `PowerEditor/src/Notepad_plus.cpp`
  - Adjusted for removed updater menu references

- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp` & `pluginsAdmin.h`
  - Removed wingup updater path initialization
  - Removed plugin update/install via wingup code
  - Simplified update handling

- `PowerEditor/src/WinControls/Preference/preference.rc`
  - Removed `"Auto-updater:"` UI control

- `PowerEditor/src/WinControls/Preference/preferenceDlg.cpp`
  - Removed auto-updater combo box initialization and handling

- `PowerEditor/src/resource.h`
  - Removed `INFO_URL` and `FORCED_DOWNLOAD_DOMAIN` defines

- `PowerEditor/installer/msi/nppInstall.wxs`
  - Removed UpdaterFolder directory and AutoUpdaterFeature

- `PowerEditor/installer/nppSetup.nsi`
  - Removed `/noUpdater` installer switch and all updater-related logic
  - Simplified installer pages

- `PowerEditor/installer/nsisInclude/binariesComponents.nsh`
  - Removed updater binary inclusion

- `PowerEditor/installer/nsisInclude/mainSectionFuncs.nsh`
  - Adjusted for removed updater section

- `PowerEditor/installer/nsisInclude/uninstall.nsh`
  - Removed updater directory cleanup

- `PowerEditor/installer/packageAll.bat`
  - Removed updater packaging steps

**Commit:** `9339613c2`

---

### 3. Added Offline Audit Reports

**Files added:**
- `offline_audit_report.md` — Security audit report (English)
- `offline_audit_report.zh-CN.md` — Security audit report (Chinese)

**Commit:** `9339613c2`

---

### 4. Documentation Updates

**Files modified:**
- `README.md` — Rewritten to describe forked nature, modifications, and bug reporting guidelines

**Files added:**
- `CHANGES.md` — This file, listing all modifications as required by GPL

**Commit:** `0fa6b6c90`

---

## License

All changes in this fork are released under the same [GNU General Public License v3](LICENSE) as the original Notepad++ project.
