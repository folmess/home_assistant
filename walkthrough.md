# Walkthrough: Mejoras del YAML en Smart Light Sala

Se ha realizado una refactorización integral del código para mejorar su eficiencia, la legibilidad y prepararlo para un crecimiento a futuro sin repetir la lógica en todo momento.

## Cambios Realizados

### 1. Template Sensor Centralizado
Se retiraron las decenas de líneas de iteración del área de luces `(*namespace leaf*)` que estaban repetidas 5 veces en todo el archivo.
En su lugar, creamos:
```yaml
      - name: "Smart Light Sala Leaf Lights"
        unique_id: smart_light_sala_leaf_lights
        state: "OK"
        attributes:
          entity_id: >-
             # (Aquí la lógica del listado calculada 1 sola vez)
```
Y en los scripts ahora simplemente dice: `{{ state_attr('sensor.smart_light_sala_leaf_lights', 'entity_id') | default([]) }}`.

### 2. Diccionarios Inline en vez de `{% if %}`
El atributo de los colores, acciones e iconos del sensor `Smart Light Sala State` eran inmensos y repetitivos. Se consolidaron a una sola línea usando `dict.get()`.
**Ejemplo de cómo quedó `next_icon`:**
```yaml
          next_icon: >-
            {{ {
              'DISABLED': 'mdi:cancel',
              'ACTIVE': 'mdi:run',
              ...
            }.get(states('input_select.smart_light_sala_state'), 'mdi:help-circle') }}
```

### 3. PRE_NOTICE Extraído a Script
Separamos el bloque de acción ultra largo de la automatización temporal:
- **`script.smart_light_sala_execute_prenotice`**: Contiene exclusivamente la lógica de fading por luz y los bucles por bombilla.
- **`smart_light_sala_pre_notice_start_parallel_restore`**: Es ahora una automatización súper limpia que solo atiende al Timer Finished y delega con un `script.turn_on`.

### 4. Valores Default Fijos
*Se priorizó el uso correcto de los input_number en lugar de tener `int(30)` como magic numbers dentro del código, exceptuando cuando no había un sensor obvio a asociar.*

---

## Resultados
- **DRY:** Menos repetición se traduce en menos puntos de falla cuando instales focos o saques partes de luz de la sala.
- **Modular:** Las lógicas están extraídas para poder probarlas manualmente si así lo requerís desde el panel de **Herramientas de Desarrollador > Servicios**.

> [!TIP]
> Dado que pasamos muchísima lógica a componentes "template:", los tiempos de recálculo (overhead) de tus sensores cuando ocurren cambios automáticos en la sala deberían mejorarse levemente.
