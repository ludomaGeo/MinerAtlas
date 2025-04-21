**MinerAtlas** es un proyecto en el marco de la maestría en ciencias de información feoespacial pensado con el objetivo de analizar y visualizar la relación entre concesiones mineras y variables socioambientale en México. 
EL proposito del proyecto es identificar zonas de riesgo, impactos territoriales y posibles conflictos derivados de la actividad minera.

---

## Herramientas utilizadas

- **PostgreSQL + PostGIS**: almacenamiento y procesamiento de datos espaciales.
- **QGIS**: visualización de resultados y edición espacial.
- **DBeaver**: cliente gráfico para bases de datos.
- **R / RStudio**: cálculo externo del índice de riesgo (se recomienda incluir el script `riesgo.Rmd`).

---

##  Estructura del repositorio

```
MinerAtlas/
│
├── Consultas_sql_minerAtlas.md      ← Documento con todas las consultas SQL, documentadas y explicadas
├── datos_usados/                    ← Carpeta con archivos geográficos utilizados (.gpkg, .csv, etc.)
│   └── readme.md                    ← Explicación de los datos utilizados
├── riesgo.Rmd           ← Script en R para cálculo del índice de riesgo
├── README.md                        ← Este archivo
```

---

## ¿Cómo replicar este proyecto?

1. Clona este repositorio:
   ```
   git clone https://github.com/ludomaGeo/MinerAtlas.git
   ```

2. Asegúrate de tener instalado:
   - PostgreSQL + PostGIS
   - QGIS
   - DBeaver
   - R + RStudio (para la parte de riesgo)

3. Carga los archivos de `datos_usados/` en tu base de datos PostGIS.

4. Ejecuta las consultas del archivo `Consultas_sql_minerAtlas.md` en el orden documentado.

5. Si deseas calcular el índice de riesgo, usa el script en R (`riesgo.Rmd`) y une el resultado con las capas espaciales en QGIS.
