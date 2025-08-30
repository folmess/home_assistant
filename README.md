# HA_pi

Configuración de Home Assistant.

![HA Config Check](https://github.com/folmess/home_assistant/actions/workflows/ha-check.yml/badge.svg)

## Estructura
- `configuration.yaml` y `packages/` para dividir por tema.
- `automations/`, `scripts/`, `scenes/`, `templates/`.


- La acción usa `.HA_VERSION` si existe, o `stable` por defecto.
- Archivo de secretos falso: `fakesecrets.yaml` (no contiene credenciales reales).
para que la validación pase sin exponer datos.



push git

Si hay cambios:

Los commitea con fecha/hora y lista de archivos modificados.

Hace git push origin main.

Crea una notificación en HA con el resultado.

Si no hay cambios: notificación “No hay cambios para subir”.

Si falla el push: notificación de error.
