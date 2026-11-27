# Phase 2: Ubuntu 24.04 VM (osboxes.org)

## Goal
Have an Ubuntu 24.04 VM running in VirtualBox, with custom credentials, an up-to-date system, SSH access and a clean base snapshot, ready to install MISP 2.5.

## Environment
- Host: Windows 11 with VirtualBox 7.2.20 (see Phase 1)
- VM: `misp-lab`, 8 GB RAM, 4 vCPU
- Image: Ubuntu 24.04 from osboxes.org (VDI format for VirtualBox)

## Where each command runs

| Prompt | Where you are | Commands that go there |
|---|---|---|
| `PS C:\...>` | Windows (PowerShell) | `VBoxManage`, `ssh`, `git` |
| `osboxes@misp-lab:~$` | Inside the VM (Ubuntu) | `sudo`, `apt`, `systemctl`, `cake` |

## 1. Download
1. Go to https://www.osboxes.org and download the Ubuntu 24.04 image for VirtualBox (VDI).
2. Extract the `.7z` (with 7-Zip) and move the `.vdi` to your own folder, e.g. `C:\VMs\misp-lab\`.
3. Record the hash of what you downloaded (the site may not publish one):

```powershell
Get-FileHash .\<file>.7z -Algorithm SHA256
```

> The `< >` signs mean "replace with your value". Do not type them.

## 2. Create the VM
With the VM powered off and in PowerShell (adjust paths and file name). It can also be done from the VirtualBox GUI.

```powershell
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
& $vbm createvm --name "misp-lab" --ostype Ubuntu_64 --register
& $vbm modifyvm "misp-lab" --memory 8192 --cpus 4 --vram 128 --graphicscontroller vmsvga --accelerate-3d off --nic1 nat
& $vbm storagectl "misp-lab" --name "SATA" --add sata --controller IntelAhci
& $vbm storageattach "misp-lab" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "C:\VMs\misp-lab\<file>.vdi"
```

- `$vbm` stores the program path in a variable (it only lives while the PowerShell window stays open).
- `&` is the call operator: it runs what the variable contains.
- `--vram 128` and `--accelerate-3d off` prevent screen flicker (see Troubleshooting).

If the VM does not boot, try toggling EFI under *Settings > System*.

## 3. Port forwarding (NAT)
```powershell
& $vbm modifyvm "misp-lab" --natpf1 "ssh,tcp,127.0.0.1,2222,,22"
& $vbm modifyvm "misp-lab" --natpf1 "https,tcp,127.0.0.1,8443,,443"
```

They only listen on `127.0.0.1`: the VM is not exposed to the local network. They are used in later phases (SSH on 2222, MISP web interface on 8443).

## 4. First boot and basic setup
Inside the VM (terminal with `Ctrl + Alt + T`):

```bash
passwd
sudo passwd root
sudo hostnamectl set-hostname misp-lab
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install openssh-server
sudo reboot
```

| Command | What it does |
|---|---|
| `passwd` | Changes the current user's password (the factory one is public) |
| `sudo passwd root` | Changes the `root` password |
| `hostnamectl set-hostname` | Changes the machine name |
| `apt update` | Downloads the list of available packages |
| `apt -y full-upgrade` | Upgrades everything, resolving dependencies (`-y` answers "yes") |
| `apt -y install openssh-server` | Installs the SSH server |
| `reboot` | Reboots to load the new kernel |

> Using a temporary password with only letters and digits while the keyboard is configured avoids lockouts (see Troubleshooting).

## 5. Verification
```bash
lsb_release -a
uname -r
free -h
df -h /
apt list --upgradable 2>/dev/null
```

Results obtained:
- Ubuntu 24.04.5 LTS (noble), kernel 7.0.0-43-generic
- 7.8 GiB of RAM
- 492 GB virtual disk with 457 GB free. It is a dynamic disk: the `.vdi` file grows on Windows as it is used, so check the host's free space.
- A few packages remain pending because of *phasing*: Ubuntu releases some updates in waves and they are not yet due. This is normal and does not affect MISP. It is confirmed by `sudo apt update && sudo apt -y full-upgrade` ("deferred due to phasing").

Optional cleanup:
```bash
sudo apt -y autoremove
```

## 6. SSH from Windows
```powershell
ssh -p 2222 osboxes@127.0.0.1
```
The first time it asks to confirm the server fingerprint (`yes`). From here on, work from PowerShell: no GUI window, and copy and paste works.

Start and stop the VM without a window:
```powershell
& $vbm startvm "misp-lab" --type headless
& $vbm controlvm "misp-lab" acpipowerbutton
```

## 7. Base snapshot
With the system up to date and the VM powered off (`sudo poweroff`):

```powershell
& $vbm snapshot "misp-lab" take "01-base-ubuntu-actualizado"
& $vbm snapshot "misp-lab" list
```

If something fails later, return to this point with:
```powershell
& $vbm snapshot "misp-lab" restore "01-base-ubuntu-actualizado"
```

## Troubleshooting (issues found)

### Screen flicker
Usual cause: low video memory and 3D acceleration on Ubuntu 24.04 over VirtualBox.
1. With the VM powered off:
```powershell
& $vbm modifyvm "misp-lab" --vram 128 --graphicscontroller vmsvga --accelerate-3d off
```
2. Inside the VM, use Xorg instead of Wayland:
```bash
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf
sudo reboot
```
3. Recommended alternative: MISP does not need a desktop. Work only over SSH with the VM in `headless` mode.

### The new password is rejected at login
Cause: the VM keyboard was set to English (US) and the password was created thinking of a Latin American keyboard. Symbols move around:

| Character | Spanish (Latam) | English (US) |
|---|---|---|
| `@` | AltGr + Q | Shift + 2 |
| `"` | Shift + 2 | Shift + ' |
| `-` | key right of `.` | key next to `0` |
| `ñ` | own key | does not exist |

Prevention: set the keyboard under *Settings > Keyboard > Input Sources* or with `sudo dpkg-reconfigure keyboard-configuration`.

### Reset the password from GRUB
1. Reboot the VM and open GRUB (hold `Shift` on BIOS or press `Esc` on EFI).
2. With the first Ubuntu entry highlighted, press `e`.
3. On the line starting with `linux`, replace `ro quiet splash $vt_handoff` with `rw init=/bin/bash`.
4. Press `Ctrl + X` to boot. You get a root console without a password prompt (for that boot only).
5. Run:
```bash
passwd osboxes
passwd root
sync
reboot -f
```

The standard recovery mode also asks for the `root` password, so it does not help if that is the one that was forgotten.

### `Command 'passswd' not found`
Typing error (`passswd` instead of `passwd`). Use `Tab` to autocomplete and `↑` to recall previous commands.

### `error: unexpected argument '-u' found` when running `sudo`
A Linux command was run in PowerShell. Windows 11 ships its own `sudo.exe`, which does not understand `-u www-data`. Check the prompt: it must read `osboxes@misp-lab:~$`.

## Evidence
Screenshots of: image hash, VM settings, `lsb_release -a`, `uname -r`, SSH connection and snapshot list. Crop the screenshots so they do not show local paths or credentials.
