# Visor Censo · Resguardo Indígena de Males – Córdoba (Nariño) 2026

Visor geográfico interactivo del censo poblacional del **Resguardo Indígena de
Males** (municipio de Córdoba, Nariño – código DANE **52215**). Muestra todas
las variables del censo mediante un **mapa temático (Leaflet)**, un **tablero
estadístico (Chart.js)** con pirámide poblacional y distribuciones, e
**indicadores demográficos** con **filtros territoriales y demográficos cruzados**.

Listo para publicarse en **GitHub Pages** (sitio estático) y con una capa
**opcional PostgreSQL/PostGIS** como base de datos central y motor de ETL.

---

## Arquitectura

```
┌─────────────────────┐   ETL (Python)   ┌──────────────────────────┐
│  CENSO ...2026.xlsx  │ ───────────────► │  web/data/*.json|geojson │
│  (fuente original)   │  procesar_censo  │  (insumos del visor)     │
└─────────────────────┘        │          └───────────┬──────────────┘
                               │                      │  fetch()
                               ▼                      ▼
                    ┌────────────────┐     ┌─────────────────────────┐
                    │ sql/datos/*.csv│     │  web/ (HTML+CSS+JS)      │
                    └───────┬────────┘     │  Leaflet + Chart.js      │
                            │ \copy        │  → GitHub Pages          │
                            ▼              └─────────────────────────┘
                    ┌────────────────┐
                    │ PostgreSQL/PostGIS │  (opcional: consultas
                    │ esquema + vistas   │   espaciales, regenerar GeoJSON)
                    └────────────────┘
```

> **GitHub Pages sirve archivos estáticos**, por lo que no ejecuta bases de
> datos. El visor funciona con los JSON/GeoJSON pre-generados por el ETL.
> PostgreSQL/PostGIS es la **fuente de verdad** desde la que se pueden
> regenerar esos archivos y hacer análisis espacial avanzado.

---

## Estructura del repositorio

```
.
├── web/                      # Sitio estático (esto se publica en Pages)
│   ├── index.html
│   ├── assets/
│   │   ├── estilos.css
│   │   ├── app.js            # filtros cruzados, mapa y gráficos
│   │   └── vendor/           # Leaflet y Chart.js locales (sin CDN)
│   └── data/
│       ├── metadata.json     # diccionarios, indicadores globales, pirámide
│       ├── sectores.geojson  # un punto por sector + indicadores agregados
│       └── microdatos.json   # microdato ANONIMIZADO para filtrado cruzado
├── etl/
│   └── procesar_censo.py     # limpia el .xlsx y genera los insumos + CSV
├── sql/
│   ├── esquema.sql           # PostGIS: tablas, códigos, vistas de indicadores
│   ├── carga.sql             # \copy de los CSV
│   └── datos/                # personas.csv, sectores.csv (generados por ETL)
└── .github/workflows/deploy.yml
```

---

## Cómo regenerar los datos

```bash
pip install openpyxl
python etl/procesar_censo.py \
  --excel "COPIA__CENSO_RESGUARDO_INDIGENA_DE_MALES_CORDOBA_2026.xlsx"
```

Esto reescribe `web/data/*.json|geojson` y `sql/datos/*.csv`.

El ETL **limpia datos sucios**: códigos guardados como texto y número, errores
tipográficos (`1O`→`10`, `KENEDY`→`Kennedy`), valores fuera de rango, fechas
en el campo de dirección, y **normaliza 144 variantes de dirección en 65
sectores/veredas canónicos**.

---

## Publicar en GitHub Pages

1. Sube el repositorio a GitHub.
2. En **Settings → Pages → Build and deployment**, elige **GitHub Actions**.
3. Haz `push` a `main`. El flujo `.github/workflows/deploy.yml` publica la
   carpeta `web/`. La URL aparece en la pestaña *Actions* / *Environments*.

Alternativa sin Actions: en *Settings → Pages* selecciona la rama `main` y la
carpeta `/web` (requiere mover `web/` a la raíz o usar la opción `/docs`).

### Probar localmente

```bash
cd web
python3 -m http.server 8000
# abrir http://localhost:8000
```
(Debe servirse por HTTP; abrir `index.html` con doble clic bloquea `fetch`.)

---

## Capa PostgreSQL/PostGIS (opcional)

```bash
createdb censo_males
psql -d censo_males -f sql/esquema.sql
psql -d censo_males -f sql/carga.sql      # ejecutar desde la raíz del proyecto
```

Incluye tablas de códigos, tabla `persona`, tabla `sector` con geometría de
punto y vistas `v_resumen`, `v_indicadores_sector` y `v_piramide`. El esquema
documenta cómo añadir **polígonos oficiales** del resguardo y cómo **exportar
el GeoJSON** directamente desde PostGIS.

---

## Notas importantes

- **Coordenadas aproximadas.** No se contó con la cartografía oficial del
  resguardo, así que los sectores se ubican como centroides generados de forma
  determinista alrededor de Córdoba (Nariño). **Reemplazar** por los polígonos
  oficiales (IGAC / Agencia Nacional de Tierras) editando
  `web/data/sectores.geojson` o cargándolos en PostGIS.
- **Privacidad.** El microdato publicado está **anonimizado**: no contiene
  nombres, números de documento ni el nombre del censista. Aun así, en
  comunidades pequeñas existe riesgo de reidentificación; se recomienda contar
  con la autorización de las autoridades del resguardo antes de publicar el
  sitio y, si se requiere mayor protección, sustituir `microdatos.json` por
  tablas únicamente agregadas.
- **Estado civil.** La guía de códigos del archivo no incluía la leyenda de
  *estado civil*; el mapeo se **infirió** por cruce con la edad y está
  centralizado para corregirlo fácilmente (`etl/procesar_censo.py` y
  `sql/esquema.sql`).

---

## Tecnologías

Leaflet 1.9 · Chart.js 4.4 · Python (openpyxl) · PostgreSQL + PostGIS ·
GitHub Pages / GitHub Actions.
