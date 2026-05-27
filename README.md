# Ingreso-Analisis-bi
Análisis de Ingresos y Compras 2021-2025 | Power BI + SQL Server + DAX | Modelo estrella, clasificación dinámica de proveedores con ALLEXCEPT, DATESINPERIOD y funciones de ventana INDEX+PARTITIONBY. Datos anonimizados.
## 📌 Descripción del Proyecto

Solución de Business Intelligence para el análisis del comportamiento de compras e ingresos de una institución durante 5 años consecutivos (2021–2025). El objetivo es identificar patrones de gasto por proveedor, clasificar proveedores por volumen de compras, analizar la variación interanual de costos y segmentar el comportamiento por bodega y tipo de clasificación.

**Problema de negocio:** La institución necesitaba visibilidad sobre el crecimiento del gasto por proveedor, clasificar proveedores estratégicos (Exclusivo, Alto, Medio, Bajo), y monitorear la variación de costos año sobre año por bodega y tipo de ingreso.

**Resultado:** Dashboard interactivo de 2 páginas con KPIs ejecutivos, matriz de clasificación con semáforo visual, análisis de variación YoY y segmentación de proveedores de alto impacto.

---

## 🔐 Privacidad y Anonimización

| Dato original | Publicado como |
|---|---|
| Nombre de empresa/proveedor | `Empresa_XXXXXX` |
| Nombre de institución | `Institución` |
| Clasificación específica | Nombres genéricos |
| Bodega específica | `MATERIAL` / `MEDICAMENTO` |

El modelo, las medidas DAX y los hallazgos son 100% reales.

---

## 🛠️ Stack Tecnológico

| Herramienta | Uso |
|---|---|
| **SQL Server** | Almacenamiento, consolidación de 7 archivos Excel |
| **Power Query (M)** | ETL, normalización, modelo estrella, DimCalendario |
| **DAX** | 10+ medidas: KPIs, inteligencia de tiempo, clasificación |
| **Power BI Desktop** | Modelado y visualización interactiva |
| **Excel** | Fuente original (7 archivos 2021–2025) |

---

## 🗂️ Fuente de Datos

- **Origen:** 7 archivos Excel exportados del sistema institucional
- **Período:** 2021 – 2025
- **Volumen:** 7,371 registros
- **Columnas originales:** Clasificacion, Facturas, Fecha, Empresa, Costo, Bodega

---

## 🏗️ Arquitectura del Proyecto

### Flujo de Datos

```
7 Archivos Excel
      ↓
  SQL Server          ← Tabla consolidada INGRESO_TOTALES
      ↓
  Power Query         ← ETL + Anonimización + Modelo Estrella
      ↓
  Modelo Estrella     ← 1 tabla de hechos + 4 dimensiones
      ↓
  DAX (10+ medidas)   ← KPIs, clasificación, inteligencia de tiempo
      ↓
  Power BI Dashboard  ← 3 páginas de análisis interactivo
```

### Modelo Estrella

```
            ┌──────────────────┐
            │  DimCalendario   │
            └────────┬─────────┘
                     │
┌──────────┐ ┌───────┴──────────┐ ┌──────────────────┐
│DimBodega │─│                  │─│  DimClasificacion│
└──────────┘ │ fctINGRESO_      │ └──────────────────┘
             │ TOTALES          │
┌──────────┐ │                  │
│DimEmpresa│─│ • Id_Compras     │
└──────────┘ │ • COSTO          │
             │ • FACTURAS       │
             │ • Clasificacion_ │
             │   Compras        │
             └──────────────────┘
```

**Tablas del modelo:**
- `fctINGRESO_TOTALES` — Tabla de hechos con 7,371 registros
- `DimEmpresa` — Dimensión de proveedores/empresas
- `DimBodega` — Dimensión de tipo de bodega (Material/Medicamento)
- `DimClasificacion` — Dimensión de clasificación de compras
- `DimCalendario` — Tabla de tiempo (Año, Mes, Semana, Período)

---

## ⚙️ Proceso ETL — Power Query

1. **Carga** — 7 archivos Excel (.xlsx → .csv) importados a SQL Server
2. **Consolidación** — Tabla única `INGRESO_TOTALES` con INSERT INTO por año
3. **Duplicación** — Consulta principal duplicada 3 veces para construir dimensiones
4. **Deduplicación** — Eliminación de duplicados por columna clave en cada dimensión
5. **Llaves sustitutas** — Columna índice como surrogate key en cada dimensión
6. **Columna Id_Compras** — Llave única de transacción: `id_Empresa & "-" & FORMAT(FECHA,"YYYYMMDD")`
7. **Anonimización** — Empresas y clasificaciones reemplazadas por identificadores genéricos
8. **DimCalendario** — Generada con script M personalizado

---

## 📐 Medidas DAX Principales

```dax
-- Base
Total Costo = SUM(fctINGRESO_TOTALES[COSTO])
Total Facturas = COUNTROWS(fctINGRESO_TOTALES)
Costos Promedio X Factura = DIVIDE([Total Costo], [Total Facturas])

-- Inteligencia de tiempo
Costos Año Anterior = CALCULATE([Total Costo],
    SAMEPERIODLASTYEAR(DimCalendario[Date]))
Variacion % vs Año Anterior =
    DIVIDE([Total Costo] - [Costos Año Anterior], [Costos Año Anterior])
Costos Acumulado por Años = CALCULATE([Total Costo],
    DATESINPERIOD(DimCalendario[Date], MAX(DimCalendario[Date]), -5, YEAR))

-- Clasificación avanzada
Compras_5_años = CALCULATE(
    DISTINCTCOUNT(fctINGRESO_TOTALES[Id_Compras]),
    KEEPFILTERS(VALUES(DimCalendario[Date])))
```

> 📄 Ver documentación completa en [`/dax/medidas_documentadas.md`](./dax/medidas_documentadas.md)

---

## 📊 KPIs Principales (2021–2025)

| Métrica | Valor |
|---|---|
| **Costos Totales** | C$ 647,238,175.23 |
| **Total Facturas** | 7,371 |
| **Costo Promedio x Factura** | C$ 87,808.73 |
| **Período analizado** | 2021 – 2025 (5 años) |

---

## 📈 Páginas del Dashboard

### Página 1 — Resumen Ejecutivo
- 3 KPI Cards: Costos Totales, Total Facturas, Costo Promedio
- Gráfico de línea: Costos Acumulados por Años (MATERIAL vs MEDICAMENTO)
- Gráfico de barras: Total Compras por Año
- Gráfico de barras horizontales: Variación % vs Año Anterior por Año y Bodega

### Página 2 — Clasificación y Rankings
- Matriz con semáforo visual: Clasificación de Compras por Año (Alto/Medio/Bajo/Proveedor Exclusivo)
- Gráfico de barras: Suma de Monto por Empresa

### Página 3 — Análisis de Tendencias
- Análisis de proveedores de alto impacto
- Comparativo por bodega y clasificación

---

## 🔍 Hallazgos Clave

1. **2022 fue el año de mayor volumen** con 873 compras — 195% más que 2021 (295 compras).
2. **Tendencia decreciente 2022-2025** — de 873 a 683 compras, sugiere optimización de proveedores o reducción de demanda.
3. **MATERIAL vs MEDICAMENTO** muestran comportamientos diferentes — MATERIAL creció 260% en 2022 vs 135% de MEDICAMENTO.
4. **Proveedores Exclusivos** representan la categoría de mayor costo concentrado — candidatos prioritarios para negociación.

---

## 📁 Estructura del Repositorio

```
📁 ingresos-analisis-bi/
│
├── 📄 README.md
├── 📁 docs/
│   ├── 01_modelo_estrella.png
│   ├── 02_dashboard_pagina1.png
│   ├── 03_dashboard_pagina2.png
│   └── 04_dashboard_pagina3.png
├── 📁 dax/
│   └── medidas_documentadas.md
├── 📁 sql/
│   └── carga_ingreso.md
└── 📁 data/
    └── muestra_anonimizada.csv
```

---

## 🚀 Estado del Proyecto

| Componente | Estado |
|---|---|
| Modelo estrella ETL | ✅ Completado |
| Anonimización | ✅ Completado |
| Medidas DAX | ✅ Completado |
| Dashboard Página 1 | ✅ Completado |
| Dashboard Página 2 | ✅ Completado |
| Dashboard Página 3 | 🔄 En progreso |
| Publicación Power BI Service | ⏳ Pendiente |

---

## 👤 Autor

**Franklin Edicson Pastora Balladares**  
📧 franklin91.fepb@gmail.com  
🔗 [LinkedIn](https://www.linkedin.com/in/franklin-pastora)  
💻 [Portafolio GitHub](https://github.com/franklinPastora)  

**Ver también:** [Proyecto Farmacia Hospitalaria](https://github.com/franklinPastora/farmacia-hospital-bi)

---

*Datos reales de institución — Completamente anonimizados | Mayo 2026*
