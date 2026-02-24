# Session State: Linux AI Hub

**Last Updated**: 2026-02-24 16:48

## Session Objective

Dejar Gmail push realtime a Telegram estable en OpenClaw (VPS Docker) y persistir hallazgos de troubleshooting.

## Current State

- [x] Alias SSH `vps-openclaw` recuperado por tailnet (`HostName srv1408623`).
- [x] Credenciales OAuth de Google cargadas para `gog` en el contenedor.
- [x] Login Gmail completado para `rafiki182.claw@gmail.com` con flujo remoto headless.
- [x] `gcloud` instalado/autenticado en contenedor y proyecto `hypnotic-camp-488413-c6` activo.
- [x] Funnel habilitado con path dedicado `/gmail-pubsub`.
- [x] `openclaw webhooks gmail setup` completado con `--push-endpoint` explicito.
- [x] Runner `openclaw webhooks gmail run` levantado y escuchando en `0.0.0.0:8788/gmail-pubsub`.
- [x] Bloqueo `pairing required` resuelto para announces de hook (pairing/scopes/permisos).
- [x] Keyring `gog` desbloqueado con `GOG_KEYRING_PASSWORD`; watcher Gmail arranca sin `no TTY available` en runtime manual.
- [x] `GOG_KEYRING_PASSWORD` persistido en `/docker/openclaw-hacw/.env` via sudo.
- [x] `OPENCLAW_HOOKS_TOKEN` persistido en `/docker/openclaw-hacw/.env`.
- [x] `docker compose up -d` ejecutado y contenedor recreado correctamente.
- [x] Script post-reboot instalado en VPS: `/usr/local/bin/openclaw-postboot-fix.sh`.
- [x] Servicio systemd habilitado: `/etc/systemd/system/openclaw-postboot-fix.service`.
- [x] Causa del bloqueo corregida en bootstrapping: se parchea `server.mjs` para usar `OPENCLAW_HOOKS_TOKEN` en lugar de forzar `hooks.token == gateway.auth.token`.
- [x] Verificado en runtime: gateway arriba (`ws://127.0.0.1:18789`), watcher Gmail arriba, push sintetico HTTP 200.
- [x] Documentacion actualizada en `logseq.md` y `journals/2026_02_24.md` con errores y aprendizajes.

## Critical Technical Context

- Entorno principal: Hostinger VPS + Docker + Tailscale Serve/Funnel.
- UI OpenClaw sigue local-only (`127.0.0.1:57086`) y se publica por tailnet.
- Endpoint publico minimo para Gmail push: `https://srv1408623.tail7009c7.ts.net/gmail-pubsub`.
- `gog` usa keyring `file`; requiere `GOG_KEYRING_PASSWORD` consistente para leer/escribir tokens.
- Estado actual: pairing/keyring/hooks resueltos; arranque estable despues de postboot fix automatizado.
- El fix postboot deja `hooks.token` distinto de `gateway.auth.token` y evita el error de validacion en reinicios.
- Seguridad SSH sigue endurecida (sin password/root) + UFW + fail2ban.

## Next Steps

1. Ejecutar prueba E2E real (email real -> Pub/Sub push -> hook -> Telegram).
2. Confirmar supervivencia del fix tras reboot completo del VPS.
3. Limpiar/rotar secretos expuestos durante troubleshooting.
4. (Opcional) reemplazar workaround por fix permanente en imagen/base de Hostinger.
