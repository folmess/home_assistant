# Sleep Detection Package - Documentación

## 📋 Descripción

Sistema inteligente de detección de sueño para Home Assistant que utiliza múltiples sensores para determinar automáticamente si estás durmiendo. Incluye controles manuales, sistema DND global y tiempos completamente configurables.

## 🎯 Características Principales

### ✅ Detección Automática Multi-Sensor
- **Score ponderado (0-100%)** basado en 7 sensores diferentes
- **Sensor nativo de sueño** (40% peso): `sensor.samsungs24u_sleep_confidence`
- **Horario de dormir** (15% peso): `input_boolean.horario_de_dormir` (22:00-02:00)
- **Sin movimiento** (15% peso): `binary_sensor.motion_sala`
- **Teléfono inactivo** (10% peso): `binary_sensor.samsungs24u_interactive`
- **Cargando** (10% peso): `binary_sensor.samsungs24u_is_charging`
- **DND activo** (5% peso): `sensor.samsungs24u_do_not_disturb_sensor`
- **En casa** (5% peso): `sensor.ubicacion_hernan`
- **Override manual** (40%+ bonus, hasta 06:00 AM): `input_boolean.manual_sleep_override`

### 🎮 Controles Manuales

| Control | Función | Cuándo Usar |
|---------|---------|-------------|
| `input_boolean.sleep_detection_enabled` | ON/OFF global del sistema | Desactivar completamente |
| `input_boolean.manual_sleep_override` | Forzar preparación para dormir (hasta 06:00 AM) | "Me voy a dormir ya" |
| `input_boolean.sleep_detection_paused` | Pausar detección temporalmente | "Estoy despierto pero quieto viendo TV" |
| `input_boolean.skip_sleep_detection_today` | Bloquear hasta mañana (6 AM) | "Hoy trasnocho" |
| `input_boolean.no_molestar` | DND global manual | Control directo del modo No Molestar |

### ⚙️ Tiempos Configurables

| Helper | Rango | Valor Inicial | Descripción |
|--------|-------|---------------|-------------|
| `input_number.sleep_inactivity_threshold` | 1-60 min | **5 min** | Tiempo sin actividad para entrar en light_sleep |
| `input_number.sleep_confirmation_time` | 1-120 min | **10 min** | Tiempo para confirmar deep_sleep |
| `input_number.sleep_confidence_threshold` | 0-100% | **50%** | Confianza mínima del sensor nativo |

## 🔄 Estados del Sistema

### 1. **awake** (Despierto)
- **Score:** 0-39%
- **Descripción:** Usuario activo y despierto
- **Comportamiento:** Sistema funcionando normalmente
- **Icono:** `mdi:account-alert`

### 2. **preparing_sleep** (Preparándose para dormir)
- **Score:** 40-59%
- **Descripción:** Indicios tempranos de preparación para dormir
- **Comportamiento:** Transición suave hacia el sueño
- **Icono:** `mdi:bed-clock`
- **Trigger:** Score >= 40% O horario de dormir ON O DND activo O override manual

### 3. **light_sleep** (Sueño ligero)
- **Score:** 60-74%
- **Descripción:** Posiblemente durmiendo, esperando confirmación
- **Comportamiento:** DND activado, esperando confirmación
- **Icono:** `mdi:sleep-off`
- **Trigger:** Score >= 60% + sin movimiento por [sleep_inactivity_threshold] minutos

### 4. **deep_sleep** (Sueño profundo)
- **Score:** 75-100%
- **Descripción:** Sueño confirmado con alta confianza
- **Comportamiento:** DND activo, modo nocturno completo
- **Icono:** `mdi:sleep`
- **Trigger:** Score >= 75% + [sleep_confirmation_time] minutos en light_sleep

### 5. **waking_up** (Despertando)
- **Score:** Variable (descendiendo)
- **Descripción:** Transición del sueño a despertar
- **Comportamiento:** Restauración gradual de servicios
- **Icono:** `mdi:alarm`
- **Trigger:** Actividad detectada después de deep_sleep

## 🔀 Flujo de Estados

```
awake
  ↓ Score >= 40% O manual_sleep_override
preparing_sleep
  ↓ Score >= 60% + [sleep_inactivity_threshold] min sin actividad
light_sleep (DND ON)
  ↓ Score >= 75% + [sleep_confirmation_time] min de confirmación
deep_sleep (DND ON)
  ↓ Actividad detectada
waking_up
  ↓ Score < 30% + fuera de horario
awake (DND OFF)
```

## 🛑 Sistema de Interrupción

### 1. Pausa Temporal (`sleep_detection_paused`)
- **Efecto:** Congela el estado actual, no permite transiciones
- **DND:** Se desactiva automáticamente
- **Uso:** Cuando estás despierto pero el sistema detecta inactividad
- **Desactivación:** Manual o automáticamente al día siguiente

### 2. Skip Hoy (`skip_sleep_detection_today`)
- **Efecto:** Fuerza estado "awake" y bloquea todas las transiciones
- **DND:** Se desactiva automáticamente
- **Uso:** Cuando sabes que vas a trasnochar
- **Desactivación:** Automática a las 6:00 AM

### 3. Cancelación Automática
- **Trigger:** Movimiento sostenido (3+ minutos) O teléfono activo (2+ minutos)
- **Efecto:** Vuelve a "awake" automáticamente
- **Estados afectados:** preparing_sleep, light_sleep
- **DND:** Se desactiva automáticamente

## 📊 Sensores Creados

### Sensores Principales
- `binary_sensor.durmiendo` - Sensor principal (ON cuando light_sleep o deep_sleep)
- `sensor.sleep_detection_score` - Score de sueño (0-100%)
- `sensor.sleep_status` - Estado detallado con atributos

### Sensores Indicadores
- `binary_sensor.indicador_horario_de_dormir`
- `binary_sensor.indicador_sin_movimiento`
- `binary_sensor.indicador_telefono_inactivo`
- `binary_sensor.indicador_cargando`
- `binary_sensor.indicador_dnd_activo`
- `binary_sensor.indicador_en_casa`
- `binary_sensor.indicador_confianza_de_sueno_alta`

## 🎨 Dashboard Card Compacta (Recomendada)

```yaml
# Card compacta con stack-in-card (recomendada)
type: custom:stack-in-card
mode: vertical
cards:
  # HEADER: Estado + Score
  - type: horizontal-stack
    cards:
      # Estado principal
      - type: custom:mushroom-template-card
        primary: |
          {% if is_state('input_boolean.skip_sleep_detection_today', 'on') %}
            🚫 Bloqueado
          {% elif is_state('input_boolean.sleep_detection_paused', 'on') %}
            ⏸️ Pausado
          {% else %}
            {{ state_attr('sensor.sleep_status', 'friendly_name') }}
          {% endif %}
        # ... (configuración completa en dashboards/sleep_detection_dashboard.yaml)

  # GRÁFICO: Mini-graph con barras
  - type: custom:mini-graph-card
    entities:
      - entity: sensor.sleep_detection_score
    bar_chart: true
    hours_to_show: 12

  # CONTROLES: Chips compactos
  - type: custom:mushroom-chips-card
    chips:
      - type: template
        icon: mdi:bed-clock
        content: Forzar
        tap_action:
          action: toggle
          entity: input_boolean.manual_sleep_override
      # ... (chips para Pausar, Skip, DND, ON/OFF)
```

## 🎨 Dashboard Completo (Opcional)

```yaml
type: vertical-stack
cards:
  - type: custom:mushroom-template-card
    primary: Estado de Sueño
    secondary: |
      {% if is_state('input_boolean.skip_sleep_detection_today', 'on') %}
        🚫 Bloqueado hasta mañana
      {% elif is_state('input_boolean.sleep_detection_paused', 'on') %}
        ⏸️ Pausado
      {% else %}
        {{ state_attr('sensor.sleep_status', 'friendly_name') }}
        {{ state_attr('sensor.sleep_status', 'duration') }}
      {% endif %}
    icon: |
      {% if is_state('input_boolean.skip_sleep_detection_today', 'on') %}
        mdi:cancel
      {% elif is_state('input_boolean.sleep_detection_paused', 'on') %}
        mdi:pause-circle
      {% else %}
        {{ state_attr('sensor.sleep_status', 'icon') }}
      {% endif %}
    icon_color: |
      {% set state = states('input_select.sleep_state') %}
      {% if state == 'deep_sleep' %}blue
      {% elif state == 'light_sleep' %}light-blue
      {% elif state == 'preparing_sleep' %}orange
      {% elif state == 'waking_up' %}yellow
      {% else %}green
      {% endif %}
    tap_action:
      action: more-info
      entity: sensor.sleep_status

  - type: custom:mini-graph-card
    entities:
      - entity: sensor.sleep_detection_score
        name: Score de Sueño
        color: '#2196F3'
    hours_to_show: 12
    line_width: 2
    points_per_hour: 4
    show:
      labels: true
      points: false

  - type: entities
    title: Controles
    entities:
      - entity: input_boolean.manual_sleep_override
        name: 🌙 Forzar preparación
      - entity: input_boolean.sleep_detection_paused
        name: ⏸️ Pausar detección
      - entity: input_boolean.skip_sleep_detection_today
        name: 🚫 No detectar hoy
      - type: divider
      - entity: input_boolean.no_molestar
        name: 🔕 No Molestar
      - entity: binary_sensor.durmiendo
        name: ¿Durmiendo?
    
  - type: entities
    title: Configuración de Tiempos
    entities:
      - entity: input_number.sleep_inactivity_threshold
        name: ⏱️ Inactividad para light_sleep
      - entity: input_number.sleep_confirmation_time
        name: ⏱️ Confirmación para deep_sleep
      - entity: input_number.sleep_confidence_threshold
        name: 📊 Confianza mínima
```

## 🔗 Integraciones (Solo Lectura)

El package **NO modifica** otros packages. Solo **lee** de:
- `input_boolean.horario_de_dormir` (yendo_a_dormir.yaml)
- `binary_sensor.motion_sala` (motion_kilo.yaml)
- `sensor.ubicacion_hernan` (tracker_hernan.yaml)
- `sensor.parte_del_dia` (sensor_parte_del_dia.yaml)

## 💡 Casos de Uso Futuros

### Notificaciones Inteligentes
```yaml
condition:
  - condition: state
    entity_id: input_boolean.no_molestar
    state: "off"
```

### Control de Luces
```yaml
condition:
  - condition: not
    conditions:
      - condition: state
        entity_id: input_select.sleep_state
        state: deep_sleep
```

### Climatización
```yaml
automation:
  - trigger:
      - platform: state
        entity_id: input_select.sleep_state
        to: light_sleep
    action:
      - service: climate.set_temperature
        data:
          temperature: 22
```

## 🚀 Instalación

1. Copiar `sleep_detection.yaml` a la carpeta `packages/`
2. Reiniciar Home Assistant
3. Verificar que todos los sensores se hayan creado
4. Ajustar los tiempos según preferencia
5. Agregar dashboard card

## 🔧 Troubleshooting

### El sistema no detecta sueño
- Verificar que `input_boolean.sleep_detection_enabled` esté ON
- Verificar que `input_boolean.skip_sleep_detection_today` esté OFF
- Verificar que `input_boolean.sleep_detection_paused` esté OFF
- Revisar el score en `sensor.sleep_detection_score`

### DND no se activa
- El DND solo se activa en `light_sleep` y `deep_sleep`
- Verificar que el estado sea correcto en `input_select.sleep_state`
- Puedes activar `input_boolean.no_molestar` manualmente

### Transiciones muy rápidas/lentas
- Ajustar `input_number.sleep_inactivity_threshold` (tiempo para light_sleep)
- Ajustar `input_number.sleep_confirmation_time` (tiempo para deep_sleep)
- Ajustar `input_number.sleep_confidence_threshold` (confianza mínima)

## 📝 Notas

- Los tiempos por defecto son 5 y 10 minutos (más cortos que el plan original)
- Todos los tiempos son configurables desde la UI
- El sistema se sincroniza automáticamente al reiniciar Home Assistant
- El skip se resetea automáticamente a las 6:00 AM
- El DND se desactiva automáticamente al despertar o cancelar

## 🎯 Próximos Pasos Sugeridos

1. Probar el sistema durante varios días
2. Ajustar umbrales según tu rutina
3. Agregar automatizaciones basadas en `input_boolean.no_molestar`
4. Integrar con `yendo_a_dormir.yaml` (opcional)
5. Agregar control de `sala_auto_bypass` (opcional)