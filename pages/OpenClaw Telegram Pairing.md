- # OpenClaw Telegram Pairing (VPS + Docker)

- estado:: activo
- actualizado:: 2026-02-24

- ## Objetivo
- Dejar Telegram operativo en OpenClaw cuando aparecen errores `pairing required` o `acceso no configurado`.

- ## Contexto de este servidor
- SSH: `ssh vps-openclaw`
- Contenedor: `openclaw-hacw-openclaw-1`
- Bot activo: `@w421claw_bot`

- ## Flujo correcto
- 1) Entrar por SSH al VPS.
- `ssh vps-openclaw`

- 2) Confirmar contenedor real.
- `docker ps`

- 3) Ver solicitudes pendientes de Telegram.
- `docker exec -it openclaw-hacw-openclaw-1 openclaw pairing list --channel telegram`

- 4) Aprobar el codigo de pairing.
- `docker exec -it openclaw-hacw-openclaw-1 openclaw pairing approve --channel telegram <CODIGO>`

- 5) Si sigue "acceso no configurado", fijar allowlist explicita para DM.
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

- 6) Reiniciar y probar.
- `sudo docker compose -f /docker/openclaw-hacw/docker-compose.yml restart`
- En Telegram enviar `/start` a `@w421claw_bot`.

- 7) Si se cambio `.env` y no toma el token nuevo, recrear contenedor.
- `sudo docker compose --env-file /docker/openclaw-hacw/.env -f /docker/openclaw-hacw/docker-compose.yml up -d --force-recreate`

- 8) Verificar token Telegram en runtime.
- `docker exec -it openclaw-hacw-openclaw-1 sh -lc 'curl -s "https://api.telegram.org/bot$TELEGRAM_BOT_TOKEN/getMe"'`
- Exito esperado: `"ok":true`.

- ## Diagnostico rapido
- Si aparece `Channel required`: falta `--channel telegram`.
- Si aparece `No such container`: el nombre de contenedor no coincide con `docker ps`.
- Si aparece `command not found: openclaw`: el comando se ejecuto fuera de `docker exec`.
- Si el codigo falla: generar codigo nuevo (expiran rapido) y aprobar de inmediato.
- Si `getMe` responde `401/404`: token Telegram invalido o no recargado; revisar `.env` y recrear contenedor.

- ## Seguridad posterior
- Rotar credenciales expuestas y volver a `allowInsecureAuth=false` tras recuperar acceso.
