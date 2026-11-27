# Fase 1: Hipervisor (VirtualBox en Windows 11)

## Objetivo
Contar con un hipervisor instalado desde el sitio oficial del fabricante y verificado, para alojar la VM de Ubuntu 24.04 donde correrá MISP 2.5.

## Entorno
- Anfitrión: Windows 11
- Hipervisor: Oracle VirtualBox 7.2.20

## 1. Descarga desde el sitio oficial
1. Ir a https://www.virtualbox.org/wiki/Downloads
2. Descargar el instalador "Windows hosts".
3. Descargar también el archivo SHA256SUMS de la misma página.

## 2. Verificar la integridad del instalador
En PowerShell, dentro de la carpeta de descargas:

```powershell
Get-FileHash .\VirtualBox-7.2.20-*-Win.exe -Algorithm SHA256
```

Comparar el resultado con la línea correspondiente en SHA256SUMS. Deben coincidir exactamente.

## 3. Verificar la firma digital (instalación existente)
```powershell
Get-AuthenticodeSignature "C:\Program Files\Oracle\VirtualBox\VirtualBox.exe" | Format-List
```
El estado debe ser `Valid` y el firmante, Oracle.

## 4. Confirmar la versión instalada
```powershell
& "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe" --version
```

## 5. Requisitos de virtualización
- Verificar en el Administrador de tareas > Rendimiento > CPU que "Virtualización" aparezca como "Habilitado".
- Si hay Hyper-V/WSL2 activos, VirtualBox usará el backend de Hyper-V (funciona, con menor rendimiento).

## Evidencia
- Captura de la página oficial de descarga
- Captura de la salida del hash y de la firma
- Captura de VirtualBox > Ayuda > Acerca de

## Problemas comunes
- **Hash distinto:** descartar el archivo y volver a descargarlo.
- **"VT-x no disponible":** habilitar virtualización en BIOS/UEFI.