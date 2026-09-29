# DatosSinCiencia

**Proyecto 1 — Bodega de datos, ETL y análisis descriptivo (RA1)**
Siniestralidad vial en Cali, 2016–2024.

## Qué hay en la carpeta

```
.
├── entrega/
│   ├── fase_a_diseño.pdf               # Fase A: diseño de la bodega
│   ├── diagrama_bodega.png             # Fase A: diagrama estrella
│   ├── fase_b_etl.ipynb                # Fase B: pipeline ETL
│   ├── fase_c_sql.ipynb                # Fase C: 6 consultas SQL
│   ├── fase_d_visualizaciones.ipynb    # Fase D: pendiente
│   └── informe.pdf                     # Fase E: pendiente
├── siniestralidad_2016_2024.csv        # datos fuente (crudos, no se sube al repo)
├── .env.example                        # plantilla de configuración
├── .gitignore
├── requirements.txt
└── README.md
```

## Lo que hace falta para correr esto

- **Python 3.11 o más nuevo** (lo probamos en 3.12).
- Una cuenta y un proyecto en **[Supabase](https://supabase.com)** — es donde
  vive la base Postgres. El plan gratis alcanza de sobra.
- El archivo `siniestralidad_2016_2024.csv` en la raíz del proyecto (o dentro
  de `entrega/`, el notebook busca en los dos lugares). Viene en codificación
  **ISO-8859-1 (latin-1)** y separado por `;` — así lo entrega el curso, no
  hay que tocarlo. **No se sube al repo** (está en `.gitignore`): pesa
  bastante y algunas filas traen datos personales metidos donde no debían.

## Cómo ponerlo a andar

### 1. Instalar dependencias

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

Si estás en VS Code, elegí el intérprete de `.venv` en la lista de kernels
antes de correr los notebooks (el que sale marcado con una estrella).

### 2. Crear el proyecto en Supabase

Si todavía no tenés uno: entrá a
[supabase.com/dashboard](https://supabase.com/dashboard), creá un proyecto
nuevo y guardá la contraseña de la base que te pide ahí (si se te olvida,
después se puede resetear desde **Settings → Database**).

### 3. Configurar las credenciales

```bash
cp .env.example .env
```

Y completá `DATABASE_URL` con el connection string de tu proyecto: en el
dashboard de Supabase, **Settings → Database → Connection string →
Connection pooling**, copiá esa URL y agregale `+psycopg2` justo después de
`postgresql` (el `.env.example` trae el formato exacto y más detalle de
dónde sacar cada parte).

`.env` es local y nunca se sube (está en `.gitignore`); lo único que se
versiona es `.env.example`, que no tiene ninguna clave real.

### 4. Correr los notebooks, en orden

```bash
jupyter lab
```

1. `entrega/fase_b_etl.ipynb` — **Kernel → Restart & Run All**. Limpia y
   carga la bodega completa en Supabase. Es repetible: si ya habían corrido
   una versión anterior de este pipeline (incluso una con otro diseño de
   tablas), esta misma celda de carga borra lo viejo antes de crear todo de
   nuevo, así que no hace falta borrar nada a mano en Supabase.
2. `entrega/fase_c_sql.ipynb` — corre después de la B, porque consulta las
   tablas que esa dejó pobladas.
3. `entrega/fase_d_visualizaciones.ipynb` — todavía no está hecho.

## Cómo queda armada la bodega

Modelo estrella: seis dimensiones alrededor de una tabla de hechos.

```
                              dim_fecha
                                   |
 dim_ubicacion  ───┐               |
                   ├──▶  hecho_siniestralidad  ◀── dim_medio_reporte
 dim_tipo_accidente┘     (un reporte)          └── dim_tipo_confirmado
                                │
                   dim_vehiculos_implicados
```

- `hecho_siniestralidad` — cada fila es **un reporte individual**,
  identificado por su código. **370.998 filas, 9 columnas.**
- Las 6 dimensiones usan **clave subrogada** entera (`id` 1..n), y cada una
  trae además una fila `id = -1` para que el hecho nunca se quede con una FK
  en `NULL`. En las cinco dimensiones de texto esa fila dice `DESCONOCIDO`;
  en `dim_fecha` la columna `fecha` queda en `NULL` (porque no existe una
  "fecha desconocida", existe una fecha que no se informó), y ahí son los
  atributos derivados (`nombre_mes`, `dia_semana`) los que dicen
  `DESCONOCIDO`.
- Las **7 tablas** tienen `PRIMARY KEY`, el hecho tiene sus **6 `FOREIGN
  KEY` reales** hacia cada dimensión (con `NOT NULL` e índice en cada una).
  Como la fila `-1` ya existe en cada dimensión antes de cargar el hecho,
  declarar las FK no choca en nada con el patrón del `-1`.
- La única medida es `numero_vehiculos`. Llega nula en más de la mitad de
  las filas y esa proporción cambia según el año, así que se carga tal cual
  viene, **sin inventar nada**, y queda documentado como limitación.

### Lo que deja cargado el pipeline

| Tabla | Filas | Qué tiene |
|---|---:|---|
| `hecho_siniestralidad` | 370.998 | 1 reporte por fila, la medida y las 6 FK `NOT NULL` |
| `dim_fecha` | 3.230 | fecha, año, mes, trimestre, día de la semana, fin de semana |
| `dim_ubicacion` | 59.349 | dirección normalizada + fila `DESCONOCIDO` |
| `dim_tipo_accidente` | 10 | tipo de accidente + `SIN ACCIDENTE (NEGATIVO)` + `NO APLICA` |
| `dim_tipo_confirmado` | 38 | clasificación general y su detalle, con `SIN DETALLE` |
| `dim_medio_reporte` | 11 | medio de reporte, con `OTRO` fusionado y narrativas anonimizadas |
| `dim_vehiculos_implicados` | 735 | combinación canónica + número de tipos y 2 banderas de moto |

## De dónde salen las 370.998 filas

| Paso | Filas |
|---|---:|
| Lectura del CSV | 371.003 |
| Se descarta 1 fila totalmente vacía | 371.002 |
| Se eliminan 4 pares de filas duplicadas exactas | **370.998** |

Los 4 duplicados son el mismo reporte cargado dos veces, idéntico en las 10
columnas. El notebook los detecta comparando fila contra fila (no solo por
código repetido) y solo borra los que coinciden en todo — si dos eventos
distintos llegaran a compartir código, los dos se conservan.

## Cosas que sabemos que le faltan a esto

- **Fase D** (visualizaciones) y **Fase E** (informe) todavía no están
  hechas.
- El mojibake (los `?` sueltos del CSV) solo se repara donde armamos un
  diccionario a mano; en `direccion_reporte` queda sin tocar a propósito,
  porque ahí es texto libre y una regla pareja arreglaría unos casos y
  dañaría otros.
- Había narrativas con nombres, cédulas y placas metidas en la columna
  "Medio de reporte". Se detectan por el largo del texto y se anonimizan
  como `OTRO`; ese contenido no se carga en ningún lado.
- El periodo 2016–2024 no es parejo para comparar año contra año: el último
  año queda incompleto y el volumen general cae bastante hacia el final.
  Todo esto queda explicado con cifras dentro del propio notebook de la
  Fase B.
