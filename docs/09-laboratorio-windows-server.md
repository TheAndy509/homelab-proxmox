# Fase 9: Laboratorio — Windows Server con Active Directory (VM 103)

Objetivo: traer al servidor la VM de Windows Server del laboratorio de la universidad (VirtualBox), con su dominio `empresa.local` y la consola de **ManageEngine Endpoint Central**, para poder usarla desde casa y desde la U sin depender de la PC.

| Dato | Valor |
|---|---|
| VM | 103 `winserver` — 2 núcleos, 8 GB de RAM, disco 60 GB en `local-lvm` |
| IP / máscara / puerta de enlace | `192.168.1.53` / `255.255.255.0` / `192.168.1.1` |
| DNS de la tarjeta | `192.168.1.53` y `127.0.0.1` (el controlador de dominio se usa a sí mismo) |
| Dominio / equipo | `empresa.local` / `WINDOWSERVER` (controlador de dominio principal) |
| Endpoint Central | `http://192.168.1.53:8020` |

## Importación

El disco de VirtualBox (`winserver.vdi`, ~40 GB) se copió a `/mnt/pve/hdd1/import/` y se importó como disco de la VM 103 en `local-lvm`. Para no cambiar de golpe el hardware que Windows ya conocía, se dejó el disco en **SATA** y la red en **e1000**; el paso a VirtIO queda para después.

Antes de tocar nada se tomó la snapshot **`importada`**. Si algo sale mal: VM 103 → Snapshots → `importada` → *Rollback*.

## 1. Sin DHCP

En la U el servidor repartía IPs con su propio DHCP. En casa eso chocaría con el del router, así que **se detuvo antes de conectar la red** (la VM arrancó con la tarjeta en *Disconnected*):

```powershell
Stop-Service DHCPServer
Set-Service DHCPServer -StartupType Disabled
```

Más tarde se desinstaló el rol del todo:

```powershell
Get-DhcpServerInDC              # anotar la IP vieja
Remove-DhcpServerInDC -DnsName windowserver.empresa.local -IPAddress <IP_vieja>
Uninstall-WindowsFeature DHCP -IncludeManagementTools
Restart-Computer
```

## 2. Red

- IP fija con los valores de la tabla (`ncpa.cpl` → IPv4).
- Proxmox → VM 103 → Hardware → `net0` → desmarcar *Disconnected*.
- Se quitó la IP automática `169.254.x.x` que quedaba en la tarjeta (`Remove-NetIPAddress`).

## 3. DNS del dominio

El servidor resuelve el dominio, y lo que no es suyo lo reenvía a DNS públicos (antes reenviaba al DNS de la U, que en casa no existe):

```powershell
Set-DnsServerForwarder -IPAddress 1.1.1.1,8.8.8.8
ipconfig /registerdns
Restart-Service Netlogon
Get-DnsServerResourceRecord -ZoneName empresa.local -RRType A
```

Se borraron los registros `A` de `@` y del equipo que apuntaban a la IP vieja de la U o a `169.254.x.x` (`Remove-DnsServerResourceRecord`), y `dcdiag /q` quedó sin errores.

## 4. Hora

Kerberos rechaza los logins si el reloj se desvía más de 5 minutos, y el controlador de dominio es la fuente de hora del dominio:

```powershell
w32tm /config /manualpeerlist:"pool.ntp.org" /syncfromflags:manual /reliable:yes /update
Restart-Service w32time; w32tm /resync
```

## 5. Escritorio remoto para los usuarios del dominio

- Usuarios añadidos al grupo **Remote Desktop Users**.
- En un controlador de dominio no basta con el grupo: en `gpmc.msc` → *Default Domain Controllers Policy* → *User Rights Assignment* → **Allow log on through Remote Desktop Services** se añadió `BUILTIN\Remote Desktop Users` (sin quitar Administrators). Luego `gpupdate /force`.
- NLA activo (`UserAuthenticationRequired = 1`) y perfil de red `DomainAuthenticated`.
- Sin licencias RDS, Windows Server permite como máximo **2 sesiones RDP a la vez**.

## 6. Endpoint Central y el cliente Windows 10

El cliente Windows 10 sigue en VirtualBox en la PC, con la red en **adaptador puente** y el DNS de la tarjeta en `192.168.1.53`. Sin ese DNS no encuentra el dominio. Con `nslookup empresa.local` → `192.168.1.53` el agente vuelve a aparecer en la consola de Endpoint Central.

## 7. Portapapeles en la consola de Proxmox

Para pegar comandos en la consola noVNC:

1. VM 103 → Hardware → Display → **Clipboard: VNC**.
2. Dentro de Windows, instalar `virtio-win-guest-tools.exe` (de [fedorapeople.org](https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/latest-virtio/), ~30 MB), que trae el agente del portapapeles.
3. Apagar y encender la VM desde Proxmox (un reinicio desde Windows no aplica el cambio de pantalla).
4. Se pega en el panel lateral de noVNC → Portapapeles, y luego clic derecho en PowerShell.

## Seguridad

- **3389 (RDP) y 8020 nunca se abren en el router.** Solo se llega por la LAN o por Tailscale: el host ya anuncia toda la `192.168.1.0/24` (fase 6), así que desde la U basta con encender Tailscale en la laptop.
- El DHCP se apagó antes de conectar la VM a la red de casa.

## Prueba

- Puertos 3389, 8020, 53 (DNS) y 88 (Kerberos) abiertos desde la PC ✅
- `empresa.local` → `192.168.1.53`, y el servidor resuelve nombres de Internet ✅
- Registro SRV `_ldap._tcp.dc._msdcs.empresa.local` → `windowserver.empresa.local` ✅
- RDP desde la laptop con un usuario del dominio ✅
- Pegar en la consola noVNC ✅

## Pendiente

- RDP desde la PC con `xfreerdp3` da `ERRCONNECT_LOGON_FAILURE` aunque la contraseña es correcta (desde la laptop entra). Revisar el evento **4625** del servidor para ver el motivo.
- Pasar disco y red a **VirtIO**, activar el **QEMU guest agent** y desinstalar las Guest Additions de VirtualBox.
- Borrar `/mnt/pve/hdd1/import/winserver.vdi` cuando todo funcione.
- Kali Linux y el resto del laboratorio.
