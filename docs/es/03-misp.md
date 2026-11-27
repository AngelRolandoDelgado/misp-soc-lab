# Fase 3: Instalación de MISP 2.5 con el script oficial

## Objetivo
Instalar MISP 2.5 en la VM Ubuntu 24.04 usando el instalador oficial del proyecto y verificar que quedó operativo.

## Entorno
- VM `misp-lab`: Ubuntu 24.04, 8 GB de RAM, 4 vCPU (ver Fase 2)
- Acceso por SSH: `ssh -p 2222 osboxes@127.0.0.1`
- Acceso web: `https://127.0.0.1:8443` (reenvío de puertos NAT hacia el 443 de la VM)
- Punto de partida: snapshot `01-base-ubuntu-actualizado`

## 1. Prerrequisitos
Conectado por SSH a la VM:

```bash
sudo apt -y install curl tmux
curl -sI https://raw.githubusercontent.com | head -n 1
tmux new -s misp
```

| Comando | Qué hace |
|---|---|
| `apt -y install curl tmux` | Instala `curl` (transferencias por URL) y `tmux` (sesiones persistentes) |
| `curl -sI <URL>` | `-s` silencioso, `-I` solo cabeceras HTTP; confirma internet y DNS |
| `head -n 1` | Muestra solo la primera línea (el código de estado, p. ej. `200`) |
| `tmux new -s misp` | Crea una sesión llamada `misp`; el proceso sobrevive si se corta SSH |

Para salir de `tmux` sin cerrarla: `Ctrl + b` y luego `d`. Para volver: `tmux attach -t misp`.

> `curl` no viene en la imagen de osboxes. Se instala con `apt` y no con `snap`: la versión snap corre confinada (*sandbox*), mezcla gestores de paquetes y se actualiza por separado.

## 2. Descargar y revisar el script
```bash
wget --no-cache -O /tmp/INSTALL.ubuntu2404.sh https://raw.githubusercontent.com/MISP/MISP/refs/heads/2.5/INSTALL/INSTALL.ubuntu2404.sh
ls -lh /tmp/INSTALL.ubuntu2404.sh
head -n 5 /tmp/INSTALL.ubuntu2404.sh
less /tmp/INSTALL.ubuntu2404.sh
```

- `wget --no-cache -O <ruta>`: descarga una copia fresca y la guarda en la ruta indicada.
- `ls -lh`: confirma que el archivo existe y no está vacío (unos 32 KB).
- `head -n 5`: debe empezar por `#!/bin/bash` y comentarios sobre MISP 2.5 en Ubuntu 24.04.
- `less`: permite leer el script antes de ejecutarlo como root (salir con `q`).

## 3. Ejecutar la instalación
```bash
sudo bash /tmp/INSTALL.ubuntu2404.sh
```

Instala Apache, MariaDB, Redis, PHP 8.3, Composer, extensiones PECL, el entorno virtual de Python y los workers, y carga las estructuras JSON (taxonomías, galaxias, objetos). Tarda entre 15 y 40 minutos según la conexión.

- Registro: `/var/log/misp_install.log`
- Credenciales iniciales: `/root/misp_settings.txt` (modo 600, solo root). **No publicar nunca este archivo ni capturas que lo muestren.**

Para comprobar el avance desde otra sesión:
```bash
pgrep -af INSTALL.ubuntu2404
sudo tail -n 20 /var/log/misp_install.log
```

El proceso terminó cuando el prompt vuelve y el script imprime `MISP setup complete`.

## 4. Primer acceso y ajustes
1. Abrir `https://127.0.0.1:8443` y aceptar el certificado autofirmado (es un laboratorio).
2. Iniciar sesión con las credenciales del archivo anterior y cambiar la contraseña del administrador.
3. Política de contraseñas de MISP: mínimo 12 caracteres con mayúscula, minúscula y número o símbolo, o bien 16 caracteres o más. Una frase de contraseña larga cumple ambas.
4. Ajustar la URL base, ya que el script usa `https://misp.local` por defecto:

```bash
sudo -u www-data /var/www/MISP/app/Console/cake Admin setSetting MISP.baseurl "https://127.0.0.1:8443"
sudo -u www-data /var/www/MISP/app/Console/cake Admin getSetting MISP.baseurl
```

- `sudo -u www-data`: ejecuta como el usuario de Apache, dueño de los archivos de MISP.
- `cake Admin setSetting / getSetting`: escribe o lee una opción desde la consola de MISP.
- La respuesta debe mostrar `"value": "https://127.0.0.1:8443"` y `"errorMessage": null`.

## 5. Verificación
```bash
systemctl is-active apache2 mariadb redis-server
```
Deben aparecer tres `active`.

En la web, *Administration > Server Settings & Maintenance*:
- **Diagnostics:** MISP v2.5.51 instalado e igual a la última versión disponible, estado OK, todos los submódulos en OK (taxonomías, galaxias, objetos, warninglists, PyMISP).
- **Workers:** los seis tipos (`cache`, `default`, `email`, `prio`, `update`, `scheduler`) con procesos saludables y 0 trabajos en cola.
- Los avisos en las pestañas *Plugin* y *AI* son habituales en una instalación nueva (servicios opcionales sin configurar) y no bloquean el uso.

## 6. Snapshot
```bash
sudo poweroff
```
```powershell
& $vbm snapshot "misp-lab" take "02-misp-instalado"
& $vbm snapshot "misp-lab" list
```

Árbol resultante:
```
01-base-ubuntu-actualizado
  └─ 02-misp-instalado  (estado actual)
```

## Troubleshooting (problemas encontrados)

### `curl: command not found`
La imagen mínima no lo incluye. Instalarlo con `sudo apt -y install curl`.

### `syntax error near unexpected token 'newline'`
Se escribió literalmente el marcador `<URL>`. Los signos `< >` en una guía significan "reemplazar por su valor"; además, `<` es un operador de redirección en bash.

### `Command not found` o `unexpected argument '-u'` al ejecutar `sudo`
El comando de Linux se ejecutó en PowerShell. Comprobar el prompt: dentro de la VM debe ser `osboxes@misp-lab:~$`. Windows 11 trae su propio `sudo.exe`, que no entiende las mismas opciones.

### La contraseña de MISP es rechazada
No cumplía la complejidad exigida. Usar una frase de contraseña de 16 caracteres o más, por ejemplo cuatro palabras separadas por guiones con un número y un símbolo. Evitar `ñ` y tildes.

### Suspensión del equipo durante la instalación
Al suspender Windows, VirtualBox congela la VM y las descargas en curso pueden expirar. Desactivar la suspensión mientras se instala (`powercfg /change standby-timeout-ac 0` en PowerShell de administrador) y restaurar el valor original después. Si el script falla a medias, restaurar el snapshot `01-base-ubuntu-actualizado` y repetir.

### Actualizaciones aplazadas por *phasing*
`apt list --upgradable` puede mostrar unos pocos paquetes que `full-upgrade` no instala. Es normal y no afecta a MISP.

## Evidencia
Capturas de: el resumen final del script (**sin credenciales**), `getSetting MISP.baseurl`, *Diagnostics*, *Workers* y la lista de snapshots. Recortar las imágenes para que no muestren rutas locales, claves API ni contraseñas.
