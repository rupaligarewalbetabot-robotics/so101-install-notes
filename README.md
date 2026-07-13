# LeRobot SO-101 Installation Guide (Windows 11 + WSL2)

A tested walkthrough for setting up the Hugging Face LeRobot stack for the SO-101 arm on Windows 11 using WSL2 Ubuntu.

The official [LeRobot installation docs](https://huggingface.co/docs/lerobot/installation) target Linux and macOS. This guide fills in the Windows-specific pieces: WSL2 setup, USB passthrough for the Feetech bus servo adapter, and the small gotchas that break the install if you follow the Linux instructions verbatim.

> **Status:** In progress. Documented steps and errors are all from my actual install. Currently through USB passthrough with both leader and follower arms attached to WSL. Motor setup and calibration coming next.

## Why WSL2 instead of native Windows

LeRobot, the Feetech SDK, and most of the robotics ecosystem assume a Unix-style environment with `/dev/ttyACM0` or `/dev/ttyUSB0` device paths. Running on WSL2 keeps you aligned with the official docs and community examples, and avoids driver-passthrough headaches down the line.

## Prerequisites

- Windows 11 with virtualization enabled in BIOS
- Admin access on your Windows account
- The SO-101 bus servo adapter (Feetech / Waveshare / CH343) connected via USB

## Step 1: Install WSL2 with Ubuntu 24.04

Open PowerShell as Administrator:

```powershell
wsl --install -d Ubuntu-24.04
```

Reboot when prompted. On first launch, Ubuntu asks you to create a Unix username and password (separate from your Windows login).

Verify you're on WSL2:

```powershell
wsl -l -v
```

You want `VERSION 2`. If it shows 1, run `wsl --set-version Ubuntu-24.04 2`.

> **Heads up (ASUS motherboards):** If `wsl --install` hangs or the required reboot loops, Armoury Crate may be stuck. See [WSL install hangs on ASUS motherboards](#wsl-install-hangs-on-asus-motherboards) in Common issues.

## Step 2: Move to your Linux home directory

If your Ubuntu terminal opens in `/mnt/c/WINDOWS/system32` or another Windows path, drop into your Linux home first:

```bash
cd ~
```

Working from the Linux filesystem (`/home/<user>`) is significantly faster for git and pip operations than working under `/mnt/c/...`.

On my install, a fresh Ubuntu terminal already opened in `/home/<user>`, so no fix was needed.

## Step 3: Update Ubuntu and install build basics

Inside the Ubuntu terminal:

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential git curl wget cmake
```

## Step 4: Install ffmpeg

`ffmpeg` is needed because LeRobot uses TorchCodec for video decoding on supported platforms.

```bash
sudo apt install -y ffmpeg
ffmpeg -version
```

> **Heads up:** On my install, this failed with `Unable to locate package ffmpeg`. Fix in [ffmpeg not found by apt](#ffmpeg-not-found-by-apt).

## Step 5: Install Miniforge

The official Miniforge command from the LeRobot docs works as written once you're inside WSL:

```bash
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```

Accept the license, accept the default install path, and say **yes** when it asks to initialize conda.

Close and reopen the terminal. You should see `(base)` at the start of your prompt, and `conda --version` should print a version number.

> **Heads up:** If `(base)` doesn't appear and `conda --version` returns "command not found", the shell init step likely got skipped. Fix in [No (base) prompt after Miniforge install](#no-base-prompt-after-miniforge-install).

## Step 6: Create the LeRobot conda environment

The current LeRobot guide requires Python >=3.12:

```bash
conda create -y -n lerobot python=3.12
conda activate lerobot
```

Your prompt should now start with `(lerobot)`. You'll need to run `conda activate lerobot` every time you open a new shell.

## Step 7: Clone and install LeRobot with Feetech support

```bash
git clone https://github.com/huggingface/lerobot.git
cd lerobot
pip install -e ".[feetech]"
```

The `[feetech]` extra pulls in the Feetech SDK that drives the STS3215 servos on the SO-101 bus.

## Step 8: Install evdev (WSL-specific)

The LeRobot docs explicitly call this out for WSL installs. Without it, teleop scripts that read keyboard or gamepad input will fail with import errors.

```bash
pip install evdev
```

## Step 9: Set up USB passthrough with usbipd-win

WSL2 does not see USB devices by default. You need to bridge the SO-101 bus adapter from Windows into WSL using `usbipd-win`.

### Install usbipd-win

The LeRobot docs suggest installing via `winget`:

```powershell
winget install --interactive --exact dorssel.usbipd-win
```

On my machine, `winget` was not recognized (see [winget command not found](#winget-command-not-found) for context), so I installed the MSI manually:

1. Go to [github.com/dorssel/usbipd-win/releases](https://github.com/dorssel/usbipd-win/releases) and download the latest `usbipd-win_<version>_x64.msi` (for AMD64 / most Windows PCs).
2. Confirm your CPU architecture in PowerShell if unsure:
   ```powershell
   echo $env:PROCESSOR_ARCHITECTURE
   ```
   Expected output for most machines: `AMD64`.
3. Install from **PowerShell as Administrator**:
   ```powershell
   msiexec /i usbipd-win_5.3.0_x64.msi
   ```
   Replace the filename with whatever version you downloaded.

> **Heads up:** Double-clicking the MSI can fail with "This installation package is not supported by this processor type", especially if you downloaded both x64 and arm64 MSIs. See [MSI installation package not supported by processor type](#msi-installation-package-not-supported-by-processor-type).

### Attach the adapters to WSL

Plug in the SO-101 bus adapter(s). Then in **PowerShell as Administrator**:

```powershell
usbipd list
```

Find the BUSID for your Feetech / CH343 adapter (it shows up as a USB Serial Device). On my setup with two arms, both adapters appear as CH343 devices, one per BUSID.

Bind and attach each adapter:

```powershell
usbipd bind --busid <BUSID>
usbipd attach --wsl --busid <BUSID>
```

### Verify in WSL

```bash
lsusb
ls /dev/tty*
```

Each attached adapter shows up as `/dev/ttyACM0`, `/dev/ttyACM1`, and so on. On my setup, I assigned leader to `/dev/ttyACM0` and follower to `/dev/ttyACM1`.

### Notes on usbipd behavior

A few things worth knowing that the official docs skim over:

- `usbipd attach --wsl` does **not** persist across reboots or WSL restarts. You'll re-run it every session after plugging in the adapters.
- `usbipd bind` **does** persist across reboots. You only need to bind each device once.
- `/dev/ttyACM*` numbering depends on the order you attach devices. Attach the leader BUSID first if you want it on `ttyACM0`.
- Two identical CH343 chips have the same VID:PID, so you can't tell leader from follower by USB metadata alone. Use `lerobot-find-port` or track by physical bus path.
- Windows-side commands (`usbipd`, `winget`, `msiexec`) run in **PowerShell as Administrator**, not inside WSL. Running `echo $env:PROCESSOR_ARCHITECTURE` inside WSL bash returns `:PROCESSOR_ARCHITECTURE` because bash doesn't understand PowerShell syntax.

## Common issues

### WSL install hangs on ASUS motherboards

**Symptom**
`wsl --install` hangs, or the required reboot loops without completing. Task Manager may show Armoury Crate stuck.

**Cause**
Armoury Crate installation blocks the WSL install cycle on some ASUS boards.

**Fix**
1. Open Windows Settings → Windows Update → Advanced options → Optional updates
2. Install any pending ASUS system updates
3. Reboot
4. Re-run `wsl --install -d Ubuntu-24.04`

**References**
- [r/ASUS: Armoury Crate installation stuck](https://www.reddit.com/r/ASUS/comments/jqb9ud/solved_armoury_crate_installation_stuck_at/)

---

### ffmpeg not found by apt

**Symptom**
```
E: Unable to locate package ffmpeg
```

**Cause**
The `universe` repository is not enabled by default in some WSL Ubuntu images.

**Fix**
Enable the universe repo, then install:
```bash
sudo add-apt-repository universe -y
sudo apt update
sudo apt install -y ffmpeg
```

---

### No (base) prompt after Miniforge install

**Symptom**
After running the Miniforge installer, opening a new terminal shows no `(base)` at the prompt, and `conda --version` returns `command not found`.

**Cause**
The installer's "update shell profile to init conda" prompt defaults to **no** when you just press Enter. If missed, conda's `init` block never gets added to `~/.bashrc`.

**Fix (soft path, if `~/miniforge3/` exists)**
```bash
~/miniforge3/bin/conda init bash
```
Close the terminal fully and reopen. `(base)` should now appear.

**Fix (clean reinstall, if the above doesn't work)**
```bash
rm -rf ~/miniforge3
wget "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-$(uname)-$(uname -m).sh"
bash Miniforge3-$(uname)-$(uname -m).sh
```
This time, answer `yes` to both the license and the shell init prompts.

---

### winget command not found

**Symptom**
```
winget : The term 'winget' is not recognized as the name of a cmdlet, function, script file, or operable program.
```

**Cause**
The App Installer package (which provides `winget`) is missing or outdated.

**Fix**
Either install App Installer from the Microsoft Store, or skip `winget` and download the usbipd-win MSI directly from [github.com/dorssel/usbipd-win/releases](https://github.com/dorssel/usbipd-win/releases).

---

### MSI installation package not supported by processor type

**Symptom**
Double-clicking the usbipd-win MSI shows:
```
This installation package is not supported by this processor type.
Contact your product vendor.
```

**Cause**
Either the arm64 MSI is being run on an AMD64 (x64) machine, or double-click is opening the wrong file when both architectures are downloaded.

**Fix**
1. Confirm your CPU architecture in PowerShell:
   ```powershell
   echo $env:PROCESSOR_ARCHITECTURE
   ```
   Expected output for most Windows PCs: `AMD64`.
2. List downloaded MSIs to see which ones you have:
   ```powershell
   dir usbipd*
   ```
3. Delete the wrong-arch MSI so it can't be confused for the right one.
4. Install the correct MSI explicitly from **Admin PowerShell**:
   ```powershell
   msiexec /i usbipd-win_5.3.0_x64.msi
   ```

**References**
- [usbipd-win releases](https://github.com/dorssel/usbipd-win/releases)

---

## Work in progress

Upcoming, tracked as I get to them:

- [x] Step 1: Install WSL2 with Ubuntu 24.04
- [x] Step 2: Move to Linux home directory
- [x] Step 3: Update Ubuntu and install build basics
- [x] Step 4: Install ffmpeg
- [x] Step 5: Install Miniforge
- [x] Step 6: Create the LeRobot conda environment
- [x] Step 7: Clone and install LeRobot with Feetech support
- [x] Step 8: Install evdev (WSL-specific)
- [x] Step 9: Set up USB passthrough with usbipd-win
- [ ] Step 10: Add user to the dialout group
- [ ] Step 11: Verify install with `lerobot-find-port`
- [ ] Step 12: Motor setup and calibration

---

If this guide helped or you hit an issue not listed here, open an issue or reach out.
