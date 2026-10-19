# Homelab Proxmox

Servidor casero sobre **Proxmox VE** en un mini PC Lenovo ThinkCentre: nube personal (Nextcloud), y más adelante laboratorio de máquinas virtuales para ciberseguridad.

Proyecto personal para aprender virtualización, almacenamiento, backups y endurecimiento de servicios, documentado paso a paso.

## Hardware

| Componente | Detalle |
|---|---|
| Equipo | Lenovo ThinkCentre Tiny |
| CPU | Intel Core i5 (9ª gen) |
| RAM | 32 GB DDR4-2666 (2×16 GB, dual channel) |
| Disco sistema | SSD 256 GB — Proxmox + discos de sistema de VMs/CTs |
| Disco datos | HDD 1 TB — archivos de Nextcloud + backups |

## Arquitectura actual

```
ThinkCentre (Proxmox VE 9.2)
├── SSD  local / local-lvm
│   ├── CT 100 nextcloud (rootfs 16 GB)
│   ├── CT 101 sismos (rootfs 4 GB, monitor de sismos)
│   └── VM 103 winserver (60 GB, Windows Server + Active Directory, laboratorio)
└── HDD  hdd1 (Directory, ext4)
    ├── mp0 de CT 100 → /mnt/data (700 GB, archivos de Nextcloud)
    └── backups vzdump (semanales)
```

## Bitácora

| # | Fase | Estado |
|---|---|---|
| 0 | [Ensayo en VirtualBox](docs/00-ensayo-virtualbox.md) | ✅ |
| 1 | [Instalación y post-instalación de Proxmox](docs/01-proxmox.md) | ✅ |
| 2 | [Almacenamiento: HDD de datos](docs/02-almacenamiento.md) | ✅ |
| 3 | [Nextcloud en contenedor LXC](docs/03-nextcloud.md) | ✅ |
| 4 | [Backups programados](docs/04-backups.md) | ✅ |
| 5 | [Endurecimiento de Nextcloud](docs/05-endurecimiento-nextcloud.md) | ✅ |
| 6 | [Acceso remoto seguro (Tailscale, sin abrir puertos)](docs/06-tailscale.md) | ✅ |
| 7 | [Monitor de sismos de Panamá (avisos al celular y Alexa)](docs/07-sismos.md) | ✅ |
| 8 | [Endurecimiento del host (SSH con llave, 2FA, fail2ban, SMART y avisos, parches automáticos; firewall en curso)](docs/08-endurecimiento-host.md) | 🔄 |
| 9 | [Laboratorio: Windows Server con Active Directory](docs/09-laboratorio-windows-server.md) (Kali pendiente) | 🔄 |

Errores y lo aprendido de ellos: [lecciones.md](docs/lecciones.md).

## Pendientes

En orden de prioridad.

**Endurecimiento del host**
- [x] SSH solo con llave (2026-10-15, [fase 8](docs/08-endurecimiento-host.md)).
- [ ] Firewall de Proxmox: política DROP de entrada; 8006 y 22 solo desde LAN y Tailscale; CT 100 solo 80/443. Crear las reglas de permitir **antes** de activarlo, con teclado y pantalla conectados al servidor.
- [x] 2FA en la web de Proxmox + usuario administrador distinto de `root@pam` (2026-10-15).
- [x] fail2ban para la web de Proxmox y el login de Nextcloud (2026-10-16, [fase 8](docs/08-endurecimiento-host.md)).
- [x] Parches de seguridad automáticos en el CT 100: ya los hacía `cron-apt` de TurnKey (2026-10-18, [fase 8](docs/08-endurecimiento-host.md)).
- [ ] Cada mes: `apt full-upgrade` en el CT 100 (point releases y PHP de sury).

**Fiabilidad y avisos**
- [x] Monitoreo SMART con autotests (2026-10-18, [fase 8](docs/08-endurecimiento-host.md)).
- [x] Avisos de Proxmox (backups, SMART) por ntfy y correo (2026-10-18).
- [ ] Enviar el correo desde una cuenta dedicada al servidor, no la personal.
- [ ] Correo SMTP en Nextcloud.
- [ ] Copia externa de los archivos (regla 3-2-1) y disco `backup1` dedicado.

**Laboratorio (VM 103)**
- [ ] RDP desde la PC con `xfreerdp3` falla (`LOGON_FAILURE`); desde la laptop sí entra.
- [ ] Pasar disco y red a VirtIO + QEMU guest agent; desinstalar las Guest Additions de VirtualBox.
- [ ] Borrar `/mnt/pve/hdd1/import/winserver.vdi`.

**Limpieza y deuda**
- [x] Borrar `/root/nextcloud-data.old` del CT 100 (2026-10-15).
- [x] PHP 8.2 → 8.4: el CT 100 ya corre PHP 8.4 y Nextcloud 35.0.1 (verificado 2026-10-15).

## Seguridad de esta documentación

- No se publican contraseñas, tokens, claves ni nombres de usuario.
- Las capturas tienen tapados los nombres de usuario de la interfaz web.
- Las IPs que aparecen son de red privada (RFC 1918, `192.168.1.0/24`), no accesibles desde Internet.
