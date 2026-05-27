# 📐 Medidas DAX — Proyecto Análisis de Ingresos y Compras

> **Proyecto:** Análisis de Ingresos y Compras Institucional  
> **Autor:** Franklin Edicson Pastora Balladares  
> **Herramientas:** Power BI Desktop | DAX | SQL Server | Power Query  
> **Período:** 2021 – 2025 | 7,371 registros  
> **Nota:** Datos completamente anonimizados.

---

## 📑 Índice

1. [Medidas Base](#1-medidas-base)
2. [Medidas de Inteligencia de Tiempo](#2-medidas-de-inteligencia-de-tiempo)
3. [Medidas de Clasificación de Proveedores](#3-medidas-de-clasificación-de-proveedores)
4. [Columnas Calculadas](#4-columnas-calculadas)
5. [Tablas Calculadas](#5-tablas-calculadas)
6. [Resumen del Arsenal DAX](#6-resumen-del-arsenal-dax)

---

## 1. Medidas Base

---

### `Total Costo`

```dax
Total Costo = SUM(fctINGRESO_TOTALES[COSTO])
```

**¿Qué hace?**
Suma el costo total de todos los ingresos en el contexto de filtro activo.

**Resultado:** C$ 647,238,175.23 (acumulado 2021–2025)

---

### `Total Facturas`

```dax
Total Facturas = COUNTROWS(fctINGRESO_TOTALES)
```

**¿Qué hace?**
Cuenta el número total de registros de facturas. Se usa `COUNTROWS` en lugar de `COUNT` para mayor precisión — cuenta todas las filas incluyendo valores nulos en otras columnas.

**Resultado:** 7,371 facturas totales

---

### `Costos Promedio X Factura`

```dax
Costos Promedio X Factura = DIVIDE([Total Costo], [Total Facturas])
```

**¿Qué hace?**
Calcula el costo promedio por factura. `DIVIDE` maneja automáticamente la división por cero.

**Resultado:** C$ 87,808.73 promedio por factura

**Insight de negocio:** Permite detectar si el costo promedio por transacción está creciendo — indicador de inflación de precios de proveedores.

---

## 2. Medidas de Inteligencia de Tiempo

---

### `Costos Año Anterior`

```dax
Costos Año Anterior = 
CALCULATE(
    [Total Costo],
    SAMEPERIODLASTYEAR(DimCalendario[Date])
)
```

**¿Qué hace?**
Calcula el costo total del mismo período del año anterior. Dinámica — responde a cualquier filtro de año o mes.

**¿Por qué SAMEPERIODLASTYEAR?**
Desplaza el contexto de fecha exactamente un año atrás manteniendo el mismo período (mes, trimestre) para comparación justa.

---

### `Variacion % vs Año Anterior`

```dax
Variacion % vs Año Anterior = 
DIVIDE(
    [Total Costo] - [Costos Año Anterior],
    [Costos Año Anterior]
)
```

**¿Qué hace?**
Porcentaje de crecimiento o decrecimiento vs el mismo período del año anterior.

**Formato:** Porcentaje con 2 decimales.

**Hallazgos:**

| Año | Variación |
|---|---|
| 2021 | Base |
| 2022 | +195% (pico máximo) |
| 2023 | -11% |
| 2024 | +2.5% |
| 2025 | -14% |

---

### `Costos Acumulado por Años`

```dax
Costos Acumulado por Años = 
CALCULATE(
    [Total Costo],
    DATESINPERIOD(
        DimCalendario[Date],
        MAX(DimCalendario[Date]),
        -5,
        YEAR
    )
)
```

**¿Qué hace?**
Calcula el costo acumulado de los últimos 5 años desde la fecha máxima en el contexto actual.

**¿Por qué DATESINPERIOD y no DATESYTD?**
`DATESINPERIOD` con `-5 YEAR` genera una ventana móvil de 5 años desde la fecha máxima del contexto. Es más flexible que `DATESYTD` que siempre empieza desde el 1 de enero del año actual. Permite ver la acumulación completa del período 2021-2025 en un solo visual.

**¿Qué es diferente vs el proyecto Farmacia?**
En Farmacia se usó `DATESYTD`. Aquí se aprendió `DATESINPERIOD` — función más versátil que permite ventanas de tiempo personalizadas. Esto representa evolución técnica real.

---

### `Compras_5_años`

```dax
Compras_5_años = 
CALCULATE(
    DISTINCTCOUNT(fctINGRESO_TOTALES[Id_Compras]),
    KEEPFILTERS(VALUES(DimCalendario[Date]))
)
```

**¿Qué hace?**
Cuenta las transacciones únicas (por Id_Compras) en los 5 años respetando los filtros del calendario activo.

**¿Por qué KEEPFILTERS?**
Preserva los filtros externos del calendario al momento de evaluar la medida. Sin `KEEPFILTERS`, `CALCULATE` podría sobreescribir los filtros del slicer de año y dar totales incorrectos.

**¿Por qué DISTINCTCOUNT sobre Id_Compras y no sobre FACTURAS?**
`Id_Compras` es la llave única construida como `id_Empresa & "-" & FORMAT(FECHA,"YYYYMMDD")`. Garantiza unicidad por empresa por día, evitando doble conteo de facturas con el mismo número en diferentes fechas.

**Uso:** Base para las columnas calculadas de clasificación de proveedores.

---

## 3. Medidas de Clasificación de Proveedores

---

### Lógica de Clasificación

El modelo implementa dos columnas de clasificación con lógica diferente usando el patrón `VAR + CALCULATE + ALLEXCEPT + SWITCH(TRUE())`:

**Clasificación Global** — considera el comportamiento del proveedor en todo el período 5 años sin filtro de año.

**Clasificación por Año** — considera el comportamiento del proveedor dentro de cada año específico.

Esta dualidad permite análisis más rico: un proveedor puede ser "Exclusivo" en el total pero "Bajo" en un año específico — señal de reducción de compras.

---

### `Clasificacion_Compras` *(Columna Calculada — Global)*

```dax
Clasificacion_Compras = 
VAR Compras =
    CALCULATE(
        [Compras_5_años],
        ALLEXCEPT(fctINGRESO_TOTALES, fctINGRESO_TOTALES[id_Empresa])
    )
RETURN
SWITCH(TRUE(),
    Compras < 10,   "Bajo",
    Compras < 30,   "Medio",
    Compras < 100,  "Alto",
    Compras < 200,  "Proveedor Exclusivo",
    TRUE,           "Bajo"
)
```

**¿Qué hace?**
Clasifica cada empresa/proveedor según el total de transacciones en los 5 años completos.

**¿Por qué ALLEXCEPT?**
`ALLEXCEPT` elimina todos los filtros de la tabla de hechos excepto el de `id_Empresa`. Esto asegura que el conteo de compras se calcula para TODA la empresa en todos los años, ignorando filtros de fecha u otros campos. Sin esto, el conteo cambiaría según el contexto de fila y la clasificación sería inconsistente.

**Segmentos:**

| Segmento | Transacciones | Descripción |
|---|---|---|
| Bajo | < 10 | Proveedor ocasional |
| Medio | 10 – 29 | Proveedor regular |
| Alto | 30 – 99 | Proveedor frecuente |
| Proveedor Exclusivo | 100 – 199 | Proveedor estratégico |

---

### `Clasificacion_Compras por Año` *(Columna Calculada — por Año)*

```dax
Clasificacion_Compras por Año = 
VAR Compras =
    CALCULATE(
        [Compras_5_años],
        ALLEXCEPT(
            fctINGRESO_TOTALES,
            fctINGRESO_TOTALES[id_Empresa],
            DimCalendario[Año]
        )
    )
RETURN
SWITCH(TRUE(),
    Compras < 10,  "Bajo",
    Compras < 30,  "Medio",
    Compras < 50,  "Alto",
    Compras >= 60, "Bajo",
    "Proveedor Exclusivo"
)
```

**¿Qué hace?**
Clasifica cada empresa dentro de cada año específico — permite ver evolución de la relación con el proveedor año a año.

**Diferencia clave vs Clasificacion_Compras:**
El `ALLEXCEPT` incluye `DimCalendario[Año]` como segunda excepción. Esto significa que el conteo respeta el filtro de año — calcula las compras de la empresa dentro de ese año específico.

**Uso en el informe:** Matriz con semáforo visual (rojo/amarillo/verde con íconos) que muestra la evolución de cada categoría de proveedor año a año de 2021 a 2025.

---

## 4. Columnas Calculadas

---

### `Id_Compras`

```dax
Id_Compras = 
fctINGRESO_TOTALES[id_Empresa] & "-" & 
FORMAT(fctINGRESO_TOTALES[FECHA], "YYYYMMDD")
```

**¿Qué hace?**
Crea un identificador único por transacción combinando el ID de la empresa con la fecha.

**Ejemplo:** Empresa 36, fecha 01-Mar-2024 → `36-20240301`

**¿Por qué es necesaria?**
El dataset original no tiene una llave primaria clara. Esta columna resuelve el problema de granularidad y permite contar transacciones únicas por empresa por día sin riesgo de doble conteo.

**Este patrón** fue aplicado previamente como `Visita_Id` en el proyecto de Farmacia — demuestra que el diseñador tiene un patrón de arquitectura propio y consistente entre proyectos.

---

## 5. Tablas Calculadas

---

### `Costos por Clasificacion y Empresas`

```dax
Costos por Clasificacion y Empresas = 
VAR tbl1 = 
    FILTER(
        ADDCOLUMNS(
            SUMMARIZE(
                fctINGRESO_TOTALES,
                DimBodega[BODEGA],
                DimClasificacion[CLASIFICACION],
                DimEmpresa[EMPRESA]
            ),
            "Monto", [Total Costo]
        ),
        [Monto] > 3000000
    )
VAR tblIndex = 
    INDEX(
        1,
        tbl1,
        ORDERBY([Monto], DESC, DimBodega[BODEGA], DESC),
        PARTITIONBY(DimEmpresa[EMPRESA], DimBodega[BODEGA])
    )
RETURN tblIndex
```

**¿Qué hace?**
Genera una tabla persistente con las combinaciones Empresa-Bodega-Clasificación cuyo costo supera C$3,000,000. Para cada combinación Empresa+Bodega identifica el registro de mayor costo usando `INDEX` con `PARTITIONBY`.

**¿Por qué C$3,000,000 como umbral?**
Representa el percentil superior del gasto — las empresas que concentran mayor impacto financiero y requieren mayor atención en negociación y seguimiento.

**Funciones avanzadas usadas:**
- `SUMMARIZE` — agrupa por múltiples dimensiones
- `ADDCOLUMNS` — agrega columna calculada a la tabla resumen
- `FILTER` — aplica umbral de costo
- `INDEX + ORDERBY + PARTITIONBY` — función de ventana DAX moderna (equivalente a ROW_NUMBER OVER PARTITION BY en SQL)

---

## 6. Resumen del Arsenal DAX

| Función / Patrón | Nivel | Medidas que lo usan |
|---|---|---|
| `SUM`, `COUNTROWS`, `DIVIDE` | Básico | Total Costo, Total Facturas, Promedio |
| `CALCULATE` | Intermedio | Costos Año Anterior, Clasificaciones |
| `SAMEPERIODLASTYEAR` | Intermedio | Costos Año Anterior |
| `DATESINPERIOD` | Intermedio-Avanzado | Costos Acumulado por Años |
| `KEEPFILTERS` | Intermedio-Avanzado | Compras_5_años |
| `ALLEXCEPT` | Avanzado | Clasificacion_Compras, por Año |
| `VAR / RETURN` | Intermedio-Avanzado | Todas las clasificaciones |
| `SWITCH(TRUE())` | Intermedio-Avanzado | Clasificacion_Compras |
| `INDEX + ORDERBY + PARTITIONBY` | Avanzado | Tabla Costos por Clasificacion |
| `SUMMARIZE + ADDCOLUMNS + FILTER` | Avanzado | Tabla Costos por Clasificacion |

---

## Evolución técnica vs Proyecto Farmacia

| Concepto | Farmacia | Ingresos | Evolución |
|---|---|---|---|
| Inteligencia de tiempo | DATESYTD | DATESINPERIOD | ✅ Nueva función |
| Clasificación | SWITCH + EARLIER | SWITCH + ALLEXCEPT | ✅ Patrón más robusto |
| Tabla calculada | INDEX simple | INDEX + SUMMARIZE + FILTER | ✅ Mayor complejidad |
| Llave única | Visita_Id | Id_Compras | ✅ Patrón propio consolidado |
| Funciones de ventana | INDEX + PARTITIONBY | INDEX + PARTITIONBY | ✅ Consistencia técnica |

---

*Documentación generada: Mayo 2026*  
*Datos anonimizados — No contiene información personal identificable*
