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
