# [[Linux AI Hub]]

- estado:: activo
- foco:: dejar OpenClaw privado, estable y operable en VPS Hostinger
- ultimo-check:: 2026-02-24

## [[OpenClaw - Configuracion estable 2026-02-23]]

- **Objetivo**: dejar OpenClaw funcionando sin exposicion publica innecesaria.
- **Resultado validado**:
  - OpenClaw arriba en Docker.
  - Puerto atado a localhost: `127.0.0.1:57086->57086`.
  - Acceso privado por Tailscale Serve: `https://srv1408623.tail7009c7.ts.net`.
  - Auditoria: `0 critical`, `0 warn`.

### Pasos que SI funcionaron

- 1) Corregir configuracion de OpenClaw en `openclaw.json`.
  - Clave corregida: `controlUi.allowInsecureAuth`.
  - Archivo real detectado y usado: `/docker/openclaw-hacw/data/.openclaw/openclaw.json`.
  - Explicacion: el error critico de auditoria venia de auth insegura del Control UI.

- 2) Asegurar acceso admin por SSH con llave.
  - Usuario operativo: `adminops`.
  - Login exitoso con llave privada local.
  - Explicacion: evita dependencia de password para operacion diaria.

- 3) Cerrar exposicion publica con firewall.
  - Reglas activas: `OpenSSH allow`, `57086 deny`.
  - Explicacion: aunque exista auth en app, cerrar puerto reduce superficie de ataque.

- 4) Habilitar acceso privado por Tailscale.
  - `tailscale up --ssh`.
  - `tailscale serve --bg http://127.0.0.1:57086`.
  - Explicacion: acceso privado por tailnet sin abrir panel a internet publica.

- 5) Defensa en profundidad en Docker Compose.
  - Cambio aplicado en `/docker/openclaw-hacw/docker-compose.yml`.
  - De `0.0.0.0` a `127.0.0.1` para el puerto `57086`.
  - Explicacion: incluso si cambia algo en firewall, el servicio sigue local-only.

### Verificacion canonica

- `docker ps`
- `sudo tailscale serve status`
- `docker exec -it openclaw-hacw-openclaw-1 openclaw security audit`

## [[Hostinger VPS - Operacion base]]

- **Entrar por SSH**:
  - `ssh vps-openclaw`
  - fallback tailnet: `ssh adminops@srv1408623`
  - nota: acceso por IP publica puede dar timeout por UFW/allowlist de IP.
- **Reinicio controlado**:
  - `sudo reboot`
- **Checks post-reinicio**:
  - `docker ps`
  - `sudo tailscale serve status`
  - `docker exec -it openclaw-hacw-openclaw-1 openclaw security audit`

## [[Tailscale - Acceso privado]]

- **Serve**: publica un servicio dentro de la tailnet.
- **Funnel**: expone a internet publica.
- Decision actual: mantener UI principal en `tailnet only` (`/` -> `127.0.0.1:57086`) y publicar solo webhook Gmail en `/gmail-pubsub`.

## [[OpenClaw + Gmail realtime - Webhook con Gog y Funnel]]

- **Objetivo**: recibir correos de Gmail en tiempo real (push) sin consulta manual.
- **Cuenta dedicada**: `rafiki182.claw@gmail.com`.
- **Estado validado**:
  - OAuth `gog` operativo con keyring file.
  - `gcloud` autenticado en proyecto `hypnotic-camp-488413-c6`.
  - Funnel activo en path dedicado: `https://srv1408623.tail7009c7.ts.net/gmail-pubsub`.
  - `openclaw webhooks gmail setup` completado (topic/subscription/push endpoint).
  - `openclaw webhooks gmail run` escuchando en `0.0.0.0:8788/gmail-pubsub`.

### Comandos que SI funcionaron

- 1) Corregir alias SSH a tailnet.
  - `~/.ssh/config` -> `HostName srv1408623`.
  - Explicacion: por IP publica `187.77.213.213:22` habia timeout por reglas/UFW.

- 2) Cargar credenciales OAuth de Google para `gog`.
  - `docker cp /home/adminops/credentials.json openclaw-hacw-openclaw-1:/home/w182/.config/gogcli/credentials.json`
  - `docker exec -it openclaw-hacw-openclaw-1 gog auth credentials /home/w182/.config/gogcli/credentials.json`

- 3) Login headless estable (remote flow).
  - `docker exec -it openclaw-hacw-openclaw-1 gog login rafiki182.claw@gmail.com --remote --step=1`
  - `docker exec -it openclaw-hacw-openclaw-1 gog login rafiki182.claw@gmail.com --remote --step=2 --auth-url '<callback-url>'`

- 4) Forzar keyring usable en servidor.
  - `docker exec -it openclaw-hacw-openclaw-1 gog auth keyring file`
  - En shell VPS: `export GOG_KEYRING_PASSWORD='<clave-fija>'`

- 5) Instalar dependencias faltantes en contenedor para setup Gmail.
  - `docker exec -it openclaw-hacw-openclaw-1 /usr/bin/apt-get update`
  - `docker exec -it openclaw-hacw-openclaw-1 /usr/bin/apt-get install -y gnupg ca-certificates`
  - Agregar repo de Google Cloud + instalar `google-cloud-cli`.
  - `docker exec -it openclaw-hacw-openclaw-1 gcloud auth login --no-launch-browser`
  - `docker exec -it openclaw-hacw-openclaw-1 gcloud config set project hypnotic-camp-488413-c6`

- 6) Publicar endpoint push dedicado por Funnel.
  - `tailscale funnel --bg --set-path /gmail-pubsub http://172.18.0.2:8788`
  - Validar: `tailscale funnel status`.

- 7) Configurar watch + Pub/Sub + hook.
  - `docker exec -e GOG_KEYRING_PASSWORD="$GOG_KEYRING_PASSWORD" -it openclaw-hacw-openclaw-1 openclaw webhooks gmail setup --account rafiki182.claw@gmail.com --project hypnotic-camp-488413-c6 --tailscale off --push-endpoint https://srv1408623.tail7009c7.ts.net/gmail-pubsub --json`

- 8) Ejecutar runner Gmail en background.
  - `docker exec -d -e GOG_KEYRING_PASSWORD="$GOG_KEYRING_PASSWORD" openclaw-hacw-openclaw-1 sh -lc 'openclaw webhooks gmail run --account rafiki182.claw@gmail.com --tailscale off --bind 0.0.0.0 --port 8788 --path /gmail-pubsub >> /tmp/gmail-webhook-run.log 2>&1'`

### Errores reales y aprendizajes

- `ssh vps-openclaw` timeout a IP publica.
  - Aprendizaje: usar tailnet para operacion (`srv1408623`) y mantener hardening por IP publica.

- `gog login` -> `expected "<email>"`.
  - Aprendizaje: el comando requiere email explicito.

- `OAuth client credentials missing`.
  - Aprendizaje: sin `credentials.json` (OAuth Desktop app) no inicia auth.

- `Error 403 access_denied` en Google.
  - Aprendizaje: app en modo Testing exige agregar cuenta en `Test users`.

- callback localhost fallando en VPS headless.
  - Aprendizaje: usar `--remote --step=1/2` y pegar callback URL manual.

- `manual auth state missing`.
  - Aprendizaje: step 2 solo sirve con callback de un step 1 vigente (sin expirar ni mezclar sesiones).

- `no TTY available for keyring file backend password prompt`.
  - Aprendizaje: siempre exportar `GOG_KEYRING_PASSWORD` en entorno no interactivo.

- `aes.KeyUnwrap(): integrity check failed`.
  - Aprendizaje: token cifrado con otra clave; resetear keyring y repetir login con una sola clave fija.

- `gcloud not installed` durante setup.
  - Aprendizaje: instalar `google-cloud-cli` dentro del contenedor OpenClaw.

- `tailscale not installed` dentro del contenedor.
  - Aprendizaje: usar `--tailscale off` en setup/run y publicar endpoint desde host con `tailscale funnel`.

- `push endpoint required` con `--tailscale off`.
  - Aprendizaje: pasar `--push-endpoint` explicito.

- `Access denied: serve config denied` en funnel.
  - Aprendizaje: ejecutar con `sudo` o definir operador con `sudo tailscale set --operator=adminops`.

- `HTTP 502` inicial en `/gmail-pubsub`.
  - Aprendizaje: ocurre si el runner no esta arriba; tras levantar `openclaw webhooks gmail run` pasa a respuesta del endpoint (404/405 segun metodo).

### Persistencia post-reboot aplicada

- Script VPS instalado: `/usr/local/bin/openclaw-postboot-fix.sh`.
- Servicio systemd habilitado: `/etc/systemd/system/openclaw-postboot-fix.service`.
- Objetivo del fix: evitar que el bootstrap Hostinger deje `hooks.token` igual a `gateway.auth.token` en cada arranque.
- Variables persistidas en compose env: `GOG_KEYRING_PASSWORD` y `OPENCLAW_HOOKS_TOKEN` en `/docker/openclaw-hacw/.env`.
- Validacion operativa: gateway arriba (`ws://127.0.0.1:18789`), watcher Gmail activo, push sintetico `/gmail-pubsub` devuelve `200`.

## [[Lecciones aprendidas - OpenClaw VPS]]

- **Error**: editar ruta equivocada de config.
  - **Aprendizaje**: siempre confirmar mounts del contenedor con `docker inspect`.
- **Error**: confundir password del usuario, passphrase de llave y token/app key.
  - **Aprendizaje**: separar credenciales por uso:
    - password de usuario (sudo/login si aplica)
    - passphrase de llave SSH (protege llave privada)
    - token de app (auth de OpenClaw)
- **Error**: usar sintaxis antigua de `tailscale serve`.
  - **Aprendizaje**: usar `tailscale serve --bg ...` y verificar con `tailscale serve status`.
- **Error**: permisos Docker denegados para `adminops`.
  - **Aprendizaje**: usar `sudo` para operaciones Docker o agregar grupo `docker`.
- **Error**: interpretar "offline" del portal como servicio caido.
  - **Aprendizaje**: validar siempre con checks locales reales (`curl`, `docker ps`, audit).
- **Error**: asumir que `docker compose restart` siempre aplica cambios de `.env`.
  - **Aprendizaje**: cuando hay dudas de variables, recrear con `docker compose ... up -d --force-recreate`.
- **Error**: diagnosticar Telegram sin validar API directa.
  - **Aprendizaje**: confirmar token con `curl https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getMe` (`ok: true`).
- **Error**: asumir que un override tardio en `sshd_config.d` siempre aplica.
  - **Aprendizaje**: en este host prevalecio `50-cloud-init.conf` con `PasswordAuthentication yes`; validar siempre con `sshd -T`.

## [[Seguridad - Respuesta a secretos expuestos]]

- Runbook dedicado: `[[OpenClaw Secretos Expuestos - Contencion y Rotacion]]`.
- Principio clave: secreto expuesto no se "borra", se rota y se revoca.

## [[Hostinger VPS Hardening - SSH y UFW]]

- Estado actual: SSH endurecido y verificado en runtime.
- `passwordauthentication no`, `permitrootlogin no`, `pubkeyauthentication yes`, `kbdinteractiveauthentication no`.
- UFW sin reglas globales `OpenSSH Anywhere`; acceso SSH permitido por IP especifica.
- Persistencia cloud-init aplicada con `ssh_pwauth: false`.
- `fail2ban` activo y validado (`pong`, jail `sshd` operativa).
- Ajuste aplicado en jail SSH: `backend=systemd`, `maxretry=5`, `findtime=10m`, `bantime=1h`, `ignoreip` con localhost + tailnet + IP operativa.

## [[Plan Git para este workspace]]

- 1) Preparar `logseq.md` con estado final (hecho).
- 2) `git init` en `@w421/linux`.
- 3) `git add .` y primer commit.
- 4) Crear repo remoto vacio (sin README idealmente).
- 5) `git remote add origin <URL>`.
- 6) `git push -u origin main`.
- 7) Segundo commit para mejoras/aprendizajes incrementales.
- 8) Cierre con registro `saver` (plan + estado sesion).

## [[Comandos canonicos]]

- SSH: `ssh vps-openclaw`
- Estado contenedor: `docker ps`
- Estado Tailscale Serve: `sudo tailscale serve status`
- Estado Tailscale Funnel: `tailscale funnel status`
- Auditoria OpenClaw: `docker exec -it openclaw-hacw-openclaw-1 openclaw security audit`
- Compose up: `sudo docker compose up -d`

## [[OpenClaw + Telegram - Pairing desde VPS (Docker)]]

- **Objetivo**: aprobar el acceso de Telegram correctamente cuando sale `pairing required` o `acceso no configurado`.
- **Contexto de este VPS**:
  - SSH: `ssh vps-openclaw`
  - Contenedor real: `openclaw-hacw-openclaw-1`
  - Bot Telegram: `@wclaw421_bot`

### Pasos operativos

- 1) Entrar al VPS por SSH (no usar consola web para este flujo).
  - `ssh vps-openclaw`

- 2) Confirmar nombre real del contenedor.
  - `docker ps`
  - Verificar que exista `openclaw-hacw-openclaw-1`.

- 3) Listar solicitudes de pairing pendientes en Telegram.
  - `docker exec -it openclaw-hacw-openclaw-1 openclaw pairing list --channel telegram`

- 4) Aprobar el codigo pendiente.
  - `docker exec -it openclaw-hacw-openclaw-1 openclaw pairing approve --channel telegram <CODIGO>`
  - Exito esperado: `Approved telegram sender <id>`.

- 5) Si sigue saliendo "acceso no configurado", forzar allowlist del ID autorizado.
  - `docker exec -i openclaw-hacw-openclaw-1 python3 - <<'PY'`
  - `import json`
  - `p="/data/.openclaw/openclaw.json"`
  - `d=json.load(open(p))`
  - `tg=d.setdefault("channels",{}).setdefault("telegram",{})`
  - `tg["dmPolicy"]="allowlist"`
  - `ids=tg.get("allowFrom",[])`
  - `if 5570879165 not in ids: ids.append(5570879165)`
  - `tg["allowFrom"]=ids`
  - `json.dump(d, open(p,"w"), indent=2)`
  - `print("OK", tg["dmPolicy"], tg["allowFrom"])`
  - `PY`

- 6) Reiniciar OpenClaw y probar.
  - `sudo docker compose -f /docker/openclaw-hacw/docker-compose.yml restart`
  - En Telegram enviar `/start` a `@wclaw421_bot`.

### Errores comunes y solucion

- `command not found: openclaw`
  - Causa: comando ejecutado fuera del contenedor.
  - Solucion: usar `docker exec -it openclaw-hacw-openclaw-1 ...`.

- `No such container`
  - Causa: nombre de contenedor incorrecto.
  - Solucion: consultar `docker ps` y usar el nombre exacto.

- `Channel required`
  - Causa: CLI exige canal explicito.
  - Solucion: usar `--channel telegram`.

- Pairing no aparece o no aprueba
  - Causa: codigo expirado.
  - Solucion: enviar mensaje nuevo al bot, listar de nuevo, aprobar en menos de 1 minuto.
