# Phase 3: MISP 2.5 installation with the official script

## Goal
Install MISP 2.5 on the Ubuntu 24.04 VM using the project's official installer and verify that it is operational.

## Environment
- VM `misp-lab`: Ubuntu 24.04, 8 GB RAM, 4 vCPU (see Phase 2)
- SSH access: `ssh -p 2222 osboxes@127.0.0.1`
- Web access: `https://127.0.0.1:8443` (NAT port forwarding to the VM's port 443)
- Starting point: snapshot `01-base-ubuntu-actualizado`

## 1. Prerequisites
Connected to the VM over SSH:

```bash
sudo apt -y install curl tmux
curl -sI https://raw.githubusercontent.com | head -n 1
tmux new -s misp
```

| Command | What it does |
|---|---|
| `apt -y install curl tmux` | Installs `curl` (URL transfers) and `tmux` (persistent sessions) |
| `curl -sI <URL>` | `-s` silent, `-I` HTTP headers only; confirms internet and DNS |
| `head -n 1` | Shows only the first line (the status code, e.g. `200`) |
| `tmux new -s misp` | Creates a session named `misp`; the process survives an SSH drop |

To leave `tmux` without closing it: `Ctrl + b`, then `d`. To return: `tmux attach -t misp`.

> `curl` is not included in the osboxes image. Install it with `apt`, not `snap`: the snap version runs confined (*sandbox*), mixes package managers and updates on its own schedule.

## 2. Download and review the script
```bash
wget --no-cache -O /tmp/INSTALL.ubuntu2404.sh https://raw.githubusercontent.com/MISP/MISP/refs/heads/2.5/INSTALL/INSTALL.ubuntu2404.sh
ls -lh /tmp/INSTALL.ubuntu2404.sh
head -n 5 /tmp/INSTALL.ubuntu2404.sh
less /tmp/INSTALL.ubuntu2404.sh
```

- `wget --no-cache -O <path>`: downloads a fresh copy and saves it at the given path.
- `ls -lh`: confirms the file exists and is not empty (about 32 KB).
- `head -n 5`: must start with `#!/bin/bash` and comments about MISP 2.5 on Ubuntu 24.04.
- `less`: lets you read the script before running it as root (quit with `q`).

## 3. Run the installation
```bash
sudo bash /tmp/INSTALL.ubuntu2404.sh
```

It installs Apache, MariaDB, Redis, PHP 8.3, Composer, PECL extensions, the Python virtual environment and the workers, and loads the JSON structures (taxonomies, galaxies, objects). It takes 15 to 40 minutes depending on the connection.

- Log: `/var/log/misp_install.log`
- Initial credentials: `/root/misp_settings.txt` (mode 600, root only). **Never publish this file or screenshots showing it.**

To check progress from another session:
```bash
pgrep -af INSTALL.ubuntu2404
sudo tail -n 20 /var/log/misp_install.log
```

The process is done when the prompt returns and the script prints `MISP setup complete`.

## 4. First login and settings
1. Open `https://127.0.0.1:8443` and accept the self-signed certificate (it is a lab).
2. Log in with the credentials from the file above and change the admin password.
3. MISP password policy: at least 12 characters with upper case, lower case and a digit or symbol, or 16 characters or more. A long passphrase meets both.
4. Set the base URL, since the script uses `https://misp.local` by default:

```bash
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting MISP.baseurl "https://127.0.0.1:8443"
sudo -u www-data /var/www/MISP/app/Console/cake Admin getSetting MISP.baseurl
```

- `sudo -u www-data`: runs as the Apache user, the owner of MISP's files.
- `cake Admin setSetting / getSetting`: writes or reads a setting from MISP's console.
- The response must show `"value": "https://127.0.0.1:8443"` and `"errorMessage": null`.

## 5. Verification
```bash
systemctl is-active apache2 mariadb redis-server
```
Three `active` lines are expected.

In the web UI, *Administration > Server Settings & Maintenance*:
- **Diagnostics:** MISP v2.5.51 installed and equal to the latest available version, status OK, all submodules OK (taxonomies, galaxies, objects, warninglists, PyMISP).
- **Workers:** all six types (`cache`, `default`, `email`, `prio`, `update`, `scheduler`) with healthy processes and 0 jobs in the queue.
- Warnings on the *Plugin* and *AI* tabs are common on a fresh install (optional services not configured) and do not block usage.

## 6. Snapshot
```bash
sudo poweroff
```
```powershell
& $vbm snapshot "misp-lab" take "02-misp-instalado"
& $vbm snapshot "misp-lab" list
```

Resulting tree:
```
01-base-ubuntu-actualizado
  └─ 02-misp-instalado  (current state)
```

## Troubleshooting (issues found)

### `curl: command not found`
The minimal image does not include it. Install it with `sudo apt -y install curl`.

### `syntax error near unexpected token 'newline'`
The `<URL>` placeholder was typed literally. The `< >` signs in a guide mean "replace with your value"; also, `<` is a redirection operator in bash.

### `Command not found` or `unexpected argument '-u'` when running `sudo`
The Linux command was run in PowerShell. Check the prompt: inside the VM it must be `osboxes@misp-lab:~$`. Windows 11 ships its own `sudo.exe`, which does not understand the same options.

### MISP rejects the password
It did not meet the required complexity. Use a passphrase of 16 characters or more, for example four words separated by hyphens plus a number and a symbol. Avoid `ñ` and accents.

### Host suspend during the installation
When Windows suspends, VirtualBox freezes the VM and in-flight downloads may time out. Disable suspend while installing (`powercfg /change standby-timeout-ac 0` in an administrator PowerShell) and restore the original value afterwards. If the script fails midway, restore the `01-base-ubuntu-actualizado` snapshot and run it again.

### Updates deferred by *phasing*
`apt list --upgradable` may list a few packages that `full-upgrade` does not install. This is normal and does not affect MISP.

## Evidence
Screenshots of: the script's final summary (**without credentials**), `getSetting MISP.baseurl`, *Diagnostics*, *Workers* and the snapshot list. Crop the images so they do not show local paths, API keys or passwords.
