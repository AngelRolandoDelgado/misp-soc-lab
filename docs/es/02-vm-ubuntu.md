# Fase 2: VM Ubuntu 24.04 (osboxes.org)

## Objetivo
Tener una VM Ubuntu 24.04 funcionando en VirtualBox, con credenciales propias, sistema actualizado, acceso por SSH y un snapshot base limpio, lista para instalar MISP 2.5.

## Entorno
- Anfitrión: Windows 11 con VirtualBox 7.2.20 (ver Fase 1)
- VM: `misp-lab`, 8 GB de RAM, 4 vCPU
- Imagen: Ubuntu 24.04 de osboxes.org (formato VDI para VirtualBox)

## Cómo distinguir dónde se ejecuta cada comando

| Prompt | Dónde estás | Qué comandos van ahí |
|---|---|---|
| `PS C:\...>` | Windows (PowerShell) | `VBoxManage`, `ssh`, `git` |
| `osboxes@misp-lab:~$` | Dentro de la VM (Ubuntu) | `sudo`, `apt`, `systemctl`, `cake` |

## 1. Descarga
1. Ir a https://www.osboxes.org y descargar la imagen de Ubuntu 24.04 para VirtualBox (VDI).
2. Descomprimir el `.7z` (con 7-Zip) y mover el `.vdi` a una carpeta propia, por ejemplo `C:\VMs\misp-lab\`.
3. Registrar el hash de lo descargado (la página puede no publicar uno):

```powershell
Get-FileHash .\<archivo>.7z -Algorithm SHA256
```

> Los signos `< >` indican "reemplazar por su valor". No se escriben.

## 2. Crear la VM
Con la VM apagada y en PowerShell (ajustar rutas y nombre de archivo). También puede hacerse desde la interfaz gráfica de VirtualBox.

```powershell
$vbm = "C:\Program Files\Oracle\VirtualBox\VBoxManage.exe"
& $vbm createvm --name "misp-lab" --ostype Ubuntu_64 --register
& $vbm modifyvm "misp-lab" --memory 8192 --cpus 4 --vram 128 --graphicscontroller vmsvga --accelerate-3d off --nic1 nat
& $vbm storagectl "misp-lab" --name "SATA" --add sata --controller IntelAhci
& $vbm storageattach "misp-lab" --storagectl "SATA" --port 0 --device 0 --type hdd --medium "C:\VMs\misp-lab\<archivo>.vdi"
```

- `$vbm` guarda la ruta del programa en una variable (solo vive mientras la ventana de PowerShell esté abierta).
- `&` es el operador de llamada: ejecuta lo que contiene la variable.
- `--vram 128` y `--accelerate-3d off` evitan el parpadeo de pantalla (ver Troubleshooting).

Si la VM no arranca, probar activar o desactivar EFI en *Configuración > Sistema*.

## 3. Reenvío de puertos (NAT)
```powershell
& $vbm modifyvm "misp-lab" --natpf1 "ssh,tcp,127.0.0.1,2222,,22"
& $vbm modifyvm "misp-lab" --natpf1 "https,tcp,127.0.0.1,8443,,443"
```

Solo escuchan en `127.0.0.1`: la VM no queda expuesta a la red local. Se usan en las fases siguientes (SSH en el 2222, interfaz web de MISP en el 8443).

## 4. Primer arranque y configuración básica
Dentro de la VM (terminal con `Ctrl + Alt + T`):

```bash
passwd
sudo passwd root
sudo hostnamectl set-hostname misp-lab
sudo apt update && sudo apt -y full-upgrade
sudo apt -y install openssh-server
sudo reboot
```

| Comando | Qué hace |
|---|---|
| `passwd` | Cambia la contraseña del usuario actual (la de fábrica es pública) |
| `sudo passwd root` | Cambia la contraseña de `root` |
| `hostnamectl set-hostname` | Cambia el nombre del equipo |
| `apt update` | Descarga la lista de paquetes disponibles |
| `apt -y full-upgrade` | Actualiza todo, resolviendo dependencias (`-y` responde "sí") |
| `apt -y install openssh-server` | Instala el servidor SSH |
| `reboot` | Reinicia para cargar el kernel nuevo |

> Usar una contraseña temporal solo con letras y números mientras se configura el teclado evita bloqueos (ver Troubleshooting).

## 5. Verificación
```bash
lsb_release -a
uname -r
free -h
df -h /
apt list --upgradable 2>/dev/null
```

Resultado obtenido:
- Ubuntu 24.04.5 LTS (noble), kernel 7.0.0-43-generic
- 7.8 GiB de RAM
- Disco virtual de 492 GB con 457 GB libres. Es un disco dinámico: el archivo `.vdi` crece en Windows a medida que se usa, así que conviene revisar el espacio libre del anfitrión.
- Quedan unos pocos paquetes sin actualizar por *phasing*: Ubuntu libera ciertas actualizaciones por tandas y aún no corresponde instalarlas. Es normal y no afecta a MISP. Se confirma con `sudo apt update && sudo apt -y full-upgrade` (aparece "deferred due to phasing").

Limpieza opcional:
```bash
sudo apt -y autoremove
```

## 6. Conexión por SSH desde Windows
```powershell
ssh -p 2222 osboxes@127.0.0.1
```
La primera vez pide confirmar la huella del servidor (`yes`). A partir de aquí se trabaja desde PowerShell, sin ventana gráfica y con copiar y pegar.

Arrancar y apagar la VM sin ventana:
```powershell
& $vbm startvm "misp-lab" --type headless
& $vbm controlvm "misp-lab" acpipowerbutton
```

## 7. Snapshot base
Con el sistema actualizado y la VM apagada (`sudo poweroff`):

```powershell
& $vbm snapshot "misp-lab" take "01-base-ubuntu-actualizado"
& $vbm snapshot "misp-lab" list
```

Si algo falla más adelante, se vuelve a este punto con:
```powershell
& $vbm snapshot "misp-lab" restore "01-base-ubuntu-actualizado"
```

## Troubleshooting (problemas encontrados)

### Parpadeo de la pantalla
Causa habitual: poca memoria de video y aceleración 3D en Ubuntu 24.04 sobre VirtualBox.
1. Con la VM apagada:
```powershell
& $vbm modifyvm "misp-lab" --vram 128 --graphicscontroller vmsvga --accelerate-3d off
```
2. Dentro de la VM, usar Xorg en lugar de Wayland:
```bash
sudo sed -i 's/#WaylandEnable=false/WaylandEnable=false/' /etc/gdm3/custom.conf
sudo reboot
```
3. Alternativa recomendada: MISP no necesita escritorio. Trabajar solo por SSH con la VM en modo `headless`.

### La contraseña nueva no entra al iniciar sesión
Causa: el teclado de la VM estaba en inglés (US) y la contraseña se creó pensando en un teclado latinoamericano. Los símbolos cambian de lugar:

| Carácter | Español Latam | Inglés (US) |
|---|---|---|
| `@` | AltGr + Q | Shift + 2 |
| `"` | Shift + 2 | Shift + ' |
| `-` | tecla a la derecha de `.` | tecla junto al `0` |
| `ñ` | tecla propia | no existe |

Prevención: configurar el teclado en *Settings > Keyboard > Input Sources* o con `sudo dpkg-reconfigure keyboard-configuration`.

### Restablecer la contraseña desde GRUB
1. Reiniciar la VM y abrir GRUB (mantener `Shift` en BIOS o pulsar `Esc` en EFI).
2. Con la primera entrada de Ubuntu resaltada, pulsar `e`.
3. En la línea que empieza por `linux`, reemplazar `ro quiet splash $vt_handoff` por `rw init=/bin/bash`.
4. Pulsar `Ctrl + X` para arrancar. Se obtiene una consola de root sin pedir contraseña (solo para ese arranque).
5. Ejecutar:
```bash
passwd osboxes
passwd root
sync
reboot -f
```

El modo de recuperación estándar (*recovery mode*) también pide la contraseña de `root`, por eso no sirve si fue esa la que se olvidó.

### `Command 'passswd' not found`
Error de tecleo (`passswd` en lugar de `passwd`). Usar `Tab` para autocompletar y `↑` para recuperar comandos anteriores.

### `error: unexpected argument '-u' found` al ejecutar `sudo`
Se ejecutó un comando de Linux en PowerShell. Windows 11 trae su propio `sudo.exe`, que no entiende `-u www-data`. Verificar el prompt: debe decir `osboxes@misp-lab:~$`.

## Evidencia
Capturas de: hash de la imagen, configuración de la VM, `lsb_release -a`, `uname -r`, conexión SSH y lista de snapshots. Recortar las capturas para que no muestren rutas locales ni credenciales.
