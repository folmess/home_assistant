# Estrategia de Control por Movimiento - Sala de Estar (v9.1)

## Descripción General

Este paquete implementa un sistema de automatización para controlar las luces de la sala de estar basado en detección de movimiento. Está diseñado para ser flexible, eficiente y adaptarse a diferentes momentos del día. Incluye temporizadores, preavisos y restauración de estados previos de las luces.

### Características Principales:
1. **Automatización por Movimiento**:
   - **Encendido**: Las luces se encienden al detectar movimiento, con diferentes comportamientos según la hora del día.
   - **Apagado**: Las luces se apagan tras un tiempo configurable sin movimiento, con un preaviso antes del apagado total.

2. **Configuración por Parte del Día**:
   - Los tiempos de apagado se ajustan según la parte del día (`mañana`, `mediodía`, `tarde`, `noche`, `madrugada`).
   - Se utiliza el sensor `sensor.parte_del_dia` para determinar la franja horaria.

3. **Preaviso**:
   - Antes del apagado total, las luces reducen su brillo en un porcentaje configurable (`input_number.sala_preaviso_target_pct`).
   - Esto permite advertir a los usuarios que las luces están por apagarse.

4. **Restauración de Estado**:
   - Si las luces estaban encendidas antes de la detección de movimiento, se guarda un "snapshot" de su estado para restaurarlo posteriormente.

5. **Timers**:
   - `timer.apagado_sala`: Controla el tiempo total antes del apagado.
   - `timer.preaviso_sala`: Controla la duración del preaviso.

6. **Configuración Flexible**:
   - Los tiempos de apagado y la reducción de brillo del preaviso son configurables a través de `input_number`.
   - Se puede definir una escena específica para la madrugada (`input_text.sala_scene_madrugada`).

---

## Flujo de las Automatizaciones

### 1. **Automatización de Encendido (`motion_detected`)**
   - **Disparador**: Movimiento detectado (`binary_sensor.motion_sala` pasa a `on`).
   - **Condición**: El interruptor `input_boolean.mantener_luces` debe estar en `off`.
   - **Acciones**:
     - Cancela cualquier temporizador activo.
     - **Madrugada**:
       - Si hay una escena configurada, se activa.
       - Si no, se encienden todas las luces al 15% de brillo.
     - **Resto del Día**:
       - Si hay un snapshot disponible, se restaura.
       - Si no, se encienden todas las luces al 60% de brillo.

### 2. **Automatización de Apagado (`motion_clear`)**
   - **Disparador**: No se detecta movimiento (`binary_sensor.motion_sala` pasa a `off` durante 5 segundos).
   - **Condición**: El interruptor `input_boolean.mantener_luces` debe estar en `off`.
   - **Acciones**:
     1. **Inicio del Timer Total**:
        - Se inicia un temporizador (`timer.apagado_sala`) con la duración configurada según la parte del día.
     2. **Espera hasta el Preaviso**:
        - Si vuelve el movimiento antes del preaviso, se cancelan los temporizadores y se detiene el flujo.
     3. **Snapshot y Preaviso**:
        - Si no vuelve el movimiento:
          - Se crea un snapshot del estado actual de las luces.
          - Se reducen todas las luces encendidas según el porcentaje configurado para el preaviso.
          - Se inicia el temporizador de preaviso (`timer.preaviso_sala`).
     4. **Espera durante el Preaviso**:
        - Si vuelve el movimiento durante el preaviso:
          - Se cancela el apagado y se restaura el snapshot.
        - Si no vuelve el movimiento:
          - Se apagan todas las luces y se limpia el snapshot.

---

## Configuración de Helpers

### **Input Booleans**
- `input_boolean.sala_snapshot_ready`: Indica si hay un snapshot listo para restaurar.

### **Input Numbers**
- `input_number.sala_off_*`: Configuran los tiempos de apagado (en minutos) para cada parte del día.
- `input_number.sala_preaviso`: Configura la duración del preaviso (en minutos).
- `input_number.sala_preaviso_target_pct`: Configura el porcentaje de reducción de brillo para el preaviso.

### **Input Text**
- `input_text.sala_scene_madrugada`: Permite definir una escena específica para la madrugada.

### **Timers**
- `timer.apagado_sala`: Temporizador para el apagado total.
- `timer.preaviso_sala`: Temporizador para el preaviso.

---

## Notas Adicionales
- **Área Configurada**: El paquete está diseñado para el área `sala_de_estar`. Si tu área tiene un nombre diferente, actualiza el valor de `area_id_sala`.
- **Fallbacks**: Si no hay configuraciones específicas (como una escena para la madrugada), se utilizan valores predeterminados (e.g., 15% de brillo).

---

## Requisitos
- Sensor de movimiento: `binary_sensor.motion_sala`.
- Sensor de parte del día: `sensor.parte_del_dia`.
- Entidades de luces en el área `sala_de_estar`.

---

Este paquete está diseñado para ser eficiente y personalizable, ofreciendo una experiencia de iluminación automatizada que se adapta a las necesidades del usuario y al contexto del día.