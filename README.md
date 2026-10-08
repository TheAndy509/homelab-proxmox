# Homelab Proxmox

Servidor casero sobre **Proxmox VE** en un mini PC Lenovo ThinkCentre: nube personal (Nextcloud), y más adelante laboratorio de máquinas virtuales para ciberseguridad.

Proyecto personal para aprender virtualización, almacenamiento, backups y endurecimiento de servicios, documentado paso a paso.

## Hardware

| Componente | Detalle |
|---|---|
| Equipo | Lenovo ThinkCentre Tiny |
| CPU | Intel Core i5 (9ª gen) |
| RAM | 4 GB (temporal, ampliación prevista) |
| Disco sistema | SSD 256 GB — Proxmox + discos de sistema de VMs/CTs |
| Disco datos | HDD 1 TB — archivos de Nextcloud + backups |

## Arquitectura actual

```
ThinkCentre (Proxmox VE 9.2)
├── SSD  local / local-lvm
│   └── CT 100 nextcloud (rootfs 16 GB)
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
| 6 | Acceso remoto seguro (Tailscale, sin abrir puertos) | ⏳ |
| 7 | Laboratorio de ciberseguridad (Windows Server, Kali…) | ⏳ |
| 8 | Minecraft (migrar el servidor existente; requiere más RAM) | ⏳ |

Errores y lo aprendido de ellos: [lecciones.md](docs/lecciones.md).

## Pendientes

En orden de prioridad.

**Fase 6 — Acceso remoto (Tailscale)**
- [ ] Instalar Tailscale en el host Proxmox como *subnet router* que anuncie `192.168.1.0/24`.
- [ ] Añadir la IP / nombre MagicDNS de Tailscale a `trusted_domains` de Nextcloud.
- [ ] Clientes: app Tailscale + app Nextcloud en móvil y portátil.
- [ ] Revisar *key expiry* de los dispositivos.
- Decisión: sin abrir puertos ni exponer Nextcloud a Internet. Compartir con gente sin Tailscale (Funnel / Cloudflare Tunnel) solo si algún día hace falta.

**Endurecimiento del host**
- [ ] SSH solo con llave; desactivar login de root por contraseña.
- [ ] Firewall de Proxmox: política DROP de entrada; 8006 y 22 solo desde LAN y Tailscale; CT 100 solo 80/443. Crear las reglas de permitir **antes** de activarlo, con teclado y pantalla conectados al servidor.
- [ ] 2FA en la web de Proxmox + usuario administrador distinto de `root@pam`.
- [ ] fail2ban para la web de Proxmox y el login de Nextcloud.
- [ ] `unattended-upgrades` en el CT 100 (parches de seguridad automáticos).

**Fiabilidad y avisos**
- [ ] Monitoreo SMART del HDD (datos y backups están en el mismo disco).
- [ ] Correo SMTP para recibir avisos de backups, SMART y Nextcloud.
- [ ] Copia externa de los archivos (regla 3-2-1) y disco `backup1` dedicado.

**Limpieza y deuda**
- [ ] Borrar `/root/nextcloud-data.old` del CT 100.
- [ ] PHP 8.2 → migrar a TurnKey/Debian 13 antes de Nextcloud 35.

## Seguridad de esta documentación

- No se publican contraseñas, tokens, claves ni nombres de usuario.
- Las capturas tienen tapados los nombres de usuario de la interfaz web.
- Las IPs que aparecen son de red privada (RFC 1918, `192.168.1.0/24`), no accesibles desde Internet.
