# [[OpenClaw CLI - GLM Quick Check]]

## Objetivo

Validar rapido desde VPS que OpenClaw sigue usando `zai/glm-4.7` y que el alias `glm` funciona.

## Comandos rapidos

```bash
CNAME="openclaw-hacw-openclaw-1"

docker exec -i "$CNAME" openclaw config get agents.defaults.model
docker exec -i "$CNAME" openclaw config get agents.defaults.models
docker exec -i "$CNAME" openclaw config get agents.list[0].model
docker exec -i "$CNAME" openclaw models list
```

## Prueba interactiva

```bash
docker exec -it "$CNAME" openclaw
```

Dentro del chat:

```text
/model list
/model glm
```

## Esperado

- `primary` en `zai/glm-4.7`.
- alias `glm` visible en modelos.
- cambio de modelo exitoso con `/model glm`.

## Troubleshooting corto

- `unknown command 'model'`: usar `openclaw models list` (plural).
- error de parser en UI: evitar pegar JS sin comillas; usar `openclaw config set`.
- no encuentra contenedor: verificar nombre real con `docker ps --format '{{.Names}}'`.
