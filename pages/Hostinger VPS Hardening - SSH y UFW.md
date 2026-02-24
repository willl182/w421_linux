- # Hostinger VPS Hardening - SSH y UFW

- estado:: activo
- actualizado:: 2026-02-24

- ## Resultado actual (verificado)
- `port 22`
- `permitrootlogin no`
- `pubkeyauthentication yes`
- `passwordauthentication no`
- `kbdinteractiveauthentication no`
- UFW activo con default `deny incoming`.
- SSH restringido por IP especifica en IPv4 (sin `OpenSSH Anywhere`).

- ## Riesgo que se detecto
- El VPS estaba con `PasswordAuthentication yes`, lo que habilita intentos de fuerza bruta sobre SSH publico.
- `fail2ban` estaba inactivo.

- ## Accion aplicada (SSH)
- 1) Se creo override local:
- `sudo tee /etc/ssh/sshd_config.d/99-hardening.conf >/dev/null <<'EOF'`
- `PasswordAuthentication no`
- `PermitRootLogin no`
- `PubkeyAuthentication yes`
- `EOF`
- 2) Se valido y recargo SSH:
- `sudo sshd -t`
- `sudo systemctl reload ssh`

- ## Hallazgo importante (cloud-init)
- El valor efectivo seguia en `passwordauthentication yes` por una regla previa:
- `/etc/ssh/sshd_config.d/50-cloud-init.conf: PasswordAuthentication yes`
- En este entorno gano la primera ocurrencia cargada.
- Solucion aplicada:
- `sudo mv /etc/ssh/sshd_config.d/50-cloud-init.conf /etc/ssh/sshd_config.d/50-cloud-init.conf.disabled`
- Verificacion final:
- `sudo sshd -T | grep -E '^(port|passwordauthentication|permitrootlogin|pubkeyauthentication|kbdinteractiveauthentication) '`

- ## Persistencia para cloud-init
- Se fijo politica para futuros reprovisionados:
- `sudo tee /etc/cloud/cloud.cfg.d/99-disable-ssh-password-auth.cfg >/dev/null <<'EOF'`
- `ssh_pwauth: false`
- `EOF`

- ## Accion aplicada (UFW)
- Se mantuvo solo SSH desde IP operativa y se eliminaron reglas globales `OpenSSH Anywhere`.
- Reglas vigentes se validan con:
- `sudo ufw status numbered`
- `sudo ufw status verbose`

- ## Accion aplicada (fail2ban ajustado)
- Estado final verificado: servicio `active`, `pong`, jail `sshd` cargada.
- Comandos ejecutados:
- `sudo apt update`
- `sudo apt install -y fail2ban`
- `sudo systemctl enable --now fail2ban`
- `sudo fail2ban-client -t` (OK)
- `sudo systemctl restart fail2ban`
- `sudo fail2ban-client status`
- `sudo fail2ban-client status sshd`

- Configuracion recomendada para este host (`/etc/fail2ban/jail.d/sshd.local`):
- `[sshd]`
- ` enabled = true`
- ` backend = systemd`
- ` maxretry = 5`
- ` findtime = 10m`
- ` bantime = 1h`
- ` ignoreip = 127.0.0.1/8 ::1 100.64.0.0/10 190.60.38.51/32`

- Verificacion de salud:
- `sudo systemctl is-active fail2ban` -> `active`
- `sudo fail2ban-client ping` -> `Server replied: pong`
- `sudo fail2ban-client status` -> `Jail list: sshd`
- `sudo fail2ban-client status sshd` -> filtro/jail operativos
