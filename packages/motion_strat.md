
# Motion Sala – v10.1 **beta** (Core + Workers + Grupo)

[![Home Assistant](https://img.shields.io/badge/Home%20Assistant-Automations-41BDF5)](https://www.home-assistant.io/)
[![Architecture](https://img.shields.io/badge/Pattern-Core%20%2B%20Workers-6f42c1)](#arquitectura-resumen)
[![Status](https://img.shields.io/badge/Status-Beta-yellow)](#estado)
[![Perf](https://img.shields.io/badge/Optimized-Low%20CPU%20%26%20No%20long%20waits-brightgreen)](#rendimiento-y-buenas-practicas)

Automatización de **Sala de estar** con arquitectura **core liviano** + **workers** (scripts) para minimizar la carga en la Raspberry Pi cuando el sensor PIR dispara eventos todo el día. Soporta **grupo** (`group.luces_sala_de_estar`), **preaviso rápido o preciso**, y **preaviso en dos etapas (opcional)**. Comentarios al final del YAML para mantener el core limpio.

> Si buscás una versión con **áreas** o **multiárea**, ver _Roadmap_ más abajo.

---

## Tabla de contenidos

- [Resumen](#resumen)
- [Arquitectura (resumen)](#arquitectura-resumen)
- [Requisitos](#requisitos)
- [Instalación](#instalación)
- [Configuración rápida](#configuración-rápida)
- [Helpers y qué hace cada uno](#helpers-y-qué-hace-cada-uno)
- [Modos de preaviso](#modos-de-preaviso)
- [Qué ver en el Dashboard](#qué-ver-en-el-dashboard)
- [Migración desde v9.x](#migración-desde-v9x)
- [Solución de problemas](#solución-de-problemas)
- [Rendimiento y buenas prácticas](#rendimiento-y-buenas-prácticas)
- [Roadmap](#roadmap)
- [Créditos y licencia](#créditos-y-licencia)

---

## Resumen

- **Core**: automatizaciones cortas, sin `wait_for_trigger` largos; sólo arrancan/cancelan **timers** y delegan a scripts.
- **Workers**: scripts que hacen el trabajo pesado **sólo cuando hace falta** (snapshot, preaviso, restore, apagado).
- **Grupo**: todo se dirige a `group.luces_sala_de_estar` → **sin `expand(area)`** en el camino caliente.
- **Preaviso**: reducción **relativa** del brillo actual (no target absoluto).  
  - **FAST**: 1 llamada de `brightness_step_pct` al grupo (ultra liviano).  
  - **PRECISE**: cálculo por luz respetando **piso mínimo** (más control, más costo).  
  - **Etapa 2 (opcional)**: segunda reducción en un porcentaje del tiempo de preaviso.
- **UI/Debug**: `input_text.sala_estado` y `input_text.sala_evento` muestran estado y últimos eventos.

---

## Arquitectura (resumen)

```
motion_on → [Core] cancela timers → restore/encender → fin
motion_off → [Core] programa timers (total y delay preaviso) → fin

timer.preaviso_delay → [Core] snapshot → preaviso (FAST/Precise) → timer.preaviso → fin
timer.preaviso_stage2 → [Core] (si activo) aplica etapa 2 → fin
timer.preaviso (fin) → [Core] apagado → fin

motion_on DURANTE preaviso → [Core] cancela timers + restore → fin
```

**Timers**:  
- `apagado_sala` (UI total), `preaviso_delay_sala`, `preaviso_sala`, `preaviso_stage2_sala` (opcional), `preaviso_cooldown_sala`.

**Workers**:  
- `sala_worker_snapshot`, `sala_worker_preaviso_fast`, `sala_worker_preaviso_precise`, `sala_worker_restore`, `sala_worker_apagado`.

---

## Requisitos

- Home Assistant 2024.x o superior.
- Un **grupo** con las luces de la sala: `group.luces_sala_de_estar`.
- `sensor.parte_del_dia` (o equivalente) para elegir tiempos por franja.
- Un sensor de movimiento: `binary_sensor.motion_sala`.

> _Tip_: Si no tenés `sensor.parte_del_dia`, podés crear uno simple por `sun.sun` o usar un helper `input_select` temporalmente.

---

## Instalación

1. Copiá el YAML `motion_sala_v10_1_beta.yaml` a tu carpeta de **packages**.  
2. Verificá que existe `group.luces_sala_de_estar` (o ajustá el nombre en el YAML).  
3. **Recargá helpers** y **Recargá automatizaciones** (o reiniciá HA).  
4. Mirá en _Ajustes → Dispositivos y servicios → Helpers_ que se hayan creado los nuevos helpers.  

> Los **comentarios** están al final del YAML para no entorpecer el core.

---

## Configuración rápida

- Definí los **tiempos por franja** (`sala_off_*`).
- Ajustá el **preaviso (min)** y la **reducción %** de etapa 1.
- (Opcional) Activá **preaviso en 2 etapas** y seteá: reducción 2 % + ratio (% del tiempo).
- Elegí **FAST** (por defecto) o **PRECISE** según tu hardware/lámparas.
- (Opcional) Seteá **escenas** `sala_scene_madrugada` / `sala_scene_dia`.
- Mostrá en tu dashboard el **estado** y los **timers**.

---

## Helpers y qué hace cada uno

**Booleans**
- `input_boolean.mantener_luces_sala`: **Bypass** total de la automatización del área.
- `input_boolean.sala_snapshot_ready`: **Interno**. Flag de snapshot lista (no tocar).
- `input_boolean.sala_preaviso_preciso`: Fuerza preaviso **PRECISE** (por luz + piso mínimo).
- `input_boolean.sala_preaviso_doble_etapa`: Activa **dos etapas** de preaviso.

**Select**
- `input_select.sala_perfil`: (Normal/Película/Limpieza/Invitados). A futuro permitirá cambiar tiempos y escenas en bloque.

**Números — tiempos por franja**
- `input_number.sala_off_manana`, `sala_off_mediodia`, `sala_off_tarde`, `sala_off_noche`, `sala_off_madrugada`: minutos hasta apagado total por franja.

**Números — preaviso**
- `input_number.sala_preaviso`: duración del preaviso (min).
- `input_number.sala_preaviso_reduccion_pct`: **% de reducción** de la **etapa 1** (relativa al brillo actual).
- `input_number.sala_preaviso_reduccion2_pct`: **% de reducción** de la **etapa 2** (si está activada).
- `input_number.sala_preaviso_stage2_ratio`: punto de **disparo** de etapa 2 (% del tiempo de preaviso).
- `input_number.sala_preaviso_piso_min_pct`: **piso mínimo** de brillo (aplica en modo PRECISE).
- `input_number.sala_preaviso_cooldown_s`: **enfriamiento** para no re-aplicar preaviso seguido.

**Números — fallbacks y debounce**
- `input_number.sala_fallback_dia_pct`: brillo al **restaurar/encender de día** si no hay escena ni snapshot.
- `input_number.sala_fallback_madrugada_pct`: brillo **de madrugada** si no hay escena.
- `input_number.sala_motion_off_debounce_s`: debounce (seg) para considerar el **OFF** de motion.

**Textos**
- `input_text.sala_scene_madrugada`: `entity_id` de escena para **madrugada** (si vacío, usa fallback).
- `input_text.sala_scene_dia`: `entity_id` de escena para **día** (si vacío, usa fallback).
- `input_text.sala_estado`: string de **estado** (para UI/debug).
- `input_text.sala_evento`: **último evento** (para UI/debug).

---

## Modos de preaviso

- **FAST (por defecto)**  
  - Toggle: `sala_preaviso_preciso` **OFF**.  
  - `brightness_step_pct` al **grupo**.  
  - **Pro**: mínimo uso de CPU; una llamada.  
  - **Con**: puede variar por driver/marca (step global).

- **PRECISE**  
  - Toggle: `sala_preaviso_preciso` **ON**.  
  - Calcula el brillo actual de **cada luz**, aplica reducción y respeta **piso mínimo**.  
  - **Pro**: resultado consistente.  
  - **Con**: más carga (loop por luz).

- **Etapa 2 (opcional)**  
  - Toggle: `sala_preaviso_doble_etapa` **ON**.  
  - Dispara a `sala_preaviso_stage2_ratio` % del tiempo de preaviso y aplica `sala_preaviso_reduccion2_pct`.

---

## Qué ver en el Dashboard

- **Estado/evento** (`input_text.sala_estado` / `input_text.sala_evento`).
- **Timers**: `timer.apagado_sala` (total) y `timer.preaviso_sala` (contadores).  
- **Chips**: `mantener_luces_sala`, `sala_preaviso_preciso`, `sala_preaviso_doble_etapa`.  
- **Sliders**: `% reducción (1 y 2)`, `piso mínimo`, `preaviso min`, `fallbacks`, `debounce`.

---

## Migración desde v9.x

- El antiguo `input_number.sala_preaviso_target_pct` (target absoluto) fue reemplazado por
  `input_number.sala_preaviso_reduccion_pct` (**% de reducción relativa**).  
- Si usabas escenas custom, ahora hay **dos**: `sala_scene_madrugada` y `sala_scene_dia`.  
- El área fija fue reemplazada por **grupo** (`group.luces_sala_de_estar`).

**Pasos**:
1. Eliminar el viejo helper si existe (`sala_preaviso_target_pct`).  
2. Ajustar las referencias en Lovelace (sliders/labels).  
3. Recargar helpers y automatizaciones.

---

## Solución de problemas

- **El preaviso parece inconsistente** → probar **PRECISE**. Subir `piso_min` si tenés parpadeos.  
- **Se vuelve a aplicar varias veces** → subir `sala_preaviso_cooldown_s`.  
- **Se apaga “injustamente”** → subir `sala_motion_off_debounce_s` o alargar tiempos por franja.  
- **No restaura como esperaba** → verificar `sala_snapshot_ready` y que `scene.sala_snapshot` exista (UI: `Escenas`).  
- **Carga alta** → usar **FAST**, desactivar etapa 2, bajar número de luces del grupo o segmentar por subgrupos.

---

## Rendimiento y buenas prácticas

- Evitar `expand(area)` en hot-path (ya reemplazado por **grupo**).  
- Usar **FAST** por defecto; activar **PRECISE** sólo cuando haga falta.  
- Mantener **debounce** > 3–5 s para PIR sensibles.  
- No abusar de la etapa 2 si no es necesaria.  
- Revisar el log sólo si hay errores; el core termina cada ejecución rápido.

---

## Roadmap

- **Blueprint multiárea** (mismo core + workers parametrizados por `group.*`).  
- **Perfiles reales** (que `sala_perfil` cambie tiempos, reducciones y escenas en bloque).  
- **Telemetría ligera** (contador de cancelaciones de preaviso para auto-tuning).

---

## Créditos y licencia

- Autor: Hernán + asistencia ChatGPT (patrón Core/Workers)  
- Licencia: uso personal / MIT (ajustá a tu preferencia antes de subir al repo).
