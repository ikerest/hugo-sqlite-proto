---
title: "Consultas SQL en tiempo de compilación"
date: 2026-11-12
description: "Cómo usar SQLite para agrupar datos antes de generar el HTML con Hugo."
tags: [sql, sqlite, hugo]
---
En esta práctica probamos a meter **SQLite** en medio del proceso de build de Hugo.
En vez de tener una base de datos corriendo en un servidor, el script `tools/index.py`
lee los ficheros Markdown, mete los datos en tablas SQL y saca los JSON que necesita la web.

```sql
SELECT tag, COUNT(*) AS total
FROM tags
GROUP BY tag
ORDER BY total DESC;
```

Así en producción solo subimos la carpeta `public/` con HTML, CSS, JS y JSON estáticos,
sin ningún proceso de base de datos corriendo por detrás.
