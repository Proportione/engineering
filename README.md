# Proportione Engineering Blog

Código fuente de [engineering.proportione.com](https://engineering.proportione.com).

## Qué es

Post-mortems, decisiones de arquitectura y material reutilizable de los proyectos de [Proportione](https://proportione.com).

## Stack

- [Jekyll](https://jekyllrb.com/) con tema [minima](https://github.com/jekyll/minima)
- Hospedado en GitHub Pages
- Dominio: `engineering.proportione.com` (CNAME)

## Publicar un post

1. Crear `_posts/YYYY-MM-DD-slug.md` con frontmatter:

```yaml
---
layout: post
title: "Título descriptivo"
date: YYYY-MM-DD HH:MM:SS +0000
categories: [post-mortem|architecture|benchmark|pattern]
tags: [tag1, tag2]
excerpt: "Una frase que explique el post y atraiga al click."
---
```

2. Cuerpo en markdown. Estilo:
   - Párrafos máximo 4 líneas.
   - Datos concretos, no generalidades.
   - Código real con rutas absolutas cuando aplique.
   - Cerrar con "Lecciones" o "Acciones pendientes" si aporta.

3. Commit a `main` → deploy automático vía GitHub Pages.

## Desarrollo local

```bash
bundle install
bundle exec jekyll serve
# → http://localhost:4000
```

## Licencia

Contenido bajo [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Código del sitio bajo [MIT](LICENSE).
