# [[OpenClaw Docker - Solo ZAI GLM 4.7]]

## Objetivo

Configurar OpenClaw en Docker para usar solo `zai/glm-4.7` (sin OpenAI), con alias `glm` y sin editar JSON manual en UI.

## Lo que fallo

- Pegar configuracion tipo JS en la UI (`defaultProfile: openclaw`) genero errores `JSON5: invalid character`.
- Setear alias por ruta con version (`agents.defaults.models.zai/glm-4.7.alias`) puede romperse por el punto en `4.7`.
- Se uso comando incorrecto `openclaw model list` (debe ser `models list`).

## Lo que si funciono

1. Detectar contenedor real:

```bash
docker ps --format '{{.Names}}'
```

2. Aplicar configuracion por CLI (sin tocar secretos aqui):

```bash
docker exec -i openclaw-hacw-openclaw-1 openclaw config set agents.defaults.model.primary "zai/glm-4.7"
docker exec -i openclaw-hacw-openclaw-1 openclaw config set agents.defaults.model.fallbacks "[]"
docker exec -i openclaw-hacw-openclaw-1 openclaw config set agents.list[0].model "zai/glm-4.7"
```

3. Setear alias por objeto completo para evitar conflicto con `4.7`:

```bash
docker exec -i openclaw-hacw-openclaw-1 openclaw config set agents.defaults.models '{"zai/glm-4.7":{"alias":"glm"}}'
```

4. Validar con comando correcto:

```bash
docker exec -i openclaw-hacw-openclaw-1 openclaw models list
```

5. Reiniciar para aplicar:

```bash
docker restart openclaw-hacw-openclaw-1
```

## Nota de ruta de config

- Dentro del contenedor: `/data/.openclaw/openclaw.json`.
- En host, la ruta depende de mounts (`docker inspect`).

## Checklist final

- `primary` en `zai/glm-4.7`.
- `fallbacks` en `[]`.
- Alias `glm` creado.
- `agents.list[0].model` en `zai/glm-4.7`.
- `models list` sin errores.
