# [[OpenClaw Docker - Plan Codex + ZAI]]

## Objetivo

Dejar **OpenAI Codex (plan ChatGPT/Codex)** y **Z.AI (GLM)** activos en la misma instalacion de OpenClaw en Docker, con:

- `zai/glm-4.7` como modelo primario.
- `openai-codex/gpt-5.3-codex` como fallback.
- alias de cambio rapido: `glm` y `codex`.

## Resultado esperado

- Login OAuth de Codex completado en el contenedor.
- `ZAI_API_KEY` guardada en configuracion.
- `openclaw.json` con `primary`, `fallbacks` y `models` definidos.
- Verificacion funcional con `/model list`, `/model codex`, `/model glm`.

## Paso 0 - Preparacion (2 minutos)

1. Entrar al VPS por SSH.
2. Confirmar contenedor real de OpenClaw.
3. Confirmar que OpenClaw CLI responde dentro del contenedor.

```bash
ssh vps-openclaw
docker ps
docker exec -it openclaw-hacw-openclaw-1 openclaw --help
```

Nota: si el nombre del contenedor no es `openclaw-hacw-openclaw-1`, usar el nombre real que salga en `docker ps`.

## Paso 1 - Autenticar OpenAI Codex (OAuth en Docker)

1. Iniciar login desde el contenedor:

```bash
docker exec -it openclaw-hacw-openclaw-1 openclaw models auth login --provider openai-codex
```

2. Copiar la URL larga que muestra la terminal (`https://auth.openai.com/...`).
3. Abrirla en el navegador de tu computadora personal.
4. Iniciar sesion en OpenAI.
5. Al final, el navegador intentara redirigir a algo como `http://127.0.0.1:1455/...` y fallara. Es normal.
6. Copiar **la URL completa final** de la barra del navegador.
7. Pegar esa URL en la terminal del VPS donde OpenClaw esta esperando.

Si el comando no funciona, usar el flujo alterno:

```bash
docker exec -it openclaw-hacw-openclaw-1 openclaw onboard
```

y seleccionar proveedor OpenAI Codex/OpenAI Code.

## Paso 2 - Configurar Z.AI (GLM)

Configurar clave API en OpenClaw:

```bash
docker exec -it openclaw-hacw-openclaw-1 openclaw config set env.ZAI_API_KEY "sk-tu-clave-zai"
```

Validar que quedo escrita (sin exponerla completa):

```bash
docker exec -it openclaw-hacw-openclaw-1 sh -lc "openclaw config get env.ZAI_API_KEY | sed 's/./*/g'"
```

## Paso 3 - Aplicar configuracion combinada (primary + fallback)

Editar `~/.openclaw/openclaw.json` dentro del contenedor para dejar este bloque:

```json
{
  "env": {
    "ZAI_API_KEY": "sk-..."
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "zai/glm-4.7",
        "fallbacks": ["openai-codex/gpt-5.3-codex"]
      },
      "models": {
        "zai/glm-4.7": { "alias": "glm" },
        "openai-codex/gpt-5.3-codex": { "alias": "codex" }
      }
    }
  }
}
```

Opciones para editar:

- `nano` dentro del contenedor.
- editar el archivo en el volumen mapeado del host.

## Paso 4 - Reinicio controlado

Reiniciar el servicio para asegurar carga limpia de configuracion:

```bash
sudo docker compose -f /docker/openclaw-hacw/docker-compose.yml restart
```

## Paso 5 - Verificacion funcional

1. Listar modelos disponibles/autenticados.
2. Cambiar a Codex por alias.
3. Volver a GLM por alias.

Comandos en chat OpenClaw:

```text
/model list
/model codex
/model glm
```

Checks tecnicos adicionales:

```bash
docker exec -it openclaw-hacw-openclaw-1 openclaw security audit
docker logs --tail 200 openclaw-hacw-openclaw-1
```

## Criterio de exito

Se considera implementado cuando:

1. `openai-codex` aparece autenticado.
2. `zai/glm-4.7` responde como modelo primario.
3. El cambio de modelo por alias funciona en la misma sesion.
4. No hay errores criticos en audit ni en logs al alternar modelos.

## Riesgos y mitigacion

- OAuth no completa: repetir paso 1 y pegar la URL final correcta (`127.0.0.1:1455/...`).
- Nombre de contenedor incorrecto: obtenerlo siempre con `docker ps`.
- Clave invalida Z.AI: reemplazar `ZAI_API_KEY` y reiniciar.
- JSON invalido: validar formato antes de reiniciar.

## Orden de ejecucion recomendado

1. Preparacion.
2. OAuth Codex.
3. Z.AI API key.
4. Config combinada.
5. Reinicio.
6. Verificacion en chat y auditoria.
