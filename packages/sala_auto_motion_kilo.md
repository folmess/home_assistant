# Package: motion_kilo

## Descripción
Auto lights en Sala de Estar: escena, reducción/fade, restore desde snapshot, y waiting arrival. timers por franja y "leaf lights".

Requiere:
- Integración HACS "var" (entidad: var.sala_snapshot_targets)
- script.gradual_brightness_change (externo)
- script.gradual_restore_from_snapshot (externo)

## Variables
- **var.sala_snapshot_targets**: Targets por-luz (JSON) - Almacena los niveles de brillo por luz en formato JSON para restauración.

## Helpers

### input_select
- **sala_auto_state**: Estado lógico del sistema (opciones: ACTIVE, IDLE_COUNTDOWN, PRE_NOTICE, RESTORE_WINDOW, WAITING_ARRIVAL, DISABLED).

### input_boolean
- **sala_auto_bypass**: Bypass general - Desactiva la automatización.
- **sala_auto_debug_mode**: Debug - Modo de depuración.
- **sala_auto_fade**: Usar fade en PRE_NOTICE - Activa transición suave en reducción de brillo.
- **sala_auto_fading_active**: Fading activo - Indica si hay un fade en curso.

### input_number
#### Timeouts (min) por franja
- **sala_auto_timeout_manana**: Timeout - Mañana (inicial: 5 min)
- **sala_auto_timeout_mediodia**: Timeout - Mediodía (inicial: 5 min)
- **sala_auto_timeout_tarde**: Timeout - Tarde (inicial: 5 min)
- **sala_auto_timeout_noche**: Timeout - Noche (inicial: 5 min)
- **sala_auto_timeout_madrugada**: Timeout - Madrugada (inicial: 3 min)

#### Arribo (ventana de restore, min) por franja
- **sala_auto_arribo_manana**: Arribo - Mañana (inicial: 4 min)
- **sala_auto_arribo_mediodia**: Arribo - Mediodía (inicial: 4 min)
- **sala_auto_arribo_tarde**: Arribo - Tarde (inicial: 4 min)
- **sala_auto_arribo_noche**: Arribo - Noche (inicial: 4 min)
- **sala_auto_arribo_madrugada**: Arribo - Madrugada (inicial: 2 min)

#### Brillos por franja (fallback encendido simple)
- **sala_auto_brightness_manana**: Brightness - Mañana (inicial: 200)
- **sala_auto_brightness_mediodia**: Brightness - Mediodía (inicial: 200)
- **sala_auto_brightness_tarde**: Brightness - Tarde (inicial: 200)
- **sala_auto_brightness_noche**: Brightness - Noche (inicial: 150)
- **sala_auto_brightness_madrugada**: Brightness - Madrugada (inicial: 50)

#### Parámetros adicionales
- **sala_auto_reduction_percentage**: Reducción en PRE_NOTICE (%) (inicial: 50%)
- **sala_auto_fade_sec**: Fade duración (s) (inicial: 30 s)
- **sala_auto_fade_reduce_pct**: Fade - Reducir (% sobre % MIN actual) (inicial: 60%)

### input_text
- **sala_auto_effect_script_name**: Effect Script - Global (inicial: sala_arribo_global)
- **sala_auto_effect_script_manana**: Effect Script - Mañana
- **sala_auto_effect_script_mediodia**: Effect Script - Mediodía
- **sala_auto_effect_script_tarde**: Effect Script - Tarde
- **sala_auto_effect_script_noche**: Effect Script - Noche
- **sala_auto_effect_script_madrugada**: Effect Script - Madrugada

## Timers
- **sala_auto_idle_timeout**: Idle countdown - Cuenta regresiva de inactividad.
- **sala_auto_dim_notice**: Pre-Notice - Notificación previa a reducción/apagado.
- **sala_auto_restore_window**: Restore Window - Ventana para restaurar luces al detectar movimiento.

## Scripts
- **sala_auto_save_scene**: Guardar escena + targets JSON (leaf) - Crea una escena y guarda niveles de brillo por luz en JSON.
- **sala_auto_reduce_brightness**: Reducir brillo instantáneo (leaf) - Reduce el brillo de las luces encendidas según porcentaje configurado.
- **sala_auto_turn_off_lights**: Apagar luces (leaf) - Apaga todas las luces del área de sala de estar.
- **sala_auto_turn_on_lights**: Encender (escena/efecto por franja, leaf) - Enciende luces usando escena guardada, efecto por franja o fallback simple.
- **sala_auto_start_idle_flow**: Idle countdown flow - Inicia el flujo de cuenta regresiva de inactividad con timers.

## Template Sensors
- **Sala Auto State**: Sensor dashboard-friendly - Muestra el estado lógico con atributos como next_action, icono, color, timer restante, etc.

## Automations
- **sala_auto - Bypass ON → DISABLED**: Al activar bypass, cambia estado a DISABLED y cancela timers.
- **sala_auto - Bypass OFF → recompute**: Al desactivar bypass, recomputa estado basado en sensor de movimiento.
- **sala_auto - Motion OFF -> IDLE_COUNTDOWN**: Al perder movimiento, inicia cuenta regresiva de inactividad.
- **sala_auto - Motion ON -> ACTIVE (V2 restore suave)**: Al detectar movimiento, activa estado ACTIVE, cancela fades y restaura suavemente si venía de PRE_NOTICE o RESTORE_WINDOW.
- **sala_auto - PRE_NOTICE START → snapshot + dim/fade + RESTORE WINDOW**: Al iniciar PRE_NOTICE, guarda snapshot, inicia fade/reducción y ventana de restore.
- **sala_auto - PRE_NOTICE FIN → apagar + RESTORE START**: Al finalizar PRE_NOTICE, apaga luces y inicia ventana de restore con duración por franja.
- **sala_auto - RESTORE FIN → WAITING_ARRIVAL**: Al finalizar ventana de restore, cambia a WAITING_ARRIVAL.