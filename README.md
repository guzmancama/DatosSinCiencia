# DatosSinCiencia

**Proyecto 1 — Bodega de datos, ETL y análisis descriptivo (RA1)**
Siniestralidad vial en Cali, 2016–2024.

## Integrantes

Grupo: **DatosSinCiencia**

| Nombre | Código estudiantil |
|---|---|
| Jesus Ediber Arenas Guzman | 2266066 |
| Juan Jose Guzman Camacho | 2380403 |
| Natalia Martinez Castañeda | 2380414 |
| Juan Pablo Rodriguez Gutierrez | 2380422 |
| Juan Jose Wallens Serna | 2380556 |

## Resumen del proyecto

Se construye una **bodega de datos dimensional (modelo estrella)** sobre los reportes de siniestralidad vial registrados en Cali entre 2016 y 2024. El proyecto incluye:

- **Fase A:** diseño conceptual/lógico de la bodega (diagrama y justificación) — `entrega/fase_a_diseño.pdf` + `entrega/diagrama_bodega.png`.
- **Fase B:** pipeline ETL idempotente en Python (pandas + SQLAlchemy + psycopg 3) que carga 7 tablas a PostgreSQL — `entrega/fase_b_etl.ipynb`.
- **Fase C:** 6 consultas SQL analíticas con interpretación de resultados — `entrega/fase_c_sql.ipynb`.
- **Fase D:** 8 visualizaciones (serie de tiempo, comparación categórica, relación bivariada, distribución univariada) con su **Insight** — `entrega/fase_d_visualizaciones.ipynb`.
- **Fase E:** informe escrito y presentación — `entrega/fase_e_informe.pdf`.

La bodega tiene **1 tabla de hechos** y **6 tablas de dimensión**, con clave subrogada entera en todas las dimensiones, 6 FK `NOT NULL` con índice en el hecho y una única medida (`numero_vehiculos`). Tras limpieza mínima (1 fila vacía + 4 duplicados exactos) quedan **370.998 reportes individuales**.

## Estructura del repositorio

```
.
├── entrega/
│   ├── fase_a_diseño.pdf               # Fase A: diseño y justificación de la bodega
│   ├── diagrama_bodega.png             # Fase A: diagrama modelo estrella
│   ├── fase_b_etl.ipynb                # Fase B: pipeline ETL (extracción-transformación-carga)
│   ├── siniestralidad_2016_2024.csv    # Datos fuente (crudos) — NO versionado
│   ├── fase_c_sql.ipynb                # Fase C: 6 consultas SQL analíticas
│   ├── fase_d_visualizaciones.ipynb    # Fase D: 8 visualizaciones + insights
│   ├── fase_e_informe.pdf              # Fase E: informe escrito (4–6 págs.)
│   └── figuras/                        # Figuras exportadas (8 PNG) usadas en informe
│       ├── v1_serie_trimestral.png
│       ├── v2_variacion_trimestral.png
│       ├── v3_tipo_accidente_todas.png
│       ├── v4_tipo_accidente_tipados.png
│       ├── v5_medio_predominante_anual.png
│       ├── v6_vehiculos_bivariada.png
│       ├── v7_combinaciones_vehiculos.png
│       └── v8_distribucion_trimestral.png
├── .env.example                        # Plantilla de configuración (DATABASE_URL)
├── .gitignore
├── requirements.txt
└── README.md
```

> **Nota de nomenclatura (rúbrica):** el entregable pide `entrega/informe.pdf`. En este repositorio el archivo se encuentra como `entrega/fase_e_informe.pdf`. Ambos corresponden al mismo informe (Fase E); el README documenta este nombre tal como está versionado en el repo.

## Datos fuente

El proyecto parte del **CSV crudo** de siniestralidad vial en Cali 2016–2024 (el dataset del catálogo del curso, sin pre-procesar). El archivo **no está versionado** en el repositorio (`.gitignore` excluye `entrega/*.csv`, son ~27 MB): se descarga de la fuente del curso y se coloca manualmente en `entrega/`.

| Detalle | Valor |
|---|---|
| Ruta esperada | `entrega/siniestralidad_2016_2024.csv` (o en la raíz del repo) |
| Separador | `;` |
| Codificación | ISO-8859-1 (**latin-1**), no UTF-8 |
| Tamaño | 371.003 filas × 10 columnas |
| Nulo real de la fuente | `.` (punto), además de vacío/espacio/`-`/`N/A` |

Los notebooks lo leen con `sep=";"`, `encoding="latin-1"` y `dtype=str`, y validan que se lean exactamente **371.003** filas. Además, si no encuentran el archivo lanzan un `FileNotFoundError` con instrucciones paso a paso de dónde colocarlo.

> La fuente trae 2 pares de columnas con el mismo nombre (`Tipo clase de accidente` y `Tipo confirmado`). Al leer con pandas, las repetidas se desambiguan con sufijo `.1`.

## Requisitos

- **Python 3.11 o superior** (probado en 3.12).
- **Una instancia de PostgreSQL accesible** (en este proyecto, una base en **Supabase**; el pipeline es agnóstico: solo necesita que `DATABASE_URL` apunte a un PostgreSQL disponible).
- **JupyterLab** (viene en `requirements.txt`) para ejecutar los tres notebooks.
- El archivo de datos `entrega/siniestralidad_2016_2024.csv` (ver sección anterior).

## Puesta en marcha

### 1. Dependencias de Python

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

En VS Code, seleccionar el intérprete de `.venv` (el que aparece con estrella en la lista de kernels) para que `Kernel → Restart & Run All` use este entorno.

### 2. Base de datos PostgreSQL

El proyecto usa una base de datos **Supabase** (el enunciado no obliga a un servidor local). Obtener la cadena de conexión desde el panel de Supabase → *Project Settings → Database → Connection string* y copiarla con el rol `session` (las consultas son SQL analítico de solo lectura, cualquier rol con permiso de escritura sobre el esquema `public` sirve).

El formato que espera el pipeline es:

```
postgresql+psycopg://usuario:clave@host:puerto/base
```

- `postgresql+psycopg` corresponde a **psycopg 3** (el driver que instala `requirements.txt`). Usar `+psycopg2` únicamente si se instaló `psycopg2` en su lugar.
- El host va como el que entrega el panel, **no** como un contenedor local.
- En Supabase conviene usar el *pooler* (host tipo `aws-<region>.pooler.supabase.com`) en lugar de la conexión directa por IPv6.

### 3. Credenciales

```bash
cp .env.example .env
```

Y editar `DATABASE_URL` en `.env` con el usuario, la contraseña, el host, el puerto y la base de su instancia (ver `.env.example`, que ya trae la plantilla del formato Supabase).

`.env` es local y está en `.gitignore`; solo se versiona `.env.example`, que **no contiene ninguna contraseña real** (solo marcadores `CAMBIAR_...`).

### 4. Ejecutar los notebooks

```bash
jupyter lab
```

Los notebooks están en `entrega/`; se abren con **Kernel → Restart & Run All** y **sin cambiar el directorio de trabajo** (el pipeline calcula la raíz del repo a partir de donde se ejecuta el kernel). El orden de las fases es obligatorio:

1. **`fase_b_etl.ipynb`** — carga la bodega en PostgreSQL (E → T → L).
2. **`fase_c_sql.ipynb`** — ejecuta las 6 consultas analíticas.
3. **`fase_d_visualizaciones.ipynb`** — genera las 8 figuras (se exportan a `entrega/figuras/`).

El pipeline de la Fase B es **idempotente**: dentro de una sola transacción hace `DROP TABLE ... CASCADE` y recrea las 7 tablas, así que se puede reejecutar tantas veces como se quiera sin duplicar datos ni dejar la base a medias. La Fase C y la Fase D fallan con un mensaje claro si la Fase B aún no se ha ejecutado.

## Reproducibilidad (checklist para quien revisa)

```bash
git clone https://github.com/guzmancama/DatosSinCiencia.git
cd DatosSinCiencia
python3 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

Luego:

1. Colocar `entrega/siniestralidad_2016_2024.csv` (ver *Datos fuente*).
2. `cp .env.example .env` y completar `DATABASE_URL`.
3. `Restart & Run All` en los notebooks `fase_b_etl.ipynb` → `fase_c_sql.ipynb` → `fase_d_visualizaciones.ipynb`.

No hay pasos manuales adicionales: el ETL crea las 7 tablas, las carga y verifica su integridad al terminar.

## Entregables por fase

### Fase A — Diseño de la bodega (25 %)

`entrega/fase_a_diseño.pdf` + `entrega/diagrama_bodega.png`

Modelo **estrella** con 1 tabla de hechos y 6 dimensiones (se cumple el mínimo de 3 dimensiones). El documento justifica por qué existe cada dimensión, cuál es su granularidad y qué atributos tiene, y explica por qué se prefirió estrella a copo de nieve.

### Fase B — Pipeline ETL (30 %)

`entrega/fase_b_etl.ipynb` — un solo **Restart & Run All**.

- **Extracción:** lectura del CSV crudo con pandas (`sep=";"`, `latin-1`, `dtype=str`), con validación de las 371.003 filas esperadas.
- **Transformación:** limpieza mínima (1 fila vacía, 4 duplicados), diccionario de reparación de texto aplicado hasta punto fijo, remapeo de 5 celdas con datos personales a `OTRO`, generación de **claves subrogadas**, combinación canónica de vehículos y conversión explícita de tipos.
- **Carga:** `to_sql` con SQLAlchemy + psycopg 3 dentro de una **única transacción**, leyendo `DATABASE_URL` de `.env`.
- **Manejo de errores:** cada fallo de conexión se traduce a una instrucción accionable (puerto, usuario, base inexistente, driver faltante, `.env` ausente, CSV ausente).
- **Controles:** reconciliación de FK, unicidad de claves, integridad referencial y enumeraciones, verificados **antes y después** de cargar; además se prueba que los controles detectan el bug reintroducido a propósito.

### Fase C — Consultas SQL analíticas (20 %)

`entrega/fase_c_sql.ipynb` — 6 consultas, cada una con pregunta de negocio, tabla de resultado e interpretación.

| # | Pregunta de negocio | Técnica SQL que demuestra |
|---|---|---|
| 1 | ¿Cuáles son los 10 tipos de accidente más frecuentes y qué % del total representan? | `JOIN` explícito + `SUM(COUNT(*)) OVER ()` |
| 2 | ¿Cómo evolucionó el volumen por año/trimestre y su variación %? | `GROUP BY` + **`LAG()`** como función de ventana |
| 3 | ¿Cuáles son las combinaciones más frecuentes de tipo × medio × severidad? | **`GROUP BY` con 3 dimensiones** + 3 `JOIN` |
| 4 | ¿Cuáles son las 10 direcciones más críticas y el tipo predominante en cada una? | CTE + **`RANK() OVER (PARTITION BY ...)`** |
| 5 | ¿Qué combinaciones de vehículos se asocian a eventos multivehículo? | `AVG`, `SUM(CASE ...)`, `HAVING` |
| 6 | ¿Cuál es el medio de reporte predominante cada año? | CTE + **`ROW_NUMBER() OVER (PARTITION BY ...)`** |

Cumple los tres mínimos de la rúbrica: `GROUP BY` con más de dos dimensiones (consulta 3), función de ventana (consultas 2, 4 y 6) y `JOIN` explícito dimensión–hecho (las 6).

### Fase D — Visualizaciones (20 %)

`entrega/fase_d_visualizaciones.ipynb` — 8 figuras (matplotlib + seaborn), cada una con título, ejes rotulados, unidades y un bloque **Insight** de 2–3 frases en Markdown. Las figuras se exportan a `entrega/figuras/`.

| # | Figura | Tipo exigido por la rúbrica | Consulta |
|---|---|---|---:|
| 1 | Reportes por trimestre | Serie de tiempo | 2 |
| 2 | Variación porcentual trimestral | Serie de tiempo | 2 |
| 3 | Reportes por tipo de accidente (todas) | Comparación categórica | 1 |
| 4 | Accidentes con tipo registrado | Comparación categórica | 1 |
| 5 | Medio de reporte predominante por año | Comparación categórica en el tiempo | 6 |
| 6 | Promedio de vehículos vs. % multivehículo | Relación bivariada | 5 |
| 7 | Combinaciones de vehículos frecuentes | Comparación categórica | 5 |
| 8 | Distribución de reportes por trimestre | Distribución univariada | 2 |

Cubre los 4 tipos obligatorios: serie de tiempo, comparación categórica, relación bivariada y distribución univariada.

### Fase E — Informe y presentación (5 %)

`entrega/fase_e_informe.pdf` — informe de 6 páginas con contexto y preguntas de negocio, diagrama de la bodega, descripción del ETL, **3 hallazgos clave** apoyados en las figuras, y limitaciones/próximos pasos. La presentación oral de 8 minutos se realiza en la Clase 5.

## Hallazgos clave

1. **La serie se parte en dos regímenes de registro, no es una tendencia.** Los reportes caen de 64.323 (2016) a 12.080 (2024), −81 %, pero no de forma gradual: hasta 2019 el trimestre se mueve entre 13.900 y 18.100, 2020-T2 se desploma (−67 %) y desde ahí sigue bajando. De los 35 trimestres completos, ninguno cae en la zona intermedia, así que la media no describe ningún trimestre real. El vacío de `medio_reporte` hasta 2020 apunta a un **cambio de cobertura del registro**, no a una mejora vial.
2. **Solo 1 de cada 9 reportes es un accidente con tipo, y 3 de cada 4 de esos son choques.** Solo 42.860 reportes (11,6 %) tienen tipo registrado; entre ellos el choque concentra el 76,1 % y el atropello el 15,4 %. Cualquier cifra de "siniestros" calculada sobre todas las filas sobreestima los accidentes cerca de nueve veces.
3. **La motocicleta interviene en más de un tercio de los reportes que informan vehículos.** Entre los 176.577 reportes (47,6 %) con `numero_vehiculos` informado, las combinaciones que incluyen motocicleta suman al menos 62.920 (35,6 %), frente al 49,9 % que concentran "automóvil solo" y "automóvil + motocicleta".

## Limitaciones (resumen)

- **2024 está incompleto**: los datos llegan al 2 de noviembre de 2024; no se compara con años completos.
- **La serie no es homogénea**: el −81 % y el vacío de `medio_reporte` sugieren un cambio de cobertura cuya causa no está documentada en el dataset.
- **Reportes, no accidentes**: el 88 % de las filas no tiene tipo de accidente (negativo, sin dato o "No aplica").
- **Muchos nulos, sin imputar**: `numero_vehiculos` 52,4 %, `medio_reporte` 77,7 %, `direccion_reporte` 49,5 %.
- **Sin `FOREIGN KEY` en el motor**: es un modelo OLAP; la integridad se valida en el ETL y después de cargar, pero PostgreSQL no la impone.
- **Faltan hora y coordenadas**: el dataset no permite análisis espacial fino ni por franja horaria.

El detalle completo está en la sección *Limitaciones y próximos pasos* del informe.

## La bodega

Modelo estrella de 6 dimensiones alrededor de una tabla de hechos.

![Diagrama del modelo estrella](entrega/diagrama_bodega.png)

```
                              dim_fecha
                                   |
 dim_ubicacion  ───┐               |
                   ├──▶  hecho_siniestralidad  ◀── dim_medio_reporte
 dim_tipo_accidente┘     (un reporte)          └── dim_tipo_confirmado
                                │
                   dim_vehiculos_implicados
```

- `hecho_siniestralidad` — granularidad: **un reporte individual** de
  siniestralidad, identificado por su código. **370.998 filas y 9 columnas**.
- Las 6 dimensiones usan **clave subrogada** entera (`id` 1..n), y cada una
  incluye una fila `id = -1` (patrón de la pre-clase del curso) para que el hecho nunca
  tenga una FK nula. En las cinco dimensiones de texto esa fila dice
  `DESCONOCIDO`; en `dim_fecha` su `fecha` queda **NULL**, porque no existe una
  "fecha desconocida": existe una fecha que no se informó, y los atributos de
  texto derivados (`nombre_mes`, `dia_semana`) son los que llevan `DESCONOCIDO`.
- Las **7 tablas** declaran `PRIMARY KEY`, y el hecho además lleva **6 FK
  `NOT NULL` con índice** (7 índices en total sobre la tabla de hechos).
- La bodega **no declara `FOREIGN KEY`**: es un modelo OLAP, la relación existe
  en el diseño, se valida en el ETL y se verifica después de cargar, pero no la
  impone el motor.
- Única medida: `numero_vehiculos`. Llega con más de la mitad de sus valores en
  nulo y su disponibilidad **cambia según el año**, así que se carga tal cual,
  **sin imputar**, y se documenta como limitación.

### Valores que produce el pipeline

| Tabla | Filas | Qué hay |
|---|---:|---|
| `hecho_siniestralidad` | 370.998 | 1 reporte por fila, 9 columnas, 6 FK `NOT NULL`, 1 medida |
| `dim_fecha` | 3.230 | fecha, año, mes, trimestre, día de la semana, fin de semana |
| `dim_ubicacion` | 59.349 | dirección normalizada + fila `DESCONOCIDO` |
| `dim_tipo_accidente` | 10 | tipo de accidente + `SIN ACCIDENTE (NEGATIVO)` + `NO APLICA` |
| `dim_tipo_confirmado` | 38 | clasificación general y su detalle, con `SIN DETALLE` |
| `dim_medio_reporte` | 11 | medio de reporte, con `OTRO` fusionado y narrativas anonimizadas |
| `dim_vehiculos_implicados` | 735 | combinación canónica + número de tipos y 2 banderas de moto |

## Datos: de dónde salen las 370.998 filas

| Paso | Filas |
|---|---:|
| Lectura del CSV | 371.003 |
| Se descarta 1 fila totalmente vacía | 371.002 |
| Se eliminan 4 filas duplicadas exactas | **370.998** |

Las 4 filas duplicadas son el mismo reporte cargado dos veces: coinciden en las
10 columnas. Tras eliminarlas, `codigo_reporte` es único, y por eso puede ser la
clave natural del hecho (aunque se prefiera la clave subrogada; el notebook
muestra la evidencia).

## Auditoría del ETL

Esta sección documenta los **bugs y decisiones de diseño** que se encontraron y corrigieron durante el desarrollo del pipeline. Se conserva aquí para que el revisor entienda por qué los números del ETL son los que son.

### Una cifra que se corrigió durante la auditoría

El conteo de `id_vehiculos_implicados = -1` **no** es 23.185 sino **23.187**, y
esa diferencia no es un error de redondeo: era un bug real.

`pandas.Series.mask()` trata un `NA` en la condición como si fuera `True`. La
condición "sin tipo de vehículo **y** reporte NEGATIVO" quedaba en `NA` en 2
filas que no son NEGATIVO, así que esas 2 filas se marcaban como
`SIN ACCIDENTE (NEGATIVO)` y no llegaban a la centinela. Con
`.fillna(False)` la condición es `True`/`False` de verdad:

| Concepto | Filas |
|---|---:|
| `id_vehiculos_implicados = -1` (DESCONOCIDO) | **23.187** |
| `SIN ACCIDENTE (NEGATIVO)` | **199.031** |
| Combinaciones de vehículos reales | **148.780** |

El notebook no fuerza ningún número: reconcilia los tres contra el CSV crudo y
falla si no cuadran.

### Otras correcciones de la auditoría

- **Diccionario de reparación hasta punto fijo.** La fuente trae 70 filas con
  `NOQUIEREINTERVENCION` pegado (sin espacio). Una sola pasada de sustitución
  no las alcanzaba porque la regla se definía en términos de la forma pegada; el
  diccionario se aplica ahora hasta punto fijo y hay un control que falla si
  alguna categoría conserva esa forma. `dim_tipo_confirmado` quedó en 38 filas,
  no en 39.
- **Meses y días en español**, con mapas explícitos y `dia_semana_num`
  (1 = lunes … 7 = domingo). No se usa `locale` ni la fecha del día en que se
  corre el notebook.
- **Sin columna `etiqueta`.** El valor de negocio ya está en su propia columna;
  duplicarlo generaba dos `VARCHAR(260)` repetidos en 59.349 filas y una
  ambigüedad en los `JOIN`.
- **`vehiculos_declarados` ya no existe** en el hecho: era 1 en ~199 mil
  reportes que no tuvieron accidente, porque contaba las filas centinela como si
  fueran un vehículo.
- **Los controles se prueban.** El notebook reintroduce a propósito el bug de
  `where` en vez de `mask` y comprueba que los tres controles lo detectan. Un
  control que nunca se ha visto fallar no es un control.
