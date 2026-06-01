# Notepad++ Complete Offline Audit Report

Generated on: 2026-06-01

Scope:
- Runtime code: `PowerEditor/src`
- Packaging and installer: `PowerEditor/installer`
- CI and release workflow: `.github/workflows`, `appveyor.yml`
- Reference/doc trees with many URL literals: `scintilla`, `lexilla`, `README.md`, `BUILD.md`

## 1. Executive Summary

Key conclusion:

1. No direct WinInet / WinHTTP / `URLDownloadToFile` / Winsock client API usage was found in `PowerEditor/src`, `lexilla`, or `scintilla`.
2. The main Notepad++ process does not appear to implement its own HTTP client in this repo.
3. Actual online behavior is mainly introduced by:
   - launching external web URLs via `ShellExecute`
   - launching the external updater `GUP.exe`
   - Plugin Admin delegating install/update work to `GUP.exe`
4. This repo contains many URL strings in docs, comments, tests, translations, and bundled examples. Those are noisy hits, not equal to runtime networking.

Raw URL-hit volume from grep:

- `PowerEditor/src`: about 825 URL-like hits
- `PowerEditor/installer`: about 253 URL-like hits
- `.github` + `appveyor.yml`: about 15 URL-like hits
- `scintilla` + `lexilla` + top-level docs: about 3130 URL-like hits

Important note:

- `GUP.exe` and the real network implementation of the updater are not in this source tree.
- The repo only contains the integration points and packaging references for `GUP.exe`.
- So "complete offline" at product level still requires auditing the shipped updater binary and shipped plugin list artifacts.

## 2. High Priority Runtime Online Entry Points

### P0. Auto-updater via external `GUP.exe`

Evidence:

- `PowerEditor/src/Parameters.h:813`
- `PowerEditor/src/Parameters.h:816`
- `PowerEditor/src/winmain.cpp:380`
- `PowerEditor/src/winmain.cpp:793`
- `PowerEditor/src/winmain.cpp:866`
- `PowerEditor/src/Parameters.cpp:8907`
- `PowerEditor/src/resource.h:33`
- `PowerEditor/src/resource.h:34`

What it does:

- Auto-update mode defaults to `autoupdate_on_startup`.
- On startup or exit, Notepad++ may launch `GUP.exe` if:
  - updater exists
  - OS is newer than XP
  - signature check passes
  - auto-update is enabled
- Updater parameters include:
  - `INFO_URL = https://notepad-plus-plus.org/update/getDownloadUrl.php`
  - `FORCED_DOWNLOAD_DOMAIN = https://github.com/notepad-plus-plus/notepad-plus-plus/`
- Certificate checks are configured before updater fetch/download.

Offline impact:

- This is the most important runtime online path.
- Even if main EXE does not implement HTTP itself, it explicitly launches an external network-capable updater.

Offline action:

1. Remove updater packaging and shipping.
2. Force auto-update mode to disabled.
3. Remove update menu entries and updater proxy configuration entry.
4. Remove updater launch path from startup and exit flow.

### P0. Plugin Admin install/update/remove path

Evidence:

- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:260`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:287`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:362`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:368`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:371`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:658`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:691`

What it does:

- Plugin Admin depends on both:
  - plugin list artifact: `nppPluginList.dll` or debug `nppPluginList.json`
  - updater binary: `GUP.exe`
- Install/update operations prepare parameters that include plugin folder, repository, and ID, then exit Notepad++ and hand work to `GUP.exe`.
- Plugin metadata includes repository and homepage fields.

Offline impact:

- Plugin installation and update path is not offline-safe.
- Even if the plugin list is local, actual plugin acquisition/update is delegated to the updater path.

Offline action:

1. Remove Plugin Admin menu and dialog from offline build.
2. Stop packaging plugin-list artifacts for offline edition.
3. Remove dependency on `GUP.exe` for plugin operations.

## 3. Medium Priority Runtime Online Entry Points

### P1. Manual "Search on Internet"

Evidence:

- `PowerEditor/src/NppCommands.cpp:820`
- `PowerEditor/src/NppCommands.cpp:838`
- `PowerEditor/src/NppCommands.cpp:843`
- `PowerEditor/src/NppCommands.cpp:847`
- `PowerEditor/src/NppCommands.cpp:851`
- `PowerEditor/src/NppCommands.cpp:855`
- `PowerEditor/src/Parameters.h:857`

What it does:

- Menu command `Search on Internet` builds a URL from selected text.
- Default search engine is Google.
- Other built-in choices include DuckDuckGo, Yahoo, and Stack Overflow.
- Custom search engine URL is allowed.

Offline impact:

- User-triggered online browsing entry point.

Offline action:

1. Remove `IDM_EDIT_SEARCHONINTERNET`.
2. Remove search-engine preference UI and config handling if strict offline behavior is required.

### P1. Clickable URLs detected inside editor text

Evidence:

- `PowerEditor/src/NppNotification.cpp:392`
- `PowerEditor/src/Parameters.h:782`
- `PowerEditor/src/Parameters.h:783`
- `PowerEditor/src/Parameters.h:699`
- `PowerEditor/src/ScintillaComponent/Buffer.cpp:784`

What it does:

- Double-clicking detected URLs triggers `ShellExecute(..., "open", url, ...)`.
- Supported schemes are broader than HTTP, including `ssh://`, `sftp://`, `slack://`, `steam://`, `spotify:`, etc.
- Large-file restriction has `_allowClickableLink = false`, but that only applies when large-file restriction mode is active.
- Normal files can still open external URLs/app handlers.

Offline impact:

- This does not make Notepad++ itself a network client.
- But it remains a direct launch point into browser or other online-capable handlers.

Offline action:

1. Disable clickable-link hotspot behavior.
2. Optionally clear or narrow `_uriSchemes`.
3. Remove URL-open handler from notification path.

### P1. About / Help / Website menu items

Evidence:

- `PowerEditor/src/NppCommands.cpp:3746`
- `PowerEditor/src/NppCommands.cpp:3751`
- `PowerEditor/src/NppCommands.cpp:3757`
- `PowerEditor/src/NppCommands.cpp:3769`
- `PowerEditor/src/NppCommands.cpp:3775`
- `PowerEditor/src/Notepad_plus.rc:1350`
- `PowerEditor/src/Notepad_plus.rc:1351`
- `PowerEditor/src/Notepad_plus.rc:1352`
- `PowerEditor/src/Notepad_plus.rc:1353`
- `PowerEditor/src/Notepad_plus.rc:1355`
- `PowerEditor/src/Notepad_plus.rc:1356`

What it does:

- Opens:
  - Notepad++ home
  - project GitHub page
  - online user manual
  - community forum
  - update entry
  - updater proxy configuration

Offline impact:

- User-triggered online entry points from main menu.

Offline action:

1. Remove online help and website items from offline build.
2. Remove update/proxy items entirely, not only hide them when updater is absent.

## 4. Lower Priority Runtime Online Entry Points

### P2. User Defined Language online help

Evidence:

- `PowerEditor/src/ScintillaComponent/UserDefineDialog.cpp:181`

What it does:

- UDL dialog contains a hardcoded online manual link.

Offline action:

- Replace with local help or remove link.

### P2. Plugin Admin repository link

Evidence:

- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:228`

What it does:

- Opens plugin-list repository page on GitHub.

Offline action:

- Remove link or replace with local metadata path.

### P2. Default user commands that target the internet

Evidence:

- `PowerEditor/src/MISC/Common/NppConstants.h:444`
- `PowerEditor/src/MISC/Common/NppConstants.h:445`
- `PowerEditor/src/MISC/Common/NppConstants.h:448`
- `PowerEditor/src/MISC/Common/NppConstants.h:449`

What it does:

- Default user-defined command template contains:
  - PHP help on the internet
  - Wikipedia search

Offline action:

- Remove these defaults from offline edition templates.

## 5. Packaging and Installer Online Content

### P1. Updater is explicitly packaged

Evidence:

- `PowerEditor/installer/msi/wingup.wxs:4`
- `PowerEditor/installer/msi/wingup.wxs:8`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:69`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:79`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:85`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:91`
- `PowerEditor/installer/packageAll.bat:343`
- `PowerEditor/installer/packageAll.bat:359`
- `PowerEditor/installer/packageAll.bat:375`

What it does:

- MSI and NSIS both package the updater payload.
- Portable packaging also copies updater files.

Offline action:

- Remove updater components from MSI, NSIS, and portable packaging definitions.

### P1. Installer opens website on unsupported platform cases

Evidence:

- `PowerEditor/installer/nsisInclude/tools.nsh:161`
- `PowerEditor/installer/nsisInclude/tools.nsh:169`
- `PowerEditor/installer/nsisInclude/tools.nsh:181`

What it does:

- Installer can open Notepad++ download pages when OS / architecture is unsupported.

Offline action:

- Replace with offline message only.
- Do not launch browser from installer.

### P2. Auto-update can be disabled by config artifact, but updater is still shipped

Evidence:

- `PowerEditor/installer/xml4Config/disableNppAutoUpdate.xml:2`
- `PowerEditor/installer/xml4Config/disableNppAutoUpdate.xml:3`

Meaning:

- Current packaging already recognizes a "disable auto update" switch.
- This is not enough for full offlineization because updater and related UI still exist.

## 6. Build / CI / Release Pipeline Online Dependencies

These are not end-user runtime networking, but they matter if you want a fully offline build/release pipeline.

### P1. CI hits PyPI and installs Python packages

Evidence:

- `.github/workflows/CI_build.yml:89`
- `.github/workflows/CI_build.yml:90`
- `.github/workflows/CI_build.yml:91`
- `.github/workflows/CI_build.yml:92`
- `.github/workflows/CI_build.yml:112`
- `.github/workflows/CI_build.yml:130`

What it does:

- Queries PyPI for version metadata.
- Installs `requests`, `rfc3987`, `pywin32`, `lxml` via `pip`.

Offline action:

1. Vendor these Python dependencies or mirror them internally.
2. Remove live version probing from workflow.

### P1. Release notifier calls GitHub API

Evidence:

- `.github/workflows/release-notifier.yml:55`
- `.github/workflows/release-notifier.yml:56`
- `.github/workflows/release-notifier.yml:61`

What it does:

- Uses `curl` to call GitHub workflow dispatch API in external repos.

Offline action:

- Remove for offline pipeline, or replace with internal eventing.

### P2. Packaging step uses external timestamp service

Evidence:

- `PowerEditor/installer/packageAll.bat:26`

What it does:

- Signing command uses `http://timestamp.globalsign.com/...`

Offline action:

- Replace with internal timestamping or offline-signing policy.

## 7. What Looks Online But Is Mostly Noise

The following areas contain many URL strings, but they are mostly not runtime networking:

1. `scintilla/doc`
   - many documentation pages and external links
2. `lexilla/*`
   - comments, syntax references, lexer examples
3. `PowerEditor/installer/nativeLang/*`
   - translation text and example search-engine strings
4. `PowerEditor/Test/*`
   - URL detection tests
5. `README.md`, `BUILD.md`, `.github/ISSUE_TEMPLATE*`
   - documentation and GitHub metadata
6. vendored headers like `json.hpp`
   - source comments with external references

Recommendation:

- Do not treat raw URL count as runtime online count.
- For offline product work, prioritize Sections 2 to 6.

## 8. Confirmed Negative Findings

The following direct-network patterns were not found in the scanned codebase areas:

- `InternetOpen`
- `WinHttp`
- `URLDownloadToFile`
- `WSAStartup`

Interpretation:

- Main source tree does not appear to perform raw HTTP/socket operations directly.
- Online behavior is primarily browser-launch based or delegated to external shipped components.

## 9. Recommended Offlineization Order

### Stage 1. Stop real update and plugin networking

1. Remove `GUP.exe` integration and packaging.
2. Disable auto-update by code, not only by config file.
3. Remove Plugin Admin from offline build.

### Stage 2. Remove browser-launch entry points

1. Remove website/help/forum/project menu items.
2. Remove `Search on Internet`.
3. Remove UDL online help and Plugin Admin repo link.
4. Remove default internet-oriented user commands.

### Stage 3. Remove passive URL launch behavior

1. Disable clickable URL opening from editor text.
2. Optionally reduce supported URI schemes.

### Stage 4. Clean installer and pipeline

1. Remove installer browser redirects.
2. Remove updater payloads from MSI/NSIS/portable.
3. Replace CI and release pipeline internet dependencies if full offline build is required.

## 10. Gap Remaining After This Audit

This audit is source-tree based. The following still need separate inspection if "complete offline" means shipped binary must have zero online capability:

1. Built `GUP.exe`
2. Built `nppPluginList.dll`
3. Any shipped plugin binaries outside this source tree
4. Final installer binaries and portable package contents

If needed, next step should be:

1. produce a patch list for removing all P0/P1 runtime online paths
2. generate a second report mapping each menu/dialog/config item to the exact source edits needed
