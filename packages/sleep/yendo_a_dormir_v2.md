# Yendo a Dormir v2

Documentación del package `packages/sleep/yendo_a_dormir_v2.yaml`.

Este package separa dos ideas que antes estaban mezcladas:

- `input_boolean.yendo_a_dormir`: la rutina temporal que está corriendo ahora.
- `input_boolean.modo_dormir_activo`: el estado nocturno persistente de la casa.

La diferencia es importante: mientras `yendo_a_dormir` está activo, Smart Lights Sala queda bloqueado para que el movimiento no suba las luces durante el fade. Cuando la rutina termina, `yendo_a_dormir` se apaga, el lock se libera, Smart Lights Sala vuelve a funcionar, y `modo_dormir_activo` queda encendido como señal de que la casa está en modo dormir.

## Resumen de comportamiento

Al iniciar `input_boolean.yendo_a_dormir`:

1. Se enciende `input_boolean.modo_dormir_activo`.
2. Dormir reclama el lock protegido de `sala_de_estar` con `owner: sleep`.
3. Se apaga `input_boolean.smart_lights_sala`.
4. Se cancelan timers activos de Smart Light Sala y el timer propio de dormir.
5. Se guarda `scene.dormir_snapshot`.
6. Se ejecuta `script.dormir_01_inicio`.
7. Se inicia `timer.dormir_countdown` con `fade + espera`.
8. Se llama a `script.gradual_brightness_change` con token de lock.

Al finalizar el countdown:

1. Se ejecuta `script.dormir_03_final`.
2. Se ejecuta `script.dormir_sequential_off`.
3. Se aplica `scene.madrugada1`.
4. Se fuerza `input_select.smart_light_sala_state` a `WAITING_ARRIVAL`.
5. Se libera el lock de Dormir.
6. Se enciende `input_boolean.smart_lights_sala`.
7. Se llama a `script.smart_light_sala_reset_runtime`.
8. Se apaga `input_boolean.yendo_a_dormir`.
9. `input_boolean.modo_dormir_activo` queda encendido.

## Entidades principales

### `input_boolean.yendo_a_dormir`

Botón temporal de rutina.

Uso esperado:

- Encenderlo para iniciar la rutina de ir a dormir.
- Apagarlo mientras la rutina está corriendo para cancelarla.
- Mostrarlo en dashboard como estado de "alguien se está yendo a dormir".

No representa que la casa está durmiendo. Sólo representa que la secuencia está en curso.

### `input_boolean.modo_dormir_activo`

Estado persistente de modo dormir/nocturno.

Uso esperado:

- Se enciende automáticamente al iniciar `yendo_a_dormir`.
- Queda encendido cuando la rutina termina bien.
- Se apaga a las 08:00 con la automatización `dormir_off`.
- Puede usarse en dashboards o futuras automatizaciones para saber que la casa está en modo noche.

Importante: este helper no bloquea Smart Lights Sala por sí mismo. Motion puede reactivar Smart Lights mientras `modo_dormir_activo` está encendido, siempre que `yendo_a_dormir` ya esté apagado.

### `input_boolean.horario_de_dormir`

Helper horario simple.

- Se enciende a las 22:00.
- Se apaga a las 02:00.
- Sirve para mostrar controles o sugerir que es horario de dormir.

No inicia la rutina por sí mismo.

### `timer.dormir_countdown`

Countdown visible en dashboard.

Se usa para controlar el final exitoso de la rutina. La duración real se calcula como:

```text
input_number.dormir_fade_sec + input_number.dormir_wait_after_sec
```

Cuando termina, dispara `Dormir - FIN countdown`.

### `scene.dormir_snapshot`

Snapshot temporal de las luces de `sala_de_estar`.

Se crea al inicio de la rutina con `script.dormir_save_scene`. Si el usuario cancela `yendo_a_dormir`, el package intenta restaurar esta escena antes de liberar Smart Lights Sala.

## Configuración

### `input_number.dormir_reduccion_pct`

Brillo objetivo del fade, en porcentaje.

Ejemplo: `10` significa bajar las luces al 10%.

### `input_number.dormir_fade_sec`

Duración del fade gradual, en segundos.

El script global `gradual_brightness_change` recibe este valor como `transition_time` y también como cantidad de pasos.

### `input_number.dormir_wait_after_sec`

Tiempo extra de espera después del fade antes de ejecutar el cierre final.

El timer total de la rutina usa:

```text
dormir_fade_sec + dormir_wait_after_sec
```

## Scripts locales

### `script.dormir_save_scene`

Guarda el estado actual de las luces leaf de `sala_de_estar` en `scene.dormir_snapshot`.

### `script.dormir_turn_off_area`

Apaga toda el área `sala_de_estar`.

Actualmente queda como utilidad manual. El flujo principal usa `dormir_sequential_off` y `scene.madrugada1`.

### `script.dormir_sequential_off`

Apaga luces encendidas de sala una por una, dejando una luz encendida.

Esto da una salida más suave que un apagado total instantáneo.

## Automatizaciones

### `Dormir - ON start`

Trigger:

```yaml
input_boolean.yendo_a_dormir -> on
```

Responsabilidades:

- Activar `modo_dormir_activo`.
- Reclamar lock protegido de sala.
- Desactivar Smart Lights Sala.
- Guardar snapshot.
- Ejecutar acciones iniciales.
- Iniciar countdown.
- Lanzar fade gradual con token de Dormir.

El lock protegido evita que Smart Lights Sala cambie luces mientras la rutina está corriendo.

### `Dormir - FIN countdown`

Trigger:

```yaml
timer.dormir_countdown finished
```

Responsabilidades:

- Ejecutar acciones finales.
- Aplicar escena final `scene.madrugada1`.
- Liberar lock.
- Reactivar Smart Lights Sala.
- Recalcular Smart Lights Sala con `script.smart_light_sala_reset_runtime`.
- Apagar `yendo_a_dormir`.

Después de este punto, motion de sala puede volver a activar Smart Lights aunque `modo_dormir_activo` siga encendido.

### `Dormir - OFF cancel + restore`

Trigger:

```yaml
input_boolean.yendo_a_dormir -> off
```

Se ejecuta sólo si el timer está activo o si el lock de sala sigue en manos de `sleep`.

Responsabilidades:

- Cancelar countdown.
- Invalidar token de Dormir.
- Restaurar `scene.dormir_snapshot`.
- Apagar `modo_dormir_activo`.
- Liberar lock.
- Reactivar Smart Lights Sala.
- Recalcular Smart Lights Sala.

Esta es la cancelación manual esperada desde dashboard.

### `Dormir - Modo dormir OFF -> release sala`

Trigger:

```yaml
input_boolean.modo_dormir_activo -> off
```

Responsabilidades:

- Si `yendo_a_dormir` sigue activo, lo apaga y deja que la cancelación limpie.
- Si no hay rutina activa, libera lock de sala por seguridad.
- Reactiva Smart Lights Sala.
- Recalcula el runtime.

### `Dormir - HA START -> recover runtime`

Trigger:

```yaml
homeassistant.start
```

Responsabilidades:

- Si HA reinicia con `yendo_a_dormir` activo y el timer activo, recupera lock protegido y mantiene Smart Lights Sala apagado.
- Si HA reinicia con `yendo_a_dormir` activo pero sin timer activo, limpia la rutina para no quedar trabado.
- Si sólo `modo_dormir_activo` está encendido, libera cualquier lock residual de `sleep`.

### Horarios

- `Horario dormir - ON at 22:00`: enciende `input_boolean.horario_de_dormir`.
- `Horario dormir - OFF at 02:00`: apaga `input_boolean.horario_de_dormir`.
- `dormir_off` a las 08:00: apaga `modo_dormir_activo` y `yendo_a_dormir`, cancela countdown y libera lock.

## Integración con Smart Lights Sala

Durante `yendo_a_dormir`:

- Dormir reclama `sala_de_estar` con `owner: sleep`.
- El lock queda protegido.
- Smart Lights Sala queda apagado con `input_boolean.smart_lights_sala`.
- Si hay motion, Smart Lights Sala no debería subir ni restaurar luces porque no puede reclamar el área protegida.

Después del final exitoso:

- Dormir libera el lock.
- Smart Lights Sala se vuelve a encender.
- `script.smart_light_sala_reset_runtime` recalcula el estado según motion, timers y contexto actual.

Esto permite que `modo_dormir_activo` siga encendido sin bloquear la sala. El modo nocturno queda como contexto; la rutina temporal es la que bloquea.

## Integración con Gradual Brightness

El fade usa:

```yaml
script.gradual_brightness_change
```

con:

```yaml
lock_token_entity: input_text.light_transition_sala_token
lock_token: "{{ sleep_lock_token }}"
```

Esto permite que los workers se detengan solos si otro flujo invalida el token del área.

La cancelación manual cambia el token con `script.light_transition_claim_area`, lo que corta los workers activos sin apagar scripts globales que podrían estar usando otra área.

## Dashboard recomendado

Mostrar como mínimo:

- `input_boolean.yendo_a_dormir`: botón principal de iniciar/cancelar rutina.
- `input_boolean.modo_dormir_activo`: estado nocturno.
- `timer.dormir_countdown`: progreso de rutina.
- `input_number.dormir_reduccion_pct`: brillo objetivo.
- `input_number.dormir_fade_sec`: duración del fade.
- `input_number.dormir_wait_after_sec`: espera post-fade.

Lectura sugerida:

- Si `yendo_a_dormir` está `on`: "Rutina de dormir activa".
- Si `yendo_a_dormir` está `off` y `modo_dormir_activo` está `on`: "Modo dormir activo; Smart Lights puede responder al movimiento".
- Si ambos están `off`: "Modo dormir apagado".

## Pruebas recomendadas

### Inicio normal

1. Encender `input_boolean.yendo_a_dormir`.
2. Confirmar que `input_boolean.modo_dormir_activo` pasa a `on`.
3. Confirmar que `input_boolean.smart_lights_sala` pasa a `off`.
4. Confirmar que `input_text.light_transition_sala_owner` queda en `sleep`.
5. Confirmar que `input_boolean.light_transition_sala_protected` queda en `on`.

### Motion durante fade

1. Iniciar `yendo_a_dormir`.
2. Disparar motion de sala durante el fade.
3. Resultado esperado: Smart Lights Sala no sube ni restaura luces normales.

### Final exitoso

1. Dejar terminar `timer.dormir_countdown`.
2. Confirmar que `input_boolean.yendo_a_dormir` queda `off`.
3. Confirmar que `input_boolean.modo_dormir_activo` queda `on`.
4. Confirmar que el lock de sala se libera.
5. Confirmar que `input_boolean.smart_lights_sala` vuelve a `on`.
6. Disparar motion de sala y confirmar que Smart Lights responde normalmente.

### Cancelación manual

1. Iniciar `yendo_a_dormir`.
2. Apagar `input_boolean.yendo_a_dormir` desde dashboard.
3. Confirmar que `timer.dormir_countdown` se cancela.
4. Confirmar que `scene.dormir_snapshot` se restaura.
5. Confirmar que `input_boolean.modo_dormir_activo` queda `off`.
6. Confirmar que Smart Lights Sala vuelve a funcionar.

### Restart de Home Assistant

1. Reiniciar HA con `yendo_a_dormir` activo y timer activo.
2. Confirmar que se recupera el lock protegido.
3. Reiniciar HA con sólo `modo_dormir_activo` activo.
4. Confirmar que no queda lock residual de `sleep`.

## Troubleshooting

### Smart Lights no responde después de dormir

Revisar:

- `input_boolean.smart_lights_sala` debe estar `on`.
- `input_text.light_transition_sala_owner` no debería quedar en `sleep`.
- `input_boolean.light_transition_sala_protected` debería estar `off`.
- Ejecutar manualmente `script.smart_light_sala_reset_runtime`.

### La rutina queda trabada

Revisar:

- `input_boolean.yendo_a_dormir`.
- `timer.dormir_countdown`.
- `input_text.light_transition_sala_owner`.
- `input_text.light_transition_sala_token`.

Si `yendo_a_dormir` está `on` sin timer activo, apagarlo desde dashboard debería disparar la limpieza.

### La cancelación no restaura luces

Revisar que exista `scene.dormir_snapshot`. Se crea al inicio de la rutina. Si la rutina se cancela demasiado temprano, antes de guardar snapshot, puede no haber escena válida para restaurar.

## Changelog

### 2026-04-28

- `input_boolean.dormir_activo` fue reemplazado por `input_boolean.yendo_a_dormir`.
- Se agregó `input_boolean.modo_dormir_activo`.
- Se separó rutina temporal de estado nocturno persistente.
- Dormir bloquea Smart Lights Sala sólo durante `yendo_a_dormir`.
- Al finalizar la rutina, se libera el lock y Smart Lights Sala vuelve a responder al movimiento.
- Se agregó recovery en `homeassistant.start`.
- La cancelación manual restaura snapshot, apaga modo dormir y libera Smart Lights Sala.
- Se integró el flujo con el coordinador global de transiciones por área.

