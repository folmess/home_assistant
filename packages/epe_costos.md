# 📘 EPE Santa Fe -- Cálculo de Costos en Home Assistant

Este archivo documenta el package `epe_costos.yaml` que armamos para
calcular el costo eléctrico **según la factura de la EPE Santa Fe**.

Incluye:\
- **Helpers** (`input_number`) editables desde la UI.\
- **Sensores template** que aplican los tramos, adicionales, impuestos y
fijos.\
- **Snippets Lovelace** para visualizar todo en Mushroom.

------------------------------------------------------------------------

## ⚙️ 1. Helpers configurables

En **Ajustes → Dispositivos y servicios → Helpers** vas a encontrar:

  --------------------------------------------------------------------------------------------------------------------------------------------------------------------
  Helper                                      Uso
  ------------------------------------------- ------------------------------------------------------------------------------------------------------------------------
  `epe_tier1_limit_kwh`                       Límite de kWh para el **primer tramo** (ej. 150 kWh).

  `epe_tier2_limit_kwh`                       Límite acumulado de kWh para el **segundo tramo** (ej. 300 kWh).

  `epe_tier1_price_ars_kwh`                   Precio \$/kWh tramo 1.

  `epe_tier2_price_ars_kwh`                   Precio \$/kWh tramo 2.

  `epe_tier3_price_ars_kwh`                   Precio
                                              $/kWh tramo 3 (todo lo que supere el tramo 2). | | `epe_cuota_servicio_mensual` | Cargo fijo mensual por servicio ($).

  `epe_cap_fijo_mensual`                      Cargo fijo mensual de **CAP (alumbrado público)**.

  `epe_fer_pct`                               \% adicional por **FER (Ley 6604)** sobre el Básico.

  `epe_ord1592_pct`                           \% adicional municipal Ord. 1592/62.

  `epe_ord1618_pct`                           \% adicional municipal Ord. 1618/62.

  `epe_ley7797_pct`                           \% adicional Ley 7797.

  `epe_iva_pct`                               IVA sobre (Básico + CAP). En la boleta es **21%**.

  `epe_ley12692_fijo`                         Cargo fijo Ley 12692 (energías renovables).
  --------------------------------------------------------------------------------------------------------------------------------------------------------------------

👉 **Ventaja**: cuando la EPE cambie tarifas, solo actualizás estos
helpers sin tocar el código.

------------------------------------------------------------------------

## 📊 2. Sensores creados

### Consumo

-   `sensor.epe_consumo_mensual_kwh` → lee del
    `utility_meter.luces_total_monthly` (kWh acumulados en el mes).\
-   `sensor.epe_kwh_tier1` → kWh consumidos en tramo 1.\
-   `sensor.epe_kwh_tier2` → kWh en tramo 2.\
-   `sensor.epe_kwh_tier3` → kWh en tramo 3.

### Costos energía

-   `sensor.epe_energia` → costo \$ de energía según los tramos.\
-   `sensor.epe_basico` → energía + cuota de servicio (base para
    cálculos de adicionales).

### Adicionales sobre Básico

-   `sensor.epe_fer_6604`\
-   `sensor.epe_ord_1592`\
-   `sensor.epe_ord_1618`\
-   `sensor.epe_ley_7797`

### Fijos e impuestos

-   `sensor.epe_cap` → CAP fijo mensual.\
-   `sensor.epe_iva` → IVA calculado como 21% sobre (Básico + CAP).\
-   `sensor.epe_ley_12692` → monto fijo de energías renovables.

### Totales

-   `sensor.epe_total_mensual` → suma de todo lo anterior, simula el
    **TOTAL de la boleta**.

### Precio marginal y aproximaciones

-   `sensor.epe_precio_marginal` → precio \$/kWh según el tramo en el
    que estás actualmente en el mes.\
-   `sensor.epe_costo_variable_diario_aprox` → gasto \$ de hoy ≈ kWh
    diarios × precio marginal.\
-   `sensor.epe_costo_variable_semanal_aprox` → gasto \$ de la semana ≈
    kWh semanales × precio marginal.

⚠️ Nota: los costos "aprox" diarios/semanales **no incluyen fijos ni
adicionales**, son solo para ver cómo viene evolucionando el mes.

------------------------------------------------------------------------

## 🖼️ 3. Dashboard Lovelace

Snippet con **chips de resumen + detalle de tramos + costo
diario/semanal**:

``` yaml
title: Costos EPE - Luces
path: costos_epe
icon: mdi:cash
type: custom:masonry-layout
cards:
  - type: custom:mushroom-chips-card
    chips:
      - type: template
        icon: mdi:flash
        content: >
          {{ (states('sensor.epe_consumo_mensual_kwh')|float) | round(1) }} kWh
      - type: template
        icon: mdi:cash
        content: >
          Básico ${{ states('sensor.epe_basico') }}
      - type: template
        icon: mdi:street-light
        content: >
          CAP ${{ states('sensor.epe_cap') }}
      - type: template
        icon: mdi:percent
        content: >
          IVA ${{ states('sensor.epe_iva') }}
      - type: template
        icon: mdi:cash-multiple
        content: >
          Total ${{ states('sensor.epe_total_mensual') }}

  - type: grid
    columns: 2
    square: false
    cards:
      - type: custom:mushroom-entity-card
        entity: sensor.epe_kwh_tier1
        name: kWh Tier1
        icon: mdi:numeric-1-circle
      - type: custom:mushroom-entity-card
        entity: sensor.epe_kwh_tier2
        name: kWh Tier2
        icon: mdi:numeric-2-circle
      - type: custom:mushroom-entity-card
        entity: sensor.epe_kwh_tier3
        name: kWh Tier3
        icon: mdi:numeric-3-circle

  - type: grid
    columns: 2
    square: false
    cards:
      - type: custom:mushroom-entity-card
        entity: sensor.epe_costo_variable_diario_aprox
        name: Variable hoy ($)
        icon: mdi:cash-clock
      - type: custom:mushroom-entity-card
        entity: sensor.epe_costo_variable_semanal_aprox
        name: Variable semana ($)
        icon: mdi:cash-clock
```

------------------------------------------------------------------------

## 🔍 4. Cómo validar

1.  Compará `sensor.epe_total_mensual` con tu boleta:\
    debería dar casi igual, salvo diferencias de redondeo (usamos 2
    decimales).\
2.  Si no coincide, revisá:
    -   Que `sensor.luces_total_monthly` tenga un valor parecido al de
        tu factura real (kWh consumidos).\
    -   Que todos los helpers (`input_number`) tengan los valores
        correctos de la última tarifa.

------------------------------------------------------------------------

## ✅ 5. Qué podés extender

-   Crear la **misma lógica para "Total casa"** (no solo luces).\
-   Prorratear costos fijos por **área** o **dispositivo**.\
-   Guardar históricos de `$` con `utility_meter` adicionales.\
-   Añadir un **sensor de costo por kWh promedio real**:\
    `(sensor.epe_total_mensual / sensor.epe_consumo_mensual_kwh)`.
