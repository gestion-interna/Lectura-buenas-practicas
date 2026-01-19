# Arquitectura Medallion con Inmon
## Normalización, Capas y Mejores Prácticas Modernas

---

## 1. Introducción

La arquitectura Medallion es el estándar moderno para diseñar Data Warehouses escalables y mantenibles. Combina décadas de mejores prácticas, utilizando la normalización de Inmon para garantizar calidad en las capas internas, y los modelos dimensionales de Kimball para optimizar el análisis en las capas de consumo.

Este documento explica los fundamentos de normalización de bases de datos (1NF, 2NF, 3NF), la estructura de la arquitectura Medallion (Bronze, Silver, Gold), y las mejores prácticas para implementarla en entornos empresariales.

---

## 2. Fundamentos: Normalización de Bases de Datos

La normalización es el proceso de organizar los datos en una base de datos para eliminar redundancia y prevenir anomalías. Se logra aplicando una serie de reglas llamadas "formas normales". Cada forma normal resuelve un tipo específico de problema.

### 2.1 El Problema: Datos sin Normalizar

Imagina que gestionas información de empleados y proyectos en una única tabla:

```
empleado_id | nombre_empleado | ciudad_empleado | proyecto      | fecha_inicio | ciudad_proyecto
------------|-----------------|-----------------|---------------|--------------|----------------
1001        | Juan García     | Madrid          | Web Intranet  | 2024-01-15   | Madrid
1001        | Juan García     | Madrid          | App Móvil     | 2024-03-01   | Barcelona
1002        | Ana López       | Valencia        | Web Intranet  | 2024-01-15   | Madrid
```

Esta tabla presenta múltiples problemas graves:

**Redundancia extrema**: El nombre y ciudad de Juan García aparecen dos veces. Si Juan tiene diez proyectos, su información se duplica diez veces.

**Anomalías de inserción**: No puedes registrar un nuevo empleado hasta que le asignes un proyecto.

**Anomalías de eliminación**: Si Ana López termina su último proyecto y lo eliminas, pierdes toda su información como empleada.

**Anomalías de actualización**: Si Madrid cambia algún atributo relacionado, necesitas actualizar múltiples registros.

### 2.2 Primera Forma Normal (1NF): Valores Atómicos

La Primera Forma Normal establece que cada columna debe contener valores atómicos (indivisibles). No puedes tener listas, arrays o múltiples valores en una sola celda.

#### Ejemplo de violación de 1NF:

```
empleado_id | nombre         | telefonos
------------|----------------|----------------------------------
1001        | Juan García    | 911222333, 622333444
1002        | Ana López      | 933444555
```

**Problema**: La columna "telefonos" contiene múltiples valores separados por comas. Esto viola 1NF porque no puedes buscar eficientemente por un teléfono específico, no puedes establecer restricciones sobre cuántos teléfonos puede tener cada empleado, y actualizar o eliminar un teléfono específico requiere manipulación de strings.

#### Solución aplicando 1NF:

```
empleado_id | nombre         | telefono
------------|----------------|------------
1001        | Juan García    | 911222333
1001        | Juan García    | 622333444
1002        | Ana López      | 933444555
```

Ahora cada celda contiene un único valor atómico. Cada teléfono está en su propia fila.

#### Reglas de la Primera Forma Normal:

Una tabla está en 1NF cuando cumple:

- Cada columna contiene valores atómicos (no listas ni arrays)
- Cada columna tiene un tipo de dato consistente
- Cada fila es única (existe una clave primaria)
- No hay grupos repetidos de columnas (como telefono1, telefono2, telefono3)

---

### 2.3 Segunda Forma Normal (2NF): Dependencia Total

La Segunda Forma Normal elimina las dependencias parciales. Una dependencia parcial ocurre cuando un atributo depende solo de parte de la clave primaria compuesta, no de toda ella.

Para entender esto, necesitas saber que algunas tablas tienen claves primarias compuestas, formadas por múltiples columnas. La 2NF dice que si tienes una clave primaria compuesta, todos los demás atributos deben depender de la clave completa, no solo de una parte.

#### Ejemplo de violación de 2NF:

Imagina una tabla de asignaciones de empleados a proyectos:

```
empleado_id | proyecto_id | nombre_empleado | ciudad_empleado | fecha_asignacion | rol_proyecto
------------|-------------|-----------------|-----------------|------------------|-------------
1001        | P100        | Juan García     | Madrid          | 2024-01-15       | Desarrollador
1001        | P200        | Juan García     | Madrid          | 2024-03-01       | Arquitecto
1002        | P100        | Ana López       | Valencia        | 2024-01-15       | Project Manager
```

La clave primaria es la combinación (empleado_id, proyecto_id), porque identifica de forma única cada asignación. Sin embargo, fíjate en las dependencias:

- **nombre_empleado y ciudad_empleado** dependen solo de empleado_id, no de la combinación completa. El nombre de Juan es el mismo independientemente del proyecto.

- **fecha_asignacion y rol_proyecto** dependen de la clave completa (empleado_id + proyecto_id).

Esta dependencia parcial causa redundancia: si Juan trabaja en cinco proyectos, su nombre y ciudad se duplican cinco veces.

#### Solución aplicando 2NF:

Separas los atributos según de qué dependen:

```sql
-- Tabla EMPLEADO (atributos que dependen solo de empleado_id)
CREATE TABLE empleado (
    empleado_id INT PRIMARY KEY,
    nombre_empleado VARCHAR(100),
    ciudad_empleado VARCHAR(50)
);

-- Datos:
-- 1001 | Juan García | Madrid
-- 1002 | Ana López   | Valencia

-- Tabla ASIGNACION (atributos que dependen de empleado_id + proyecto_id)
CREATE TABLE asignacion (
    empleado_id INT,
    proyecto_id VARCHAR(10),
    fecha_asignacion DATE,
    rol_proyecto VARCHAR(50),
    PRIMARY KEY (empleado_id, proyecto_id),
    FOREIGN KEY (empleado_id) REFERENCES empleado(empleado_id)
);

-- Datos:
-- 1001 | P100 | 2024-01-15 | Desarrollador
-- 1001 | P200 | 2024-03-01 | Arquitecto
-- 1002 | P100 | 2024-01-15 | Project Manager
```

Ahora cada atributo depende completamente de su clave primaria. El nombre de Juan aparece una sola vez.

#### Reglas de la Segunda Forma Normal:

Una tabla está en 2NF cuando:

- Está en 1NF
- Todos los atributos no clave dependen de la clave primaria completa
- Si la clave primaria es simple (una sola columna), automáticamente cumples 2NF

---

### 2.4 Tercera Forma Normal (3NF): Eliminando Dependencias Transitivas

La Tercera Forma Normal elimina las dependencias transitivas. Una dependencia transitiva ocurre cuando un atributo no clave depende de otro atributo no clave, en lugar de depender directamente de la clave primaria.

En términos simples: si A determina B, y B determina C, entonces A determina C de forma transitiva. La 3NF dice que esto no está permitido.

#### Ejemplo de violación de 3NF:

Consideremos nuestra tabla de empleados que ya cumple 2NF:

```
empleado_id | nombre_empleado | ciudad_empleado | codigo_postal | provincia
------------|-----------------|-----------------|---------------|----------
1001        | Juan García     | Madrid          | 28001         | Madrid
1002        | Ana López       | Valencia        | 46001         | Valencia
1003        | Carlos Ruiz     | Getafe          | 28901         | Madrid
```

Analicemos las dependencias:

- **empleado_id → nombre_empleado**: Correcto, depende directamente de la PK
- **empleado_id → ciudad_empleado**: Correcto
- **ciudad_empleado → codigo_postal**: PROBLEMA. El código postal depende de la ciudad, no del empleado
- **codigo_postal → provincia**: PROBLEMA. La provincia depende del código postal, no del empleado

Esto crea una cadena: empleado_id → ciudad_empleado → codigo_postal → provincia. Las dos últimas son dependencias transitivas.

El problema práctico es que si tienes cincuenta empleados en Madrid, guardas cincuenta veces que Madrid tiene código 28001 y está en la provincia de Madrid.

#### Solución aplicando 3NF:

Extraes las entidades que tienen sus propias características independientes:

```sql
-- Tabla EMPLEADO (solo atributos propios del empleado)
CREATE TABLE empleado (
    empleado_id INT PRIMARY KEY,
    nombre_empleado VARCHAR(100),
    ciudad_id INT,
    FOREIGN KEY (ciudad_id) REFERENCES ciudad(ciudad_id)
);

-- Datos:
-- 1001 | Juan García | C001
-- 1002 | Ana López   | C002
-- 1003 | Carlos Ruiz | C003

-- Tabla CIUDAD (atributos propios de la ciudad)
CREATE TABLE ciudad (
    ciudad_id INT PRIMARY KEY,
    nombre_ciudad VARCHAR(100),
    codigo_postal VARCHAR(10),
    provincia VARCHAR(50)
);

-- Datos:
-- C001 | Madrid  | 28001 | Madrid
-- C002 | Valencia| 46001 | Valencia
-- C003 | Getafe  | 28901 | Madrid
```

Ahora cada tabla contiene solo atributos que dependen directamente de su clave primaria. La información de Madrid se guarda una sola vez.

#### Reglas de la Tercera Forma Normal:

Una tabla está en 3NF cuando:

- Está en 2NF
- No tiene dependencias transitivas (todos los atributos no clave dependen únicamente de la clave primaria, no de otros atributos no clave)

La 3NF es el nivel de normalización que se considera estándar en bases de datos OLTP y es el que utiliza Inmon para construir Data Warehouses.

---

### 2.5 Tabla Comparativa: Formas Normales

| Forma Normal | Requisito Principal | Qué Elimina | Ejemplo Violación | Uso Típico |
|--------------|-------------------|-------------|-------------------|------------|
| **1NF** | Valores atómicos en cada celda | Listas, arrays, valores múltiples | `telefono: '911222, 622333'` | Mínimo requerido |
| **2NF** | Dependencia total de PK completa | Dependencias parciales en claves compuestas | `nombre` depende solo de `empleado_id` en PK `(empleado_id, proyecto_id)` | Tablas intermedias |
| **3NF** | Sin dependencias transitivas | Atributos no clave que dependen de otros no clave | `ciudad → codigo_postal → provincia` | Standard OLTP y DW Silver (Inmon) |

---

## 3. Arquitectura Medallion: Capas Bronze-Silver-Gold

La arquitectura Medallion organiza los datos en tres capas progresivamente refinadas, cada una optimizada para un propósito específico. Este enfoque separa responsabilidades y permite evolucionar cada capa independientemente.

### Flujo de Datos en Arquitectura Medallion:

```
┌──────────────────────────────────────────────────────────┐
│  FUENTES DE DATOS                                        │
│  ├─ ERP (SAP, Oracle)                                    │
│  ├─ CRM (Salesforce)                                     │
│  ├─ Bases de datos transaccionales                       │
│  └─ APIs externas, archivos Excel, CSVs                  │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│  CAPA BRONZE - Datos Crudos                              │
│  ────────────────────────────────                        │
│  • Sin transformaciones                                  │
│  • Inmutable (nunca se modifica)                         │
│  • Trazabilidad completa                                 │
│  • Formato: JSON, Parquet, CSV, tablas raw               │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│  CAPA SILVER - Inmon 3NF (Fuente Única de Verdad)       │
│  ──────────────────────────────────────────              │
│  • Normalizado (3NF)                                     │
│  • Validado y limpio                                     │
│  • Gobernanza de datos aplicada                          │
│  • Histórico completo (SCD Tipo 2)                       │
│  • Integración de múltiples fuentes                      │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│  CAPA GOLD - Kimball Dimensional (Business Ready)       │
│  ─────────────────────────────────────────               │
│  • Desnormalizado (esquema estrella)                    │
│  • Optimizado para consultas analíticas                 │
│  • Data Marts por área de negocio                       │
│  • Métricas pre-calculadas                              │
└──────────────────────────────────────────────────────────┘
                           ↓
┌──────────────────────────────────────────────────────────┐
│  CONSUMO                                                 │
│  ├─ Dashboards (Power BI, Tableau)                      │
│  ├─ Reportes ejecutivos                                 │
│  ├─ Modelos de Machine Learning                         │
│  └─ Aplicaciones analíticas                             │
└──────────────────────────────────────────────────────────┘
```

---

### 3.1 Capa Bronze: Inmutabilidad y Trazabilidad

Bronze es tu red de seguridad. Captura datos exactamente como llegan, sin modificaciones. Esto garantiza que siempre puedes volver al origen y reprocesar si descubres errores.

#### Características de Bronze:

**Inmutabilidad absoluta**: Una vez que escribes un dato en Bronze, nunca lo modificas ni eliminas. Si descubres un error en la ingesta, no corriges los datos en Bronze, en su lugar, vuelves a ingestar creando una nueva versión.

**Formato crudo**: Los datos se guardan en el formato más cercano posible a cómo llegaron. Si vienen como JSON desde una API, los guardas como JSON. Si vienen de una base de datos, puedes guardarlos en tablas pero manteniendo nombres y tipos originales.

**Metadatos de ingesta**: Cada registro incluye información sobre cuándo fue ingerido, desde qué sistema fuente, qué versión del proceso lo capturó.

**Permite reprocesamiento**: Si descubres un bug en tu lógica de transformación, puedes volver a Bronze y reprocesar todos los datos desde cero.

#### Ejemplo de estructura Bronze:

```sql
CREATE TABLE bronze.empleados_sap_raw (
    ingesta_id BIGINT IDENTITY(1,1) PRIMARY KEY,
    fecha_ingesta DATETIME2 NOT NULL DEFAULT GETDATE(),
    sistema_origen VARCHAR(50) NOT NULL,  -- 'SAP-HR'
    version_ingesta VARCHAR(20) NOT NULL, -- 'v2.3.1'
    datos_raw NVARCHAR(MAX) NOT NULL      -- JSON crudo desde SAP
);

-- Ejemplo de registro:
-- ingesta_id: 1001
-- fecha_ingesta: 2024-11-08 02:15:33
-- sistema_origen: SAP-HR
-- version_ingesta: v2.3.1
-- datos_raw: {"PERNR":"1001","VORNA":"Juan","NACHN":"García","CITY":"Madrid",...}
```

#### Por qué Bronze es crucial:

**Capacidad de reprocesamiento**: Si descubres un bug en tu lógica de transformación en Silver o Gold, puedes volver a Bronze y reprocesar todos los datos desde cero.

**Auditabilidad**: Si alguien cuestiona un número en un reporte, puedes rastrearlo hasta Bronze y demostrar exactamente qué dato llegó desde el sistema fuente.

**Evolución de esquemas**: Si el sistema fuente agrega nuevos campos, Bronze los captura automáticamente sin romper tu pipeline.

---

### 3.2 Capa Silver: Fuente Única de Verdad con Inmon 3NF

La capa Silver es donde aplicas normalización Inmon (3NF), reglas de calidad, estandarizaciones y construyes tu fuente única de verdad corporativa. Silver es el corazón de tu arquitectura Medallion.

#### Características de Silver:

**Normalización 3NF**: Aplicas las formas normales. Cada entidad tiene su propia tabla (empleados, departamentos, ciudades, proyectos), las relaciones se establecen mediante claves foráneas, eliminas redundancia estructural.

**Calidad de datos**: Implementas reglas de validación y limpieza. Los emails se validan con regex, los números de teléfono se normalizan a un formato estándar, los NIFs se verifican, las fechas imposibles se rechazan.

**Estandarización**: Los nombres de ciudades se unifican contra un catálogo maestro, los nombres de departamentos siguen la nomenclatura oficial, las unidades se convierten a un estándar corporativo.

**Histórico con SCD Tipo 2**: Para dimensiones críticas, implementas Slowly Changing Dimensions Tipo 2, manteniendo todas las versiones históricas con fechas de validez.

**Integración de fuentes**: Si el mismo empleado aparece en SAP y en un sistema de nóminas, Silver reconcilia ambas fuentes y crea una única representación consistente.

#### Ejemplo de estructura Silver con Inmon 3NF:

```sql
-- Tabla de dimensión EMPLEADO con SCD Tipo 2
CREATE TABLE silver.dim_empleado (
    empleado_sk BIGINT IDENTITY(1,1) PRIMARY KEY,  -- Surrogate key
    empleado_id INT NOT NULL,                       -- Business key
    nombre VARCHAR(100) NOT NULL,
    apellidos VARCHAR(200) NOT NULL,
    email VARCHAR(100) NOT NULL,
    departamento_id INT NOT NULL,                   -- FK normalizada
    puesto_id INT NOT NULL,                         -- FK normalizada
    ciudad_id INT NOT NULL,                         -- FK normalizada
    gerente_empleado_sk BIGINT NULL,                -- FK a otro empleado
    
    -- Columnas de SCD Tipo 2
    fecha_inicio DATE NOT NULL,
    fecha_fin DATE NULL,
    es_actual BIT NOT NULL DEFAULT 1,
    
    -- Metadatos de gobernanza
    usuario_carga VARCHAR(100) NOT NULL,
    fecha_carga DATETIME2 NOT NULL DEFAULT GETDATE(),
    fuente_sistema VARCHAR(50) NOT NULL,
    
    -- Constraints
    CONSTRAINT fk_emp_departamento FOREIGN KEY (departamento_id)
        REFERENCES silver.dim_departamento(departamento_id),
    CONSTRAINT fk_emp_puesto FOREIGN KEY (puesto_id)
        REFERENCES silver.dim_puesto(puesto_id),
    CONSTRAINT fk_emp_ciudad FOREIGN KEY (ciudad_id)
        REFERENCES silver.dim_ciudad(ciudad_id),
    CONSTRAINT fk_emp_gerente FOREIGN KEY (gerente_empleado_sk)
        REFERENCES silver.dim_empleado(empleado_sk)
);

-- Tabla DEPARTAMENTO (normalizada, elimina redundancia)
CREATE TABLE silver.dim_departamento (
    departamento_id INT PRIMARY KEY,
    nombre_departamento VARCHAR(100) NOT NULL,
    area_negocio VARCHAR(50) NOT NULL,
    centro_costos VARCHAR(20) NOT NULL
);

-- Tabla CIUDAD (elimina dependencia transitiva de 3NF)
CREATE TABLE silver.dim_ciudad (
    ciudad_id INT PRIMARY KEY,
    nombre_ciudad VARCHAR(100) NOT NULL,
    codigo_postal VARCHAR(10) NOT NULL,
    provincia VARCHAR(50) NOT NULL,
    comunidad_autonoma VARCHAR(50) NOT NULL
);
```

#### Proceso de transformación Bronze → Silver:

```sql
-- ETL que transforma Bronze a Silver

-- 1. Extraer y parsear JSON de Bronze
WITH datos_parseados AS (
    SELECT 
        JSON_VALUE(datos_raw, '$.PERNR') AS empleado_id,
        JSON_VALUE(datos_raw, '$.VORNA') AS nombre,
        JSON_VALUE(datos_raw, '$.NACHN') AS apellidos,
        JSON_VALUE(datos_raw, '$.EMAIL') AS email_raw,
        JSON_VALUE(datos_raw, '$.CITY') AS ciudad_raw
    FROM bronze.empleados_sap_raw
    WHERE fecha_ingesta >= @ultima_carga
),

-- 2. Aplicar limpieza y validaciones
datos_limpios AS (
    SELECT
        CAST(empleado_id AS INT) AS empleado_id,
        UPPER(LTRIM(RTRIM(nombre))) AS nombre,      -- Estandarizar
        UPPER(LTRIM(RTRIM(apellidos))) AS apellidos,
        LOWER(email_raw) AS email,                   -- Normalizar a minúsculas
        ciudad_raw
    FROM datos_parseados
    WHERE email_raw LIKE '%@%'  -- Validar formato básico
),

-- 3. Lookup de dimensiones normalizadas
datos_enriquecidos AS (
    SELECT
        dl.empleado_id,
        dl.nombre,
        dl.apellidos,
        dl.email,
        c.ciudad_id  -- Obtener FK de tabla normalizada
    FROM datos_limpios dl
    INNER JOIN silver.dim_ciudad c 
        ON UPPER(c.nombre_ciudad) = UPPER(dl.ciudad_raw)
)

-- 4. Insertar en Silver con lógica SCD Tipo 2
INSERT INTO silver.dim_empleado (
    empleado_id, nombre, apellidos, email, ciudad_id,
    fecha_inicio, fecha_fin, es_actual,
    usuario_carga, fecha_carga, fuente_sistema
)
SELECT 
    empleado_id, nombre, apellidos, email, ciudad_id,
    GETDATE() AS fecha_inicio,
    NULL AS fecha_fin,
    1 AS es_actual,
    SYSTEM_USER AS usuario_carga,
    GETDATE() AS fecha_carga,
    'SAP-HR' AS fuente_sistema
FROM datos_enriquecidos;
```

#### Por qué Silver es el corazón del sistema:

**Fuente única de verdad**: Cuando finanzas pregunta cuántos empleados hay en Madrid y recursos humanos hace la misma pregunta, ambos obtienen la misma respuesta porque consultan Silver.

**Calidad garantizada**: Todos los datos en Silver han pasado por validaciones. No hay emails inválidos, no hay fechas imposibles, no hay inconsistencias.

**Histórico completo**: Con SCD Tipo 2, puedes responder preguntas como "¿cuántos empleados había en cada departamento el trimestre pasado?"

**Base para múltiples Data Marts**: Silver es la base desde la cual construyes todos tus Data Marts en Gold, garantizando consistencia.

---

### 3.3 Capa Gold: Optimización para Consumo con Kimball

Gold es donde los datos se convierten en información lista para el consumo. Aquí abandonas la normalización de Inmon y adoptas modelos dimensionales de Kimball, optimizados específicamente para análisis y visualización.

#### Características de Gold:

**Desnormalización estratégica**: Tomas las tablas normalizadas de Silver y las combinas en estructuras denormalizadas más fáciles y rápidas de consultar.

**Modelos dimensionales (estrella)**: Creas esquemas en estrella con tablas de hechos en el centro y dimensiones alrededor. Las tablas de hechos contienen métricas cuantificables, las dimensiones proporcionan el contexto.

**Agregaciones pre-calculadas**: En lugar de calcular totales cada vez que alguien abre un dashboard, pre-calculas y guardas las métricas más comunes.

**Data Marts especializados**: Creas diferentes Data Marts optimizados para diferentes áreas de negocio.

**Puede ser Views o Tablas**: Implementas como views para datos en tiempo real o como tablas físicas para dashboards de alto tráfico.

#### Ejemplo de Data Mart en Gold (esquema estrella):

```sql
-- DIMENSIÓN EMPLEADO (desnormalizada para consultas rápidas)
CREATE TABLE gold.dim_empleado (
    empleado_key INT PRIMARY KEY,  -- Clave de dimensión simplificada
    empleado_id INT NOT NULL,
    nombre_completo VARCHAR(300) NOT NULL,  -- Nombre + apellidos juntos
    email VARCHAR(100) NOT NULL,
    
    -- Desnormalizado: traemos toda la jerarquía
    departamento VARCHAR(100) NOT NULL,
    area_negocio VARCHAR(50) NOT NULL,
    puesto VARCHAR(100) NOT NULL,
    ciudad VARCHAR(100) NOT NULL,
    provincia VARCHAR(50) NOT NULL,
    comunidad_autonoma VARCHAR(50) NOT NULL,
    nombre_gerente VARCHAR(300) NULL,
    
    fecha_actualizacion DATETIME2 NOT NULL
);

-- DIMENSIÓN FECHA (tabla calendario pre-poblada)
CREATE TABLE gold.dim_fecha (
    fecha_key INT PRIMARY KEY,  -- YYYYMMDD: 20241108
    fecha DATE NOT NULL,
    anio INT NOT NULL,
    trimestre INT NOT NULL,
    mes INT NOT NULL,
    nombre_mes VARCHAR(20) NOT NULL,
    semana_anio INT NOT NULL,
    dia_semana INT NOT NULL,
    nombre_dia_semana VARCHAR(20) NOT NULL,
    es_fin_semana BIT NOT NULL,
    es_festivo BIT NOT NULL
);

-- TABLA DE HECHOS: Headcount mensual (métricas pre-calculadas)
CREATE TABLE gold.fact_headcount_mensual (
    empleado_key INT NOT NULL,
    fecha_key INT NOT NULL,  -- Primer día del mes
    
    -- Métricas pre-calculadas
    dias_activo INT NOT NULL,
    salario_base DECIMAL(10,2) NOT NULL,
    complementos DECIMAL(10,2) NOT NULL,
    costo_total DECIMAL(10,2) NOT NULL,
    horas_trabajadas DECIMAL(8,2) NOT NULL,
    
    PRIMARY KEY (empleado_key, fecha_key),
    FOREIGN KEY (empleado_key) REFERENCES gold.dim_empleado(empleado_key),
    FOREIGN KEY (fecha_key) REFERENCES gold.dim_fecha(fecha_key)
);
```

#### Consultas simples y rápidas en Gold:

```sql
-- ¿Cuál fue el costo total de personal por departamento en Q1 2024?
SELECT 
    e.departamento,
    SUM(f.costo_total) AS costo_total_q1
FROM gold.fact_headcount_mensual f
INNER JOIN gold.dim_empleado e ON f.empleado_key = e.empleado_key
INNER JOIN gold.dim_fecha d ON f.fecha_key = d.fecha_key
WHERE d.anio = 2024 AND d.trimestre = 1
GROUP BY e.departamento
ORDER BY costo_total_q1 DESC;

-- Nota: Esta consulta es extremadamente rápida porque:
-- 1. Los datos ya están desnormalizados (solo 3 tablas, 2 JOINs)
-- 2. Las métricas están pre-calculadas (costo_total)
-- 3. La tabla está optimizada para agregaciones
```

#### Implementación: ¿Views o tablas físicas?

Gold puede implementarse de dos formas:

**Como Views sobre Silver**:
- Para datos que necesitan estar en tiempo real
- Para Data Marts con bajo volumen de consultas
- Ejemplo: Vista de empleados actuales

```sql
CREATE VIEW gold.vw_empleados_actuales AS
SELECT 
    e.empleado_id,
    e.nombre || ' ' || e.apellidos AS nombre_completo,
    d.nombre_departamento,
    c.nombre_ciudad
FROM silver.dim_empleado e
INNER JOIN silver.dim_departamento d ON e.departamento_id = d.departamento_id
INNER JOIN silver.dim_ciudad c ON e.ciudad_id = c.ciudad_id
WHERE e.es_actual = 1;
```

**Como Tablas físicas pobladas por ETL**:
- Para dashboards con alto tráfico
- Para métricas complejas que requieren mucho procesamiento
- Para datos históricos que no cambian frecuentemente
- Ejemplo: Tabla de headcount mensual (mostrada arriba)

---

### 3.4 Tabla Comparativa: Bronze vs Silver vs Gold

| Aspecto | Bronze | Silver | Gold |
|---------|--------|--------|------|
| **Propósito** | Ingesta de datos crudos | Fuente única de verdad corporativa | Consumo analítico optimizado |
| **Formato** | Raw (JSON, CSV, tal cual llega) | Normalizado 3NF (Inmon) | Dimensional (Kimball, estrella) |
| **Calidad de datos** | Sin validación | Validado, limpio, estandarizado | Heredada de Silver |
| **Transformaciones** | Ninguna o mínimas | Limpiezas, normalizaciones, integraciones | Desnormalizaciones, agregaciones |
| **Histórico** | Snapshot del momento de ingesta | SCD Tipo 2 para dimensiones críticas | Optimizado para consultas temporales |
| **Mutabilidad** | Inmutable (nunca se modifica) | Versionado (SCD), auditable | Puede regenerarse desde Silver |
| **Usuarios típicos** | Ingenieros de datos | Ingenieros de datos, gobernanza | Analistas, herramientas BI |
| **Complejidad consultas** | No diseñada para consultas | Media-alta (múltiples JOINs) | Baja (JOINs simples) |
| **Rendimiento consultas** | No relevante | Medio (normalizado) | Alto (desnormalizado) |
| **Tamaño típico** | Grande (todo el histórico raw) | Grande (histórico normalizado) | Medio (datos agregados) |
| **Frecuencia actualización** | Continua o por lotes frecuentes | Diaria o varias veces al día | Desde tiempo real a mensual |
| **Ejemplo tabla** | `bronze.empleados_sap_raw` | `silver.dim_empleado` | `gold.dm_headcount_mensual` |

---

## 4. Fortalezas de la Arquitectura Medallion con Inmon

### 4.1 Separación de Responsabilidades

Una de las fortalezas más importantes es que separa claramente diferentes responsabilidades en diferentes capas. Cada capa tiene un propósito específico y está optimizada para ese propósito, sin comprometer los demás.

**Bronze se preocupa solo de ingesta**: Su único trabajo es capturar datos de forma confiable y rápida. No necesita preocuparse por validaciones, transformaciones o rendimiento de consultas.

**Silver se preocupa solo de calidad**: Su trabajo es garantizar que los datos sean correctos, consistentes y completos. No necesita preocuparse por el rendimiento de dashboards.

**Gold se preocupa solo de usabilidad**: Su trabajo es hacer que los datos sean fáciles y rápidos de consultar. No necesita preocuparse por integridad referencial.

Esta separación significa que puedes evolucionar cada capa independientemente. Si necesitas mejorar la calidad de datos, trabajas en Silver sin afectar Bronze o Gold.

---

### 4.2 Trazabilidad Completa

Con esta arquitectura, puedes rastrear cualquier dato desde su origen hasta su consumo final. Si un ejecutivo cuestiona un número en un dashboard, puedes seguir el rastro:

1. El número viene de `gold.dm_ventas` en el Data Mart de ventas
2. Esa tabla se pobla desde `silver.fact_ventas` mediante un ETL diario
3. La tabla en Silver se alimenta de registros en `bronze.ventas_sap_raw`
4. Puedes ver el JSON exacto que llegó desde SAP

Esta trazabilidad es invaluable para auditorías, resolución de problemas y garantizar confianza en los datos.

---

### 4.3 Capacidad de Reprocesamiento

Imagina que descubres un bug en tu lógica de cálculo que ha estado activo durante seis meses. Con arquitectura Medallion, la solución es directa:

1. Bronze tiene todos los datos originales intactos
2. Corriges la lógica de transformación en tu ETL
3. Truncas los datos afectados en Silver
4. Reprocesas los últimos seis meses desde Bronze
5. Regeneras Gold desde el Silver corregido

Sin Bronze, habrías perdido los datos originales y no podrías corregir el histórico.

---

### 4.4 Gobernanza Centralizada

Al tener Silver como fuente única de verdad normalizada, toda tu gobernanza de datos se aplica en un solo lugar. Las reglas de validación, la normalización, la estandarización: todo se implementa una vez en el ETL de Bronze a Silver.

Cuando creas un nuevo Data Mart en Gold, automáticamente hereda toda la calidad de datos de Silver. Esto garantiza consistencia: todos los Data Marts usan las mismas definiciones y las mismas reglas de calidad.

---

### 4.5 Escalabilidad Natural

La arquitectura escala naturalmente a medida que tu volumen de datos y número de usuarios crece:

- **Bronze puede escalar horizontalmente**: Añadir más fuentes es simplemente agregar más pipelines de ingesta
- **Silver puede optimizarse con particionamiento**: Particionar tablas por fecha o región para mejorar rendimiento
- **Gold puede duplicarse por área**: Si el Data Mart de finanzas tiene mucho tráfico, puede tener su propia base de datos física

---

### 4.6 Flexibilidad ante Cambios

Los negocios cambian constantemente. La arquitectura Medallion maneja estos cambios con gracia:

- **Nueva fuente de datos**: Agregas pipeline a Bronze, extiendes ETL a Silver
- **Nueva regla de negocio**: Modificas lógica en Silver, reprocesas si necesario
- **Nuevo Data Mart**: Creas estructuras en Gold consumiendo desde Silver
- **Cambio de esquema en fuente**: Bronze captura automáticamente, decides cuándo incorporar en Silver

---

## 5. Mejores Prácticas de Implementación

### 5.1 Para la Capa Bronze

- **Nunca modifiques datos una vez escritos**, solo añade nuevas versiones
- **Incluye siempre metadatos**: `fecha_ingesta`, `sistema_origen`, `version_proceso`
- **Usa formatos eficientes** para almacenamiento masivo (Parquet, Delta Lake)
- **Implementa retención de datos** según políticas corporativas (ej: 7 años)
- **Documenta el esquema de cada fuente** incluso si es JSON sin estructura fija

### 5.2 Para la Capa Silver

- **Aplica normalización 3NF estricta** para garantizar integridad
- **Usa SCD Tipo 2** para dimensiones críticas (empleados, clientes, productos)
- **Implementa validaciones exhaustivas** (tipo de dato, formato, rangos)
- **Documenta todas las reglas de negocio** y transformaciones aplicadas
- **Usa catálogos maestros** para estandarizar dimensiones (ciudades, países)
- **Implementa tests de calidad** que se ejecuten automáticamente
- **Mantén audit trail** de quién modificó qué y cuándo

### 5.3 Para la Capa Gold

- **Crea Data Marts específicos** por área de negocio (finanzas, ventas, RRHH)
- **Usa views para datos en tiempo real**, tablas físicas para dashboards de alto tráfico
- **Pre-calcula métricas comunes** (totales, promedios) para optimizar rendimiento
- **Implementa tabla de fechas** (dim_fecha) con atributos útiles (trimestre, festivos)
- **Desnormaliza solo lo necesario**, manteniendo balance entre rendimiento y complejidad
- **Documenta la semántica** de cada métrica para evitar confusiones

---

## 6. Comparación con Enfoques Alternativos

| Aspecto | Medallion + Inmon | DW Monolítico (Inmon puro) | Data Marts directos (Kimball puro) |
|---------|-------------------|----------------------------|-----------------------------------|
| **Tiempo implementación** | Medio (capas incrementales) | Lento (todo a la vez) | Rápido (por departamento) |
| **Capacidad reprocesamiento** | Excelente (Bronze inmutable) | Limitada (sin raw data) | Pobre (sin histórico raw) |
| **Gobernanza** | Centralizada en Silver | Centralizada | Distribuida por Data Mart |
| **Rendimiento analítico** | Excelente (Gold optimizado) | Medio-bajo (normalizado) | Excelente (dimensional) |
| **Consistencia entre áreas** | Garantizada (Silver único) | Garantizada | Requiere disciplina |
| **Complejidad** | Media | Alta | Baja inicialmente |
| **Escalabilidad** | Excelente | Media | Media |
| **Flexibilidad ante cambios** | Alta | Baja | Media |
| **Adecuado para** | Mayoría de organizaciones modernas | Grandes empresas reguladas | Proyectos ágiles pequeños |

---

## 7. Conclusión

La arquitectura Medallion con Inmon en Silver y Kimball en Gold representa el estado del arte en diseño de Data Warehouses modernos. Combina lo mejor de décadas de experiencia:

- **Inmutabilidad** de Bronze para trazabilidad y reprocesamiento
- **Normalización 3NF** de Inmon en Silver para gobernanza y calidad
- **Modelos dimensionales** de Kimball en Gold para análisis optimizado
- **Separación de responsabilidades** para escalabilidad y mantenibilidad

Esta arquitectura es:
- Suficientemente **probada** para ser confiable
- Suficientemente **flexible** para adaptarse a necesidades específicas
- Suficientemente **estándar** para encontrar herramientas, talento y recursos fácilmente

Las formas normales (1NF, 2NF, 3NF) no son solo teoría académica: son la base que permite construir Silver como una fuente de verdad confiable, eliminando redundancia y garantizando integridad referencial.

Si estás diseñando un nuevo Data Warehouse o modernizando uno existente, la arquitectura Medallion es el punto de partida recomendado por la industria. Es el enfoque que utilizan Databricks, Snowflake, Azure Synapse y las organizaciones data-driven más avanzadas del mundo.

---

**Recuerda**: El objetivo final no es la arquitectura en sí misma, sino empoderar a tu organización con datos confiables, accesibles y accionables. La arquitectura Medallion con Inmon te da la estructura para lograrlo de forma sostenible y escalable.
