Este archivo contiene el conjunto completo y documentado de consultas SQL utilizadas en el desarrollo del sistema de información geografica **MinerAtlas**. Su objetivo principal es permitir el análisis territorial de las concesiones mineras en México con base en su relación espacial con cuerpos de agua, áreas naturales protegidas, conflictos socioambientales, tenencia de la tierra y materiales extraídos.

## Herramientas utilizadas

Para ejecutar correctamente las consultas y replicar el análisis del proyecto **MinerAtlas**, es importante contar con el siguiente entorno y herramientas:

- **PostgreSQL 16 + PostGIS**  
  Para la gestión de bases de datos espaciales y ejecución de todas las consultas SQL.  
  Se recomienda instalar:
  - PostgreSQL
  - PostGIS 

- **QGIS 3. 30**  
  Usado para visualizar capas geográficas, realizar uniones espaciales, exportar resultados y generar mapas finales. También se utilizó para unir capas generadas desde R con datos espaciales.

- **DBeaver 24.3.3**  
  Cliente de base de datos utilizado para interactuar de forma gráfica con PostgreSQL/PostGIS. Facilita la escritura, prueba y depuración de consultas.

- **Rstudio**  
  Utilizado en la etapa de estimación del índice de riesgo. El cálculo fue externo a SQL y los resultados se integraron en QGIS para análisis espacial. Se recomienda tener instalado RStudio si se desea replicar esta parte.

## Estructura general del archivo

Este docuemnto se presenta en tres grandes bloques, que explican a grandes rasgos la logica del procesamiento:

## 1. Preparación y procesamiento general
En este primer apartado se nncluye la creación de la base de datos, activación de extensiones, verificación de SRID y generación de índices espaciales. Son pasos previos necesarios para optimizar la base de datos.

## 2. Consultas de análisis espacial
Este bloque contiene las consultas principales que analizan la relación espacial entre las concesiones mineras y capas de interés: conflictos, cuencas, ANP, cuerpos de agua, núcleos agrarios y materiales extraídos.

## 3. Estimación del índice de riesgo (proceso externo en R)
El índice de riesgo socioambiental fue calculado en R a partir de las variables normalizadas exportadas desde SQL. Los resultados fueron posteriormente integrados con QGIS para su análisis espacial y visualización.  

---

### 1.1 Creación de base de datos y extensión espacial

Se crea una base de datos para el proyecto y se habilita la extensión PostGIS para trabajar con geometrías espaciales.
``` sql
CREATE DATABASE sig_conflictos;
CREATE EXTENSION postgis;
```

---

### 1.2  Verificación de SRID y creación de índices espaciales

Consulta para revisar si las capas utilizan el mismo sistema de coordenadas (SRID), requisito fundamental para hacer operaciones espaciales correctas, se presenta la estructura con dos ejemplos pero se verifica con todas las capas que se utilizaorn
```sql
SELECT DISTINCT ST_SRID(geom) FROM public."Concesiones_mineras";
SELECT DISTINCT ST_SRID(geom) FROM public.conflictos_mineros_2;
```

Se crean índices espaciales que permiten acelerar las consultas geográficas como `ST_Intersects` o `ST_DWithin`.
```sql
CREATE INDEX idx_anp_geom ON "ANP — anp" USING GIST (geom);
CREATE INDEX idx_concesiones_geom ON "Concesiones_mineras" USING GIST (geom);
CREATE INDEX idx_ran_geom ON "Ran_nacional" USING GIST (geom);
CREATE INDEX idx_conflictos_geom ON conflictos_mineros_2 USING GIST (geom);
CREATE INDEX idx_cuencas_geom ON cuencas USING GIST (geom);
CREATE INDEX idx_cuerpos_agua_geom ON cuerpos_agua USING GIST (geom);
CREATE INDEX idx_estados_geom ON estados USING GIST (geom);
CREATE INDEX idx_minas_geom ON minas_puntuales USING GIST (geom);
CREATE INDEX idx_municipios_geom ON municipios USING GIST (geom);
```

---
## 2. Consultas de análisis espacial
### 2.1 Concesiones y conflictos

Se cuenta cuántos conflictos hay cerca de cada concesión minera (radio de 5 km) y se almacena en una nueva tabla.
```sql
CREATE TABLE conflictos_concesion AS
SELECT c.titulo AS concesion, c.nombrelote, c.titular, COUNT(cm.fid) AS total_conflictos_cercanos
FROM public."Concesiones_mineras" c
LEFT JOIN conflictos_mineros_2 cm ON ST_DWithin(c.geom, cm.geom, 5000)
GROUP BY concesion, c.nombrelote, c.titular
ORDER BY total_conflictos_cercanos DESC;
```

Se identifican las empresas titulares de concesiones con mayor número de lotes mineros registrados.
```sql
CREATE TABLE tabla_empresas_concesion AS
SELECT c.titular AS empresa, COUNT(*) AS numero_concesiones
FROM "Concesiones_mineras" c
GROUP BY c.titular
ORDER BY numero_concesiones DESC;
```

Consulta para mostrar las 10 concesiones dentro de cuencas con mayor cantidad de conflictos mineros cercanos.
```sql
SELECT cu.cuenca, c.titulo, c.nombrelote, c.titular, COUNT(cm.fid) AS total_conflictos_cercanos
FROM public."Concesiones_mineras" c
LEFT JOIN conflictos_mineros_2 cm ON ST_DWithin(c.geom, cm.geom, 5000)
LEFT JOIN cuencas cu ON ST_Intersects(c.geom, cu.geom)
GROUP BY cu.cuenca, c.titulo, c.nombrelote, c.titular
ORDER BY total_conflictos_cercanos DESC
LIMIT 10;
```

Ranking de cuencas con mayor número de conflictos mineros en su interior.
```sql
SELECT cu.cuenca, COUNT(cm.fid) AS total_conflictos
FROM "cuencas" cu
JOIN conflictos_mineros_2 cm ON ST_Intersects(cm.geom, cu.geom)
GROUP BY cu.cuenca
ORDER BY total_conflictos DESC
LIMIT 10;
```

Se obtiene un top 10 de concesiones mineras con más conflictos cercanos para análisis puntual.
```sql
SELECT c.titulo AS concesion, c.nombrelote, c.titular, COUNT(cm.fid) AS total_conflictos_cercanos
FROM public."Concesiones_mineras" c
LEFT JOIN conflictos_mineros_2 cm ON ST_DWithin(c.geom, cm.geom, 5000)
GROUP BY c.titulo, c.nombrelote, c.titular
ORDER BY total_conflictos_cercanos DESC
LIMIT 10;
```

---

### 2.1 Cercanía a cuerpos de agua

Consulta que mide cuántos cuerpos de agua hay cerca de cada concesión minera, en un radio de 2 km.
```sql
CREATE TABLE tabla_cuerpos_agua AS
SELECT c.titulo, c.nombrelote, c.titular, COUNT(ca.id) AS cuerpos_agua_cercanos
FROM "Concesiones_mineras" c
LEFT JOIN cuerpos_agua ca ON ST_DWithin(c.geom, ca.geom, 2000)
GROUP BY c.id, c.titulo, c.nombrelote, c.titular
ORDER BY cuerpos_agua_cercanos DESC;
```

---

### 2.2 Relación con Áreas Naturales Protegidas (ANP)

Consulta simple que cuenta cuántas ANP están cercanas a cada concesión minera en un rango de 3 km.
```sql
SELECT c.titulo, c.nombrelote, COUNT(a.id_anp) AS anp_cercanas
FROM "Concesiones_mineras" c
LEFT JOIN "ANP — anp" a ON ST_DWithin(c.geom, a.geom, 3000)
GROUP BY c.nombrelote, c.titulo
ORDER BY anp_cercanas DESC;
```

Consulta más avanzada que agrega los nombres de las ANP cercanas por concesión y los guarda en una tabla.
```sql
CREATE TABLE tabla_resultados_anp_total AS
SELECT c.titulo, c.nombrelote, COUNT(DISTINCT a.id_anp) AS anp_cercanas,
       STRING_AGG(DISTINCT a.nombre, ', ') AS nombres_anp
FROM "Concesiones_mineras" c
LEFT JOIN "ANP — anp" a ON ST_DWithin(c.geom, a.geom, 3000)
GROUP BY c.titulo, c.nombrelote
ORDER BY anp_cercanas DESC;
```

---

### 2.3 Ocupación de núcleos agrarios

Cuenta cuántos núcleos agrarios tienen al menos una concesión minera dentro de su territorio.
```sql
SELECT COUNT(DISTINCT r."Clv_Unica") AS nucleos_con_concesiones
FROM public."Ran_nacional" r
JOIN "Concesiones_mineras" c ON ST_Intersects(c.geom, r.geom);
```

Agrega una columna con el área de cada núcleo agrario en kilómetros cuadrados.
```sql
ALTER TABLE public."Ran_nacional" ADD COLUMN area_km2 NUMERIC;
UPDATE public."Ran_nacional" SET area_km2 = ST_Area(geom) / 1000000.0;
```

Calcula la superficie ocupada por concesiones mineras dentro de cada núcleo agrario.
```sql
SELECT r."Clv_Unica" AS id_nucleo, r."NOM_NUC" AS nucleo, r."tipo",
       ROUND(SUM(ST_Area(ST_Intersection(c.geom, r.geom))::numeric) / 1000000.0, 2) AS area_concesion_km2
FROM public."Ran_nacional" r
JOIN "Concesiones_mineras" c ON ST_Intersects(c.geom, r.geom)
GROUP BY r."Clv_Unica", r."NOM_NUC", r."tipo"
ORDER BY area_concesion_km2 DESC;
```

Consulta extendida que calcula también el porcentaje del núcleo ocupado por minería.
```sql
SELECT r."Clv_Unica" AS id_nucleo, r."NOM_NUC" AS nucleo, r."tipo" AS tipo_tenencia,
       c.titulo AS id_concesion, c.nombrelote AS lote, c.titular AS empresa,
       ROUND(ST_Area(r.geom)::numeric / 1000000.0, 2) AS area_nucleo_km2,
       ROUND(ST_Area(ST_Intersection(c.geom, r.geom))::numeric / 1000000.0, 2) AS area_ocupada_km2,
       ROUND(ST_Area(ST_Intersection(c.geom, r.geom))::numeric / NULLIF(ST_Area(r.geom)::numeric, 0) * 100.0, 2) AS porcentaje_ocupacion_nucleo
FROM public."Ran_nacional" r
JOIN "Concesiones_mineras" c ON ST_Intersects(c.geom, r.geom)
ORDER BY porcentaje_ocupacion_nucleo DESC;
```

---

### 2.4 Asignación de materiales extraídos

Añade una nueva columna para registrar el mineral asociado a cada concesión según la intersección con minas puntuales.
```sql
ALTER TABLE "Concesiones_mineras" ADD COLUMN material TEXT;

UPDATE "Concesiones_mineras" c
SET material = mp."MINERALES"
FROM minas_puntuales mp
WHERE ST_Intersects(mp.geom, c.geom);
```

Consulta para verificar qué concesiones ya tienen mineral asignado.
```sql
SELECT titulo, material
FROM "Concesiones_mineras"
WHERE material IS NOT NULL;
```

---
## 3. Estimación del índice de riesgo (proceso externo en R)

Después de realizar los procesos de consulta y preparación de datos en DBeaver, la información fue trasladada a R, donde se calculó el índice de riesgo combinando distintas variables relevantes. Este cálculo permitió obtener un valor que resume el nivel de riesgo asociado a cada unidad espacial. Posteriormente, los resultados fueron exportados y utilizados en QGIS para generar un mapa que representa visualmente las zonas con mayor y menor riesgo. A continuación, se presenta el código utilizado para el cálculo del índice en R.
## 3.1 Cálculo del índice de riesgo socioambiental


