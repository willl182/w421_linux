# Plan: Recuperar SSH + Gmail realtime con Funnel

**Created**: 2026-02-24 08:41
**Status**: in_progress
**Slug**: ssh-gmail-funnel

## Objetivo

Recuperar acceso operativo por `ssh vps-openclaw` y dejar Gmail en OpenClaw con notificaciones en tiempo real usando endpoint publico por Tailscale Funnel.

## Fases

### Fase 1: Recuperar SSH operativo por alias (tailnet)

**Explicacion**

El alias `vps-openclaw` apunta a la IP publica y hoy falla por timeout en puerto 22. Para volver a operar ya mismo, el alias se debe mover a Tailscale (MagicDNS), donde el nodo esta activo.

**Resultado esperado**

- `ssh vps-openclaw` conecta al VPS por tailnet.
- Se mantiene hardening actual (sin abrir SSH global en internet).

**Prompts/comandos**

```bash
ssh adminops@srv1408623
```

```sshconfig
Host vps-openclaw
    HostName srv1408623
    User adminops
```

```bash
ssh vps-openclaw
```

### Fase 2: Verificacion base en VPS antes de Gmail

**Explicacion**

Confirmar que OpenClaw y skill `gog` estan disponibles para evitar mezclar errores de infraestructura con errores de OAuth/webhook.

**Resultado esperado**

- Contenedor gateway arriba.
- Skill `gog` detectado.

**Prompts/comandos**

```bash
docker ps
docker exec -it openclaw-hacw-openclaw-1 openclaw skills list
tailscale status
```

### Fase 3: OAuth Gmail

**Explicacion**

Iniciar login desde VPS, abrir URL en navegador local, autorizar y pegar codigo/callback en la terminal del VPS.

Antes del login, `gog` requiere credenciales OAuth de Google Cloud (archivo JSON de cliente OAuth tipo Desktop app).

**Resultado esperado**

- Autenticacion Gmail completada para `gog`.

**Estado actual**

- Bloqueado temporalmente: faltan credenciales OAuth (`OAuth client ID JSON`).
- Error observado: `OAuth client credentials missing ... run: gog auth credentials <credentials.json>`.
- Credenciales ya cargadas en contenedor; nuevo bloqueo: `Error 403: access_denied` por app en modo Testing sin tester autorizado.

**Prompts/comandos**

```bash
docker exec -it openclaw-hacw-openclaw-1 gog auth credentials /ruta/a/credentials.json
docker exec -it openclaw-hacw-openclaw-1 gog login rafiki182.claw@gmail.com
```

```bash
docker exec -it openclaw-hacw-openclaw-1 gog login
```

### Fase 4: Exponer endpoint publico con Funnel

**Explicacion**

Google Pub/Sub push requiere URL HTTPS publica; `tailscale serve` privado no alcanza para notificacion realtime.

**Resultado esperado**

- Funnel activo y URL publica funcional.

**Estado actual**

- Nodo en `tailnet only` (aun sin Funnel publico).
- Para activar Funnel en esta tailnet se requiere habilitarlo en admin console y aplicar con privilegios:
  `sudo tailscale funnel --bg http://127.0.0.1:57086`.

**Prompts/comandos**

```bash
sudo tailscale funnel --bg http://127.0.0.1:57086
sudo tailscale funnel status
```

### Fase 5: Setup webhook Gmail en OpenClaw

**Explicacion**

Crear watch de Gmail y asociar entrega de eventos a OpenClaw via endpoint publico.

**Resultado esperado**

- `watch` activo para la cuenta.
- Eventos de correo entrante procesados automaticamente.

**Prompts/comandos**

```bash
docker exec -it openclaw-hacw-openclaw-1 openclaw webhooks gmail setup --account tu-email@gmail.com
```

### Fase 6: Validacion end-to-end realtime

**Explicacion**

Enviar correo de prueba desde otra cuenta y verificar llegada automatica sin consulta manual.

**Resultado esperado**

- Notificacion visible en OpenClaw/Telegram en segundos.

**Prompts/comandos**

```bash
docker logs --since=10m openclaw-hacw-openclaw-1
```

## Log de Ejecucion

- [x] Plan creado y guardado en `logs/plans/`.
- [x] Fase 1 iniciada.
- [x] Fase 1 completada.
- [x] Fase 2 iniciada.
- [x] Fase 2 completada.
- [x] Fase 3 iniciada.
- [x] Fase 3 completada.
- [x] Fase 4 iniciada.
- [x] Fase 4 completada.
- [x] Fase 5 iniciada.
- [x] Fase 5 completada.
- [x] Fase 6 iniciada.
- [ ] Fase 6 completada.
