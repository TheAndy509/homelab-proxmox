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
| 5 | Endurecimiento de Nextcloud (avisos de seguridad) | ⏳ |
| 6 | Acceso remoto seguro (Tailscale, sin abrir puertos) | ⏳ |
| 7 | Minecraft (requiere más RAM) | ⏳ |
| 8 | Laboratorio de ciberseguridad (Windows Server, Kali…) | ⏳ |

Errores y lo aprendido de ellos: [lecciones.md](docs/lecciones.md).

## Seguridad de esta documentación

- No se publican contraseñas, tokens, claves ni nombres de usuario.
- Las capturas tienen tapados los nombres de usuario de la interfaz web.
- Las IPs que aparecen son de red privada (RFC 1918, `192.168.1.0/24`), no accesibles desde Internet.
