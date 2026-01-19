# Guía Completa: Data Warehouses y Slowly Changing Dimensions

## Introducción: El Problema que Resolvemos

Cuando trabajas con bases de datos transaccionales (OLTP), el sistema está diseñado para procesar operaciones del día a día de forma rápida y eficiente. Si un cliente cambia de dirección, simplemente actualizas el registro. Si un empleado cambia de departamento, modificas ese campo. El sistema solo mantiene el estado actual de las cosas.

Pero imagina que te hacen esta pregunta: *"¿Cuál fue la rentabilidad de nuestros proyectos en Madrid durante el primer trimestre de 2023?"* O esta otra: *"¿Cómo ha evolucionado la distribución geográfica de nuestros clientes en los últimos cinco años?"*

Tu base de datos transaccional no puede responder estas preguntas porque no conserva el histórico. Solo sabe cómo están las cosas ahora, no cómo estaban antes. Aquí es donde entran los Data Warehouses: repositorios especializados diseñados específicamente para almacenar datos históricos y facilitar su análisis.

Sin embargo, construir un Data Warehouse no es tan simple como copiar datos. Hay decisiones arquitectónicas fundamentales que tomar, y cada una tiene sus propias ventajas y desafíos. Vamos a explorar los tres enfoques principales y luego profundizaremos en cómo gestionar los cambios en el tiempo.

---

## Parte 1: Los Tres Enfoques para Construir un Data Warehouse

### 1.1 Kimball: Construyendo desde el Análisis (Bottom-up)

Ralph Kimball propuso una filosofía pragmática: empieza pequeño, entrega valor rápido y crece de forma incremental. En lugar de intentar modelar toda la empresa de golpe, construyes lo que Kimball llama "Data Marts" por áreas de negocio.

#### Cómo Funciona

Imagina que el departamento de ventas necesita analizar sus datos urgentemente. Con Kimball, empezarías creando un Data Mart específico para ventas. Este Data Mart usaría un modelo dimensional, típicamente con un esquema en estrella.

En el centro de la estrella está la tabla de hechos, que contiene las métricas cuantificables: importe de venta, cantidad vendida, descuento aplicado. Estas son las cosas que puedes sumar, promediar o contar. Alrededor de esta tabla central están las dimensiones, que proporcionan el contexto: quién compró (cliente), qué se vendió (producto), cuándo ocurrió (fecha), dónde se realizó (región o tienda).

```
       DIMENSIÓN_TIEMPO
              |
              |
DIMENSIÓN_PRODUCTO --- TABLA_HECHOS_VENTAS --- DIMENSIÓN_CLIENTE
              |
              |
       DIMENSIÓN_REGIÓN
```

Después de implementar el Data Mart de ventas y ver su éxito, el departamento de finanzas pide el suyo. Construyes otro Data Mart para analizar costos y presupuestos. Luego viene recursos humanos con sus necesidades. Gradualmente, estos Data Marts se conectan mediante dimensiones conformadas (dimensiones compartidas que significan lo mismo en todos los Data Marts), creando un Data Warehouse empresarial.

#### Las Ventajas del Enfoque Kimball

La principal ventaja es la velocidad de implementación. En pocas semanas puedes tener un Data Mart funcionando y entregando valor al negocio. El modelo dimensional es extremadamente intuitivo: un analista de negocio sin conocimientos técnicos profundos puede entender rápidamente qué representa cada tabla y cómo relacionarlas.

Las consultas son también muy eficientes. Cuando un usuario quiere ver "ventas por producto y región en el último trimestre", la estructura en estrella permite responder esta pregunta con joins simples y directos. Las herramientas de BI como Tableau, Power BI o Qlik se conectan naturalmente a estos esquemas.

#### Los Desafíos

El problema surge cuando tienes diez, quince o veinte Data Marts construidos por diferentes equipos en diferentes momentos. La dimensión "Cliente" del Data Mart de ventas podría no coincidir exactamente con la del Data Mart de finanzas. Uno tiene el nombre completo del cliente, el otro solo las iniciales. Uno actualiza la ciudad inmediatamente, el otro mantiene histórico.

Integrar estos Data Marts requiere disciplina y gobernanza. Sin dimensiones conformadas bien gestionadas, terminas con islas de información que no se comunican bien entre sí.

---

### 1.2 Inmon: La Visión Corporativa desde el Principio (Top-down)

Bill Inmon tenía una visión diferente. Para él, el Data Warehouse debe ser la fuente única de verdad de la organización, construida de arriba hacia abajo. Primero diseñas y construyes un Data Warehouse corporativo completo y normalizado. Solo después derivas Data Marts específicos para cada departamento.

#### Cómo Funciona

Con Inmon, empiezas con un análisis exhaustivo de toda la empresa. Identificas todas las entidades de negocio relevantes: clientes, productos, empleados, transacciones, contratos, proyectos. Luego diseñas un modelo relacional normalizado en tercera forma normal (3NF).

La normalización elimina redundancias estructurales. Si mil clientes viven en Madrid, no guardas "Madrid" mil veces. En su lugar, tienes una tabla de ciudades con un único registro para Madrid, y los clientes referencian ese registro mediante una clave foránea. Cada dato se almacena en un solo lugar, lo que garantiza consistencia.

Este Data Warehouse corporativo actúa como el repositorio maestro. Cuando finanzas necesita un Data Mart, lo derivas del Data Warehouse. Cuando ventas necesita el suyo, también lo derivas del mismo lugar. Esto garantiza que ambos departamentos trabajen con definiciones consistentes de cliente, producto o cualquier otra entidad compartida.

#### Las Ventajas del Enfoque Inmon

La ventaja principal es la gobernanza. Tienes una única fuente de verdad, perfectamente auditada y controlada. Los datos son consistentes en toda la organización. Si el departamento legal necesita rastrear el histórico completo de un cliente para cumplir con regulaciones, ese histórico está completo y confiable.

La normalización también facilita agregar nuevas fuentes de datos sin duplicar información. Si integras datos de un nuevo sistema, simplemente los incorporas a las tablas normalizadas existentes.

#### Los Desafíos

El principal desafío es el tiempo. Construir un Data Warehouse corporativo completo puede llevar años. Durante ese tiempo, los departamentos de negocio siguen necesitando análisis, y la presión para entregar resultados es constante.

Además, las consultas analíticas sobre un modelo normalizado son más complejas. Para responder "ventas por región y producto", podrías necesitar hacer joins a través de cinco o seis tablas. Esto hace las consultas más lentas y difíciles de escribir para usuarios no técnicos. Por eso Inmon propone derivar Data Marts dimensionales desde el Data Warehouse normalizado: el mejor de ambos mundos, aunque con mayor complejidad arquitectónica.

---

### 1.3 Data Vault: Lo Mejor de Ambos Mundos (Híbrido)

Data Vault es un enfoque más moderno desarrollado por Dan Linstedt que intenta tomar lo mejor de Kimball e Inmon. Reconoce la necesidad de agilidad de Kimball y la necesidad de gobernanza de Inmon, y propone una estructura única que facilita ambas.

#### Cómo Funciona

Data Vault divide los datos en tres tipos fundamentales de estructuras:

**Hubs** representan las entidades de negocio principales. Un Hub contiene solo identificadores únicos: el ID del cliente, el ID del producto, el ID del pedido. Piensa en los Hubs como las "identidades" fundamentales de tu negocio. Son inmutables: una vez que creas un Hub para el cliente 1001, ese Hub nunca cambia ni se elimina.

**Links** capturan las relaciones entre Hubs. Si un cliente hace un pedido, creas un Link entre el Hub del cliente y el Hub del pedido. Si ese pedido incluye productos, creas Links entre el Hub del pedido y los Hubs de los productos. Los Links son también inmutables: representan hechos que ocurrieron (el cliente 1001 hizo el pedido 5000 el 15 de marzo de 2024).

**Satellites** contienen todos los atributos descriptivos y su historia. El nombre del cliente, su dirección, su teléfono: todo esto vive en Satellites adjuntos al Hub del cliente. Cada vez que un atributo cambia, se inserta un nuevo registro en el Satellite con la nueva versión y una marca temporal. Los Satellites son la única parte mutable de Data Vault, porque capturan los cambios en el tiempo.

```
HUB_CLIENTE ─────┐
    │            │
    │            LINK_CLIENTE_PEDIDO ───── HUB_PEDIDO
    │                                          │
SATELLITE_CLIENTE                        SATELLITE_PEDIDO
(nombre, dirección)                      (fecha, total)
con fechas de validez                    con fechas de validez
```

#### Las Ventajas del Enfoque Data Vault

La gran ventaja de Data Vault es su flexibilidad extrema. Si mañana necesitas integrar datos de un nuevo sistema CRM, simplemente agregas nuevos Satellites al Hub de cliente existente. No necesitas rediseñar la estructura principal. Si descubres una nueva relación de negocio que no habías modelado, agregas un nuevo Link sin tocar los Hubs ni Satellites existentes.

Esta estructura es también completamente auditable. Cada cambio queda registrado con su marca temporal. Puedes reconstruir el estado exacto de tu negocio en cualquier momento del pasado.

Data Vault funciona excepcionalmente bien en entornos con múltiples fuentes de datos heterogéneas que cambian frecuentemente. Es común en grandes empresas con sistemas legados, fusiones y adquisiciones, o en organizaciones que necesitan adaptarse rápidamente a cambios regulatorios.

#### Los Desafíos

La principal desventaja es la complejidad. Una consulta simple como "ventas del último mes" puede requerir joins a través de múltiples Hubs, Links y Satellites. El modelo no está optimizado directamente para análisis.

Por esta razón, Data Vault típicamente se usa en las capas internas de una arquitectura de datos moderna (la capa "Silver" en arquitecturas Medallion), y luego se derivan modelos dimensionales tipo Kimball en las capas de consumo (la capa "Gold") para facilitar el análisis.

---

## Tabla Comparativa: Kimball vs Inmon vs Data Vault

| Aspecto | Kimball | Inmon | Data Vault |
|---------|---------|-------|------------|
| **Filosofía** | Entrega rápida de valor, construcción incremental | Visión corporativa completa, gobernanza central | Flexibilidad y auditabilidad extrema |
| **Dirección** | Bottom-up (de abajo hacia arriba) | Top-down (de arriba hacia abajo) | Híbrido (centro hacia afuera) |
| **Modelo de datos** | Dimensional (estrella/copo de nieve) | Relacional normalizado (3NF) | Hubs, Links, Satellites |
| **Tiempo de implementación inicial** | Rápido (semanas) | Lento (meses/años) | Medio (meses) |
| **Complejidad de consultas** | Baja (joins simples) | Alta (múltiples joins) | Muy alta (muchos joins) |
| **Gestión del histórico** | Limitada (requiere técnicas SCD) | Completa pero compleja | Completa y natural |
| **Facilidad de cambios** | Media (puede requerir rediseño) | Baja (impacta toda la estructura) | Alta (agregar sin modificar existente) |
| **Intuitivo para usuarios** | Muy intuitivo | Poco intuitivo | Poco intuitivo |
| **Rendimiento analítico** | Excelente | Medio/Bajo | Bajo (requiere capa derivada) |
| **Ideal para** | Organizaciones orientadas al análisis rápido | Organizaciones con fuertes requisitos de gobernanza | Entornos complejos con múltiples fuentes cambiantes |
| **Mantenimiento** | Relativamente simple | Complejo | Muy complejo |
| **Escalabilidad** | Media (requiere dimensiones conformadas) | Media (monolítico) | Alta (modular) |

---

## Parte 2: Slowly Changing Dimensions - Gestionando el Cambio en el Tiempo

Independientemente del enfoque arquitectónico que elijas, enfrentarás un desafío fundamental: cómo gestionar los cambios en las dimensiones a lo largo del tiempo. Este es uno de los problemas más importantes en el diseño de Data Warehouses, y las Slowly Changing Dimensions (SCD) son las técnicas que lo resuelven.

### 2.1 El Problema Fundamental

En un sistema transaccional, cuando un cliente se muda de Madrid a Valencia, simplemente actualizas su campo "ciudad". El sistema ahora refleja que el cliente está en Valencia, y el hecho de que estuvo en Madrid se pierde para siempre.

En un Data Warehouse, esta pérdida de información es problemática. Si estás analizando la rentabilidad por región y ese cliente generó ventas mientras vivía en Madrid, ¿deberías atribuir esas ventas históricas a Madrid o a Valencia? La respuesta correcta depende de tu pregunta de negocio.

Si la pregunta es "¿dónde están mis clientes ahora?", solo te importa Valencia. Pero si la pregunta es "¿qué regiones generaron más ventas en 2023?", necesitas recordar que ese cliente estaba en Madrid durante 2023.

Las técnicas SCD te permiten gestionar estos escenarios de forma precisa y deliberada.

---

### 2.2 SCD Tipo 0: Atributos Inmutables

Algunos atributos simplemente no cambian. El país de nacimiento de una persona es el que es y será siempre. El género asignado al nacer (en contextos donde esto sea relevante legalmente) no cambia. El número de identificación fiscal de una empresa permanece constante.

Para estos atributos, la solución es simple: no implementas ninguna lógica de cambio. Guardas el valor una vez y lo dejas ahí permanentemente.

**Ejemplo práctico:**

```sql
CREATE TABLE dim_pais (
    pais_id INT PRIMARY KEY,
    codigo_iso CHAR(2) NOT NULL,
    nombre_oficial VARCHAR(100) NOT NULL,
    nombre_comun VARCHAR(100) NOT NULL
);

-- España siempre será España con código ES
INSERT INTO dim_pais VALUES (1, 'ES', 'Reino de España', 'España');
```

Este tipo de dimensión es la más simple de gestionar porque no hay cambios que rastrear. La usas cuando estás absolutamente seguro de que el atributo es inmutable por naturaleza o por decisión de negocio.

---

### 2.3 SCD Tipo 1: Sobrescritura sin Histórico

El SCD Tipo 1 es apropiado cuando los cambios representan correcciones de datos o cuando simplemente no te importa el valor anterior. Si un cliente actualiza su número de teléfono, probablemente no necesitas saber cuál era el anterior. Si alguien corrigió un error tipográfico en el nombre de un producto, definitivamente no quieres conservar la versión incorrecta.

**Cómo funciona:**

Simplemente actualizas el registro existente. El valor antiguo se sobrescribe y se pierde para siempre.

**Ejemplo práctico:**

```sql
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    telefono VARCHAR(20),
    email VARCHAR(100)
);

-- Cliente 1001 tiene un teléfono antiguo
INSERT INTO dim_cliente VALUES (1001, 'María González', '911222333', 'maria@email.com');

-- María actualiza su teléfono - simplemente lo sobrescribimos
UPDATE dim_cliente 
SET telefono = '911444555'
WHERE cliente_id = 1001;

-- El número antiguo '911222333' se ha perdido permanentemente
```

**Ventajas:**

La principal ventaja es la simplicidad. No hay lógica compleja, no hay tablas auxiliares, no hay versionado. La tabla se mantiene pequeña porque cada entidad tiene exactamente un registro. Las consultas son directas y rápidas.

**Desventajas:**

Pierdes la capacidad de hacer análisis histórico preciso. Si ese número de teléfono era importante para entender patrones de comunicación pasados, esa información ya no está disponible.

**Cuándo usarlo:**

Usa SCD Tipo 1 para atributos donde los cambios son raros y cuando ocurren son típicamente correcciones. Ejemplos incluyen correcciones de nombres mal escritos, actualizaciones de información de contacto no crítica, o cualquier atributo donde el valor actual es suficiente para todos los análisis posibles.

---

### 2.4 SCD Tipo 2: Versionado Completo con Histórico

El SCD Tipo 2 es la técnica más poderosa y más utilizada en Data Warehouses empresariales serios. Cuando un atributo cambia, no sobrescribes el registro existente. En su lugar, marcas el registro antiguo como "inactivo" e insertas un nuevo registro con los valores actualizados.

**Cómo funciona:**

Cada versión de la entidad recibe una nueva fila en la tabla de dimensión. Para distinguir entre múltiples versiones de la misma entidad de negocio, usas lo que se llama una "surrogate key" (clave sustituta): un identificador artificial que es único para cada fila, independientemente de la entidad de negocio que representa.

Además, agregas columnas para rastrear la validez temporal de cada versión: cuándo comenzó a ser válida y cuándo dejó de serlo.

**Ejemplo práctico:**

```sql
CREATE TABLE dim_cliente (
    cliente_sk INT PRIMARY KEY IDENTITY(1,1),  -- Surrogate key (auto-incremental)
    cliente_id INT NOT NULL,                    -- Business key (ID de negocio)
    nombre VARCHAR(100) NOT NULL,
    ciudad VARCHAR(50) NOT NULL,
    fecha_inicio DATE NOT NULL,                 -- Inicio de validez
    fecha_fin DATE NULL,                        -- Fin de validez (NULL = actual)
    es_actual BIT NOT NULL DEFAULT 1            -- Flag de versión actual
);

-- Versión 1: María vive en Madrid desde enero 2020
INSERT INTO dim_cliente (cliente_id, nombre, ciudad, fecha_inicio, fecha_fin, es_actual)
VALUES (1001, 'María González', 'Madrid', '2020-01-01', NULL, 1);

-- En enero 2024, María se muda a Valencia
-- Paso 1: Cerrar la versión antigua
UPDATE dim_cliente
SET fecha_fin = '2023-12-31',
    es_actual = 0
WHERE cliente_id = 1001 AND es_actual = 1;

-- Paso 2: Insertar nueva versión
INSERT INTO dim_cliente (cliente_id, nombre, ciudad, fecha_inicio, fecha_fin, es_actual)
VALUES (1001, 'María González', 'Valencia', '2024-01-01', NULL, 1);

-- Ahora tenemos dos registros:
-- cliente_sk=1: María en Madrid (2020-01-01 a 2023-12-31, inactivo)
-- cliente_sk=2: María en Valencia (2024-01-01 a NULL, activo)
```

**El concepto de Surrogate Key:**

Este es crucial para entender. En la tabla de dimensión, `cliente_sk` es la clave primaria, no `cliente_id`. Esto es necesario porque la misma María (cliente_id 1001) aparece en múltiples filas.

Las tablas de hechos se relacionan con `cliente_sk`, no con `cliente_id`. Esto significa que cuando registras una venta, guardas qué versión específica del cliente estaba vigente en ese momento. Una venta de marzo 2023 apuntaría a `cliente_sk=1` (María en Madrid), mientras que una venta de marzo 2024 apuntaría a `cliente_sk=2` (María en Valencia).

**Consultas poderosas que habilita:**

```sql
-- ¿Cuántos clientes teníamos en Madrid en junio 2023?
SELECT COUNT(DISTINCT cliente_id)
FROM dim_cliente
WHERE ciudad = 'Madrid'
  AND fecha_inicio <= '2023-06-30'
  AND (fecha_fin IS NULL OR fecha_fin >= '2023-06-30');

-- ¿Qué clientes se han mudado de región en los últimos 2 años?
SELECT c1.cliente_id, c1.ciudad AS ciudad_anterior, c2.ciudad AS ciudad_actual
FROM dim_cliente c1
INNER JOIN dim_cliente c2 ON c1.cliente_id = c2.cliente_id
WHERE c1.es_actual = 0
  AND c2.es_actual = 1
  AND c1.ciudad <> c2.ciudad
  AND c2.fecha_inicio >= DATEADD(year, -2, GETDATE());

-- Ventas por ciudad, atribuidas correctamente según dónde vivía el cliente
SELECT dc.ciudad, SUM(hv.importe) AS ventas_totales
FROM hechos_ventas hv
INNER JOIN dim_cliente dc ON hv.cliente_sk = dc.cliente_sk
WHERE hv.fecha_venta BETWEEN '2023-01-01' AND '2023-12-31'
GROUP BY dc.ciudad;
```

**Ventajas:**

El SCD Tipo 2 proporciona un historial completo y preciso. Puedes responder preguntas sobre cualquier punto en el tiempo. Es perfecto para auditorías, cumplimiento regulatorio y análisis de tendencias a largo plazo.

**Desventajas:**

La complejidad aumenta significativamente. Los procesos ETL deben detectar cambios y gestionar el versionado correctamente. La tabla de dimensión crece más rápido porque entidades estables tienen múltiples registros. Las consultas requieren filtros por fecha o por el flag `es_actual` para obtener la versión correcta.

Además, las actualizaciones masivas son más costosas. Si cambias un atributo para mil clientes, necesitas cerrar mil registros e insertar mil nuevos.

**Cuándo usarlo:**

Usa SCD Tipo 2 para dimensiones críticas donde el histórico importa. Ejemplos clásicos incluyen ubicación de clientes (para análisis regional), jerarquías organizacionales (para entender reporting structures pasados), categorías de productos (para análisis de mezcla de productos a lo largo del tiempo), o estados de empleados (activo/inactivo, departamento, puesto).

---

### 2.5 SCD Tipo 3: Memoria Limitada del Cambio Anterior

El SCD Tipo 3 es un compromiso intermedio. En lugar de crear múltiples versiones de registros, agregas columnas adicionales para almacenar el valor anterior del atributo que cambia. Esto te permite comparar el estado actual con el inmediatamente anterior, pero no más allá.

**Cómo funciona:**

Para cada atributo que quieres rastrear, creas dos (o tres) columnas: el valor actual, el valor anterior, y opcionalmente la fecha del último cambio.

**Ejemplo práctico:**

```sql
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    ciudad_actual VARCHAR(50) NOT NULL,
    ciudad_anterior VARCHAR(50) NULL,
    fecha_cambio_ciudad DATE NULL
);

-- María comienza en Madrid (sin ciudad anterior)
INSERT INTO dim_cliente VALUES 
(1001, 'María González', 'Madrid', NULL, NULL);

-- María se muda a Valencia
UPDATE dim_cliente
SET ciudad_anterior = ciudad_actual,
    ciudad_actual = 'Valencia',
    fecha_cambio_ciudad = '2024-01-01'
WHERE cliente_id = 1001;

-- Resultado:
-- cliente_id=1001, ciudad_actual='Valencia', ciudad_anterior='Madrid', 
-- fecha_cambio_ciudad='2024-01-01'

-- Si María se muda nuevamente a Sevilla
UPDATE dim_cliente
SET ciudad_anterior = ciudad_actual,  -- Valencia se convierte en "anterior"
    ciudad_actual = 'Sevilla',
    fecha_cambio_ciudad = '2025-01-01'
WHERE cliente_id = 1001;

-- Madrid se pierde - solo recordamos Valencia y Sevilla
```

**Ventajas:**

El modelo es más simple que SCD Tipo 2. No hay surrogate keys, no hay múltiples registros por entidad, no hay lógica de validez temporal. Las consultas son directas: para obtener el valor actual, simplemente accedes a `ciudad_actual`; para comparar con el anterior, accedes a `ciudad_anterior`.

Esto es útil para análisis del tipo "¿cuántos clientes cambiaron de región recientemente?" o "¿cuál es el impacto en ventas de clientes que se mudaron de zona rural a urbana?"

**Desventajas:**

Solo conservas un nivel de histórico. Si María ha vivido en tres ciudades en los últimos dos años, solo sabrás las dos más recientes. No puedes hacer análisis temporal verdadero como "¿dónde estaban nuestros clientes en cada trimestre de 2023?"

**Cuándo usarlo:**

Usa SCD Tipo 3 cuando necesitas comparar el estado actual con el inmediatamente anterior, pero no necesitas histórico completo. Ejemplos incluyen segmentos de cliente que cambian ocasionalmente (Premium→Gold), categorías de riesgo en análisis crediticio, o niveles de inventario (Normal→Crítico).

---

### 2.6 SCD Tipo 4: Separación de Histórico en Tabla Auxiliar

El SCD Tipo 4 adopta un enfoque diferente: en lugar de mezclar registros actuales e históricos en la misma tabla, los separas completamente. La tabla principal contiene solo el estado actual de cada entidad, mientras que una tabla histórica separada contiene todas las versiones pasadas.

**Cómo funciona:**

Mantienes dos tablas: una para el estado actual (optimizada para consultas rápidas del presente) y otra para el histórico (optimizada para almacenamiento y consultas temporales).

**Ejemplo práctico:**

```sql
-- Tabla principal: solo estado actual
CREATE TABLE dim_cliente (
    cliente_id INT PRIMARY KEY,
    nombre VARCHAR(100) NOT NULL,
    ciudad VARCHAR(50) NOT NULL,
    ultima_actualizacion DATETIME NOT NULL
);

-- Tabla histórica: todas las versiones pasadas
CREATE TABLE hist_cliente (
    hist_id INT PRIMARY KEY IDENTITY(1,1),
    cliente_id INT NOT NULL,
    nombre VARCHAR(100) NOT NULL,
    ciudad VARCHAR(50) NOT NULL,
    fecha_inicio DATE NOT NULL,
    fecha_fin DATE NOT NULL
);

-- Proceso cuando María se muda de Madrid a Valencia:
-- Paso 1: Mover versión actual a histórico
INSERT INTO hist_cliente (cliente_id, nombre, ciudad, fecha_inicio, fecha_fin)
SELECT cliente_id, nombre, ciudad, '2020-01-01', '2023-12-31'
FROM dim_cliente
WHERE cliente_id = 1001;

-- Paso 2: Actualizar tabla actual
UPDATE dim_cliente
SET ciudad = 'Valencia',
    ultima_actualizacion = '2024-01-01'
WHERE cliente_id = 1001;
```

**Ventajas:**

La tabla actual permanece pequeña y rápida. Las consultas sobre el estado presente no se ven afectadas por años de histórico. Esto es especialmente valioso en dimensiones muy grandes donde la mayoría de las consultas solo necesitan datos actuales.

La tabla histórica puede ser indexada y optimizada de forma diferente, incluso almacenada en medios más lentos pero más económicos si el acceso es infrecuente.

**Desventajas:**

La lógica de los procesos ETL se complica porque debes mantener sincronizadas dos tablas. Las consultas que necesitan combinar estado actual con histórico deben hacer UNION entre ambas tablas.

Las tablas de hechos solo pueden referenciar la tabla actual, lo que significa que pierdes la capacidad de hacer joins directos para análisis histórico profundo. Necesitas lógica adicional para reconstruir el contexto temporal correcto.

**Cuándo usarlo:**

Usa SCD Tipo 4 cuando tienes dimensiones muy grandes (millones de registros) donde el rendimiento de las consultas actuales es crítico, pero aún necesitas acceso ocasional al histórico completo. Ejemplos incluyen catálogos de productos enormes donde la mayoría de las consultas son sobre productos activos, o bases de clientes masivas donde los análisis históricos profundos son raros.

---

### 2.7 SCD Tipo 6: Híbrido de Tipos 1, 2 y 3

El SCD Tipo 6 (a veces llamado SCD Tipo 1+2+3) combina elementos de los tres primeros tipos para ofrecer máxima flexibilidad. Mantiene versiones completas como Tipo 2, pero también incluye columnas con valores actuales y anteriores como Tipo 3, y permite sobrescribir ciertos atributos no críticos como Tipo 1.

**Cómo funciona:**

Creas múltiples registros para cada entidad (como Tipo 2), pero cada registro también contiene columnas que siempre reflejan los valores actuales y anteriores, independientemente de la fecha de validez de ese registro.

**Ejemplo práctico:**

```sql
CREATE TABLE dim_cliente (
    cliente_sk INT PRIMARY KEY IDENTITY(1,1),     -- Surrogate key
    cliente_id INT NOT NULL,                       -- Business key
    nombre VARCHAR(100) NOT NULL,
    
    -- Columnas históricas (como SCD Tipo 2)
    ciudad_historica VARCHAR(50) NOT NULL,         -- Ciudad válida en este período
    fecha_inicio DATE NOT NULL,
    fecha_fin DATE NULL,
    es_actual BIT NOT NULL,
    
    -- Columnas current/previous (como SCD Tipo 3)
    ciudad_actual VARCHAR(50) NOT NULL,            -- Siempre la ciudad actual
    ciudad_anterior VARCHAR(50) NULL               -- Siempre la ciudad previa
);

-- Versión 1: María en Madrid (2020-2023)
INSERT INTO dim_cliente VALUES 
(1001, 'María González', 'Madrid', '2020-01-01', '2023-12-31', 0, 
 'Valencia', 'Madrid');  -- Ya incluye Valencia como actual

-- Versión 2: María en Valencia (2024+)
INSERT INTO dim_cliente VALUES 
(1001, 'María González', 'Valencia', '2024-01-01', NULL, 1,
 'Valencia', 'Madrid');  -- Mismos valores current/previous

-- La magia: todas las versiones históricas también saben cuál es la ciudad actual
```

Cuando María se muda, no solo insertas un nuevo registro: también actualizas las columnas `ciudad_actual` y `ciudad_anterior` en TODOS los registros históricos de esa cliente.

**Ventajas:**

Tienes lo mejor de todos los mundos. Puedes hacer análisis temporal preciso usando las columnas históricas y las fechas de validez. Al mismo tiempo, cualquier consulta puede acceder rápidamente al valor actual sin necesidad de filtrar por `es_actual=1`. Las comparaciones entre estado actual y anterior son directas sin importar qué versión histórica estés consultando.

**Desventajas:**

La complejidad es máxima. Cada cambio requiere no solo insertar un nuevo registro sino también actualizar múltiples registros históricos. El almacenamiento aumenta porque duplicas ciertos atributos. Los procesos ETL son significativamente más complejos y propensos a errores.

El rendimiento de escritura es peor porque cada UPDATE afecta potencialmente muchas filas.

**Cuándo usarlo:**

Usa SCD Tipo 6 solo en escenarios donde múltiples equipos analíticos tienen necesidades contradictorias que no puedes satisfacer de otra forma. Por ejemplo, un equipo necesita análisis histórico detallado mientras otro necesita acceso ultra-rápido a valores actuales. O cuando las regulaciones exigen que puedas auditar el histórico completo pero también necesitas rendimiento de consulta excepcional.

En la práctica, SCD Tipo 6 es raro. La mayoría de las organizaciones encuentran que SCD Tipo 2 con vistas materializadas para el estado actual proporciona mejor balance entre capacidades y complejidad.

---

## Tabla Comparativa Completa de SCD

| Tipo | Método | Historia Conservada | Estructura | Complejidad ETL | Complejidad Consultas | Tamaño Tabla | Velocidad Escritura | Velocidad Lectura | Ideal Para |
|------|--------|---------------------|------------|-----------------|----------------------|--------------|---------------------|-------------------|------------|
| **Tipo 0** | Valor fijo | No aplica (inmutable) | 1 fila por entidad | Mínima | Mínima | Mínimo | Muy rápida (insert único) | Muy rápida | Atributos inmutables por naturaleza |
| **Tipo 1** | Sobrescritura | Ninguna | 1 fila por entidad | Muy baja | Muy baja | Mínimo | Rápida (update simple) | Muy rápida | Correcciones, atributos no críticos |
| **Tipo 2** | Versionado | Completa | Múltiples filas por entidad | Alta | Media/Alta | Grande | Media (insert + update) | Media | Análisis temporal, auditoría, regulaciones |
| **Tipo 3** | Columna anterior | Solo último cambio | 1 fila por entidad | Baja/Media | Baja | Mínimo | Media (update con lógica) | Rápida | Comparaciones actual vs anterior |
| **Tipo 4** | Tabla separada | Completa (en tabla auxiliar) | 1 fila actual + n filas históricas | Alta | Alta (requiere UNION) | Variable | Media | Actual rápida, histórico lento | Dimensiones grandes con acceso histórico infrecuente |
| **Tipo 6** | Híbrido (1+2+3) | Completa + referencias actuales | Múltiples filas con columnas duplicadas | Muy alta | Baja/Media | Muy grande | Lenta (insert + updates múltiples) | Rápida | Requisitos analíticos múltiples y contradictorios |

---

## Ejemplos Prácticos Comparados

Para ilustrar las diferencias de forma concreta, veamos cómo manejarían las diferentes técnicas SCD un escenario real.

### Escenario: Seguimiento de Empleados

Tienes una dimensión de empleados donde necesitas rastrear cambios de departamento, cambios de puesto y cambios de gerente. Vamos a ver cómo lo abordarías con cada técnica.

**Situación inicial:** Juan Pérez es Analista en el departamento de Finanzas, reporta a María López.

**Cambio 1 (enero 2024):** Juan es promovido a Senior Analista en el mismo departamento.

**Cambio 2 (junio 2024):** Juan se transfiere a Operaciones manteniendo su puesto, ahora reporta a Carlos Ruiz.

#### Implementación SCD Tipo 1

```sql
CREATE TABLE dim_empleado_tipo1 (
    empleado_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    puesto VARCHAR(50),
    departamento VARCHAR(50),
    gerente_id INT
);

-- Estado inicial
INSERT INTO dim_empleado_tipo1 VALUES 
(1001, 'Juan Pérez', 'Analista', 'Finanzas', 2001);

-- Cambio 1: Sobrescribimos puesto
UPDATE dim_empleado_tipo1 
SET puesto = 'Senior Analista'
WHERE empleado_id = 1001;

-- Cambio 2: Sobrescribimos departamento y gerente
UPDATE dim_empleado_tipo1 
SET departamento = 'Operaciones', gerente_id = 3001
WHERE empleado_id = 1001;

-- Resultado final: solo sabemos el estado actual
-- empleado_id=1001, puesto='Senior Analista', 
-- departamento='Operaciones', gerente_id=3001
-- Se perdió toda información sobre Finanzas y la gerente anterior
```

**Análisis imposibles:**
- ¿Cuánto tiempo estuvo Juan en Finanzas?
- ¿Quién reportaba a María en marzo 2024?
- ¿Cuál era la estructura organizacional en el primer trimestre?

#### Implementación SCD Tipo 2

```sql
CREATE TABLE dim_empleado_tipo2 (
    empleado_sk INT PRIMARY KEY IDENTITY(1,1),
    empleado_id INT NOT NULL,
    nombre VARCHAR(100),
    puesto VARCHAR(50),
    departamento VARCHAR(50),
    gerente_id INT,
    fecha_inicio DATE,
    fecha_fin DATE NULL,
    es_actual BIT
);

-- Versión 1: Estado inicial
INSERT INTO dim_empleado_tipo2 VALUES 
(1001, 'Juan Pérez', 'Analista', 'Finanzas', 2001, 
 '2020-01-01', '2023-12-31', 0);

-- Versión 2: Después de promoción
INSERT INTO dim_empleado_tipo2 VALUES 
(1001, 'Juan Pérez', 'Senior Analista', 'Finanzas', 2001,
 '2024-01-01', '2024-05-31', 0);

-- Versión 3: Después de transferencia
INSERT INTO dim_empleado_tipo2 VALUES 
(1001, 'Juan Pérez', 'Senior Analista', 'Operaciones', 3001,
 '2024-06-01', NULL, 1);

-- Ahora tenemos historial completo
```

**Análisis posibles:**

```sql
-- ¿Quiénes reportaban a María (gerente_id=2001) en marzo 2024?
SELECT nombre, puesto
FROM dim_empleado_tipo2
WHERE gerente_id = 2001
  AND fecha_inicio <= '2024-03-01'
  AND (fecha_fin IS NULL OR fecha_fin >= '2024-03-01');

-- ¿Cuánto tiempo estuvo Juan en cada departamento?
SELECT departamento,
       DATEDIFF(month, fecha_inicio, COALESCE(fecha_fin, GETDATE())) AS meses
FROM dim_empleado_tipo2
WHERE empleado_id = 1001;

-- Headcount por departamento en cualquier momento
SELECT departamento, COUNT(DISTINCT empleado_id) AS empleados
FROM dim_empleado_tipo2
WHERE fecha_inicio <= '2024-03-31'
  AND (fecha_fin IS NULL OR fecha_fin >= '2024-03-31')
GROUP BY departamento;
```

#### Implementación SCD Tipo 3

```sql
CREATE TABLE dim_empleado_tipo3 (
    empleado_id INT PRIMARY KEY,
    nombre VARCHAR(100),
    puesto_actual VARCHAR(50),
    puesto_anterior VARCHAR(50),
    departamento_actual VARCHAR(50),
    departamento_anterior VARCHAR(50),
    gerente_actual_id INT,
    gerente_anterior_id INT,
    fecha_ultimo_cambio DATE
);

-- Estado inicial
INSERT INTO dim_empleado_tipo3 VALUES 
(1001, 'Juan Pérez', 'Analista', NULL, 'Finanzas', NULL, 2001, NULL, NULL);

-- Cambio 1: Promoción
UPDATE dim_empleado_tipo3
SET puesto_anterior = puesto_actual,
    puesto_actual = 'Senior Analista',
    fecha_ultimo_cambio = '2024-01-01'
WHERE empleado_id = 1001;

-- Cambio 2: Transferencia
UPDATE dim_empleado_tipo3
SET departamento_anterior = departamento_actual,
    departamento_actual = 'Operaciones',
    gerente_anterior_id = gerente_actual_id,
    gerente_actual_id = 3001,
    fecha_ultimo_cambio = '2024-06-01'
WHERE empleado_id = 1001;

-- Resultado:
-- puesto_actual='Senior Analista', puesto_anterior='Analista'
-- departamento_actual='Operaciones', departamento_anterior='Finanzas'
-- gerente_actual_id=3001, gerente_anterior_id=2001
```

**Análisis posibles:**

```sql
-- ¿Cuántos empleados han cambiado de departamento recientemente?
SELECT COUNT(*) AS empleados_transferidos
FROM dim_empleado_tipo3
WHERE departamento_actual <> departamento_anterior
  AND fecha_ultimo_cambio >= DATEADD(month, -6, GETDATE());

-- Comparar salarios promedio entre departamento actual y anterior
SELECT departamento_actual, 
       AVG(salario_actual) AS salario_promedio
FROM dim_empleado_tipo3
GROUP BY departamento_actual
UNION ALL
SELECT departamento_anterior,
       AVG(salario_anterior)
FROM dim_empleado_tipo3
WHERE departamento_anterior IS NOT NULL
GROUP BY departamento_anterior;
```

**Análisis imposibles:**
- No puedes saber dónde estaba Juan antes de Finanzas
- No puedes reconstruir la estructura organizacional de hace un año
- Pierdes detalle de cambios intermedios

---

## Decisiones de Diseño: Cómo Elegir

### Criterios de Selección de Arquitectura (Kimball vs Inmon vs Data Vault)

**Elige Kimball si:**
- Necesitas entregar valor rápidamente
- Tu organización está orientada al análisis de negocio
- Tienes recursos limitados de desarrollo
- Los usuarios necesitan acceso directo a los datos mediante herramientas de BI
- Tu estructura de datos es relativamente estable
- Puedes implementar gobernanza a través de dimensiones conformadas

**Elige Inmon si:**
- La gobernanza de datos es crítica (regulación, auditoría)
- Necesitas una fuente única de verdad certificada
- Tu organización puede esperar implementaciones de largo plazo
- Tienes múltiples sistemas fuente con definiciones conflictivas
- La consistencia de datos es más importante que la velocidad de consultas
- Planeas construir múltiples Data Marts derivados con lógica consistente

**Elige Data Vault si:**
- Integras datos de muchas fuentes heterogéneas
- Tu negocio cambia frecuentemente (fusiones, adquisiciones, nuevos productos)
- Necesitas trazabilidad completa de todos los cambios
- Trabajas en industrias altamente reguladas (finanzas, salud)
- Tu organización ya tiene madurez en ingeniería de datos
- Planeas implementar una arquitectura Medallion (Bronze-Silver-Gold) donde Data Vault esté en Silver

### Criterios de Selección de SCD

**Usa SCD Tipo 1 para:**
- Atributos que cambian por corrección de errores
- Información de contacto no crítica
- Atributos descriptivos que no afectan análisis histórico
- Dimensiones con pocos cambios donde el histórico no aporta valor

**Usa SCD Tipo 2 para:**
- Jerarquías organizacionales (departamentos, gerentes, ubicaciones)
- Categorías de cliente o producto que afectan análisis
- Estados o clasificaciones que determinan reglas de negocio
- Cualquier atributo donde necesites análisis temporal preciso
- Cuando existen requisitos de auditoría o regulatorios

**Usa SCD Tipo 3 para:**
- Atributos donde solo importa comparar actual vs inmediatamente anterior
- Análisis de migración (de un estado a otro)
- Escenarios donde el espacio de almacenamiento es muy limitado
- Cuando la simplicidad es más importante que el detalle histórico completo

**Usa SCD Tipo 4 para:**
- Dimensiones muy grandes (millones de registros) con cambios frecuentes
- Cuando el 95%+ de consultas solo necesitan el estado actual
- Requisitos de rendimiento muy exigentes en consultas actuales
- Cuando tienes almacenamiento diferenciado (rápido para actual, lento para histórico)

**Usa SCD Tipo 6 para:**
- Nunca, a menos que tengas requisitos extremadamente específicos que no puedes satisfacer de otra forma
- Considera primero si vistas materializadas sobre SCD Tipo 2 resuelven tu problema

---

## Conclusión

Los Data Warehouses son herramientas poderosas para transformar datos históricos en insights de negocio. La elección entre Kimball, Inmon y Data Vault determina la estructura fundamental de tu arquitectura y afecta la velocidad de implementación, la flexibilidad y el costo de mantenimiento.

Las técnicas de Slowly Changing Dimensions son igualmente fundamentales porque determinan qué tan bien puedes hacer análisis temporal. Un Data Warehouse sin gestión apropiada del histórico es simplemente una base de datos grande sin valor analítico real.

La clave está en entender tus requisitos de negocio específicos. ¿Necesitas entregar dashboards rápidamente o construir una fortaleza de gobernanza de datos? ¿Necesitas saber dónde estaban tus clientes hace dos años o solo quieres saber dónde están ahora? Las respuestas a estas preguntas guían tus decisiones técnicas.

En arquitecturas modernas de datos (Data Lakehouse, arquitecturas Medallion), es común combinar estos enfoques: usar Data Vault en capas de integración (Bronze/Silver) para flexibilidad y trazabilidad, y luego derivar modelos Kimball en capas de consumo (Gold) para análisis optimizado. Esto te da lo mejor de ambos mundos: la robustez y flexibilidad de Data Vault con la simplicidad analítica de Kimball.

Recuerda que no hay respuestas universales. Cada organización tiene necesidades diferentes, madurez diferente en datos, y restricciones diferentes. Lo importante es entender las opciones, sus trade-offs, y tomar decisiones informadas que se alineen con tus objetivos de negocio.
