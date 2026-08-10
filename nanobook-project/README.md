---
title: "README"
description: "Documentación de Nanobook"
date: 2026-07-30
author: "Nanobook Team"
tags: ["documentation"]
draft: false
index: false
---

# Nanobook

Nanobook es un sitio estático construido con Astro para publicar contenido técnico organizado en libros, capítulos y artículos. Usa una estructura de carpetas y archivos Markdown con frontmatter para generar automáticamente índices de navegación.

## Características

- Índices jerárquicos generados automáticamente a partir de la estructura de carpetas.
- Contenido en Markdown con frontmatter.
- Renderizado de páginas estáticas con Astro.
- Diseño con Tailwind CSS.
- Soporte para temas claro y oscuro.

## Stack

![Astro](https://img.shields.io/badge/Astro-5-BC52EE?logo=astro&logoColor=white&style=plastic)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-4-06B6D4?logo=tailwindcss&logoColor=white&style=plastic)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?logo=typescript&logoColor=white&style=plastic)
![Node.js](https://img.shields.io/badge/Node.js-22+-339933?logo=nodedotjs&logoColor=white&style=plastic)
![Markdown](https://img.shields.io/badge/Markdown-000000?logo=markdown&logoColor=white&style=plastic)

## Requisitos

- Node.js >= 22.12.0

## Instalación

```bash
npm install
npm run dev
```

## Scripts

- `npm run dev` — Inicia el servidor de desarrollo.
- `npm run build` — Genera el sitio estático en `dist/`.
- `npm run preview` — Previsualiza el sitio generado.

## Estructura de contenido

El contenido se ubica en `src/content/` y sigue una jerarquía basada en carpetas:

```text
src/content/
├── index.md                              → /
├── blog/
│   ├── index.md                          → /blog/
│   └── overthinking.md                   → /blog/overthinking/
└── books/
    ├── index.md                          → /books/
    └── responsive-development/
        ├── index.md                      → /books/responsive-development/
        ├── overview.md                   → /books/responsive-development/overview/
        └── chapter-1.md                  → /books/responsive-development/chapter-1/
```

## Convenciones de frontmatter

```md
---
title: "Título del documento"
description: "Breve descripción del contenido"
date: 2026-07-30
author: "Autor"
tags: ["tag1", "tag2"]
draft: false
index: true
---
```

Campos:

- `title` (obligatorio): Título del documento o carpeta.
- `description` (obligatorio): Descripción corta.
- `date` (obligatorio): Fecha de publicación.
- `author` (obligatorio): Autor del contenido.
- `tags` (obligatorio): Lista de etiquetas.
- `draft` (opcional): Si es `true`, el documento no se genera.
- `index` (obligatorio para `index.md`): Marca el archivo como representación de una carpeta.

## Reglas de indexado

- Cada carpeta que quiera aparecer en la navegación debe contener un archivo `index.md` con `index: true`.
- Las carpetas sin `index.md` no son visibles en los índices superiores.
- Los archivos que no son `index.md` se renderizan como documentos finales.
- Los documentos se listan en el índice de su carpeta padre inmediata.

## Arquitectura del proyecto

- `src/content/` — Contenido en Markdown.
- `src/pages/[...slug].astro` — Ruta dinámica universal que decide si renderizar un índice o un documento.
- `src/components/IndexList.astro` — Componente de listado de índices.
- `src/layouts/Layout.astro` — Layout base del sitio.
- `src/content.config.ts` — Configuración de la colección de contenido.

## Extensibilidad

El sistema usa la API de loaders de Astro 5. Actualmente el contenido se carga desde el directorio local con `glob`, pero el loader puede reemplazarse por un loader custom que obtenga archivos desde GitHub, un CMS headless, S3, Notion u otra fuente externa.

## Licencia

[Agregar licencia si aplica]
