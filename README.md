# kasumigaura-lakehouse-poc

Kasumigaura Lakehouse PoC

TFM — Máster Universitario en Análisis y Visualización de Datos Masivos / Visual Analytics and Big Data
Universidad Internacional de La Rioja (UNIR)

Descripción

Prueba de Concepto (PoC) de una arquitectura Lakehouse con Medallion Architecture (Bronze → Silver → Gold) aplicada al dominio del monitoreo ambiental hídrico.

El proyecto automatiza la ingesta, validación de calidad físico-química y explotación analítica de datos históricos de calidad del agua del Lago Kasumigaura (Japón), utilizando el dataset público del Instituto Nacional de Estudios Ambientales de Japón (NIES), con registros de monitoreo continuo desde 1977 hasta 2024.

Autora

Kathiana Murillo García
Máster en Análisis y Visualización de Datos Masivos — UNIR (2025–2027)

Dataset
Fuente: Lake Kasumigaura Database — NIES (National Institute for Environmental Studies, Japan)
Archivo: 2_water quality_eng_v20250925.xlsx (2,22 MB)
Período: 1977–2024
Registros: 6.220 filas · 83 variables · 10 estaciones de monitoreo
URL oficial: https://db.cger.nies.go.jp/gem/moni-e/inter/GEMS/database/kasumi/contents/datalist.html

Leer los Terms of Use del NIES antes de usar los datos.

Stack tecnológico
Herramienta	Uso
Python 3.x + pandas	Lectura del Excel y transformaciones
Apache Spark (PySpark)	Motor de procesamiento distribuido
Delta Lake	Almacenamiento transaccional ACID (capas Bronze, Silver, Gold)
Spark SQL	Construcción de indicadores analíticos en Gold
Databricks Community Edition	Entorno de desarrollo en la nube
matplotlib	Visualización de resultados
Estructura del repositorio
kasumigaura-lakehouse-poc/
├── data/
│   └── 2_water quality_eng_v20250925.xlsx   # Dataset NIES
├── notebooks/
│   ├── 01_bronze_ingesta.ipynb              # Ingesta y conversión a Delta Lake
│   ├── 02_silver_calidad.ipynb              # Reglas de calidad físico-química
│   └── 03_gold_indicadores.ipynb            # Indicadores analíticos Spark SQL
└── README.md
Arquitectura Medallion
NIES Excel
    │
    ▼
┌─────────────────────────────────────┐
│  BRONZE — Ingesta raw               │
│  Datos tal como llegan del NIES     │
│  Formato Delta Lake, append-only    │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  SILVER — Calidad y estandarización │
│  Reglas físico-químicas (pH, DO,    │
│  temperatura, nutrientes)           │
│  Flag: VALID / NULL / ANOMALY /     │
│         DUPLICATE                   │
└──────────────────┬──────────────────┘
                   │
                   ▼
┌─────────────────────────────────────┐
│  GOLD — Indicadores analíticos      │
│  Indicador 1: Índice de Oxigenación │
│  Indicador 2: Tendencia Eutrofiz.   │
│  Construidos con Spark SQL          │
└─────────────────────────────────────┘
Variables clave del dataset
Variable	Nombre en dataset	Unidad	Uso en pipeline
Oxígeno disuelto (superficie)	DO0m	mg/L	Silver + Gold Ind.1
pH (superficie)	pH0m	—	Silver
Temperatura agua (superficie)	WT0m	°C	Silver
Fósforo total	T-P	µg/L	Silver + Gold Ind.2
Nitrógeno total	T-N	µg/L	Silver + Gold Ind.2
Clorofila-a	Chl-a	µg/L	Silver + Gold Ind.2
Transparencia (Secchi)	Transp(Secchi depth)	cm	Silver
Conductividad eléctrica	E.C.	µS/cm	Silver
Estación de monitoreo	ST.	—	Todas las capas
Fecha de muestreo	Date	—	Todas las capas
Indicadores analíticos (Gold)

Indicador 1 — Índice de Oxigenación Mensual por Estación
Promedio mensual de DO0m por estación, clasificado en tres estados ecológicos:

hipoxia_critica: DO0m < 4 mg/L
aceptable: 4 ≤ DO0m < 6 mg/L
optimo: DO0m ≥ 6 mg/L

Indicador 2 — Tendencia de Eutrofización Anual
Agregación anual de T-P, T-N y Chl-a por estación (1984–2024), para detectar patrones históricos de degradación ecológica.

Alcance del PoC

Este proyecto es una Prueba de Concepto (PoC). No contempla:

Despliegue en producción
Procesamiento en streaming
Orquestación automática (Apache Airflow)
Modelos de Machine Learning o LLMs
Licencia y citación del dataset

Los datos del NIES se utilizan bajo los términos de uso académico del Lake Kasumigaura Database. Citar como:

NIES — National Institute for Environmental Studies (2025). Lake Kasumigaura Database. https://db.cger.nies.go.jp/gem/moni-e/inter/GEMS/database/kasumi/
