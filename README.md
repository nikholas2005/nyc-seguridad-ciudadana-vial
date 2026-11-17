# Big Data aplicado a la seguridad ciudadana y vial en Nueva York

Diagnóstico territorial de **arrestos** y **choques de tránsito** en la ciudad de Nueva York entre 2021 y 2026, procesado con **Apache Spark** por distrito y por las **55 PUMAs** del censo. Se cruzan los dos indicadores con condiciones sociales (pobreza, desempleo, educación), acceso digital y espacio público, y se entrenan modelos de aprendizaje automático con **MLlib** para proponer un plan de acción al gobierno de la ciudad.

El proyecto sigue la metodología **CRISP-DM** y se desarrolló en dos entregas.

---

## Equipo y curso

| | |
|---|---|
| **Integrantes** | Nikholas Barsky Angulo · Laura Zoet Guacaneme Gómez |
| **Curso** | Procesamiento de Datos a Gran Escala (1255) |
| **Profesor** | John Corredor Franco, PhD |
| **Universidad** | Pontificia Universidad Javeriana, Bogotá |
| **Periodo** | Segundo semestre de 2026 |

---

## Estructura del repositorio

```
README.md
entrega-1/   Entendimiento del negocio y de los datos
entrega-2/   Preparación de datos, modelado, evaluación y plan de acción
```

| Carpeta | Contenido | Fases de CRISP-DM |
|---|---|---|
| [`entrega-1`](entrega-1) | Informe de la primera entrega (PDF), cuaderno de Jupyter y presentación ejecutiva | 1. Entendimiento del negocio · 2. Entendimiento de los datos · 3. Avance de la preparación |
| [`entrega-2`](entrega-2) | Informe final (PDF), cuaderno final `proyecto_NY_entrega_final.ipynb` y presentación ejecutiva final | 3. Preparación de los datos · 4. Modelado · 5. Evaluación · 6. Implantación |

El informe final es la continuación del informe de la primera entrega, con un resumen inicial y las secciones nuevas.

---

## Problema de negocio

El gobierno de Nueva York considera prioritarios dos indicadores: **la cantidad de arrestos** y **la cantidad de choques viales**. El equipo actúa como consultor y su objetivo es:

> Diseñar un plan de acción basado en datos para que el gobierno de Nueva York reduzca los arrestos y los choques con víctimas, priorizando zonas, horarios y grupos de población.

**Objetivos específicos**

1. Identificar las PUMAs prioritarias.
2. Identificar las condiciones que hacen que un choque deje heridos o muertos.
3. Relacionar la pobreza, la educación y el acceso digital con los arrestos para orientar programas preventivos.

**Criterios de éxito**

- **Negocio:** un grupo de PUMAs que no pase del 20 % de las zonas y concentre al menos el 40 % de los arrestos o de los choques con víctimas; cada recomendación ligada a un hallazgo.
- **Data mining:** AUC-ROC de prueba de al menos 0.70 (meta propia del equipo: 0.75), superando a la regresión logística y a la clase mayoritaria; silueta de KMeans mayor a 0.30 con grupos que se puedan describir.

---

## Fuentes de datos

Todas las fuentes son oficiales. El periodo de análisis es 2021-2026: 2025 es el año completo más reciente y 2026 es un año parcial.

| Fuente | Años | Uso |
|---|---|---|
| NYPD Arrest Data, YTD y Historic (NYC Open Data) | 2021-2026 | Primer indicador: arrestos |
| Motor Vehicle Collisions: Crashes, Vehicles y Person (NYC Open Data) | 2021-2026 | Segundo indicador: choques, vehículos, causas y víctimas |
| ACS PUMS, personas (U.S. Census Bureau) | 2024 | Población, pobreza, desempleo y educación; denominador de todas las tasas |
| School Quality Reports, SAT (NYC Department of Education) | 2023-24 y 2024-25 | Desempeño educativo |
| Citywide Public Computer Centers (NYC Open Data) | Año más reciente | Acceso digital |
| NYC Parks Active and Passive Recreation (NYC Open Data) | 2026 | Espacio público según el manual CEQR |
| Límites de las PUMAs (geografía 2020) | 2020 | Asignación de puntos a zonas y mapas |
| Población del NYSDOH, por web scraping | Año más reciente | Validación de la población (bono 1 de la primera entrega) |
| OpenWeatherMap, API | Tiempo real | Contexto climático (bono 2 de la primera entrega) |
| Open-Meteo, Historical Weather API (ERA5) | 2021-2026, por hora | Clima de la hora de cada choque, como variable del modelo |

---

## Infraestructura

- **Apache Spark en modo Standalone**, con un Master y varios Workers compartidos por el curso.
- **Almacenamiento compartido por NFS** y bases intermedias guardadas en **Parquet**, para no recalcular lo que ya existe.
- Driver con 8 GB de memoria y ejecutores con 2 núcleos y 2 GB cada uno.
- Spark 4.2.0 en el clúster y PySpark 4.0.4 en el Driver. Como los Workers no ejecutan código Python, todo el procesamiento distribuido usa funciones nativas de Spark SQL y de MLlib.
- Zona horaria de la sesión: `America/New_York`.

---

## Qué se hizo

**Primera entrega: entendimiento del negocio y de los datos**

- Contexto de la ciudad e indicadores macroeconómicos.
- Selección, carga y descripción de las fuentes en el clúster: tipos de datos, significado de los atributos y contenido.
- Exploración con estadísticos descriptivos, tablas y gráficas.
- Reporte de calidad con conteo de faltantes y técnicas para tratarlos.
- Planteamiento de 10 preguntas de negocio.
- Primeros filtros, limpieza y transformaciones.

**Segunda entrega: preparación, modelado, evaluación e implantación**

- Filtros y transformaciones finales: asignación de cada evento a su PUMA, tasas por cada 100,000 habitantes, variables de calendario y de clima.
- Respuesta a las 10 preguntas de negocio con consultas en Spark, tablas y gráficas.
- Preparación para el modelado: control de fuga, eliminación de variables muy correlacionadas, normalización y selección de variables finales.
- **Modelo supervisado:** clasificar si un choque deja víctimas con RandomForest, bagging, regresión logística y una red neuronal (perceptrón multicapa), con búsqueda de parámetros mediante TrainValidationSplit.
- **Modelo no supervisado:** agrupar las 55 PUMAs en tipos de zona con KMeans, comparado con GaussianMixture.
- Evaluación en una partición de prueba separada desde el inicio y verificación de los criterios de éxito.
- Plan de acción por tipo de zona, con un indicador de seguimiento para cada acción.

---

## Resultados principales

**Modelo supervisado** (154,401 choques de prueba)

| Modelo | AUC-ROC de prueba |
|---|---|
| Red neuronal (capas ocultas 128, 64 y 32) | 0.7940 |
| RandomForest (100 árboles, profundidad 15) | 0.7893 |
| Bagging | 0.7890 |
| Regresión logística | 0.7435 |
| Clase mayoritaria | 0.5000 |

- Con su umbral, la red neuronal detecta el 81 % de los choques con víctimas.
- Las variables que más pesan en RandomForest son el número de vehículos y la presencia de bicicleta o e-bike.

**Modelo no supervisado**

- KMeans con 6 grupos y silueta de 0.2953, superior a GaussianMixture (0.1209).
- Los grupos se leen como tendencias, porque las zonas de la ciudad cambian de forma gradual.

**Hallazgos para el plan**

- Las 10 PUMAs con mayor tasa de arrestos concentran el 33.35 % de los arrestos de 2025 con el 17.86 % de la población.
- Cinco PUMAs están a la vez en el 25 % más alto de arrestos y de choques con víctimas: 4221 y 4263 del Bronx, 4165 y 4111 de Manhattan y 4305 de Brooklyn.
- Los peatones tienen la mayor letalidad: 13.74 muertos por cada 1,000 lesionados.
- Las motos y la velocidad insegura están muy sobrerrepresentadas en los choques con muertos.
- Los choques con víctimas se concentran en las tardes de los días laborales, entre las 15:00 y las 17:00.

**Criterios de éxito:** se cumplen 5 de 7.

- **Sí se cumplen:** el AUC, superar a la regresión logística, superar a la clase mayoritaria, responder las 10 preguntas y tener grupos describibles.
- **No se cumplen, aunque quedan cerca:**
  - Concentración: 38.72 % frente al 40 % que pedía el criterio.
  - Silueta: 0.2953 frente a 0.30.

---

## Cómo reproducir

1. Descargar las fuentes listadas arriba en la carpeta de datos compartida del clúster.
2. Abrir `entrega-2/proyecto_NY_entrega_final.ipynb` en Jupyter, desde el Driver o por un túnel SSH.
3. Ajustar la ruta de datos (`DATA_PATH`) y la dirección del Master si el clúster es otro.
4. Ejecutar el cuaderno en orden.
   - Con `REUSAR_PARQUET = True`, las bases ya procesadas se leen desde Parquet y no se recalculan.
   - Los modelos entrenados se guardan, para no reentrenarlos si se reinicia el kernel.

La clave de la API de OpenWeatherMap no está en el repositorio. Se lee de un archivo aparte en la carpeta compartida.

---

## Limitaciones

- Los resultados muestran **asociaciones, no causas**. Por ejemplo, la relación entre pobreza y arrestos no prueba que una cause la otra.
- El modelo de choques usa información que se conoce después del choque, así que sirve para **priorizar zonas y condiciones de riesgo**, no para predecir en tiempo real.
- La base no registra variables clave como la velocidad real o el uso de cinturón. Por eso los modelos se quedan cerca de un AUC de 0.79.
- Las comparaciones por distrito tienen solo 5 casos y se leen como pistas.

---

## Metodología

El trabajo sigue las seis fases de CRISP-DM: entendimiento del negocio, entendimiento de los datos, preparación de los datos, modelado, evaluación e implantación. Las referencias se gestionaron con Zotero y aparecen en formato APA en los informes.
