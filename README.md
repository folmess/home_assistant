# HA_pi

Configuración de Home Assistant.

## Estructura
- `configuration.yaml` y `packages/` para dividir por tema.
- `automations/`, `scripts/`, `scenes/`, `templates/`.

## CI
Este repo corre un chequeo de configuración en cada push/PR usando
[frenck/action-home-assistant](https://github.com/frenck/action-home-assistant).

- La acción usa `.HA_VERSION` si existe, o `stable` por defecto.
- Archivo de secretos falso: `fakesecrets.yaml` (no contiene credenciales reales).

## Secretos
Definí **las mismas claves** en `fakesecrets.yaml` que en tu `secrets.yaml`
para que la validación pase sin exponer datos.
