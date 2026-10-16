# Fase 8: Endurecimiento del host Proxmox

Se hace por partes, de la más simple a la que más riesgo tiene de dejarte sin acceso: SSH con llave → firewall → 2FA → fail2ban.

## 1. SSH solo con llave

La llave `ed25519` de la PC ya estaba instalada en el host desde la fase 7. Quedaba desactivar el login por contraseña.

En vez de editar `/etc/ssh/sshd_config`, se añadió un archivo en `sshd_config.d/`. La línea `Include` está al principio del archivo principal y en sshd **gana el primer valor que aparece**, así que este archivo tiene prioridad sobre `PermitRootLogin yes`:

```sh
cat > /etc/ssh/sshd_config.d/00-solo-llave.conf <<'EOF'
PermitRootLogin prohibit-password
PasswordAuthentication no
KbdInteractiveAuthentication no
EOF
sshd -t && systemctl reload ssh
```

`sshd -t` valida la sintaxis antes de recargar. Si hay un error, el servicio no se toca. `reload` no corta las sesiones que ya están abiertas.

### Qué no cambia

- La **web de Proxmox (8006)** usa su propio login, no SSH. Se entra con usuario y contraseña igual que antes.
- La **Shell del nodo** en la web tampoco pasa por SSH. Si se pierde la llave, se puede revertir desde ahí borrando el archivo.

### Prueba

- `sshd -T` → `permitrootlogin without-password`, `passwordauthentication no` ✅
- Conexión nueva con llave → entra ✅
- Conexión forzando contraseña → `Permission denied (publickey)` ✅
- Web 8006 → HTTP 200 ✅

## 2. Usuario administrador propio y 2FA

Para el uso diario se creó un usuario en el realm **`pve`** (Proxmox VE authentication server), no en PAM:

- Con `pve`, la contraseña la guarda Proxmox y el usuario **no existe en Linux**, así que no puede entrar al host por SSH.
- Con PAM habría que crear antes el usuario con `adduser` y además podría entrar por SSH.

Pasos:

1. *Datacenter → Permissions → Users → Add*, realm *Proxmox VE authentication server*, con contraseña.
2. *Datacenter → Permissions → Add → User Permission*: path `/`, rol `Administrator`, *Propagate* marcado. El botón *Permissions* de la lista de usuarios solo **muestra** permisos, no los asigna.
3. 2FA **TOTP** (app de autenticación) para el usuario nuevo **y** para `root@pam`. `root@pam` se conserva como acceso de emergencia, no se borra.

### Prueba

- `pveum acl list` → `/`, `Administrator`, propagate 1 ✅
- `pveum user tfa list` → TOTP en los dos usuarios ✅
- Login del usuario nuevo con código TOTP → OK ✅

### Si se pierde el celular

Desde SSH con la llave (paso 1): `pveum user tfa delete <usuario>@<realm>` y se vuelve a configurar.
