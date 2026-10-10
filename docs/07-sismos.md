# 7. Monitor de sismos de Panamá (CT 101)

Objetivo: un contenedor pequeño que revise cada minuto la página de sismos del [Instituto de Geociencias de la UP](https://sismosgeociencias.up.ac.pa/) y, por cada sismo nuevo, avise al celular (ntfy) y lo anuncie en voz alta por Alexa (Voice Monkey). También sirve un mapa web con los sismos del día.

Código: repo privado `TheAndy509/sismos-panama` (Python, solo biblioteca estándar).

## Contenedor

Creado desde la Shell del host:

```sh
pveam update
pveam download local debian-13-standard_13.6-1_amd64.tar.zst

pct create 101 local:vztmpl/debian-13-standard_13.6-1_amd64.tar.zst \
  --hostname sismos --cores 1 --memory 512 --swap 512 --rootfs local-lvm:4 \
  --net0 name=eth0,bridge=vmbr0,firewall=1,ip=192.168.1.51/24,gw=192.168.1.1 \
  --unprivileged 1 --features nesting=1 --onboot 1 \
  --ssh-public-keys /root/pc.pub --timezone America/Panama
pct start 101
```

| Ajuste | Por qué |
|---|---|
| Debian 13 | Es la base de Proxmox 9 y trae Python 3.13 |
| 512 MB de RAM | El script dura menos de un segundo cada minuto; el servidor solo tiene 4 GB |
| Sin privilegios | Si alguien rompe el contenedor, no es root en el host |
| IP fija `.51` | Junto a Nextcloud (`.50`); se comprobó antes con `ping` que estaba libre |
| Solo llave SSH | Se creó sin contraseña de root; se entra con la llave de la PC o con `pct enter 101` |
| `onboot` | Arranca solo si se reinicia el servidor |

## Dentro del contenedor

```sh
apt update && apt install -y python3 tzdata cron curl
mkdir -p /opt/sismos-panama   # se copian sismos.py y plantilla.html con scp
python3 /opt/sismos-panama/sismos.py --test   # ok
python3 /opt/sismos-panama/sismos.py          # 1ª vez: marca los sismos actuales como vistos, sin avisar
```

Los tokens van en `/etc/sismos.env` (permisos `600`, solo root), nunca en el repo:

```sh
NTFY_TOPIC=…
VM_TOKEN=…
VM_DEVICE=…
```

Cron de root:

```
* * * * * flock -n /run/sismos.lock sh -c "set -a; . /etc/sismos.env; python3 /opt/sismos-panama/sismos.py" >> /var/log/sismos.log 2>&1
```

Mapa web como servicio `sismos-web` (puerto 8080). Usa `DynamicUser=yes`, así que el servidor web corre con un usuario temporal sin privilegios y no como root:

```ini
[Service]
ExecStart=/usr/bin/python3 -m http.server 8080 -d /opt/sismos-panama/web
Restart=always
DynamicUser=yes
```

## Carga sobre el sitio de la UP

Revisar cada minuto no satura al servidor de Geociencias: el script descarga la página (~500 KB) solo cuando cambió, usando `ETag`. Si no hay cambios, el servidor responde `304` con 0 bytes.

## Acceso

| Servicio | Dirección |
|---|---|
| Mapa de sismos | `http://192.168.1.51:8080` |

Por Tailscale (fase 6) el mapa también se ve fuera de casa, porque el host ya anuncia toda la LAN.

## Prueba

- `curl http://192.168.1.51:8080` → HTTP 200 ✅
- Primera ejecución contra el sitio real: 200 sismos marcados como vistos ✅
- Aviso de prueba por ntfy: HTTP 200 ✅
- Aviso real (último sismo, M3.8 a 46 km al sur de Santiago) enviado al celular y a Alexa; Voice Monkey respondió `{"success":true}` ✅

## Voice Monkey

- Se usa la **v3** (`app.voicemonkey.io`, API `https://api-v3.voicemonkey.io/announce` con el parámetro `speech`). La consola vieja (`console.voicemonkey.io`) es v2 y ya no crea cuentas.
- Skill vinculada en el marketplace **US**. Se creó un dispositivo tipo **Speaker** y, en la app de Alexa, una rutina: *Casa inteligente → Alexa Voice Monkey v3 → VM Speakers → sismos* → acción *Skills → Voice Monkey*, desde el Echo que habla.
