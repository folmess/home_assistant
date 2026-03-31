# Smart Climate Core — Documentación

## Qué hace

Controla el aire acondicionado de la sala (`climate.aire_acondicionado` — LG ThinQ) de forma automática basándose en:

- **Presencia**: si estás o no en casa (`person.hernan`)
- **Temperatura ambiente**: sensor configurable con offset
- **Estado de sueño**: `input_select.sleep_state`
- **Modo operativo manual**: seleccionable por el usuario

El sistema **nunca llama directamente** a servicios de `climate.*`. Toda acción pasa por scripts atómicos con reintentos y fallback a IR (`climate.aire_ir`).

---

## Arquitectura en capas

```
[Sensores de contexto]          binary_sensor.smart_climate_is_home
                                sensor.smart_climate_effective_mode
                                binary_sensor.smart_climate_action_allowed
          ↓
[Controlador]                   automation.smart_climate_controller
          ↓
[Script coordinador]            script.smart_climate_exec_apply
          ↓
[Scripts atómicos]              smart_climate_exec_power_on/off
                                smart_climate_exec_set_temperature
                                smart_climate_exec_set_cooling
```

---

## Modos efectivos (`sensor.smart_climate_effective_mode`)

| Modo | Cuándo aplica | Acción del controlador |
|---|---|---|
| `disabled` | `input_boolean.smart_climate` = off | Nada |
| `error_lock` | `input_boolean.smart_climate_error_lock` = on | Apagar (failsafe) |
| `away` | No estás en casa, sin precooling | **Apagar siempre** (ignora guardrails) |
| `away_precool` | No estás en casa + precooling activo | Enfriar al `precool_temp` |
| `precooling` | En casa + timer precooling activo | Enfriar al `precool_temp` |
| `manual_hold` | Timer `smart_climate_manual_hold` activo | No tocar (hold manual) |
| `sleeping` | Durmiendo y no modo sofá | Apagar si `allow_sleep_shutdown=on` |
| `auto` | Modo normal en casa despierto | Control por histéresis |
| `override` | Modo override seleccionado | Control por histéresis |
| `off` | Modo off seleccionado | Apagar |

---

## Settings y qué hacen

### Switches principales

| Entidad | Descripción |
|---|---|
| `input_boolean.smart_climate` | **Master switch** — apaga todo el sistema si está off |
| `input_boolean.smart_climate_debug` | Activa logs en el logbook de HA |
| `input_boolean.smart_climate_notifications` | Habilita notificaciones al usuario |
| `input_boolean.smart_climate_allow_sleep_shutdown` | Si ON, apaga el aire cuando te dormís |
| `input_boolean.smart_climate_error_lock` | Lock de fallos — bloquea todo excepto el apagado |
| `input_boolean.smart_climate_user_precool_request` | Solicitud manual de precooling (estando fuera) |

### Temperaturas

| Entidad | Default | Descripción |
|---|---|---|
| `input_number.smart_climate_target_temp_home` | 25°C | Temperatura objetivo en casa despierto |
| `input_number.smart_climate_hysteresis` | 0.5°C | Banda de histéresis para evitar oscilación |
| `input_number.smart_climate_precool_temp` | 19°C | Temperatura agresiva durante precooling |
| `input_number.smart_climate_temp_offset` | 0°C | Corrección del sensor de temperatura |

### Modo manual

| Entidad | Opciones | Descripción |
|---|---|---|
| `input_select.smart_climate_modo` | `auto / manual / override / precooling / off` | Modo seleccionado por el usuario |

> **Nota**: `manual` activa `timer.smart_climate_manual_hold` (1h 30min por defecto). Al vencer, vuelve a `auto`.

### Timers

| Timer | Duración | Propósito |
|---|---|---|
| `smart_climate_min_on` | 10 min | Tiempo mínimo encendido antes de poder apagar |
| `smart_climate_min_off` | 6 min | Tiempo mínimo apagado antes de poder encender |
| `smart_climate_manual_hold` | 1h 30min | Protege ajustes manuales del usuario |
| `smart_climate_precooling` | 30 min | Duración del ciclo de precooling |
| `smart_climate_ir_cooldown` | 5 min | Cooldown entre usos del fallback IR |

---

## Cómo funciona el apagado al salir

1. `person.hernan` pasa a `not_home` / `oficina` / `away`
2. `binary_sensor.smart_climate_is_home` → `off`
3. `sensor.smart_climate_effective_mode` → `away`
4. `automation.smart_climate_controller` se dispara y decide `desired_action = off`
5. El modo `away` **bypasea todos los guardrails** (`execute_allowed = true` incluso si `action_allowed = false`)
6. Se llama `script.smart_climate_exec_apply` con `power: "off"`
7. El script intenta apagar por LG (3 reintentos), y si falla usa IR como fallback

### Por qué puede fallar

- El AC ya estaba apagado cuando se detectó la salida → `last_action = "no action"` (correcto, no hay bug)
- `input_boolean.smart_climate` está en `off` → el script aborta al principio
- La integración LG falla 3 veces **Y** el IR está en cooldown → no hay fallback disponible
- `person.hernan` tarda en actualizarse (tracker lento) → el modo `away` llega tarde

---

## Debug paso a paso

### Sensores clave a monitorear

```
sensor.smart_climate_effective_mode    → ¿en qué modo está?
sensor.smart_climate_action_block_reason → ¿qué lo bloquea?
binary_sensor.smart_climate_action_allowed → ¿puede actuar?
input_text.smart_climate_last_decision → ¿qué decidió el controller?
input_text.smart_climate_last_action   → ¿qué ejecutó finalmente?
input_number.smart_climate_api_fail_count → ¿cuántas veces falló la API de LG?
```

### Tabla de diagnóstico

| `last_decision` | `last_action` | Diagnóstico |
|---|---|---|
| `away -> force off (failsafe)` | `apply: power=off` | ✅ Todo OK |
| `away -> force off (failsafe)` | `no action` | AC ya estaba apagado cuando se detectó la salida |
| `away -> force off (failsafe)` | `skip off: already off` | Ídem, confirmado por estado de HA |
| `away -> force off (failsafe)` | `blocked: ...` | Bug en guardrails — revisar `block_reason` |
| `disabled` | `no action` | `input_boolean.smart_climate` está OFF |
| `manual_hold` | `no action` | Timer de hold activo — el sistema no toca nada |

### Activar logs

```yaml
# Poner en ON:
input_boolean.smart_climate_debug: on
```

Los logs aparecen en **Configuración → Logbook** filtrando por `smart_climate_core`.

---

## Scripts disponibles (llamables manualmente)

```yaml
# Apagar con reintentos + fallback IR
service: script.smart_climate_exec_power_off

# Encender con reintentos + fallback IR
service: script.smart_climate_exec_power_on

# Cambiar temperatura (solo LG, 3 reintentos)
service: script.smart_climate_exec_set_temperature
data:
  temperature: 24

# Aplicar estado completo (wrapper recomendado)
service: script.smart_climate_exec_apply
data:
  power: "on"
  hvac_mode: "cool"
  temperature: 24
```

---

## Precooling manual (estando fuera)

Para enfriar la casa antes de llegar:

```yaml
service: input_boolean.turn_on
target:
  entity_id: input_boolean.smart_climate_user_precool_request
```

Esto inicia un ciclo de 30 minutos enfriando a `smart_climate_precool_temp` (19°C por defecto).  
Al terminar el timer, el aire se apaga automáticamente si seguís fuera.

---

## Entidades de solo lectura (no tocar)

```
sensor.smart_climate_temp_control           — temperatura efectiva con offset
sensor.smart_climate_effective_mode         — modo calculado automáticamente
sensor.smart_climate_action_block_reason    — razón de bloqueo activa
sensor.smart_climate_policy_summary         — resumen compacto del estado
binary_sensor.smart_climate_action_allowed  — si el controlador puede actuar
binary_sensor.smart_climate_is_home         — mirror de person.hernan
binary_sensor.smart_climate_is_sleeping_effective — sueño considerando modo sofá
```
