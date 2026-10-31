# Lecciones aprendidas

| Problema | Causa | Solución |
|---|---|---|
| Web UI inaccesible en el ensayo | El instalador puso una IP de otra subred | Editar `vmbr0` y `/etc/hosts`, aplicar con `ifreload -a` |
| "No Proxmox VE repository is enabled" | Se deshabilitó `enterprise` sin añadir `no-subscription` | Añadir el repositorio No-Subscription |
| `apt update` falla con error 401 | El repo `ceph enterprise` seguía activo | Deshabilitarlo también |
| No se pueden hacer snapshots del CT | Mount point en un storage de tipo Directory | Backups vzdump, o LVM-Thin en el HDD |
| `chown` falla en `lost+found` | Carpeta del host en un CT unprivileged | Ignorarlo, no afecta |
| `^[[200~` al pegar comandos | Bracketed paste en la consola noVNC | Pegar línea por línea |
| TurnKey pide reiniciar por el kernel | Un LXC no tiene kernel propio | *Skip*; el kernel lo actualiza el host |
| `sudo: command not found` en el CT | TurnKey no instala `sudo` | Como root: `runuser -u www-data -- php occ …` |
| Nextcloud 29 sin parches de seguridad | La plantilla TurnKey trae una versión vieja | Updater oficial, una versión mayor cada vez |
| Falla la integridad en `layout.guest.php` | TurnKey añade su pie de página al login | Comparar con el original (`diff`); el updater lo restaura |
| `app:update` no actualiza nada | Se lanzó con el modo mantenimiento activo | Desactivar el mantenimiento y repetir |
| Avisos falsos en `occ setupchecks` | Desde la terminal no hay petición web | La lista que vale es la de Administración → Vista general |
| El log seguía en la carpeta de datos antigua | TurnKey fija `logfile` en `config.php` | Mover el log a `/var/log/nextcloud` y cambiar `logfile` |
| `git clone` falla en el CT 101 pidiendo usuario | El repo del código es privado y el CT no tiene credenciales de GitHub | Copiar los archivos con `scp` desde la PC (no poner tokens de GitHub en el servidor) |
| Voice Monkey responde 403 `error code: 1010` | Cloudflare bloquea el User-Agent por defecto de Python (`Python-urllib`) | Enviar un User-Agent propio (el script usa `sismos-panama/1`) |
| Alexa responde "no tengo claro cómo ayudarte con eso" en vez de leer el aviso | La rutina usa la acción personalizada `open Voice Monkey`, en inglés, y el Echo está en español | Cambiar la acción por *Skills → Voice Monkey* |
| fail2ban del CT 100 no arrancaba (`Have not found any log file for nextcloud jail`) | Se movió el log de Nextcloud y el jail de TurnKey seguía apuntando a la ruta vieja | Corregir `logpath`; al mover un log, buscar quién más lo lee (`grep -r` en `/etc`) |
| fail2ban no veía los logins fallidos de Nextcloud | Con `loglevel` 3 solo se anotan errores, y un login fallido es *warning* | `loglevel` 2 |
| `fail2ban-client status` justo tras `restart` da `Failed to access socket path` | El servicio todavía estaba arrancando | Esperar un par de segundos y repetir |
| Se iba a instalar `unattended-upgrades` en el CT 100 | TurnKey ya trae `cron-apt` instalando los parches de seguridad cada noche | Antes de instalar una herramienta, revisar qué trae ya la plantilla (`ls /etc/cron.d`, `/etc/apt/apt.conf.d`) |
| La web de Proxmox da `ERR_CONNECTION_REFUSED` aunque el servidor responde al ping | fail2ban baneó la IP de la propia PC tras 3 logins fallidos | `fail2ban-client set proxmox unbanip <IP>` por SSH; con `findtime = 2d` los fallos se acumulan dos días |
| `C:\vgt.exe`: *The file or directory is corrupted and unreadable* | `Invoke-WebRequest` dejó la descarga incompleta | Comprobar el tamaño con `(Get-Item archivo).Length` y bajarlo de nuevo con el navegador |
| La guía de pasos usaba `windowsserver.empresa.local` | El equipo se llama `windowserver` (una sola "s") | Mirar el nombre real con `hostname` o el registro SRV en vez de escribirlo de memoria |
| No se puede pegar en la consola noVNC de la VM Windows | Falta el agente de portapapeles en el invitado | Display → Clipboard: VNC + `virtio-win-guest-tools`, y apagar/encender la VM desde Proxmox |
| La VM de Wazuh perdía conexiones (timeouts con GitHub, una sesión SSH cortada) | Un celular tenía la misma IP: el DHCP del router repartía toda la red (`.2–.254`) | Rango del DHCP en `.100–.199` y las IPs fijas fuera de él. Antes de fijar una IP, hacer `ping` con el equipo apagado |
| `ssh-import-id gh:usuario` da `urlopen error timed out` | La VM tenía la IP duplicada (fila anterior) | Copiar la llave desde la PC por SSH y revisar la red |
| El instalador de Ubuntu dejó `/` con 24 GB en un disco de 50 | Con LVM, la instalación guiada solo asigna la mitad del grupo de volúmenes | En *Storage configuration*, ampliar `ubuntu-lv` al máximo (o borrarlo y crearlo de nuevo con todo el espacio) |
| Se iba a usar Ubuntu 25.04 para Wazuh | Las versiones no LTS tienen 9 meses de soporte (la 25.04 terminó en enero de 2026) y Wazuh solo soporta LTS | En servidores, solo versiones LTS |
| La llave SSH de la laptop no se podía usar | Passphrase olvidada: no se puede recuperar | Llave nueva por equipo (`-C "usuario@equipo"`), passphrase en el gestor de contraseñas y `ssh-agent` |
| Contraseña "incorrecta" en la consola noVNC desde el celular | El teclado del celular pone mayúscula y autocorrige | Entrar por SSH con una app (Termius) |
| El dashboard de Wazuh da `ERR_CONNECTION_REFUSED` | Se abrió con `http://` (puerto 80); el dashboard solo escucha en el 443 | Escribir `https://` |
| El agente de Wazuh queda "unconfigured" al instalarlo en Proxmox con `dpkg -i` | Falta `lsb-release`, que Proxmox no trae (y tampoco `sudo`) | `WAZUH_MANAGER=… WAZUH_AGENT_NAME=… apt install -f -y` instala la dependencia y termina la configuración con los datos correctos |
| `NET START Wazuh` falla justo después de instalar el agente | `msiexec` vuelve enseguida y sigue instalando en segundo plano | Esperar un minuto y repetir (o instalar con `Start-Process msiexec -Wait`) |

## Login de Proxmox: "authentication failed" con el usuario nuevo

**Qué pasó:** el usuario nuevo no podía entrar a la web aunque la contraseña era correcta.
**Causa:** en la pantalla de login seguía elegido el realm *Linux PAM*. El log (`journalctl -u pvedaemon`) lo mostraba: `no such user ('…@pam')`, porque el usuario existe en el realm `pve`.
**Lección:** en el login se elige el realm que corresponde al usuario. Ante un fallo de autenticación, revisar primero el log de `pvedaemon`, que dice exactamente qué usuario y realm se intentó.

## Gmail: "No hemos podido verificar que eres tú" con la cuenta nueva

**Qué pasó:** con la cuenta de Gmail recién creada para el servidor, Google no dejaba activar la verificación en 2 pasos y sin ella no se puede crear la contraseña de aplicación ("La opción de configuración que buscas no está disponible para tu cuenta").
**Causa:** un filtro antifraude de Google. Una cuenta nueva que cambia su seguridad desde un navegador sin historial parece sospechosa.
**Lección:** crear la cuenta de avisos con tiempo, hacer la configuración de seguridad desde un dispositivo donde ya tenga la sesión iniciada (el celular) y no reintentar seguido. Mientras tanto se envía desde otra cuenta y se cambia cuando Google la libere.
