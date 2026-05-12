# Especificación Técnica — Integración Dashboard UEH
**Hospital San José · Unidad de Emergencia Hospitalaria**
**Versión 1.1 · Mayo 2026**

---

## 1. Propósito

Este documento especifica los indicadores, visualizaciones y datos que el sistema de TI del hospital debe exponer para integrar las vistas del Dashboard UEH en la plataforma institucional. El dashboard actual está operativo como aplicación web standalone (`index.html`) y persiste los datos en Google Sheets. Este documento define exactamente qué mostrar para reproducir o integrar esas vistas.

---

## 2. Fuente de datos

| Atributo | Valor |
|---|---|
| **Tipo** | Google Sheets via Google Apps Script |
| **Endpoint actual** | `https://script.google.com/macros/s/AKfycbwdOX-pNUtrGzrybfrOXX84IyBCPdqOLvcG2uSF2hHpnnSSuE0-OpI1nksEp45C2ItK/exec` |
| **Protocolo** | HTTP GET con parámetro `payload` (JSON URL-encoded) |
| **Autenticación** | Ninguna (script público) |
| **Google Sheet** | ID `155OvZG0JbjfA7Sl9ISApUeaWcDakI_d7BYjyCLNLFEE` |
| **Hoja** | `gid=181251294` |

### 2.1 Estructura de un registro

Cada turno genera un registro con los siguientes campos. Todos son strings en la API; los numéricos deben castearse a float al consumirlos.

| Campo | Tipo lógico | Descripción |
|---|---|---|
| `ts` | integer (ms) | Timestamp Unix en milisegundos. **Clave primaria.** |
| `turno` | string | Letra del turno: A / B / C / D |
| `fecha` | string | Formato `DD/MM/YYYY` |
| `hora` | string | Formato `HH:MM` (24 h) |
| `jefe` | string | Nombre de EU jefe de turno |
| `total_proceso` | integer | Pacientes totales en el proceso UEH |
| `espera` | integer | Pacientes en espera de atención |
| `tmax_espera` | integer | Tiempo máximo de espera (horas) |
| `ambulancias_espera` | integer | Ambulancias con pacientes en espera |
| `esi1_esp_n` | integer | Pacientes ESI 1 en espera (cantidad) |
| `esi2_esp_n` | integer | Pacientes ESI 2 en espera (cantidad) |
| `esi3_esp_n` | integer | Pacientes ESI 3 en espera (cantidad) |
| `esi45_esp_n` | integer | Pacientes ESI 4/5 en espera (cantidad) |
| `esi1_esp_h` | integer | Horas de espera ESI 1 (máx) |
| `esi2_esp_h` | integer | Horas de espera ESI 2 (máx) |
| `esi3_esp_h` | integer | Horas de espera ESI 3 (máx) |
| `esi45_esp_h` | integer | Horas de espera ESI 4/5 (máx) |
| `med_espera` | integer | Pacientes Medicina en espera |
| `cir_espera` | integer | Pacientes Cirugía en espera |
| `tmt_espera` | integer | Pacientes TMT en espera |
| `atencion` | integer | Pacientes en proceso de atención (ambulatorio) |
| `box_lt360` | integer | Pacientes en box < 6 horas (< 360 min) |
| `box_gt360` | integer | Pacientes en box > 6 horas (> 360 min) |
| `tmax_atencion` | string | Tiempo máximo en atención. Ej: `"4h30m"` |
| `ambulancias_aten` | integer | Usuarios de ambulancia en atención |
| `esi1_aten_n` | integer | Pacientes ESI 1 en atención |
| `esi2_aten_n` | integer | Pacientes ESI 2 en atención |
| `esi3_aten_n` | integer | Pacientes ESI 3 en atención |
| `esi45_aten_n` | integer | Pacientes ESI 4/5 en atención |
| `hosp` | integer | Total hospitalizados en UEH (boarding) |
| `uci` | integer | Hospitalizados con criterio UCI |
| `uti` | integer | Hospitalizados con criterio UTI |
| `uco` | integer | Hospitalizados con criterio UCO |
| `pabellon` | integer | Hospitalizados en espera de Pabellón |
| `med_hosp` | integer | Hospitalizados con criterio Medicina |
| `cir_hosp` | integer | Hospitalizados con criterio Cirugía |
| `uro_tmt` | integer | Hospitalizados URO/TMT |
| `cmq` | integer | Hospitalizados CMQ |
| `uce` | integer | Hospitalizados UCE |
| `hosdom` | integer | Hospitalizados HOSDOM |
| `ugcc` | integer | Hospitalizados UGCC |
| `hosmet` | integer | Hospitalizados HOSMET |
| `alta` | integer | Pacientes con alta pendiente |

---

## 3. Indicadores calculados (KPIs)

Estos valores **no se almacenan** en la fuente de datos. Se calculan en tiempo real a partir de los campos crudos.

### 3.1 Presión de camas

```
presion = hosp > 0 ? espera / hosp : 0
```

| Valor | Interpretación | Color |
|---|---|---|
| < 0.70 | Holgado | Verde `#15803d` |
| 0.70 – 0.99 | Moderado | Ámbar `#b45309` |
| ≥ 1.00 | Presión alta | Rojo `#b91c1c` |

### 3.2 Ratio ambulatorio

```
ratio = espera > 0 ? atencion / espera : 0
```

| Valor | Interpretación | Color |
|---|---|---|
| ≥ 1.00 | Capacidad suficiente | Verde |
| 0.80 – 0.99 | Tensión leve | Ámbar |
| < 0.80 | Déficit de atención | Rojo |

### 3.3 Porcentaje Box > 6 horas

```
pct_box = atencion > 0 ? (box_gt360 / atencion) * 100 : 0
```

| Valor | Interpretación | Color |
|---|---|---|
| < 10% | Aceptable | Verde |
| 10% – 19.9% | Alerta | Ámbar |
| ≥ 20% | Crítico | Rojo |

### 3.4 IS-UEH — Índice de Saturación UEH (escala 0–9)

Compuesto por tres dimensiones (0–3 puntos cada una):

**Dimensión 1 — Boarding (d1)**
| Condición | Puntos |
|---|---|
| hosp < 45 | 0 |
| 45 ≤ hosp ≤ 55 | 1 |
| 56 ≤ hosp ≤ 65 | 2 |
| hosp > 65 | 3 |

**Dimensión 2 — % Box > 6h (d2)**
| Condición | Puntos |
|---|---|
| pct_box < 10% | 0 |
| 10% ≤ pct_box ≤ 15% | 1 |
| 15% < pct_box ≤ 20% | 2 |
| pct_box > 20% | 3 |

**Dimensión 3 — Ratio ambulatorio (d3)**
| Condición | Puntos |
|---|---|
| ratio > 1.0 | 0 |
| 0.80 ≤ ratio ≤ 1.0 | 1 |
| 0.50 ≤ ratio < 0.80 | 2 |
| ratio < 0.50 | 3 |

```
IS-UEH = d1 + d2 + d3
```

| Puntaje | Estado | Color |
|---|---|---|
| 0–1 | Normal | Verde |
| 2–3 | Tensión | Ámbar |
| 4–6 | Saturado | Naranja |
| 7–9 | Crítico | Rojo |

### 3.5 Estado del turno (semáforo)

Derivado de presion y ratio:

| Condición | Estado | Icono |
|---|---|---|
| presion < 1 AND ratio ≥ 1 | Óptimo | 🟢 |
| presion ≥ 1 AND ratio < 1 | Crítico | 🔴 |
| Cualquier otro caso | Intermedio | 🟡 |

---

## 4. Vistas requeridas

### 4.1 Tabla de historial de turnos

Mostrar todos los registros ordenados por fecha/hora descendente (más reciente primero). El registro más reciente debe destacarse visualmente (fondo azul claro).

**Columnas a mostrar:**

| Columna | Campo(s) | Formato |
|---|---|---|
| # | — | Número de fila |
| Fecha | `fecha` | DD/MM/YYYY |
| T | `turno` | Badge: A / B / C / D |
| Hora | `hora` | HH:MM |
| Jefe | `jefe` | Texto |
| Espera | `espera` | Entero |
| Aten. | `atencion` | Entero |
| Hosp | `hosp` | Entero |
| Box <6h | `box_lt360` | Entero |
| Box >6h | `box_gt360` | Entero |
| Presión | calculado | 2 decimales, color por umbral |
| Ratio | calculado | 2 decimales, color por umbral |
| %Box | calculado | 1 decimal + %, color por umbral |
| IS-UEH | calculado | Puntaje 0–9 con badge de color |
| Estado | calculado | Badge: Óptimo / Intermedio / Crítico |

### 4.2 Gráficos de tendencia

Todos son gráficos de línea, eje X = fecha/hora de cada turno, ordenados cronológicamente. Deben poder filtrarse por turno (Todos / A / B / C / D).

---

#### Gráfico 1 — Pacientes en Espera y Atención

**Tipo:** Línea dual  
**Series:**
- `espera` → color rojo `#dc2626`
- `atencion` → color azul `#1d4ed8`

**Eje Y:** Número de pacientes (entero, desde 0)  
**Leyenda:** visible (Espera / Atención)  
**Nota:** El punto más reciente de cada serie debe destacarse (mayor tamaño, color diferenciado)

---

#### Gráfico 2 — Hospitalizados (Boarding)

**Tipo:** Línea única  
**Serie:** `hosp` → color ámbar `#b45309`  
**Eje Y:** Número de pacientes hospitalizados en UEH (desde 0)  
**Leyenda:** no necesaria

---

#### Gráfico 3 — % Box > 6 horas

**Tipo:** Línea única con línea de alerta  
**Serie principal:** `pct_box` (calculado) → color púrpura `#7c3aed`  
**Línea de alerta:** valor fijo 30% → gris punteado `#9ca3af`  
**Eje Y:** Porcentaje (0–100, con sufijo %)  
**Leyenda:** no necesaria

---

#### Gráfico 4 — Hospitalizados por Unidad

**Tipo:** Línea múltiple  
**Series:**
- `med_hosp` → "Medicina" → azul `#1d4ed8`
- `uti` → "UTI" → verde `#15803d`
- `pabellon` → "Pabellón" → ámbar `#b45309`

**Eje Y:** Número de hospitalizados por criterio (desde 0), con label "Pacientes hospitalizados"  
**Leyenda:** visible (Medicina / UTI / Pabellón)

---

## 5. Paleta de colores institucional

| Variable | Hex | Uso |
|---|---|---|
| Azul primario | `#1d4ed8` | Cabecera, botones, turno activo |
| Azul oscuro | `#1e40af` | Hover |
| Azul claro | `#dbeafe` | Fila más reciente |
| Verde | `#15803d` | Estado óptimo, valor seguro |
| Ámbar | `#b45309` | Estado intermedio, alerta |
| Naranja | `#c2410c` | IS-UEH Saturado |
| Rojo | `#b91c1c` | Estado crítico |
| Rojo vivo | `#dc2626` | Punto más reciente en gráfico |
| Púrpura | `#7c3aed` | % Box >6h |
| Gris fondo | `#f9fafb` | Fondo general |
| Gris borde | `#e5e7eb` | Bordes de tarjetas |

**Tipografía:** IBM Plex Sans (UI) + IBM Plex Mono (valores numéricos)

---

## 6. Frecuencia y actualización de datos

- Los reportes se ingresan **4 veces al día** (turnos A, B, C, D).
- No hay actualización automática en tiempo real; el dashboard carga los datos al abrirse.
- Si el sistema de TI consume la API directamente, se recomienda un TTL de caché de 30 minutos mínimo para no saturar el Apps Script.

---

## 7. Exportación

El dashboard actual permite exportar los 4 gráficos como imagen JPEG (1600×1060 px) con:
- Cabecera institucional azul con nombre del hospital y turno activo
- 4 gráficos en grilla 2×2
- Pie de página con timestamp

Si el sistema de TI ofrece exportación propia, no es obligatorio replicar este formato, pero sí incluir los 4 gráficos especificados en la sección 4.2.

---

## 8. Contacto

Desarrollado por la Unidad de Emergencia Hospitalaria — Hospital San José.  
Responsable clínico: EU jefe UEH / Subdirección de Enfermería.
