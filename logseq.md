# [[Linux AI Hub]]

- estado:: activo
- foco:: dejar OpenClaw privado, estable y operable en VPS Hostinger
- ultimo-check:: 2026-02-23

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
  - fallback: `ssh -i ~/.ssh/hostinger-srv1408623 adminops@187.77.213.213`
- **Reinicio controlado**:
  - `sudo reboot`
- **Checks post-reinicio**:
  - `docker ps`
  - `sudo tailscale serve status`
  - `docker exec -it openclaw-hacw-openclaw-1 openclaw security audit`

## [[Tailscale - Acceso privado]]

- **Serve**: publica un servicio dentro de la tailnet.
- **Funnel**: expone a internet publica (no usado en este caso).
- Decision tomada: sin Funnel, solo `tailnet only`.

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
- Auditoria OpenClaw: `docker exec -it openclaw-hacw-openclaw-1 openclaw security audit`
- Compose up: `sudo docker compose up -d`
