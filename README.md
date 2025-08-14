# HA_pi

Configuración de Home Assistant.

![HA Config Check](https://github.com/folmess/home_assistant/actions/workflows/ha-check.yml/badge.svg)

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



push git

Si hay cambios:

Los commitea con fecha/hora y lista de archivos modificados.

Hace git push origin main.

Crea una notificación en HA con el resultado.

Si no hay cambios: notificación “No hay cambios para subir”.

Si falla el push: notificación de error.

💡 Con esto no tenés que instalar nada extra en el contenedor de HA. Solo asegurate de:

Que la URL http://localhost:8123 sea la correcta desde tu contenedor/instancia.

Que el token tenga permisos completos.

Que el repo esté inicializado y configurado con la clave SSH que ya hicimos antes.