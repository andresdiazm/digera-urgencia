# Dashboard UEH — Hospital San José

Dashboard web de turno para la Unidad de Emergencia Hospitalaria (UEH). Procesa los reportes de enfermería enviados por WhatsApp, extrae métricas clave y las persiste en Google Sheets para análisis histórico y visualización de tendencias.

## Funcionalidades

- **Captura de reportes**: pega el texto del reporte de turno y extrae automáticamente los valores mediante expresiones regulares.
- **KPIs calculados en tiempo real**:
  - Presión de camas (espera / hospitalizados)
  - Ratio ambulatorio (atención / espera)
  - % Box > 6 h
  - Índice de Saturación UEH (IS-UEH, escala 0–9)
- **Historial de registros**: tabla ordenada por fecha/hora con estado de turno (Óptimo / Intermedio / Crítico).
- **Gráficos de tendencia** filtrados por turno (A / B / C / D):
  - Pacientes en espera y en atención
  - Hospitalizados (boarding)
  - % Box > 6 h
  - Hospitalizados por unidad (Medicina / UTI / Pabellón)
- **Sincronización con Google Sheets** mediante Google Apps Script (GET con payload JSON).

## Estructura del proyecto

```
Report urgencia/
└── index.html   # Aplicación completa (HTML + CSS + JS en un único archivo)
```

## Uso

1. Abre `index.html` en cualquier navegador moderno (no requiere servidor local).
2. En la pestaña **① Captura**, pega el reporte de WhatsApp y presiona **Procesar**.
3. Revisa los valores extraídos y presiona **Guardar** para enviarlos a Google Sheets.
4. En la pestaña **② Gráficos**, filtra por turno y revisa las tendencias históricas.

## Formato esperado del reporte

El parser reconoce el formato estándar del reporte de enfermería de turno. Campos clave:

| Patrón en el texto | Campo extraído |
|---|---|
| `Turno A 26-04` | turno, fecha |
| `Reporte 08:00 hrs` | hora |
| `Total: 45` | total_proceso |
| `Espera de Atención: 12` | espera |
| `ESI 1 = 2` | esi1_esp_n |
| `(50 hospitalizados)` | hosp |
| `Con criterio: UCI: 3 …` | unidades de boarding |
| `EU Nombre Apellido` | jefe de turno |

## Dependencias externas

| Recurso | Uso |
|---|---|
| [Chart.js 4.4.1](https://www.chartjs.org/) | Gráficos de línea |
| IBM Plex Sans / IBM Plex Mono | Tipografía (Google Fonts) |
| Google Apps Script | Backend de persistencia (Google Sheets) |

## IS-UEH — Índice de Saturación

| Puntuación | Estado |
|---|---|
| 0–1 | Normal |
| 2–3 | Tensión |
| 4–6 | Saturado |
| 7–9 | Crítico |

Compuesto por tres dimensiones (0–3 cada una): hospitalizados totales, % box > 6 h y ratio ambulatorio.
