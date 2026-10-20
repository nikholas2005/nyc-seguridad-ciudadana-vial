# Big Data aplicado a la seguridad ciudadana y vial en Nueva York

Diagnóstico territorial de **arrestos** y **choques de tránsito** en la ciudad de Nueva York entre 2021 y 2026, procesado con **Apache Spark** por distrito y por las **55 PUMAs** del censo. Se cruzan los dos indicadores con condiciones sociales y servicios públicos.

Proyecto del curso **Procesamiento de Datos a Gran Escala (clase 1255)**, Pontificia Universidad Javeriana, Bogotá.
Profesor: John Corredor Franco, PhD.

**Equipo consultor:**
- Nikholas Barsky Angulo
- Laura Zoet Guacaneme Gómez

---

## Caso de negocio

El equipo actúa como consultor contratado por el estado de Nueva York para proponer un plan de acción, basado en procesamiento de datos, que mejore dos indicadores: la cantidad de arrestos y la cantidad de accidentes viales.

**Objetivo del consultor:** construir con datos públicos un diagnóstico territorial por distrito y por PUMA que identifique zonas críticas, patrones temporales y factores asociados, para sustentar un plan de acción basado en evidencia.

## Metodología: CRISP-DM

| Fase | Estado |
|---|---|
| 1. Entendimiento del negocio | Completa |
| 2. Entendimiento de los datos | Completa |
| 3. Preparación de los datos | En avance |
| 4. Modelado | Entrega final |
| 5. Evaluación | Entrega final |
| 6. Implantación | Entrega final |

## Infraestructura

| Componente | Detalle |
|---|---|
| Modo | Apache Spark Standalone, Nube Privada Javeriana |
| Nodos | Un Master y un Driver; los Workers se comparten con otros estudiantes del curso |
| Versiones | Spark 4.2.0 en el clúster; PySpark 4.0.4 y Python 3.9 en el Driver |
| Ejecutor | 2 núcleos y 2 GB |
| Almacenamiento | Carpeta compartida por NFS (el clúster del curso no tiene HDFS) |
| Salida | Bases limpias en Parquet |

Los Workers no tienen Python, así que todo el procesamiento distribuido usa **funciones nativas de Spark SQL**, sin UDF de Python. El único cálculo sin función nativa es ubicar cada punto en su PUMA: se hace en el Driver con `contains_points` de matplotlib y se une a los datos con un *broadcast join*.

## Fuentes de datos

| Fuente | Años | Rol |
|---|---|---|
| [NYPD Arrest Data, Year to Date](https://data.cityofnewyork.us/d/uip8-fykc) | 2026 | Arrestos recientes |
| [NYPD Arrests Data, Historic](https://data.cityofnewyork.us/d/8h9b-rp9u) | 2021-2025 | Tendencia de arrestos |
| [Motor Vehicle Collisions: Crashes](https://data.cityofnewyork.us/d/h9gi-nx95) | 2021-2026 | Choques, heridos y muertos |
| [Motor Vehicle Collisions: Vehicles](https://data.cityofnewyork.us/d/bm4k-52h4) | 2021-2026 | Tipo de vehículo y causas |
| [Motor Vehicle Collisions: Person](https://data.cityofnewyork.us/d/f55k-p6yu) | 2021-2026 | Peatones, ciclistas y ocupantes |
| [ACS PUMS, U.S. Census Bureau](https://www.census.gov/programs-surveys/acs/microdata.html) | 2024 | Población, pobreza, desempleo y educación |
| School Quality Reports (SAT), NYC Department of Education | 2023-24 y 2024-25 | Educación |
| [Citywide Public Computer Centers](https://data.cityofnewyork.us/d/sejx-2gn3) | 2026 | Acceso digital |
| [NYC Parks Active and Passive Recreation](https://data.cityofnewyork.us/d/kcqe-vnci) | 2026 | Espacio público |
| [2020 Public Use Microdata Areas (PUMAs)](https://data.cityofnewyork.us/d/pikk-p9nv) | 2020 | Geografía de las 55 PUMAs |
| NYSDOH, Vital Statistics (web scraping) | 2023 | Validación de la población (bono 1) |
| [OpenWeatherMap API](https://openweathermap.org/api) | Tiempo real | Clima (bono 2) |

En total son cerca de 4.7 millones de registros. El análisis empieza en 2021 porque desde abril de 2020 la NYPD no registra los choques que solo dejan daños materiales; así todo el periodo tiene un único régimen de registro.

## Calidad y preparación de los datos

| Problema | Tratamiento | Resultado |
|---|---|---|
| Edad y sexo faltan en el 52.78% de los arrestos de 2025 | "NO ESPECIFICADO"; perfiles con 2021-2024 y 2026 | Ningún arresto eliminado |
| El distrito falta en el 29.77% de los choques | Voto mayoritario de su celda de 0.01° (cerca de 1 km²) | Baja a 5.34%, con consistencia de 99.32% |
| La hora 00:00 tiene entre 27 y 49 veces más choques que un minuto normal | Se marca como hora de relleno y sale del análisis por hora | Desaparece el pico falso de medianoche |
| El dato de muertos falta en el 24.37% de los choques de 2026 | Los muertos de 2026 no se comparan | Sin conclusiones falsas |
| En parques, cada parte repite el área total | Una fila por `GISPROPNUM` | De 2,936 filas a 1,811 parques; de 63,971 a 27,041 acres |
| Ocupantes imposibles en vehículos | Tope por tipo de vehículo | 111 valores anulados, el 0.01% |

Ningún faltante se imputa con la media: los vacíos vienen de cómo están hechos los formularios, no son al azar.

## Hallazgos principales

**Arrestos**
- 278,953 arrestos en 2025, un 79.4% más que en 2021.
- Tasa de la ciudad: 3,290.2 por cada 100,000 habitantes; el Bronx tiene la más alta, 4,534.6.
- Por PUMA la tasa va de 690 a 9,841. La más alta, la PUMA 4221 del Bronx, es también la más pobre (43.77%).
- 13 de 62 tipos de delito explican el 80.43% de los arrestos de 2025.
- El 69.2% de los arrestos de menores de 18 años es por delito grave.

**Seguridad vial, 2021-2025**
- 487,914 choques, 261,634 heridos y 1,364 muertos.
- La velocidad está en el 4.82% de los vehículos en choques, pero en el 23.90% de los vehículos en choques mortales.
- Los peatones son el 16.93% de los heridos, pero el 45.23% de los muertos (52.0% en 2025). Su letalidad es 4.4 veces la de los ocupantes.
- Las motos tienen 7.5 veces más choques mortales por vehículo que los sedanes.

**Territorio y servicios**
- Pobreza: 18.15% en la ciudad y 28.96% en el Bronx (ACS PUMS 2024).
- Arrestos y pobreza por PUMA: r = 0.51, y 0.70 sin Manhattan. Choques con víctimas y pobreza: r = 0.03.
- La ciudad tiene 0.66 acres de recreación activa por cada 1,000 habitantes, un tercio de la meta CEQR de 2.0; 52 de 55 PUMAs están por debajo.

Son asociaciones exploratorias y no causas.

## Estructura del repositorio

```
├── README.md
├── notebook/
│   └── proyecto_NY_final.ipynb
├── informe/
│   └── Informe_Proyecto_NY.pdf
└── presentacion/
    └── Big_Data_seguridad_ciudadana_y_vial_NY.pdf
```

Los datos no se incluyen por su tamaño; se descargan desde los enlaces de la tabla de fuentes.

## Cómo ejecutar el notebook

1. Tener un clúster Spark en modo Standalone y una carpeta de datos visible desde todos los nodos.
2. Descargar las fuentes en esa carpeta con los nombres que indica la celda `ARCHIVOS` del notebook.
3. En la sección de configuración, ajustar `SPARK_MASTER_IP` y `DATA_PATH`.
4. Para el bono del clima, guardar la API key de OpenWeatherMap en un archivo `.owm_api_key` dentro de `DATA_PATH` o en la carpeta personal. **La clave nunca se escribe en el notebook ni se sube al repositorio.**
5. Con `REUSAR_PARQUET = True`, las bases limpias se guardan en Parquet y las corridas siguientes las reutilizan.

**Librerías del Driver:** PySpark 4.0.4, pandas 2.3.3, matplotlib 3.9.4, seaborn 0.13.2, numpy y requests.

## Siguiente paso (entrega final)

- Un modelo supervisado y uno no supervisado con Spark MLlib sobre la tabla de 55 PUMAs.
- Cruce del clima histórico con los choques.
- Respuesta a las 10 preguntas de negocio.
- Propuesta de plan de acción focalizado para el equipo de gobierno.
