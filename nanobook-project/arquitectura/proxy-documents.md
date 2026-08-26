---
title: "Documentos proxy"
description: "Cómo funciona el sistema de referencias `ref` para crear documentos que reflejan el contenido de otros documentos internos, locales o de proyectos ajenos."
date: 2026-08-23
author: "Nanobook Team"
tags: ["arquitectura", "proxy", "ref", "documentos"]
draft: false
index: false
---

## Propósito

Un **documento proxy** es un documento en `src/content/` que no aporta contenido propio, sino que **refleja el contenido y metadatos de otro documento**. Mantiene su propio `id`, `slug` y `parentId`, por lo que se genera en su propia URL, pero el cuerpo y la mayoría de metadatos provienen del documento destino.

Casos de uso típicos:

- **Alias**: publicar el mismo contenido en varias URLs sin duplicar el archivo.
- **README del proyecto**: crear una página en el sitio que muestre el `README.md` de la raíz del repo.
- **Documentos de proyectos ajenos**: traer contenido Markdown desde un repositorio de GitHub y publicarlo como parte del sitio.

## Cómo se define

En el frontmatter de cualquier documento de la colección se añade el campo `ref`:

```yaml
---
title: "Generador de Demos (alias)"
description: "Alias del proyecto generador-demos"
date: 2026-08-22
author: "Nanobook"
tags: ["proxy"]
draft: false
index: false
position: 1
ref: ./generador-demos.md
---
```

El documento puede tener cuerpo vacío; se ignorará porque el proxy reemplaza el contenido por el del destino.

## Tipos de referencias

El campo `ref` acepta tanto strings como objetos estructurados. El sistema resuelve la referencia mediante un registro de resolutores.

### 1. Referencia interna

Apunta a otro documento dentro de la misma colección de contenido. Se resuelve de forma relativa al documento origen.

```yaml
# Mismo directorio
ref: ./generador-demos.md

# Subdirectorio
ref: ./subcarpeta/index.md

# Directorio padre
ref: ../otro-documento.md
```

La resolución respeta el campo `index` del frontmatter:

- Si el documento origen tiene `index: true`, la referencia se resuelve desde el propio ID (comportamiento de directorio).
- Si tiene `index: false`, se resuelve desde el directorio padre del ID (comportamiento de archivo).

### 2. Referencia local

Apunta a un archivo Markdown fuera de `src/content/`, relativo a la raíz del proyecto.

```yaml
# Forma corta
ref: /README.md

# Forma estructurada
ref:
  source: local
  path: README.md
```

Las rutas se validan para que no salgan de la raíz del proyecto. El archivo destino no necesita tener frontmatter completo: los metadatos faltantes se toman del documento proxy.

### 3. Referencia a GitHub

Apunta a un archivo Markdown en un repositorio de GitHub. Requiere un token (`GITHUB_TOKEN`) para repositorios privados o para evitar límites de rate.

```yaml
# Forma corta (rama main por defecto)
ref: github:astro/astro/CONTRIBUTING.md

# Forma corta con rama explícita
ref: github:astro/astro/CONTRIBUTING.md@main

# Forma estructurada
ref:
  source: github
  owner: astro
  repo: astro
  path: CONTRIBUTING.md
  branch: main
```

## Metadatos del documento resultante

El proxy mezcla metadatos del documento origen y del documento destino:

- El destino aporta `title`, `description`, `content`, `tags`, `cover`, `author` y `date`.
- El origen conserva `position` e `index`, porque definen dónde aparece el proxy en la navegación.
- El campo `ref` no se hereda: un proxy no puede apuntar a otro proxy (no se permiten cadenas).

De este modo, un `README.md` sin frontmatter puede publicarse usando los metadatos del documento proxy:

```yaml
---
title: "README del proyecto"
description: "Documentación de alto nivel de Nanobook"
date: 2026-08-23
author: "Nanobook Team"
tags: ["documentación"]
draft: false
index: false
ref: /README.md
---
```

## Limitaciones

- **No se permiten cadenas**: un documento proxy no puede referenciar a otro documento proxy.
- **No se renderiza el cuerpo del origen**: el cuerpo del documento proxy se descarta y se reemplaza por el del destino.
- **El destino debe ser Markdown**: el sistema parsea el frontmatter del archivo destino; no está pensado para otros formatos.

## Implementación

El flujo de resolución está en `src/document/reference/`:

```text
AstroCollectionRepository.list()
        │
        ▼
resolveProxy(sourceEntry, resolver)
        │
        ▼
CompositeReferenceResolver
        │
        ├── InternalReferenceResolver  → entriesById
        ├── LocalFileReferenceResolver → filesystem
        └── GitHubReferenceResolver    → GitHub API
```

Cada resolutor decide si puede manejar el valor de `ref` y devuelve una `ContentEntry`. Si ninguno resuelve la referencia, `resolveProxy` devuelve `null` y el documento origen se renderiza como un documento normal.
