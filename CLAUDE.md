# CLAUDE.md — Dashboard UEH · Hospital San José

## Qué es este proyecto

Dashboard web de turno para la Unidad de Emergencia Hospitalaria (UEH) del Hospital San José.
Procesa reportes de enfermería enviados por WhatsApp, extrae métricas y las persiste en Google Sheets.

Toda la aplicación vive en **un único archivo**: `index.html` (HTML + CSS + JS inline).
No hay build step, bundler ni dependencias npm. Para probar, abre el archivo en un navegador.

## Arquitectura

```
index.html
  ├── <style>          CSS con design tokens en :root
  ├── Tab "Captura"    Textarea → parseReport() → renderExtracted() → saveRecord()
  ├── Tab "Gráficos"   renderCharts() con Chart.js 4.4.1
  └── <script>
        ├── CONSTANTS  API URL, HEADERS array (columnas Google Sheets)
        ├── parseReport(text)  → objeto con todos los campos
        ├── calcKPIs(r)        → { presion, ratio, pct_box }
        ├── calcISUEH(r)       → { score 0-9, label, cls }
        ├── getStatus(r)       → { icon, label, cls }
        ├── API helpers        loadRecords / saveRecord / deleteRecord / wipeAll
        └── renderTable / renderCharts / exportCharts
```

## Modelo de datos

`HEADERS` define el orden exacto de columnas en Google Sheets:

| Grupo | Campos |
|---|---|
| Identificación | ts, turno, fecha, hora, jefe |
| Proceso general | total_proceso, espera, tmax_espera, ambulancias_espera |
| ESI en espera (n) | esi1_esp_n, esi2_esp_n, esi3_esp_n, esi45_esp_n |
| ESI en espera (h) | esi1_esp_h, esi2_esp_h, esi3_esp_h, esi45_esp_h |
| Espera por especialidad | med_espera, cir_espera, tmt_espera |
| Ambulatorio atención | atencion, box_lt360, box_gt360, tmax_atencion, ambulancias_aten |
| ESI en atención (n) | esi1_aten_n, esi2_aten_n, esi3_aten_n, esi45_aten_n |
| Hospitalizados | hosp, uci, uti, uco, pabellon, med_hosp, cir_hosp |
| Unidades boarding | uro_tmt, cmq, uce, hosdom, ugcc, hosmet, alta |

`ts` es timestamp Unix en milisegundos (string). Es la clave primaria para delete.

## KPIs — fórmulas exactas

```js
presion  = hosp > 0 ? espera / hosp : 0          // Presión de camas
ratio    = espera > 0 ? atencion / espera : 0      // Ratio ambulatorio
pct_box  = atencion > 0 ? (box_gt360 / atencion) * 100 : 0  // % Box > 6h
```

**IS-UEH** (Índice de Saturación, escala 0–9):

| Dimensión | 0 | 1 | 2 | 3 |
|---|---|---|---|---|
| d1 hosp | < 45 | 45–55 | 56–65 | > 65 |
| d2 %box | < 10% | 10–15% | 15–20% | > 20% |
| d3 ratio | > 1.0 | 0.8–1.0 | 0.5–0.8 | < 0.5 |

`score = d1 + d2 + d3` → Normal (0–1) / Tensión (2–3) / Saturado (4–6) / Crítico (7–9)

**Estado turno** (semáforo):
- Óptimo 🟢: presion < 1 AND ratio ≥ 1
- Crítico 🔴: presion ≥ 1 AND ratio < 1
- Intermedio 🟡: cualquier otro caso

## Google Sheets / API

- Endpoint: `https://script.google.com/macros/s/AKfycbwdOX-pNUtrGzrybfrOXX84IyBCPdqOLvcG2uSF2hHpnnSSuE0-OpI1nksEp45C2ItK/exec`
- Google Sheet: `https://docs.google.com/spreadsheets/d/155OvZG0JbjfA7Sl9ISApUeaWcDakI_d7BYjyCLNLFEE/`
- Protocolo: GET con `?payload=<JSON encodeURIComponent>`
- Acciones: `{action:'get'}` / `{action:'add', record:{...}}` / `{action:'delete', ts:'...'}` / `{action:'wipe'}`
- El Apps Script responde siempre JSON con `{ok: true/false, ...}`

## Parser

`parseReport(text)` usa regex sobre el texto plano del reporte WhatsApp.
El texto se divide en dos bloques en el emoji `🅰️` (o `🅰` sin variation selector):
- **blockEsp**: todo lo anterior → datos de espera y ESI en espera
- **blockAten**: todo lo posterior → datos de atención ambulatoria

Tolerancias del parser: fechas con mes en texto ("16-Abril"), espacios en horas, coma decimal.

## Gráficos

Todos son `Chart.js` tipo `line`. Los cuatro activos en la vista:

| ID canvas | Datos | Color |
|---|---|---|
| ch-espera-atencion | espera + atencion (dual) | rojo + azul |
| ch-hosp | hosp | ámbar |
| ch-pctbox | pct_box (%) | púrpura, alerta en 30% |
| ch-unidades | med_hosp + uti + pabellon | azul + verde + ámbar |

Filtro activo (`activeFilter`): 'Todos' | 'A' | 'B' | 'C' | 'D' — filtra por campo `turno`.
El último punto de cada serie se pinta en rojo (`#dc2626`) y radio 5 para destacar el turno más reciente.

## Convenciones al editar

- Todos los valores numéricos se procesan con `num(v)` — convierte string a float, tolerando coma decimal.
- El orden de `HEADERS` es sagrado: define la columna en Google Sheets. No reordenar.
- Al agregar un campo nuevo: añadir a HEADERS, al parser, a renderExtracted() y a renderCharts() si aplica.
- No usar módulos ES6 ni import/export — el archivo se abre directo en el navegador sin servidor.
- CSS: usar las variables de `:root` (`--blue`, `--red`, etc.), no colores hardcodeados.

## Versiones

- v1.0: Parser + KPIs + tabla + gráficos + sync Google Sheets
- v1.1 (actual): Campos extraídos editables, recálculo KPIs en tiempo real, exportación JPEG 2×2
