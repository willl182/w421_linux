# Session State: Linux AI Hub

**Last Updated**: 2026-02-24 23:20

## Session Objective

Dejar OpenClaw en modo solo Z.AI GLM 4.7 y documentar el runbook sin exponer secretos.

## Current State

- [x] Se confirmo que el alias SSH `vps-openclaw` existe y resuelve con `ssh -G`.
- [x] Se detecto contenedor activo: `openclaw-hacw-openclaw-1`.
- [x] Se aplico por CLI `agents.defaults.model.primary = zai/glm-4.7`.
- [x] Se aplico por CLI `agents.defaults.model.fallbacks = []`.
- [x] Se aplico por CLI `agents.list[0].model = zai/glm-4.7`.
- [x] Se identifico que `openclaw model list` es incorrecto; comando correcto: `openclaw models list`.
- [x] Se identifico causa de validaciones fallidas en UI: contenido pegado como objeto JS sin comillas validas de JSON/JSON5.
- [x] Se documento workaround robusto: usar `openclaw config set` en vez de editar JSON manualmente en UI.
- [ ] Pendiente cargar API key real de Z.AI (omitida deliberadamente en esta sesion).

## Critical Technical Context

- Error `invalid character 'o' at 5:21` proviene de valores string sin comillas (ej: `defaultProfile: openclaw`).
- Error al setear alias por ruta con `4.7` ocurre por parsing de paths con punto; conviene setear objeto completo de `agents.defaults.models` en un solo comando JSON.
- Ruta de config `/data/.openclaw/openclaw.json` aplica dentro del contenedor; en host depende de mounts de Docker.
- `openclaw config set ...` sobreescribe config y crea backup automatico (`openclaw.json.bak`).

## Next Steps

1. Cargar `env.ZAI_API_KEY` real por CLI en contenedor.
2. Reiniciar contenedor `openclaw-hacw-openclaw-1`.
3. Verificar con `openclaw models list` y cambio de alias `/model glm`.
