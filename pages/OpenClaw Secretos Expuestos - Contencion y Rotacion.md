- # OpenClaw Secretos Expuestos - Contencion y Rotacion

- estado:: activo
- actualizado:: 2026-02-24

- ## Respuesta corta
- Si, los secretos quedaron expuestos en esta sesion (salieron en consola y en chat).
- No se puede "desexponer" un secreto ya mostrado; la accion correcta es rotarlo y revocar el anterior.

- ## Que estuvo expuesto
- `OPENCLAW_GATEWAY_TOKEN`
- `TELEGRAM_BOT_TOKEN`
- `NEXOS_API_KEY` (si se mantiene, tratarla como comprometida)

- ## Se puede borrar de la memoria
- En terminal local: se puede limpiar historial y archivos propios.
- En servicios externos (chat/plataformas): no hay garantia de borrado retroactivo inmediato.
- Regla operativa: asumir compromiso y rotar.

- ## Plan de contencion (orden recomendado)
- 1) Rotar secretos primero.
- 2) Reiniciar servicios para aplicar nuevos valores.
- 3) Validar acceso con credenciales nuevas.
- 4) Limpiar rastros locales (historial y backups con secretos viejos).

- ## Paso A - Rotar `OPENCLAW_GATEWAY_TOKEN` (SSH en VPS)
- Entrar: `ssh vps-openclaw`
- Backup controlado: `sudo cp /docker/openclaw-hacw/.env /docker/openclaw-hacw/.env.bak.$(date +%Y%m%d_%H%M%S)`
- Generar token nuevo: `NEW_GATEWAY_TOKEN=$(openssl rand -hex 24)`
- Aplicar en `.env`: `sudo sed -i "s|^OPENCLAW_GATEWAY_TOKEN=.*|OPENCLAW_GATEWAY_TOKEN=${NEW_GATEWAY_TOKEN}|" /docker/openclaw-hacw/.env`
- Mostrar solo el nuevo token para login inicial: `echo "$NEW_GATEWAY_TOKEN"`

- ## Paso B - Rotar `TELEGRAM_BOT_TOKEN`
- En Telegram con `@BotFather`: revocar/regenerar token del bot.
- Aplicar en `.env`: `sudo sed -i "s|^TELEGRAM_BOT_TOKEN=.*|TELEGRAM_BOT_TOKEN=PEGA_TOKEN_NUEVO|" /docker/openclaw-hacw/.env`

- ## Paso C - (Opcional pero recomendado) Rotar `NEXOS_API_KEY`
- Generar nueva key en panel Nexos.
- Aplicar en `.env`: `sudo sed -i "s|^NEXOS_API_KEY=.*|NEXOS_API_KEY=PEGA_KEY_NUEVA|" /docker/openclaw-hacw/.env`

- ## Paso D - Volver a modo seguro y reiniciar
- Forzar auth segura en OpenClaw: `docker exec -i openclaw-hacw-openclaw-1 python3 - <<'PY'`
- `import json`
- `p="/data/.openclaw/openclaw.json"`
- `d=json.load(open(p))`
- `d.setdefault("gateway",{}).setdefault("controlUi",{})["allowInsecureAuth"]=False`
- `json.dump(d, open(p,"w"), indent=2)`
- `print("allowInsecureAuth=false")`
- `PY`
- Reiniciar: `sudo docker compose -f /docker/openclaw-hacw/docker-compose.yml restart`

- ## Paso E - Verificacion
- `docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"`
- `curl -I http://127.0.0.1:57086 | head -n 1`
- `sudo tailscale serve status`
- `docker logs --tail 60 openclaw-hacw-openclaw-1`

- ## Paso F - Higiene de rastros locales
- Bash history (usuario actual): `history -c && history -w`
- Revisar si hay backups sensibles en `/docker/openclaw-hacw/` (`.env.bak.*`, `openclaw.json.bak*`).
- Mantener solo backups necesarios y moverlos a almacenamiento cifrado.

- ## Criterio de cierre
- Login web funciona con token nuevo.
- Telegram responde con token nuevo.
- `allowInsecureAuth=false` confirmado.
- Secretos viejos revocados.
