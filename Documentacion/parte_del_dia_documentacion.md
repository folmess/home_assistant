# Package: Parte del Día (Home Assistant)
Este paquete crea un sistema flexible para determinar la **parte del día** (madrugada, mañana, mediodía, tarde, noche) usando helpers ajustables, y un sensor global que calcula el estado actual, icono dinámico y rango horario.

---

## 1. Helpers (`input_datetime`)
Definimos un `input_datetime` para el inicio de cada tramo del día.
- No incluyen fecha (`has_date: false`), solo hora (`has_time: true`).
- Los iconos pueden personalizarse y se usarán luego en el sensor para mostrar icono dinámico.

```yaml
input_datetime:
  parte_madrugada_inicio:
    name: Inicio Madrugada
    icon: mdi:theme-light-dark
    has_date: false
    has_time: true
    initial: "00:00:00"

  parte_manana_inicio:
    name: Inicio Mañana
    icon: mdi:weather-sunset-up
    has_date: false
    has_time: true
    initial: "06:00:00"

  parte_mediodia_inicio:
    name: Inicio Mediodía
    icon: mdi:weather-sunny
    has_date: false
    has_time: true
    initial: "11:00:00"

  parte_tarde_inicio:
    name: Inicio Tarde
    icon: mdi:weather-sunset
    has_date: false
    has_time: true
    initial: "15:00:00"

  parte_noche_inicio:
    name: Inicio Noche
    icon: mdi:weather-night
    has_date: false
    has_time: true
    initial: "20:00:00"
```

---

## 2. Sensor `parte_del_dia`
Crea un sensor que:
1. Calcula el **estado** en función de la hora actual y los `input_datetime`.
2. Ajusta correctamente los cambios que pasan por medianoche.
3. Usa **icono dinámico** según el icono configurado en cada helper.
4. Añade atributo `range` con el horario de inicio y fin del tramo actual, con formato `HH:MM → HH:MM`.

```yaml
template:
  - sensor:
      - name: Parte del día
        unique_id: parte_del_dia_sensor
        state: >
          {% set t_now = now() %}
          {% set t_madr = today_at(states('input_datetime.parte_madrugada_inicio')) %}
          {% set t_man  = today_at(states('input_datetime.parte_manana_inicio')) %}
          {% set t_med  = today_at(states('input_datetime.parte_mediodia_inicio')) %}
          {% set t_tar  = today_at(states('input_datetime.parte_tarde_inicio')) %}
          {% set t_noc  = today_at(states('input_datetime.parte_noche_inicio')) %}

          {% set madr_end = t_man if t_man > t_madr else t_man + timedelta(days=1) %}
          {% set man_end  = t_med if t_med > t_man else t_med + timedelta(days=1) %}
          {% set med_end  = t_tar if t_tar > t_med else t_tar + timedelta(days=1) %}
          {% set tar_end  = t_noc if t_noc > t_tar else t_noc + timedelta(days=1) %}

          {% if t_now >= t_madr and t_now < madr_end %}Madrugada
          {% elif t_now >= t_man and t_now < man_end %}Mañana
          {% elif t_now >= t_med and t_now < med_end %}Mediodía
          {% elif t_now >= t_tar and t_now < tar_end %}Tarde
          {% else %}Noche
          {% endif %}
        icon: >
          {% set mapa = {
            'Madrugada': state_attr('input_datetime.parte_madrugada_inicio','icon'),
            'Mañana'   : state_attr('input_datetime.parte_manana_inicio','icon'),
            'Mediodía' : state_attr('input_datetime.parte_mediodia_inicio','icon'),
            'Tarde'    : state_attr('input_datetime.parte_tarde_inicio','icon'),
            'Noche'    : state_attr('input_datetime.parte_noche_inicio','icon')
          } %}
          {{ mapa.get(this.state, 'mdi:clock-outline') }}
        attributes:
          range: >
            {% set madr = states('input_datetime.parte_madrugada_inicio')[0:5] %}
            {% set man  = states('input_datetime.parte_manana_inicio')[0:5] %}
            {% set med  = states('input_datetime.parte_mediodia_inicio')[0:5] %}
            {% set tar  = states('input_datetime.parte_tarde_inicio')[0:5] %}
            {% set noc  = states('input_datetime.parte_noche_inicio')[0:5] %}

            {% set t_now = now() %}
            {% set t_madr = today_at(states('input_datetime.parte_madrugada_inicio')) %}
            {% set t_man  = today_at(states('input_datetime.parte_manana_inicio')) %}
            {% set t_med  = today_at(states('input_datetime.parte_mediodia_inicio')) %}
            {% set t_tar  = today_at(states('input_datetime.parte_tarde_inicio')) %}
            {% set t_noc  = today_at(states('input_datetime.parte_noche_inicio')) %}

            {% set madr_end = t_man if t_man > t_madr else t_man + timedelta(days=1) %}
            {% set man_end  = t_med if t_med > t_man else t_med + timedelta(days=1) %}
            {% set med_end  = t_tar if t_tar > t_med else t_tar + timedelta(days=1) %}
            {% set tar_end  = t_noc if t_noc > t_tar else t_noc + timedelta(days=1) %}

            {% if t_now >= t_madr and t_now < madr_end %}
              {{ madr }} → {{ man }}
            {% elif t_now >= t_man and t_now < man_end %}
              {{ man }} → {{ med }}
            {% elif t_now >= t_med and t_now < med_end %}
              {{ med }} → {{ tar }}
            {% elif t_now >= t_tar and t_now < tar_end %}
              {{ tar }} → {{ noc }}
            {% else %}
              {{ noc }} → {{ madr }}
            {% endif %}
```

---

## 3. Ejemplo de uso en tarjeta Lovelace (Mushroom)
```yaml
type: custom:mushroom-template-card
entity: sensor.parte_del_dia
primary: "{{ states('sensor.parte_del_dia') }}"
secondary: "{{ state_attr('sensor.parte_del_dia', 'range') }}"
icon: "{{ state_attr('sensor.parte_del_dia','icon') }}"
```
Esto mostrará el nombre de la parte del día, el horario de inicio/fin y el icono asociado.

---

## Ventajas de este enfoque
- Horarios totalmente configurables desde la UI.
- Iconos dinámicos sin modificar el código.
- Lógica preparada para cambios que cruzan medianoche.
- Atributo extra `range` ideal para mostrar en dashboards.

