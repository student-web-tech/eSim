# eSim 2.5 Installation on Ubuntu 25+ : Issue Report

## Environment
- Host: Windows with WSL2 (kernel 6.6.87.2-microsoft-standard-WSL2)
- Test OS: Ubuntu 26.04.1 LTS (resolute). Ubuntu 25.04 is end-of-life and not offered by `wsl --list --online`, so 26.04 was used as the "Ubuntu 25 and above" target.
- Baseline: Ubuntu 24.04 (WSL) used for cloning and Git setup only.
- Package tested: eSim-2.5 (esim.fossee.in/downloads) plus the `installers` branch of FOSSEE/eSim.
- Method: every run logged with `./install-eSim.sh --install 2>&1 | tee <log>`.

## Summary

| # | Issue | Location | Status |
|---|-------|----------|--------|
| 1 | Syntax error, installer cannot run on any Ubuntu version | Ubuntu/install-eSim.sh | Fixed (d6431fe) |
| 2 | 26.04 not supported by the version dispatcher | Ubuntu/install-eSim.sh | Fixed (4c2cdbe) |
| 3 | KiCad step fails: PPA 404 and pinned version | install-eSim-26.04.sh | Fixed (4c2cdbe) |
| 4 | Bare `installers` branch lacks library/, nghdl.zip etc. | Setup | Documented (run from full package) |
| 5 | Package has no 25.04/26.04 script; NGHDL dispatcher rejects 26.04 | nghdl/install-nghdl.sh | Workaround applied |
| 6 | `libcanberra-gtk-module` removed from Ubuntu 26.04 | install-nghdl-24.04.sh line 68 | Fixed (local patch) |

## Issue 1: Syntax error in install-eSim.sh
- Symptom: `bash -n install-eSim.sh` gives `line 70: syntax error near unexpected token '}'`.
- Evidence: the file on GitHub is identical to the downloaded copy (`diff` printed SAME), so the bug is in the repo.
- Cause: `run_version_script` loses its closing `else`/`fi`/`}` after `bash "$SCRIPT" "$ARGUMENT"`. Orphaned lines from an old config-writing function (`echo "[eSim]" >> $config_dir/$config_file ...`) were left in their place.
- Fix: deleted the orphaned lines 62-69 and added `else ... exit 1; fi`.
- Result: `bash -n` passes and the script runs.

## Issue 2: Ubuntu 26.04 unsupported
- Symptom: `Unsupported Ubuntu version: 26.04 (26.04.1)`.
- Cause: the `case $VERSION_ID` block only lists 22.04, 23.04, 24.04 and 25.04.
- Fix: added a `"26.04")` case pointing to a new `install-eSim-26.04.sh`.

## Issue 3: KiCad installation fails
- Symptom: `404 Not Found ... ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu resolute Release`, then "Aborting Installation".
- Cause: any version other than 24.04/25.04 falls into the `else` branch, which selects the KiCad 6.0 PPA. That PPA has no packages for `resolute`. The script also pins `kicad=8.0.8+dfsg-1`, which Ubuntu 26.04 does not offer.
- Fix: on 26.04 skip the PPA and install `kicad kicad-footprints kicad-libraries kicad-symbols kicad-templates` from the default repository (KiCad 9.0.8). I removed the broken PPA entry beforehand.
- Result: KiCad 9.0.8 installed; later runs report it already installed.

## Issue 4: Running from the bare installers branch
- Symptom: `tar: library/kicadLibrary.tar.xz: Cannot open: No such file or directory`.
- Cause: the `installers` branch only holds installer scripts. The installer needs `library/`, `images/`, `nghdl.zip` and `src/` from the full eSim 2.5 package.
- Resolution: copy the fixed scripts into the extracted eSim-2.5 package and run from there. Not a script bug, but worth documenting.

## Issue 5: No 25.04/26.04 script in the 2.5 package; NGHDL rejects 26.04
- The eSim-2.5 package only ships `install-eSim-22.04.sh`, `23.04.sh` and `24.04.sh`.
- `nghdl.zip` has its own dispatcher (`install-nghdl.sh`) with the same limitation and prints `Unsupported Ubuntu version: 26.04`.
- Workaround: changed line 46 to `"24.04"|"26.04")` so 26.04 reuses the 24.04 NGHDL script, and repacked `nghdl.zip`.

## Issue 6: libcanberra-gtk-module missing
- Symptom: `Package 'libcanberra-gtk-module' has no installation candidate`.
- Cause: line 68 of `install-nghdl-24.04.sh` installs `libcanberra-gtk-module libcanberra-gtk3-module`. The GTK2 package no longer exists in Ubuntu 26.04.
- Fix: install only `libcanberra-gtk3-module` (candidate 0.30-18ubuntu3).

## Other observations (not yet verified on a real run)
- `FULL_VERSION` is read with `lsb_release -d | grep -oP '\d+\.\d+\.\d+'`, which is empty on releases without a point number.
- The 25.04 script still uses a KiCad PPA that may not exist for newer releases.
- `install-eSim-26.04.sh` still contains the old pinned KiCad line; it is unreachable on 26.04 but should be cleaned up.

## Current status of the install
## Issue 7: NGHDL/GHDL build fails on LLVM 21
- Symptom: `Unhandled version llvm 21.1.8` from GHDL 4.1.0 configure.
- Cause: Ubuntu 26.04 installs LLVM 21; the bundled GHDL 4.1.0 does not recognise it.
- Status: NGHDL disabled with a TODO comment (`installNghdl` call commented out in install-eSim-26.04.sh). Possible fix (untested): install llvm-18 (available on 26.04) or upgrade GHDL.

## Issue 8: Desktop shortcut copy fails
- Symptom: `cp: cannot create regular file '/home/sound/Desktop/': Not a directory`.
- Cause: WSL has no ~/Desktop. The `esim` command and the .desktop entry were already installed.
- Possible fix: `mkdir -p ~/Desktop` or check that it exists before copying. Environment quirk, not a core bug.

## Issue 9 (minor): Welcome page not found
- Symptom: `QTextBrowser: No document for file:///.../src/browser/library/browser/welcome.html`. The Welcome pane is blank.
- Cause: wrong relative path to welcome.html. Cosmetic.

## Final result
The main eSim 2.5 GUI installs and launches on Ubuntu 26.04.1 (Python 3.14.4, KiCad 9.0.8). NGHDL (optional) is disabled because of Issue 7.
Evidence: Ubuntu/screenshots/ (splash screen and main window). Patch files for the NGHDL changes: Ubuntu/patches/.

## Logs
install_attempt1.txt to install_attempt5.txt, issue2_unsupported_version.txt# eSim 2.5 Installation on Ubuntu 25+ : Issue Report

## Environment
- Host: Windows with WSL2 (kernel 6.6.87.2-microsoft-standard-WSL2)
- Test OS: Ubuntu 26.04.1 LTS (resolute). Ubuntu 25.04 is end-of-life and not offered by `wsl --list --online`, so 26.04 was used as the "Ubuntu 25 and above" target.
- Baseline: Ubuntu 24.04 (WSL) used for cloning and Git setup only.
- Package tested: eSim-2.5 (esim.fossee.in/downloads) plus the `installers` branch of FOSSEE/eSim.
- Method: every run logged with `./install-eSim.sh --install 2>&1 | tee <log>`.

## Summary

| # | Issue | Location | Status |
|---|-------|----------|--------|
| 1 | Syntax error, installer cannot run on any Ubuntu version | Ubuntu/install-eSim.sh | Fixed (d6431fe) |
| 2 | 26.04 not supported by the version dispatcher | Ubuntu/install-eSim.sh | Fixed (4c2cdbe) |
| 3 | KiCad step fails: PPA 404 and pinned version | install-eSim-26.04.sh | Fixed (4c2cdbe) |
| 4 | Bare `installers` branch lacks library/, nghdl.zip etc. | Setup | Documented (run from full package) |
| 5 | Package has no 25.04/26.04 script; NGHDL dispatcher rejects 26.04 | nghdl/install-nghdl.sh | Workaround applied |
| 6 | `libcanberra-gtk-module` removed from Ubuntu 26.04 | install-nghdl-24.04.sh line 68 | Fixed (local patch) |

## Issue 1: Syntax error in install-eSim.sh
- Symptom: `bash -n install-eSim.sh` gives `line 70: syntax error near unexpected token '}'`.
- Evidence: the file on GitHub is identical to the downloaded copy (`diff` printed SAME), so the bug is in the repo.
- Cause: `run_version_script` loses its closing `else`/`fi`/`}` after `bash "$SCRIPT" "$ARGUMENT"`. Orphaned lines from an old config-writing function (`echo "[eSim]" >> $config_dir/$config_file ...`) were left in their place.
- Fix: deleted the orphaned lines 62-69 and added `else ... exit 1; fi`.
- Result: `bash -n` passes and the script runs.

## Issue 2: Ubuntu 26.04 unsupported
- Symptom: `Unsupported Ubuntu version: 26.04 (26.04.1)`.
- Cause: the `case $VERSION_ID` block only lists 22.04, 23.04, 24.04 and 25.04.
- Fix: added a `"26.04")` case pointing to a new `install-eSim-26.04.sh`.

## Issue 3: KiCad installation fails
- Symptom: `404 Not Found ... ppa.launchpadcontent.net/kicad/kicad-6.0-releases/ubuntu resolute Release`, then "Aborting Installation".
- Cause: any version other than 24.04/25.04 falls into the `else` branch, which selects the KiCad 6.0 PPA. That PPA has no packages for `resolute`. The script also pins `kicad=8.0.8+dfsg-1`, which Ubuntu 26.04 does not offer.
- Fix: on 26.04 skip the PPA and install `kicad kicad-footprints kicad-libraries kicad-symbols kicad-templates` from the default repository (KiCad 9.0.8). I removed the broken PPA entry beforehand.
- Result: KiCad 9.0.8 installed; later runs report it already installed.

## Issue 4: Running from the bare installers branch
- Symptom: `tar: library/kicadLibrary.tar.xz: Cannot open: No such file or directory`.
- Cause: the `installers` branch only holds installer scripts. The installer needs `library/`, `images/`, `nghdl.zip` and `src/` from the full eSim 2.5 package.
- Resolution: copy the fixed scripts into the extracted eSim-2.5 package and run from there. Not a script bug, but worth documenting.

## Issue 5: No 25.04/26.04 script in the 2.5 package; NGHDL rejects 26.04
- The eSim-2.5 package only ships `install-eSim-22.04.sh`, `23.04.sh` and `24.04.sh`.
- `nghdl.zip` has its own dispatcher (`install-nghdl.sh`) with the same limitation and prints `Unsupported Ubuntu version: 26.04`.
- Workaround: changed line 46 to `"24.04"|"26.04")` so 26.04 reuses the 24.04 NGHDL script, and repacked `nghdl.zip`.

## Issue 6: libcanberra-gtk-module missing
- Symptom: `Package 'libcanberra-gtk-module' has no installation candidate`.
- Cause: line 68 of `install-nghdl-24.04.sh` installs `libcanberra-gtk-module libcanberra-gtk3-module`. The GTK2 package no longer exists in Ubuntu 26.04.
- Fix: install only `libcanberra-gtk3-module` (candidate 0.30-18ubuntu3).

## Other observations (not yet verified on a real run)
- `FULL_VERSION` is read with `lsb_release -d | grep -oP '\d+\.\d+\.\d+'`, which is empty on releases without a point number.
- The 25.04 script still uses a KiCad PPA that may not exist for newer releases.
- `install-eSim-26.04.sh` still contains the old pinned KiCad line; it is unreachable on 26.04 but should be cleaned up.

## Current status of the install
## Issue 7: NGHDL/GHDL build fails on LLVM 21
- Symptom: `Unhandled version llvm 21.1.8` from GHDL 4.1.0 configure.
- Cause: Ubuntu 26.04 installs LLVM 21; the bundled GHDL 4.1.0 does not recognise it.
- Status: NGHDL disabled with a TODO comment (`installNghdl` call commented out in install-eSim-26.04.sh). Possible fix (untested): install llvm-18 (available on 26.04) or upgrade GHDL.

## Issue 8: Desktop shortcut copy fails
- Symptom: `cp: cannot create regular file '/home/sound/Desktop/': Not a directory`.
- Cause: WSL has no ~/Desktop. The `esim` command and the .desktop entry were already installed.
- Possible fix: `mkdir -p ~/Desktop` or check that it exists before copying. Environment quirk, not a core bug.

## Issue 9 (minor): Welcome page not found
- Symptom: `QTextBrowser: No document for file:///.../src/browser/library/browser/welcome.html`. The Welcome pane is blank.
- Cause: wrong relative path to welcome.html. Cosmetic.

## Final result
The main eSim 2.5 GUI installs and launches on Ubuntu 26.04.1 (Python 3.14.4, KiCad 9.0.8). NGHDL (optional) is disabled because of Issue 7.
Evidence: Ubuntu/screenshots/ (splash screen and main window). Patch files for the NGHDL changes: Ubuntu/patches/.

## Logs
install_attempt1.txt to install_attempt5.txt, issue2_unsupported_version.txt
