# Kasumigaura Lakehouse PoC

> Arquitectura Medallion (Bronze → Silver → Gold) aplicada al monitoreo del Lago Kasumigaura (Japón, 1977–2024).

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

**TFM** · Máster en Análisis y Visualización de Datos Masivos — UNIR  
**Autora:** Kathiana Murillo García (2026–2027)

---

## ¿De qué trata este proyecto?

Este proyecto construye un pipeline de datos completo para analizar **47 años de registros de calidad del agua** del Lago Kasumigaura, uno de los lagos más monitoreados de Japón.

Usando una **Arquitectura Medallion** (tres capas de datos progresivamente más refinadas), el pipeline pasa de un archivo Excel crudo del NIES hasta indicadores analíticos listos para interpretar.

**Resultados del pipeline ejecutado en Databricks:**

| Capa | Registros | Detalle |
|------|-----------|---------|
| 🥉 Bronze | 6,220 | Datos raw del Excel NIES (1977–2024) |
| 🥈 Silver | 6,218 VALID · 2 ANOMALY | 99.97% de registros válidos |
| 🥇 Gold Ox | 3,158 | Índice de oxigenación mensual por estación |
| 🥇 Gold Eu | 410 | Tendencia de eutrofización anual (1984–2024) |

---

## Arquitectura del pipeline

```
┌─────────────────────────────────────────────────────────────────┐
│                     FUENTE DE DATOS                             │
│   NIES — Lake Kasumigaura Database                              │
│   2_water quality_eng_v20250925.xlsx                            │
│   6,220 registros · 83 variables · 10 estaciones · 1977–2024   │
└───────────────────────────┬─────────────────────────────────────┘
                            │  pandas read_excel
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥉  CAPA BRONZE — Ingesta raw                                  │
│  Notebook: 01_bronze_ingesta.ipynb                              │
│                                                                 │
│  • Lee el Excel con pandas (sheet "water quality", header=0)    │
│  • Normaliza nombres de columna (clean_col_name)                │
│  • Convierte a Spark DataFrame y añade metadatos:               │
│      _ingestion_ts · _dataset_version · _source_url             │
│  • Escribe en Delta Lake (modo overwrite, idempotente)          │
│                                                                 │
│  Resultado: workspace.kasumigaura.bronze_water_quality          │
│             6,220 filas · 87 columnas                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │  spark.table(TABLE_BRONZE)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥈  CAPA SILVER — Validación de calidad                        │
│  Notebook: 02_silver_calidad.ipynb                              │
│                                                                 │
│  Aplica 10 reglas físico-químicas a cada registro:              │
│  ┌────┬──────────────────────────┬──────────────────────────┐   │
│  │ R1 │ Estación (ST)            │ No nula                  │   │
│  │ R2 │ Fecha (Date)             │ No nula                  │   │
│  │ R3 │ Oxígeno disuelto (DO0m)  │ 0–28 mg/L                │   │
│  │ R4 │ pH superficial (pH0m)    │ 6.0–11.0                 │   │
│  │ R5 │ Temperatura agua (WT0m)  │ 0–35 °C                  │   │
│  │ R6 │ Fósforo total (T_P)      │ 0–1,300 µg/L             │   │
│  │ R7 │ Nitrógeno total (T_N)    │ 0–5,000 µg/L             │   │
│  │ R8 │ Transparencia Secchi     │ > 0 cm                   │   │
│  │ R9 │ Clorofila-a (Chl_a)      │ 0–500 µg/L               │   │
│  │R10 │ Duplicados (ST + Date)   │ Solo primer registro     │   │
│  └────┴──────────────────────────┴──────────────────────────┘   │
│                                                                 │
│  Etiqueta cada registro con dq_flag:                            │
│    VALID      → pasa todas las reglas          → 6,218 (99.97%)│
│    ANOMALY    → valor fuera de rango           →     2 ( 0.03%)│
│    NULL_KEY   → ST o Date nulos                →     0          │
│    DUPLICATE  → ST + Date repetidos            →     0          │
│                                                                 │
│  Resultado: workspace.kasumigaura.silver_water_quality          │
└───────────────────────────┬─────────────────────────────────────┘
                            │  filter(dq_flag == "VALID")
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│  🥇  CAPA GOLD — Indicadores analíticos (Spark SQL)             │
│  Notebook: 03_gold_indicadores.ipynb                            │
│                                                                 │
│  Indicador 1 — Índice de Oxigenación Mensual                    │
│  ─────────────────────────────────────────────                  │
│  Promedio mensual de DO0m por estación, clasificado en:         │
│    optimo          → DO0m ≥ 6 mg/L                              │
│    aceptable       → 4 ≤ DO0m < 6 mg/L                         │
│    hipoxia_critica → DO0m < 4 mg/L                              │
│  → workspace.kasumigaura.gold_oxigenacion_mensual (3,158 reg.)  │
│                                                                 │
│  Indicador 2 — Tendencia de Eutrofización Anual                 │
│  ─────────────────────────────────────────────                  │
│  Promedio anual de T_P, T_N y Chl_a por estación (1984–2024),  │
│  clasificado en:                                                │
│    oligotrofico    → T_P < 10 µg/L                             │
│    mesotrofico     → 10 ≤ T_P < 35 µg/L                        │
│    eutrofico       → 35 ≤ T_P < 100 µg/L                       │
│    hipereutrofico  → T_P ≥ 100 µg/L                            │
│  → workspace.kasumigaura.gold_eutrofizacion_anual (410 reg.)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Stack tecnológico

| Herramienta | Versión | Uso |
|-------------|---------|-----|
| Python + pandas | 3.12 | Lectura del Excel NIES |
| Apache Spark (PySpark) | 3.x | Motor de procesamiento distribuido |
| Delta Lake | — | Almacenamiento ACID por capas |
| Spark SQL | — | Construcción de indicadores Gold |
| Databricks Community Edition | — | Entorno cloud de ejecución |
| matplotlib | — | Visualización de resultados |

---

## Estructura del repositorio

```
kasumigaura-lakehouse-poc/
├── data/
│   └── 2_water quality_eng_v20250925.xlsx   # Dataset NIES (no incluido en el repo)
├── notebooks/
│   ├── 01_bronze_ingesta.ipynb              # Ingesta → Delta Lake Bronze
│   ├── 02_silver_calidad.ipynb              # Validación → Delta Lake Silver
│   └── 03_gold_indicadores.ipynb            # Indicadores Spark SQL → Gold
└── README.md
```

> **Nota:** el archivo Excel no está en el repositorio por tamaño. Descárgalo desde la [URL oficial del NIES](https://db.cger.nies.go.jp/gem/moni-e/inter/GEMS/database/kasumi/contents/datalist.html) y colócalo en la carpeta `data/`.

---

## Cómo ejecutar

1. Sube el notebook a **Databricks Community Edition**
2. Coloca el Excel en `/Workspace/Users/<tu-email>/kasumigaura-lakehouse-poc/data/`
3. Ejecuta los notebooks **en orden**: `01` → `02` → `03`

Cada notebook detecta automáticamente la ruta del archivo y el catálogo activo.

---

## Dataset

- **Fuente:** [Lake Kasumigaura Database — NIES](https://db.cger.nies.go.jp/gem/moni-e/inter/GEMS/database/kasumi/contents/datalist.html)
- **Período:** 1977–2024
- **Registros:** 6,220 filas · 83 variables · 10 estaciones de monitoreo
- **Citación:** NIES (2025). *Lake Kasumigaura Database*. National Institute for Environmental Studies, Japan.

Consulta los [Terms of Use del NIES](https://db.cger.nies.go.jp/gem/moni-e/inter/GEMS/database/kasumi/contents/datalist.html) antes de usar los datos.

---

## Licencia

El código fuente está bajo la [licencia MIT](LICENSE).  
Los datos del NIES se usan bajo sus propios términos de uso académico.
