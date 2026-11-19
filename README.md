# hugo-sqlite-proto

Prototipo didáctico: **Hugo** genera HTML estático desde Markdown y **SQLite** trabaja
**solo en tiempo de build** (indexar y agregar). En producción: ficheros estáticos, nada más.

Continuación de la práctica
[P1.2 — Tu mini sitio web con Hugo](https://www.librebits.info/2asir/iaw/mini-sitio-web-con-hugo/).

**Demo en vivo:** [fp-bits.github.io/hugo-sqlite-proto](https://fp-bits.github.io/hugo-sqlite-proto/)

## Hugo build 
El generador de sitios web estático [Hugo](https://gohugo.io/documentation) construye 'las páginas web html (+ css, +javascript)' cuando ejecutas 'hugo serve' desde la terminal.
Ese comando detecta cambios en `content/` y `layouts/` y reconstruye el sitio automáticamente, sin que tengas que parar y volver a arrancar nada.
 
## Idea ([Unix](https://es.wikipedia.org/wiki/Unix): cada pieza hace una cosa, se unen con ficheros, a modo 'tuberías')

```
content/*.md --> tools/index.py --> build/site.db --+--> static/search.json --> navegador (search.js)
 (Markdown)       (parsea + INSERT)   (SQLite, local) +--> data/tags.json -----> Hugo (layouts/index.html)

                                     hugo --> public/   <- lo único que se despliega
```

`build/`, `static/search.json` y `data/tags.json` están en `.gitignore`: se **regeneran** en cada build.

## Requisitos

- [Hugo](https://gohugo.io/installation/) (versión estándar, no hace falta *extended*)
- Python 3.8+ (solo biblioteca estándar: [`sqlite3`](https://docs.python.org/3/library/sqlite3.html), `json`, `re`)
- `make`

## Uso

```bash
make            # == make help
make serve    # índice + servidor local en http://localhost:1313/
make build    # índice + hugo --gc --minify  → public/
make check    # valida que los JSON generados son JSON válido
make clean
make check-invites   # [profesor] invitaciones de colaborador pendientes (gh CLI)
```

## Estructura

```
content/            Markdown con front matter YAML (el borrador no se indexa ni se publica)
layouts/            Plantillas Hugo: baseof, single, list, index + parciales header/footer
static/search.js    Búsqueda en el navegador: vainilla ES6+, estilo funcional
static/style.css    CSS3 con variables y modo oscuro
tools/index.py      Markdown → SQLite → JSON (funciones puras + efectos al final)
Makefile            index · build · serve · check · clean
```

## Dónde mirar

- **SQL como generador de JSON**: `Q_SEARCH` y `Q_TAGS` en `tools/index.py`
  ([JSON1](https://www.sqlite.org/json1.html), `GROUP BY`, `COUNT`).
- **Datos de build en plantillas**: `hugo.Data.tags` en `layouts/index.html`
  ([Hugo data](https://gohugo.io/content-management/data-sources/)).
- **Búsqueda sin servidor**: `static/search.js` ([`fetch`](https://developer.mozilla.org/es/docs/Web/API/Fetch_API),
  [`filter`](https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/Array/filter),
  [`normalize`](https://developer.mozilla.org/es/docs/Web/JavaScript/Reference/Global_Objects/String/normalize)).

## Ejercicios

1. Añade un post con `tags: [sql]` y comprueba cómo cambia el conteo de etiquetas en la home (`make serve`).
2. `hugo server -D` muestra el borrador, pero la búsqueda no lo encuentra. ¿Por qué? Añade un
   flag `--drafts` a `tools/index.py` para que coincidan.
3. Añade una tabla `words(url, n)` con el número de palabras de cada post y muestra el total en el pie.
4. Cambia la búsqueda para ordenar los resultados por número de coincidencias (sigue siendo una función pura).
5. Despliega `public/` con `rsync` a tu VPS o con GitHub Pages. ¿Qué proceso corre en el servidor? (Ninguno de tuyos.)

## Ejercicios resueltos P1.1 (Iker Estebaranz Corcobado)

1. **Post nuevo con etiqueta `sql`**: añadido `content/posts/consultas-sql.md` con `tags: [sql, sqlite, hugo]`. Al lanzar `make index` (o `make serve`), `tools/index.py` inserta las etiquetas en la tabla `tags` de SQLite y regenera `data/tags.json` con la consulta `Q_TAGS` (`GROUP BY tag`), por lo que el conteo en la portada se actualiza solo.
2. **Borradores en la búsqueda (`--drafts`)**: `hugo server -D` enseña los posts con `draft: true`, pero el buscador no los encontraba porque `read_all()` en `tools/index.py` filtraba siempre `if not page["draft"]` antes de meterlos en SQLite. He añadido el parámetro `--drafts` a `tools/index.py` y la regla `make serve-drafts` en el `Makefile` para lanzar las dos cosas a la vez.
3. **Tabla `words(url, n)` y conteo de palabras**: añadida la tabla `words` al esquema de SQLite en `tools/index.py`. Al leer cada fichero Markdown se calcula `len(text.split())`, se guarda en `words` y la consulta `Q_WORDS` exporta `data/words.json` con la suma total (`SUM(n)`) y el desglose por URL. Hugo lee `hugo.Data.words` para mostrar el total de palabras en `layouts/partials/footer.html` y, como aportación extra, las palabras de cada post junto a la fecha en `layouts/_default/single.html`.
4. **Ordenar la búsqueda por coincidencias**: en `static/search.js` he añadido la función pura `countHits`, que cuenta cuántas veces aparecen los términos buscados en cada post usando `reduce`, y ordena una copia de los resultados (`[...posts.filter(...)].sort(...)`) de mayor a menor número de apariciones sin mutar el array original.

## Migración a Chyrp Lite + SQLite — P1.2 (rama `chyrp-lite`)

En la rama `chyrp-lite` se añade el proceso ETL para migrar las entradas y páginas Markdown del prototipo a la base de datos SQLite de **Chyrp Lite**:

- `tools/migrar.py`: extrae los ficheros Markdown (`content/posts/*.md` y `content/*.md`), los transforma con funciones puras (`slugify`, `fecha_chyrp`, `etiquetas_json`, `transformar`, `transformar_pagina`) al esquema relacional + EAV (`posts`, `post_attributes` y `pages`) y los carga en una única transacción de forma idempotente.
- `tools/test_migrar.py`: pruebas unitarias (`unittest`) para validar las funciones puras sin tocar disco ni base de datos (`make test`).
- `Makefile`: añadidos los objetivos `make test` y `make migrar DESTINO=/ruta/copia.sqlite [SIMULACRO=1]`.

### Ejercicios resueltos en P1.2

- **Ejercicio 5 (Límite con etiquetas numéricas como `2026`)**: el módulo *Tagginator* de Chyrp Lite (`modules/tags/tags.php`) antepone el carácter Unicode `U+FE0E` (`\ufe0e`) a las etiquetas numéricas para evitar que PHP convierta la clave del array asociativo en un índice entero, y calcula el hash `md5` como *slug* cuando el *slug* solo contiene dígitos o guiones. Se ha actualizado `etiquetas_json` en `tools/migrar.py` y añadido `test_etiqueta_numerica` en `tools/test_migrar.py`.
- **Ejercicio 2 (Migración de páginas estáticas a `pages`)**: añadidas `transformar_pagina` (función pura), `extraer_paginas` y `cargar_paginas` para migrar páginas como `content/about.md` a la tabla `pages` de Chyrp Lite, junto con la suite `TestTransformarPagina`.
- **Ejercicio 4 (Soporte de autor por `--autor` y `author` en *front matter*)**: `leer_entrada` extrae el campo `author` si aparece en el *front matter* YAML y `transformar` resuelve su `user_id` a partir del diccionario de usuarios de Chyrp Lite (o usa el `--autor` indicado por línea de comandos), manteniendo `transformar` pura y probada con `test_autor_en_front_matter`.

## Límites a propósito (KISS)

`tools/index.py` entiende front matter YAML simple (`clave: valor` y `tags: [a, b]`); no TOML ni listas
multilínea. Para algo más serio, usa una biblioteca de YAML: aquí lo importante es el *flujo*.

## Entregable

Haz un **fork** (o clona y crea tu propio repo) de este proyecto, resuelve los `Ejercicios` que quieras
de los de arriba (no hace falta hacerlos todos) y haz **varios commits** a medida que avanzas.

La práctica se considera entregada en el momento en que **invitas al profesor** (usuario de GitHub
`fenix011`) **como colaborador** de tu repositorio. Sigue estos pasos de la documentación oficial de
GitHub en español:
[Invitar a colaboradores a un repositorio personal](https://docs.github.com/es/repositories/managing-your-repositorys-settings-and-features/repository-access-and-collaboration/inviting-collaborators-to-a-personal-repository).

No hace falta sFTP ni capturas: con la invitación aceptada, el profesor ya puede clonar tu repo y revisarlo.

---

# RUBRICA DE CALIFICACIÓN
## Criterios de contenido (10 puntos)

### Implementación técnica — 7 puntos

| **Criterio** | **Peso** | **Qué se valora** |
|:---|:---:|:---|
| **Entorno funcionando** | 1p | `make serve`, `make build` y `make check` corren sin errores sobre tu fork/copia |
| **Ejercicios resueltos** | 3p | Al menos 2 de los 5 `Ejercicios` propuestos, funcionando de verdad (no hace falta hacer los 5) |
| **Comprensión del flujo** | 2p | Se entiende cómo encajan `tools/index.py` → SQLite → JSON → Hugo en los cambios hechos (visible en el código o explicado en el README) |
| **Calidad del código añadido** | 1p | Estilo coherente con el proyecto (funciones puras donde aplica, nombres claros) |

### Más allá de lo técnico — 3 puntos

| **Criterio** | **Peso** | **Qué se valora** |
|:---|:---:|:---|
| **Aportación propia** | 1p | Algo más allá del mínimo pedido en los ejercicios elegidos |
| **Documentación del cambio** | 1p | El README (o un commit) explica qué se ha hecho y por qué |
| **Entregable completo y verificable** | 1p | Invitación a `fenix011` aceptada, repo clonable, commits reales |

## Convención en vivo vs. online, con matiz de `git log`

Esta práctica se entrega vía repositorio GitHub: 
- **10/10** — demo en vivo (en el aula) **y** `git log` con trabajo real incremental (varios commits, marcas de
  tiempo realmente separadas, mensajes no genéricos. Es decir, decriptivos de los cambios realizados!).
- **Intermedio (hasta 9/10)** — sin demo en vivo, pero con esa misma evidencia en el `git log`.
- **Tope 7/10** — sin demo en vivo y sin evidencia en el `git log` (un commit, mensajes genéricos o
  timestamps agrupados).

## Escala orientativa

- **Excelente (9-10):** varios ejercicios resueltos y funcionando, código claro, demo en vivo y/o
  `git log` con evidencia real.
- **Notable (7-8.9):** 2+ ejercicios resueltos y funcionando, algún detalle menor.
- **Suficiente (5-6.9):** el entorno corre pero los ejercicios resueltos son mínimos o tienen fallos.
- **Insuficiente (<5):** el entorno no llega a funcionar (`make build` falla) o no hay ningún ejercicio
  resuelto.

## Lista de comprobación rápida (para nosotros, el profesorado :-)

```bash
git clone https://github.com/<alumno>/<repo>.git && cd <repo>
make check && make build
git log --format='%h %ad %s' --date=iso | head -20
```

## Plantilla de feedback

```
## Feedback
- Qué ha ido bien: …
- A mejorar: …
- ¿Has pensado en usar `git` desde la terminal para hacer los cambios... a lo Pro? ;-)
```

"Recuerda ... KISS!" :-)
