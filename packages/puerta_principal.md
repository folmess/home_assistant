# Puerta Principal

Documentacion del package `packages/puerta_principal.yaml`.

## Resumen

Este package interpreta eventos de la puerta principal y los convierte en senales utiles para la casa:

- entrada detectada;
- salida detectada;
- puerta abierta demasiado tiempo;
- doble apertura/cierre rapido;
- arribo a sala con escena de bienvenida.

La parte nueva importante es el arribo por puerta. Cuando `input_boolean.flag_acceso_entro` pasa a `on`, el package asume que alguien esta llegando a casa y toma control temporal de la sala para ejecutar una bienvenida visual sin que Smart Light Sala la pise por movimiento.

## Entidades

### `binary_sensor.puerta_entrada`

Sensor fisico de apertura de la puerta principal.

- `on`: puerta abierta.
- `off`: puerta cerrada.

### `binary_sensor.motion_sala_1_occupancy`

Sensor usado por este package para distinguir entrada de salida.

Si la puerta se abre y no hubo movimiento reciente, se considera entrada. Si la puerta se abre con movimiento presente, se considera salida.

### `input_boolean.flag_acceso_entro`

Flag temporal de entrada detectada.

- Se enciende cuando la puerta se abre sin movimiento reciente en sala.
- Se apaga automaticamente despues de 10 minutos.
- Dispara el flujo de arribo por puerta.

### `input_boolean.flag_acceso_salio`

Flag temporal de salida detectada.

- Se enciende cuando la puerta se abre con movimiento presente.
- Se apaga automaticamente despues de 10 minutos.

### `input_boolean.flag_puerta_abierta_x_min`

Flag de puerta abierta demasiado tiempo.

Se enciende si `binary_sensor.puerta_entrada` permanece abierto mas tiempo que el valor configurado en `input_number.puerta_abierta_minutos`.

### `input_number.puerta_abierta_minutos`

Minutos necesarios para considerar que la puerta quedo abierta.

### `input_boolean.sala_arribo_puerta_activo`

Indica que esta corriendo el flujo de arribo por puerta.

Es util para dashboard/debug. No deberia usarse como boton principal de usuario.

### `input_number.sala_arribo_hold_seconds`

Segundos de pausa despues de ejecutar `script.sala_arribo_puerta`.

Durante esa pausa, el lock de sala sigue protegido para que Smart Light Sala no aplique mood ni restaure luces.

### `timer.sala_arribo_puerta_hold`

Timer de la pausa post arribo.

Cuando termina, el package libera el lock y devuelve el control a Smart Light Sala.

## Arribo por Puerta

El flujo empieza con:

```text
input_boolean.flag_acceso_entro -> on
```

Luego `script.puerta_principal_arribo_sala_start` hace:

1. Verifica que `input_boolean.yendo_a_dormir` este `off`.
2. Cancela timers activos de Smart Light Sala.
3. Reclama `sala_de_estar` con `owner: puerta_principal_arribo`.
4. Marca el lock como protegido.
5. Setea `input_select.smart_light_sala_state` en `ACTIVE`.
6. Ejecuta `script.sala_arribo_puerta`.
7. Arranca `timer.sala_arribo_puerta_hold`.

Mientras ese lock protegido esta activo, Smart Light Sala ve que otro owner controla el area y no intenta prender, restaurar ni cambiar mood aunque `binary_sensor.motion_sala` detecte movimiento.

## Script Editable

La receta visual vive en `scripts.yaml`:

```text
script.sala_arribo_puerta
```

Ese script esta pensado para editarse desde la UI de Home Assistant. Puede prender luces una por una, usar delays, cambiar color, brillo o llamar escenas.

El package no define la estetica final; solo coordina el momento y protege la sala mientras la bienvenida ocurre.

## Handoff a Smart Light Sala

Cuando termina `timer.sala_arribo_puerta_hold`, corre `script.puerta_principal_arribo_sala_finish`.

Comportamiento:

- Libera el lock de `sala_de_estar`.
- Apaga `input_boolean.sala_arribo_puerta_activo`.
- Si `binary_sensor.motion_sala` sigue `on`, deja Smart Light Sala en `ACTIVE` sin reaplicar mood.
- Si `binary_sensor.motion_sala` esta `off`, inicia el flujo normal de idle/countdown.

Esto conserva la escena de bienvenida si la persona sigue en sala y evita un cambio brusco de mood justo al entrar.

## Relacion con Dormir

Si `input_boolean.yendo_a_dormir` esta `on`, el arribo por puerta no toma control ni prende luces.

La razon: durante la rutina de dormir, `sleep` tiene un lock protegido sobre `sala_de_estar`. Esa rutina tiene prioridad sobre motion y sobre arribo.

`input_boolean.modo_dormir_activo` no bloquea por si mismo. Solo bloquea la rutina temporal `yendo_a_dormir`.

## Relacion con Modo Cine

El handoff evita reactivar Smart Light Sala si `input_boolean.modo_cine` esta `on`.

Modo cine sigue siendo responsable de apagar/reactivar `input_boolean.smart_lights_sala` desde `automations.yaml`.

## Debug Rapido

Para entender que esta pasando, mirar:

- `input_boolean.flag_acceso_entro`
- `input_boolean.sala_arribo_puerta_activo`
- `timer.sala_arribo_puerta_hold`
- `input_text.light_transition_sala_owner`
- `input_boolean.light_transition_sala_protected`
- `input_select.smart_light_sala_state`
- `binary_sensor.motion_sala`

Valores esperados durante arribo:

- `input_boolean.sala_arribo_puerta_activo`: `on`
- `input_text.light_transition_sala_owner`: `puerta_principal_arribo`
- `input_boolean.light_transition_sala_protected`: `on`

## Pruebas Recomendadas

1. Activar `input_boolean.flag_acceso_entro`.
2. Confirmar que se ejecuta `script.sala_arribo_puerta`.
3. Confirmar que `input_boolean.sala_arribo_puerta_activo` pasa a `on`.
4. Disparar motion en sala durante la bienvenida.
5. Confirmar que Smart Light Sala no cambia mood ni restaura snapshot.
6. Esperar que termine `timer.sala_arribo_puerta_hold`.
7. Confirmar que el lock se libera.
8. Confirmar que Smart Light Sala queda `ACTIVE` si hay motion o entra en idle si no hay motion.
