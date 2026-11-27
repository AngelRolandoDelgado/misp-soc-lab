# Phase 1: Hypervisor (VirtualBox on Windows 11)

## Goal
Have a hypervisor installed from the vendor's official site and verified, to host the Ubuntu 24.04 VM that will run MISP 2.5.

## Environment
- Host: Windows 11
- Hypervisor: Oracle VirtualBox 7.2.20

## 1. Download from the official site
1. Go to https://www.virtualbox.org/wiki/Downloads
2. Download the "Windows hosts" installer.
3. Also download the SHA256SUMS file from the same page.

## 2. Verify installer integrity
In PowerShell, inside the downloads folder:

```powershell
Get-FileHash .\VirtualBox-7.2.20-*-Win.exe -Algorithm SHA256
```

Compare the output with the matching line in SHA256SUMS. They must match exactly.

## 3. Verify the digital signature (existing install)
```powershell
Get-AuthenticodeSignature "C:\Program Files\Oracle\VirtualBox\VirtualBox.exe" | Format-List
```
The status must be `Valid` and the signer Oracle.

## 4. Confirm the installed version
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
```

## 5. Virtualization requirements
- In Task Manager > Performance > CPU, "Virtualization" should show "Enabled".
- If Hyper-V/WSL2 are active, VirtualBox will use the Hyper-V backend (works, with lower performance).

## Evidence
- Screenshot of the official download page
- Screenshot of the hash and signature output
- Screenshot of VirtualBox > Help > About

## Common issues
- **Hash mismatch:** discard the file and download it again.
- **"VT-x not available":** enable virtualization in BIOS/UEFI.