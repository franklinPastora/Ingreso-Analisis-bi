# 🗄️ Proceso SQL Server — Carga de Datos INGRESOS

## Descripción

Los datos originales provenían de 7 archivos Excel exportados
del sistema institucional (2021–2025). El proceso de carga a
SQL Server se realizó para practicar y consolidar el flujo
completo de trabajo: Excel → SQL Server → Power BI.

---

## Entorno

- **Motor:** SQL Server (localhost)
- **Base de datos:** INGRESOS
- **Tabla principal:** INGRESO_TOTALES
- **Total registros:** 7,371

---

## Etapa 1 — Conversión de formato

Los 7 archivos `.xlsx` fueron convertidos a formato `.csv`
para facilitar la importación a SQL Server mediante el
Asistente de Importación de Datos Planos.

---

## Etapa 2 — Creación de tabla maestra

```sql
USE INGRESOS;

CREATE TABLE INGRESO_TOTALES (
    CLASIFICACION  VARCHAR(50),
    FACTURAS       INT,
    FECHA          DATE,
    EMPRESA        VARCHAR(140),
    COSTO          NUMERIC(18,4),
    BODEGA         VARCHAR(50)
);
```

---

## Etapa 3 — Importación por año

Cada archivo CSV fue importado usando el Asistente de
Importación de SQL Server:

```
Año 2021 → tabla temporal → INSERT INTO INGRESO_TOTALES
Año 2022 → tabla temporal → INSERT INTO INGRESO_TOTALES
Año 2023 → tabla temporal → INSERT INTO INGRESO_TOTALES
Año 2024 → tabla temporal → INSERT INTO INGRESO_TOTALES
Año 2025 → tabla temporal → INSERT INTO INGRESO_TOTALES
```

---

## Etapa 4 — Verificación de registros

```sql
-- Verificar total de registros
SELECT COUNT(*) FROM INGRESO_TOTALES;
-- Resultado: 7,371 registros

-- Verificar registros por año
SELECT
    YEAR(FECHA)      AS Anio,
    COUNT(*)         AS Total_Registros,
    SUM(COSTO)       AS Costo_Total
FROM INGRESO_TOTALES
GROUP BY YEAR(FECHA)
ORDER BY Anio;
```

**Resultado esperado:**

| Año | Registros | Costo Total |
|---|---|---|
| 2021 | 295 | C$ XX,XXX,XXX |
| 2022 | 873 | C$ XX,XXX,XXX |
| 2023 | 778 | C$ XX,XXX,XXX |
| 2024 | 798 | C$ XX,XXX,XXX |
| 2025 | 683 | C$ XX,XXX,XXX |

---

## Etapa 5 — Consultas de análisis usadas

```sql
-- Vista general de los datos
SELECT * FROM INGRESO_TOTALES;

-- Top empresas por costo total
SELECT
    EMPRESA,
    COUNT(*)         AS Total_Facturas,
    SUM(COSTO)       AS Costo_Total,
    AVG(COSTO)       AS Costo_Promedio
FROM INGRESO_TOTALES
GROUP BY EMPRESA
ORDER BY Costo_Total DESC;

-- Análisis por año y bodega
SELECT
    YEAR(FECHA)      AS Anio,
    BODEGA,
    COUNT(*)         AS Total_Compras,
    SUM(COSTO)       AS Costo_Total
FROM INGRESO_TOTALES
GROUP BY YEAR(FECHA), BODEGA
ORDER BY Anio, Costo_Total DESC;

-- Variacion entre años
SELECT
    YEAR(FECHA)      AS Anio,
    SUM(COSTO)       AS Costo_Total
FROM INGRESO_TOTALES
GROUP BY YEAR(FECHA)
ORDER BY Anio;
```

---

## Etapa 6 — Conexión a Power BI

Desde Power BI Desktop:
- Inicio → Obtener datos → SQL Server
- Servidor: localhost
- Base de datos: INGRESOS
- Tabla: INGRESO_TOTALES
- Modo: Importación

---

## Diferencia vs Proyecto Farmacia

| Aspecto | Farmacia | Ingresos |
|---|---|---|
| Archivos fuente | 26 Excel | 7 Excel |
| Registros | 1,739,518 | 7,371 |
| Objetivo SQL | Necesidad real de escala | Práctica deliberada |
| Resultado | Misma arquitectura | Proceso consolidado |

El proyecto Ingresos usó SQL Server no por necesidad de
volumen sino para consolidar el flujo de trabajo completo
y practicar el proceso que se aplicó en Farmacia con
1.7 millones de registros.

---

*Nota: Los valores en la columna EMPRESA fueron anonimizados
durante el proceso ETL en Power Query antes de la publicación.*
