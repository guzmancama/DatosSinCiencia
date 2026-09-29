# DatosSinCiencia

**Proyecto 1 — Bodega de datos, ETL y análisis descriptivo (RA1)**
Siniestralidad vial en Cali, 2016–2024.

## Estructura

```
.
├── entrega/
│   ├── 01_preclase.ipynb               # pre-clase del profesor
│   ├── Proyecto 1.md                   # enunciado y rúbrica
│   ├── fase_a_diseño.pdf               # Fase A: diseño de la bodega
│   ├── diagrama_bodega.png             # Fase A: diagrama estrella
│   ├── fase_b_etl.ipynb                # Fase B: pipeline ETL
│   ├── siniestralidad_2016_2024.csv    # datos fuente (crudos)
│   ├── fase_c_sql.ipynb                # Fase C (pendiente)
│   └── fase_d_visualizaciones.ipynb    # Fase D (pendiente)
├── .env.example                        # plantilla de configuración
├── .gitignore
├── requirements.txt
└── README.md
```

## Requisitos

- **Python 3.11 o superior** (probado en 3.12).
- **Docker**, para la base de datos.
- El archivo `entrega/siniestralidad_2016_2024.csv`, en codificación
  **ISO-8859-1 (latin-1)** y separado por **`;`**. El notebook lo detecta y avisa
  si no lo encuentra.

## Puesta en marcha

### 1. Dependencias de Python

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

En VS Code, seleccionar el intérprete de `.venv` (el que aparece con estrella en
la lista de kernels) para que `Kernel → Restart & Run All` use este entorno.

### 2. Base de datos PostgreSQL

El puerto **5432 suele estar ocupado** por otros proyectos locales, por eso la
bodega del curso usa el **5433**. El usuario, la contraseña y el nombre de la base
son los que elijas; deben coincidir con los de `DATABASE_URL`.

```bash
docker run -d --name bodega-siniestralidad \
  -e POSTGRES_USER=bodega \
  -e POSTGRES_PASSWORD=CAMBIAR_CLAVE \
  -e POSTGRES_DB=siniestralidad \
  -p 5433:5432 postgres:17
```

Para volver a levantarla más adelante:

```bash
docker start bodega-siniestralidad
```

### 3. Credenciales

```bash
cp .env.example .env
```

Y editar `DATABASE_URL` con el usuario y la contraseña del paso 2. En la URL el
host va como `localhost`, nunca como el nombre del contenedor.

`.env` es local y está en `.gitignore`; solo se versiona `.env.example`, que no
contiene ninguna contraseña real.

### 4. Ejecutar los notebooks

```bash
jupyter lab
```

Abrir `entrega/fase_b_etl.ipynb` y hacer **Kernel → Restart & Run All**. El
pipeline es **idempotente**: usa `if_exists="replace"`, así que se puede volver
a ejecutar tantas veces como se quiera sin duplicar datos.

El orden de las fases es **B → C → D**: la B carga la bodega, la C consulta, la D
visualiza.

## La bodega

Modelo estrella de 6 dimensiones alrededor de una tabla de hechos.

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
  incluye una fila `id = -1` (patrón de la pre-clase) para que el hecho nunca
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

## Advertencia: una cifra que se corrigió durante la auditoría

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

## Otras correcciones de la auditoría

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
