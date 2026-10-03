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
