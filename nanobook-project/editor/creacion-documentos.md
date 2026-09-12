---
title: "Creación de documentos"
description: "Decisión e implementación del flujo para crear nuevos documentos e índices de directorio en Nanobook."
date: 2026-08-28
author: "Nanobook"
tags: ["nanobook", "editor", "documentos", "decision"]
draft: false
index: false
---

## Contexto

Hasta ahora el editor integrado solo permitía modificar documentos existentes. Para completar el CRUD mínimo de contenido, se necesita una forma de crear documentos nuevos tanto desde la API como desde la UI.

## Decisiones

### 1. `create()` y `update()` en lugar de `save()`

El método `save()` de `ContentRepository` era ambiguo: no quedaba claro si debía crear o actualizar. Se separó en dos métodos con semántica explícita:

- `create(document)`: el documento no debe existir; si existe, lanza `DocumentAlreadyExistsError`.
- `update(document)`: el documento debe existir; si no, lanza `DocumentNotFoundError`.

Esto se aplica a `FileSystemRepository`, `GitHubRepository` y `MemoryRepository`.

### 2. Normalización de ids de índice

Los índices de directorio se representan en el modelo con el id de la carpeta, no con el segmento final `index`:

- `src/content/blog/index.md` → id `blog`
- `src/content/index.md` → id `index`

Por tanto, cuando el usuario envía `blog/index`, se normaliza a `blog`. La función `normalizeDocumentId()` centraliza esta regla y `buildNewDocument()` la aplica automáticamente.

### 3. Inferencia de `index` desde el id

Si el id original termina en `/index` (o es `index`), `buildNewDocument()` fuerza `metadata.index = true`. Si el usuario intenta crear `blog/post` con `index: true`, se rechaza porque es inconsistente.

### 4. Validación de padres

Al crear un documento, se valida que:

- El padre implícito exista (salvo para el índice raíz).
- El padre sea un índice (`metadata.index === true`).

Esto evita crear documentos huérfanos o colgar contenido de documentos que no son carpetas.

### 5. Persistencia inmediata + redirección al editor

Se siguió el principio de persistir el documento vacío inmediatamente y redirigir a la vista de edición. Esto evita estados intermedios complejos y reutiliza el flujo de previsualización existente.

### 6. Endpoints

- `POST /api/documents`: crea un documento nuevo.
- `PATCH /api/[...slug]`: actualiza un documento existente (devuelve 404 si no existe).

Ambos endpoints invalidan el cache de páginas renderizadas del documento afectado y, en creación, también del padre.

### 7. UI

- Botón "Nuevo" en el header de todas las páginas.
- Página `/{slug}/new` con un formulario que muestra la ruta actual como prefijo fijo y un input para el nombre del nuevo elemento.
- Dos opciones de tipo semánticas:
  - **Documento**: crea un archivo Markdown directamente bajo la ruta actual.
  - **Directorio**: crea un directorio con un `index.md` que actúa como índice.
- Al crear, se redirige a `/{id}/edit`.

## Archivos clave

- `src/document/model/types.ts`: contrato `ContentRepository` con `create` y `update`.
- `src/document/model/errors.ts`: errores de dominio.
- `src/document/parse/path.ts`: `normalizeDocumentId`, `isIndexId`, `validateDocumentId`, `idToFilePath`.
- `src/document/adapters/repository/document-builder.ts`: `buildNewDocument`.
- `src/document/adapters/repository/file-system-repository.ts`: implementación filesystem.
- `src/document/adapters/repository/github-repository.ts`: implementación GitHub.
- `src/document/api/document.ts`: handlers de POST y PATCH.
- `src/edition/ui/components/NewDocumentForm/NewDocumentForm.astro`: formulario de creación.
- `src/pages/[...slug]/new.astro`: página de creación.

## Próximos pasos

- Añadir capa de autenticación para proteger los endpoints y el botón "Nuevo".
- Considerar un modal o drawer para la creación en lugar de una página aparte.
- Añadir soporte para arrastrar y soltar archivos Markdown o importar desde clipboard.
