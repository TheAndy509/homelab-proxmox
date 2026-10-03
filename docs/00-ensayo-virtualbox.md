# 0. Ensayo en VirtualBox

Antes de tocar el hardware real, instalé Proxmox dentro de una VM de VirtualBox para practicar todo el proceso sin riesgo.

## Configuración de la VM

- 16 GB RAM, 2 CPU, **VT-x anidado** activado (para poder crear VMs dentro de Proxmox).
- Red en modo **bridge** sobre la tarjeta física, para que Proxmox tenga IP propia en la LAN.
- ISO: Proxmox VE 9.2.

## Problema: IP en otra subred

El instalador dejó Proxmox en `192.168.100.2`, pero mi LAN es `192.168.1.0/24`: la web (`:8006`) no era accesible.

**Solución:** cambiar la IP del bridge `vmbr0` en `/etc/network/interfaces` y en `/etc/hosts`, y aplicar con:

```sh
ifreload -a
```

Después: snapshot de VirtualBox, repositorio `pve-no-subscription` y `apt full-upgrade`.

## Qué saqué del ensayo

- Revisar que la IP y la puerta de enlace del instalador coincidan con la red real.
- La VM de ensayo y el servidor real **no pueden encenderse a la vez** si usan la misma IP.
