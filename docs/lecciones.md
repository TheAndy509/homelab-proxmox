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

## Login de Proxmox: "authentication failed" con el usuario nuevo

**Qué pasó:** el usuario nuevo no podía entrar a la web aunque la contraseña era correcta.
**Causa:** en la pantalla de login seguía elegido el realm *Linux PAM*. El log (`journalctl -u pvedaemon`) lo mostraba: `no such user ('…@pam')`, porque el usuario existe en el realm `pve`.
**Lección:** en el login se elige el realm que corresponde al usuario. Ante un fallo de autenticación, revisar primero el log de `pvedaemon`, que dice exactamente qué usuario y realm se intentó.
