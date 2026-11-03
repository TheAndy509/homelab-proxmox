# Fase 10: SIEM con Wazuh (VM 104)

Objetivo: un servidor **Wazuh** (SIEM + XDR) en su propia VM, para el Parcial #2 de Ciberseguridad II y, después, para vigilar el resto del homelab. Instalación "todo en uno": servidor, indexador y dashboard en la misma VM; los equipos envían sus eventos con el agente de Wazuh.

| Dato | Valor |
|---|---|
| VM | 104 `wazuh` — 4 núcleos (tipo `host`), 8 GB de RAM sin ballooning, disco 50 GB en `local-lvm` |
| Sistema | Ubuntu Server 24.04.5 LTS |
| IP / puerta de enlace / DNS | `192.168.1.54/24` / `192.168.1.1` / `192.168.1.1` |
| Wazuh | 4.14.9, todo en uno |
| Dashboard | `https://192.168.1.54` |
| Snapshots | `Ubuntu_Base` (Ubuntu actualizado, 29 oct) · `wazuh_instalado` (Wazuh sin configurar, 30 oct) |

## Por qué así

| Decisión | Motivo |
|---|---|
| VM dedicada, no en el host | Wazuh instala muchos paquetes (OpenSearch, Java) que podrían chocar con los de Proxmox, aumenta la superficie de ataque del host y en una VM hay snapshots |
| VM, no LXC | El indexador (OpenSearch) suele dar problemas en contenedores sin privilegios |
| Ubuntu 24.04 LTS, no 25.04 | 25.04 no es LTS: tuvo 9 meses de soporte y terminó en enero de 2026. Wazuh solo soporta versiones LTS |
| Disco en el SSD | El indexador necesita disco rápido |
| Discard activado | Cuando Ubuntu borra archivos, devuelve el espacio al pool thin del SSD |
| Sin ballooning | El indexador reserva su memoria al arrancar y se lleva mal con que Proxmox se la quite |
| 4 vCPU, 8 GB, 50 GB | Es lo que Wazuh recomienda para hasta 25 agentes |

## 1. Espacio en el SSD antes de crear la VM

`pvesm status` (29 oct):

| Almacenamiento | Disco | Total | Usado | Libre |
|---|---|---|---|---|
| `local-lvm` | SSD | 144 GiB | 84 GiB (58 %) | 60 GiB |
| `local` | SSD | 69 GiB | 7 GiB | 58 GiB |
| `hdd1` | HDD | 916 GiB | 367 GiB (40 %) | 502 GiB |

- El pool `local-lvm` ya está **sobreasignado**: sumando los discos thin asignados da unos 172 GB sobre 144. Es normal con thin, pero si el pool llega al 100 % todas las VMs y CTs que viven en él pueden dar errores de disco. Hay que vigilar el `Data%` de `lvs` y mantenerlo por debajo del 80 %.
- El snapshot `importada` de la VM 103 retiene unos 20 GiB de bloques viejos.
- Plan: Wazuh en el SSD; el cliente Windows y Kali en `hdd1` con formato **qcow2**, que en un almacenamiento tipo Directory sí permite snapshots de VMs.

## 2. Ubuntu Server

La ISO se bajó directo al servidor: **local → ISO Images → Download from URL**, con `https://releases.ubuntu.com/24.04/ubuntu-24.04.5-live-server-amd64.iso` y su SHA-256 de `SHA256SUMS`.

Creación de la VM: SCSI Controller **VirtIO SCSI single**, ✅ Qemu Agent, disco SCSI con ✅ Discard, ✅ SSD emulation, ✅ IO thread, red VirtIO en `vmbr0` con ✅ Firewall.

En el instalador:

- **Red:** IPv4 manual con los datos de la tabla.
- **Almacenamiento:** con "Use an entire disk" + LVM, el instalador dejó `ubuntu-lv` (la raíz) con **24 GB de 48** y el resto como espacio libre. Se borró `ubuntu-lv` y se volvió a crear con todo el espacio (47.996 GB, ext4, montado en `/`).
- **Sin cifrado LUKS:** pediría la contraseña en la consola en cada arranque y la VM no volvería sola tras un reinicio.
- **SSH:** OpenSSH instalado. La importación de la llave desde GitHub falló con `urlopen error timed out` (la causa resultó ser el conflicto de IP de la sección 3).

Después de instalar:

```sh
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y qemu-guest-agent
sudo timedatectl set-timezone America/Panama
sudo reboot
df -h /          # Size ≈ 47G
```

Proxmox muestra la IP de la VM en **Summary** (el guest agent funciona). Snapshot **`Ubuntu_Base`** sin RAM.

## 3. Incidente: IP duplicada

**Síntomas:** la importación de la llave desde GitHub daba timeout, una sesión SSH se cortó justo después de entrar y algunas conexiones HTTPS no respondían, aunque `ping 1.1.1.1` funcionaba.

**Diagnóstico:**

1. Con la VM 104 **apagada**, desde el host: `ping -c 3 192.168.1.54` → **respondía** (`ttl=64`). Otro equipo tenía la misma IP.
2. `ip neigh show 192.168.1.54` → una MAC **administrada localmente** (aleatoria), típica de un celular con "dirección privada de Wi-Fi".
3. En la configuración DHCP del router, el rango era `192.168.1.2 – 192.168.1.254`: el router repartía **toda la red**, incluidas las IPs fijas de los servidores.

**Solución:** rango del DHCP en `192.168.1.100 – 192.168.1.199`. De la `.2` a la `.99` y de la `.200` a la `.254` quedan para IPs fijas.

**Prueba:**

- Lista de clientes DHCP: el celular pasó a una IP `.1xx` ✅
- `ping 192.168.1.54` con la VM apagada → sin respuesta ✅
- Desde la VM: `curl https://packages.wazuh.com/4.x/` → 403 (conecta; la carpeta no permite listar) y `curl -4 https://api.github.com` → 200 ✅. Por IPv6, `api.github.com` no resuelve: GitHub no tiene IPv6.

Esto protege también a Nextcloud (`.50`), sismos (`.51`) y el Windows Server (`.53`), que tenían el mismo riesgo.

## 4. Acceso SSH: una llave por equipo

La llave de la laptop tenía una passphrase olvidada y no se puede recuperar, así que se generó una nueva, con un comentario que dice de qué equipo es:

```powershell
ssh-keygen -t ed25519 -C "anndre@laptop"
Get-Service ssh-agent | Set-Service -StartupType Automatic
Start-Service ssh-agent
ssh-add $env:USERPROFILE\.ssh\id_ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh anndre@192.168.1.54 "cat > ~/.ssh/authorized_keys"
```

El agente SSH de Windows pide la passphrase una vez por sesión. Cada equipo tiene su propia llave: si se pierde uno, se borra solo su línea en los servidores.

Por ahora la VM **todavía acepta contraseña** por SSH. Antes de desactivarla faltan las llaves de la PC de casa y del celular (Termius genera la suya). La configuración prevista sigue el mismo patrón que el host en la [fase 8](08-endurecimiento-host.md):

```sh
printf 'PasswordAuthentication no\nKbdInteractiveAuthentication no\nPermitRootLogin no\n' | sudo tee /etc/ssh/sshd_config.d/00-hardening.conf
sudo sshd -t
sudo sshd -T | grep -Ei 'passwordauthentication|kbdinteractiveauthentication|permitrootlogin'
sudo systemctl restart ssh
```

## 5. Instalación de Wazuh

Por SSH, dentro de `tmux` para que un corte de conexión no deje la instalación a medias (`tmux new -s wazuh`, y si se corta, `tmux attach -t wazuh`):

```sh
curl -sO https://packages.wazuh.com/4.14/wazuh-install.sh
sudo bash ./wazuh-install.sh -a
```

`-a` instala los tres componentes en esta VM. Al final muestra el usuario `admin` y su contraseña generada, que se guardó en el gestor de contraseñas.

> El instalador deja `wazuh-install-files.tar` en la carpeta actual, con todas las contraseñas internas y los certificados. No se copia sin cifrar ni se sube a ningún repositorio. Si se pierde la contraseña de `admin`: `sudo tar -O -xvf wazuh-install-files.tar wazuh-install-files/wazuh-passwords.txt`.

Comprobación de los servicios: `sudo systemctl status wazuh-manager wazuh-indexer wazuh-dashboard --no-pager` → los tres `active (running)`.

Wazuh recomienda desactivar su repositorio para que un `apt full-upgrade` no actualice un componente y no los otros:

```sh
sudo sed -i "s/^deb /#deb /" /etc/apt/sources.list.d/wazuh.list
sudo apt update
```

El dashboard se abre en `https://192.168.1.54`. El aviso de certificado es esperado (lo genera Wazuh). Con `http://` da `ERR_CONNECTION_REFUSED`: solo escucha en el 443.

| Puerto | Para qué | Quién se conecta |
|---|---|---|
| 443/tcp | Dashboard | El navegador, solo desde LAN o Tailscale |
| 1514/tcp | Envío de eventos | Agentes |
| 1515/tcp | Registro de agentes | Agentes, al instalarse |
| 55000/tcp | API del servidor | Dashboard y herramientas de administración |

## 6. Primer agente: Windows Server (`winserver-dc`)

En el dashboard: **Endpoints → Deploy new agent** → Windows MSI, server address `192.168.1.54`, nombre `winserver-dc` (no se puede cambiar después), grupo `default`. El dashboard genera el comando para PowerShell como administrador:

```powershell
Invoke-WebRequest -Uri https://packages.wazuh.com/4.x/windows/wazuh-agent-4.14.9-1.msi -OutFile $env:tmp\wazuh-agent
msiexec.exe /i $env:tmp\wazuh-agent /q WAZUH_MANAGER='192.168.1.54' WAZUH_AGENT_NAME='winserver-dc'
NET START Wazuh
```

El primer `NET START Wazuh` falló: `msiexec` devuelve el control enseguida y sigue instalando en segundo plano. Esperando un minuto y repitiendo, el servicio arrancó.

El agente es de la misma versión que el servidor (4.14.9). La regla de Wazuh es que el agente nunca sea más nuevo que el servidor.

**Línea base**, sin ataques, en las primeras 24 horas: 623 alertas de nivel medio (7–11) y 1 321 de nivel bajo (0–6), 0 altas y 0 críticas. Gran parte viene de **Configuration Assessment** (revisión del servidor contra una guía CIS) y de eventos normales de Windows. Sirve de referencia para distinguir, en las simulaciones, lo que genera cada técnica.

## 7. Agente en el host Proxmox (`pve-host`)

**Endpoints → Deploy new agent** → Linux DEB amd64, server `192.168.1.54`, nombre `pve-host`. El comando se ejecuta en **server → Shell**, que ya es `root`. Proxmox no trae `sudo`, así que hay que quitarlo del comando que genera el dashboard:

```sh
wget https://packages.wazuh.com/4.x/apt/pool/main/w/wazuh-agent/wazuh-agent_4.14.9-1_amd64.deb
WAZUH_MANAGER='192.168.1.54' WAZUH_AGENT_NAME='pve-host' dpkg -i ./wazuh-agent_4.14.9-1_amd64.deb
```

`dpkg` dejó el paquete sin configurar: **falta `lsb-release`**, que Proxmox no trae. `apt install -f` instala la dependencia y termina la configuración; se le pasan otra vez las variables para que la tome con la IP y el nombre correctos:

```sh
WAZUH_MANAGER='192.168.1.54' WAZUH_AGENT_NAME='pve-host' apt install -f -y
grep -nE "<address>|<agent_name>" /var/ossec/etc/ossec.conf
systemctl daemon-reload
systemctl enable --now wazuh-agent
```

Proxmox 9 guarda los logs en el *journal* de systemd, no en `/var/log/auth.log`. Para que el agente vea los inicios de sesión falta instalar `rsyslog` y añadir los logs propios de Proxmox (pendiente):

```xml
<localfile>
  <location>/var/log/pveproxy/access.log</location>
  <log_format>syslog</log_format>
</localfile>
<localfile>
  <location>/var/log/fail2ban.log</location>
  <log_format>syslog</log_format>
</localfile>
```

## 8. Regla propia: baneos de fail2ban

Los logins fallidos de la web de Proxmox ya generaban alertas propias de Wazuh (regla **87201**, y **87203** para los logins correctos). Pero cuando fail2ban baneaba una IP no aparecía nada: Wazuh solo guarda lo que genera alerta.

**Investigación** con `wazuh-logtest` en la VM 104, pegando la línea real del baneo:

```
2026-11-01 18:35:55,010 fail2ban.actions        [1032]: NOTICE  [proxmox] Ban 192.168.1.185
```

→ `No decoder matched`. El evento llegaba, pero ningún decoder de Wazuh entiende el formato de fail2ban.

**Decoder** (`/var/ossec/etc/decoders/local_decoder.xml`). Reconoce la línea por `] Ban ` / `] Unban `: al separar la fecha, Wazuh no deja `fail2ban.actions` en la parte del mensaje que revisan los decoders.

```xml
<decoder name="fail2ban-custom">
  <prematch>] Ban |] Unban </prematch>
</decoder>

<decoder name="fail2ban-custom-action">
  <parent>fail2ban-custom</parent>
  <regex>[(\w+)]\s+(\w+)\s+(\S+)</regex>
  <order>jail,action,srcip</order>
</decoder>
```

**Reglas** (`/var/ossec/etc/rules/local_rules.xml`; los IDs desde 100000 son para reglas propias):

```xml
<group name="fail2ban,proxmox,">
  <rule id="100110" level="8">
    <decoded_as>fail2ban-custom</decoded_as>
    <action>Ban</action>
    <description>fail2ban bloqueó la IP $(srcip) en el jail $(jail)</description>
    <mitre>
      <id>T1110</id>
    </mitre>
  </rule>

  <rule id="100111" level="3">
    <decoded_as>fail2ban-custom</decoded_as>
    <action>Unban</action>
    <description>fail2ban desbloqueó la IP $(srcip) en el jail $(jail)</description>
  </rule>
</group>
```

Procedimiento seguro para aplicar cambios en decoders y reglas:

```sh
sudo /var/ossec/bin/wazuh-analysisd -t      # revisa la configuración sin reiniciar
sudo systemctl restart wazuh-manager        # solo si no hay ERROR ni CRITICAL
sudo /var/ossec/bin/wazuh-logtest           # pegar la línea de log y revisar las fases 2 y 3
```

Prueba con una IP de documentación (`192.0.2.10`, reservada para ejemplos), en la Shell de Proxmox:

```sh
fail2ban-client set proxmox banip 192.0.2.10
fail2ban-client set proxmox unbanip 192.0.2.10
```

`wazuh-logtest` → decoder `fail2ban-custom` (`action: Ban`, `jail: proxmox`, `srcip`), regla **100110** nivel 8, MITRE **T1110 Brute Force** (táctica *Credential Access*).

## Prueba

- Dashboard en `https://192.168.1.54` con `admin` ✅
- `wazuh-manager`, `wazuh-indexer` y `wazuh-dashboard` activos ✅
- **Agents summary: Active (1)**, `winserver-dc` ✅
- `pve-host` activo en **Endpoints** (2026-10-31) ✅
- Login fallido en la web de Proxmox → alerta **87201**; login correcto → **87203** ✅
- `banip` → alerta **100110** "fail2ban bloqueó la IP 192.0.2.10 en el jail proxmox"; `unbanip` → **100111** (2026-11-03) ✅

## Pendiente

- Snapshot `wazuh_configurado` con los agentes, los grupos, el usuario y la regla de fail2ban (`qm snapshot 104 wazuh_configurado`).
- Grupo `windows` con su `agent.conf`, para que los agentes Windows recojan Sysmon y PowerShell:

  ```xml
  <agent_config>
    <localfile>
      <location>Microsoft-Windows-Sysmon/Operational</location>
      <log_format>eventchannel</log_format>
    </localfile>
    <localfile>
      <location>Microsoft-Windows-PowerShell/Operational</location>
      <log_format>eventchannel</log_format>
    </localfile>
  </agent_config>
  ```

- Sysmon en el Windows Server y en el cliente, con la configuración del profesor.
- GPO *Turn on PowerShell Script Block Logging* (evento 4104).
- Cliente Windows 10/11 unido al dominio y Kali, en `hdd1` con qcow2.
- Red aislada para el laboratorio (`vmbr1`), cuentas `pve` limitadas a un pool para el grupo y sus IPs en el `ignoreip` de fail2ban, antes de darles acceso.
- Llaves SSH de la PC de casa y del celular, y desactivar la contraseña en el SSH de la VM.
- Retención de índices (por ejemplo, 30 días) para que el disco no crezca sin límite.
- Comprobar si el router deja pasar conexiones IPv6 entrantes hacia la VM.
- En `pve-host`: instalar `rsyslog` y añadir los logs de `pveproxy` y fail2ban. Si los logins fallidos de la web llegan pero no generan alerta, escribir una regla propia.
- Agentes en el CT 100 y el CT 101 (`pct enter`), y el syslog del router hacia Wazuh.
