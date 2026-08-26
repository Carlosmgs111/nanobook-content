---
title: "Plan de refactorización: estructura y organización de módulos"
description: Roadmap de refactorización incremental para reorganizar el código fuente de Nanobook para una arquitectura modular y escalable.
date: 2026-08-23
author: Nanobook Team
tags:
  - arquitectura
  - build
  - incremental
  - refactoring
  - roadmap
  - decision
draft: false
index: false
---

# Plan de refactorización: estructura y organización de módulos

## Objetivo

Reorganizar `src/` para que cada pieza de código resida junto a su dominio y responsabilidad real, eliminando la confusión actual entre `core`, `adapters` y componentes de UI/cliente. No se busca una arquitectura hexagonal maximalista, sino una organización coherente y escalable.

## Principios rectores

1. **Dominio primero**: organizar por capacidad de negocio (`document`, `navigation`, `rendering`, `edition`).
2. **Separar modelo, lógica y efectos secundarios**:
   - `model/` → tipos y contratos puros.
   - `parse/`, `change/`, `graph/`, `page/` → lógica de aplicación sin dependencias de Astro ni de I/O.
   - `adapters/` → implementaciones concretas (Astro, filesystem, red, base de datos).
   - `client/` → código que solo corre en el navegador (`sessionStorage`, `BroadcastChannel`, `Worker`, CodeMirror).
   - `ui/` → componentes Astro.
3. **Evitar `core` como cajón de sastre**: `core` debe desaparecer en favor de nombres que describan qué hace el módulo.
4. **No romper funcionalidad**: cada fase termina con `pnpm test` y `pnpm build`.

---

## Diagnóstico del estado actual

### 1. `core` usado como cajón de sastre

`src/document/` mezcla:

- Tipos puros: `types.ts`
- Parsing: `frontmatter.ts`, `document.ts`
- Estructura de rutas: `path.ts`
- Referencias: `reference/*`
- Cambios incrementales: `snapshot.ts`, `change-service.ts`
- Caché de Astro: `astro-cache.ts` (depende de `astro:content`, es adapter)
- Utilidad de UI: `scroll-spy.ts` (no tiene nada que ver con documentos)
- Proxy: `proxy.ts`

### 2. Rendering mezcla renderizado con edición

- `src/edition/client/render-service.ts` gestiona `sessionStorage` y `BroadcastChannel` solo para el flujo de edición; no es un servicio de renderizado genérico.
- `src/rendering/client/workers/` son workers de Markdown usados exclusivamente por la edición en cliente.
- `rendering/core/` mezcla el modelo (`types.ts`) y la lógica de página (`page-renderer.ts`, `page-cache.ts`).

### 3. Referencias con side effects escondidas en `core`

- `document/reference/internal-resolver.ts` depende de `astro:content`.
- `document/reference/local-file-resolver.ts` depende de `node:fs`.
- Ambos son adapters, no lógica de dominio pura.

### 4. Duplicación en navegación

- `navigation/graph/builder.ts` y `navigation/service/service.ts` tienen funciones casi idénticas (`getBreadcrumbs`, `getSidebarEntries`, `getImmediateChildren`).
- El builder construye un `NavigationTree` que luego se mapea a `DocumentGraph`; hay dos abstraciones superpuestas.

### 5. Snapshot persistence en scripts

- `scripts/lib/content-workflow.ts` contiene `loadSnapshot()` y `saveSnapshot()`.
- Estas funciones pertenecen al dominio de detección de cambios, no a scripts.

---

## Estructura objetivo

```text
src/
  content.config.ts
  content/                          # Markdown de contenido

  document/                         # Dominio: documentos
    model/
      types.ts                      # Document, DocumentMetadata, ContentRepository, etc.
    parse/
      frontmatter.ts
      document.ts                   # parseDocument, extractHeadings, toDocument
      path.ts                       # idToFilePath, getParentId
      proxy.ts                      # resolveProxy
    change/
      snapshot.ts                   # hashDocument, computeDocumentChanges, load/save snapshot
      change-service.ts             # ContentChangeService
    reference/
      types.ts                      # ReferenceResolverPlugin, ReferenceResolutionContext
      path-resolver.ts              # resolveDocumentReference
      resolver.ts                   # CompositeReferenceResolver
    adapters/
      repository/                   # Implementaciones de ContentRepository
        astro-collection-repository.ts
        file-system-repository.ts
        memory-repository.ts
        database-repository.ts
      cache/
        astro-cache.ts              # getAstroEntries
      loader/
        github-loader/              # loader de Astro + resolutor de referencias
      reference/                    # Resolutores de referencias con side effects
        internal-reference-resolver.ts
        local-file-reference-resolver.ts
        github-reference-resolver.ts
    api/
      document.ts                   # API route PATCH /api/:slug
    ui/
      DocumentPage.astro
      components/
        IndexList/
        TableOfContents/

  navigation/                       # Dominio: navegación
    model/
      types.ts                      # NavigationNode, Crumb, DocumentGraph, etc.
    graph/
      graph.ts                      # DocumentGraphImpl, buildDocumentGraph
      invalidation.ts               # computeInvalidatedIds, shouldRebuildNavigationTree
      builder.ts                    # buildNavigationTree (o tree.ts)
    service/
      service.ts                    # NavigationServiceImpl, createNavigationService

  rendering/                        # Dominio: renderizado
    model/
      types.ts                      # RenderedDocument, DocumentRenderer
    page/
      page-renderer.ts              # PageRenderer
      page-cache.ts                 # RenderedPageCache
    adapters/
      markdown/
        unified-markdown.ts
        markdown-it.ts
        astro-markdown.ts
      cache/
        file-system-page-cache.ts
    client/
      workers/
        index.ts
        work.ts

  edition/                          # Dominio: edición de documentos
    client/
      stage-document.ts             # sessionStorage
      save-document.ts              # fetch a API
      parse-staged-document.ts      # buildDocumentFromContent
      render-service.ts             # sessionStorage + BroadcastChannel (movido desde rendering)
      code-mirror-editor.ts         # inicialización de CodeMirror
    ui/
      EditPage.astro
      PreviewPage.astro
      components/
        DocumentEditor/

  shared/                           # Utilidades transversales
    utils/
      debounce.ts
      scroll-spy.ts                 # movido desde document/core

  pages/                            # Rutas Astro
    [...slug]/
      index.astro
      edit.astro
      preview.astro
  api/                              # API routes de Astro (opcional, alternativa a document/api)

  layouts/                          # Layouts Astro
  theme/                            # Componentes de tema

  scripts/                          # Scripts CLI
    content.ts
    lib/
      content-workflow.ts
```

---

## Fases de refactorización

### Fase 0: Preparación

- Asegurar que `git status` esté limpio o que los cambios actuales estén committeados.
- Ejecutar `pnpm test` y `pnpm build` como baseline.
- Documentar la estructura actual.

### Fase 1: Consolidar snapshot persistence (quick win)

**Qué mover**:

- `scripts/lib/content-workflow.ts` → `src/document/change/snapshot.ts`:
  - `loadSnapshot()`
  - `saveSnapshot()`
  - `SNAPSHOT_PATH` (con ruta configurable y default `.nanobook/content-snapshot.json`)

**Por qué**: la persistencia del snapshot es parte del dominio de detección de cambios, no del CLI.

**Pasos**:

1. Crear `src/document/change/snapshot.ts` con `loadSnapshot`, `saveSnapshot`, `hashDocument`, `computeDocumentChanges`.
2. `scripts/lib/content-workflow.ts` importa desde `src/document/change/snapshot.ts`.
3. Actualizar `scripts/content.ts` si es necesario.
4. Tests y build.

### Fase 2: Renombrar y limpiar archivos mal ubicados

**Qué mover**:

- `src/shared/utils/scroll-spy.ts` → `src/shared/utils/scroll-spy.ts`
- `src/document/core/astro-cache.ts` → `src/document/adapters/cache/astro-cache.ts`
- `src/edition/client/render-service.ts` → `src/edition/client/render-service.ts`
- `src/rendering/client/workers/` → `src/rendering/client/workers/`
- `src/document/parse/document.ts` → `src/document/parse/document.ts`

**Por qué**:

- `scroll-spy` es una utilidad genérica de cliente.
- `astro-cache` depende de Astro, por tanto es adapter.
- `render-service` es específico del flujo de edición.
- `workers` son infraestructura de cliente.
- `document.ts` contiene funciones de parseo, no la entidad Document.

**Pasos**:

1. Mover cada archivo usando `git mv` para preservar historial.
2. Actualizar todas las importaciones.
3. Tests y build.

### Fase 3: Reorganizar por dominio

**Document**:

- Crear carpetas:
  - `src/document/model/`
  - `src/document/parse/`
  - `src/document/change/`
  - `src/document/reference/`
- Mover:
  - `src/document/model/types.ts` → `src/document/model/types.ts`
  - `src/document/parse/frontmatter.ts` → `src/document/parse/frontmatter.ts`
  - `src/document/parse/document.ts` (desde Fase 2)
  - `src/document/parse/path.ts` → `src/document/parse/path.ts`
  - `src/document/parse/proxy.ts` → `src/document/parse/proxy.ts`
  - `src/document/change/snapshot.ts` (desde Fase 1)
  - `src/document/change/change-service.ts` → `src/document/change/change-service.ts`
  - `src/document/reference/types.ts` → `src/document/reference/types.ts`
  - `src/document/reference/path-resolver.ts` → `src/document/reference/path-resolver.ts`
  - `src/document/reference/resolver.ts` → `src/document/reference/resolver.ts`
- Reorganizar `src/document/adapters/` en subcarpetas semánticas:
  - `src/document/adapters/repository/` para `astro-collection-repository.ts`, `file-system-repository.ts`, `memory-repository.ts`, `database-repository.ts`
  - `src/document/adapters/cache/` para `astro-cache.ts`
  - `src/document/adapters/loader/` para `github-loader/`
  - `src/document/adapters/reference/` para los resolutores de referencias
- Actualizar todos los imports.

**Navigation**:

- Crear carpetas:
  - `src/navigation/model/`
  - `src/navigation/graph/`
  - `src/navigation/service/`
- Mover:
  - `src/navigation/model/types.ts` → `src/navigation/model/types.ts`
  - `src/navigation/graph/graph.ts` → `src/navigation/graph/graph.ts`
  - `src/navigation/graph/invalidation.ts` → `src/navigation/graph/invalidation.ts`
  - `src/navigation/graph/builder.ts` → `src/navigation/graph/builder.ts`
  - `src/navigation/service/service.ts` → `src/navigation/service/service.ts`
- Actualizar imports.

**Rendering**:

- Crear carpetas:
  - `src/rendering/model/`
  - `src/rendering/page/`
  - `src/rendering/adapters/markdown/`
  - `src/rendering/adapters/cache/`
  - `src/rendering/client/workers/`
- Mover:
  - `src/rendering/model/types.ts` → `src/rendering/model/types.ts`
  - `src/rendering/page/page-renderer.ts` → `src/rendering/page/page-renderer.ts`
  - `src/rendering/page/page-cache.ts` → `src/rendering/page/page-cache.ts`
  - `src/rendering/adapters/markdown/unified-markdown.ts` → `src/rendering/adapters/markdown/unified-markdown.ts`
  - `src/rendering/adapters/markdown/markdown-it.ts` → `src/rendering/adapters/markdown/markdown-it.ts`
  - `src/rendering/adapters/markdown/astro-markdown.ts` → `src/rendering/adapters/markdown/astro-markdown.ts`
  - `src/rendering/adapters/cache/file-system-page-cache.ts` → `src/rendering/adapters/cache/file-system-page-cache.ts`
  - `src/rendering/client/workers/` (desde Fase 2)
- Actualizar imports.

**Edition**:

- Crear carpetas:
  - `src/edition/client/`
  - `src/edition/ui/`
- Mover:
  - `src/edition/client/stage-document.ts` → `src/edition/client/stage-document.ts`
  - `src/edition/client/save-document.ts` → `src/edition/client/save-document.ts`
  - `src/edition/client/parse-staged-document.ts` → `src/edition/client/parse-staged-document.ts`
  - `src/edition/client/render-service.ts` (desde Fase 2)
  - `src/edition/client/code-mirror-editor.ts` → `src/edition/client/code-mirror-editor.ts`
  - `src/edition/EditPage.astro` → `src/edition/ui/EditPage.astro`
  - `src/edition/PreviewPage.astro` → `src/edition/ui/PreviewPage.astro`
  - `src/edition/ui/components/` → `src/edition/ui/components/`
- Actualizar imports.

### Fase 4: Consolidar navigation builder/service

**Opción A (recomendada): mantener dos niveles pero clarificar**

- `navigation/graph/` → estructura de árbol/grafo y dependencias.
- `navigation/service/` → operaciones de navegación para UI (breadcrumbs, sidebar).
- Eliminar duplicación: `builder.ts` expone funciones puras sobre `NavigationTree`; `service.ts` las consume.
- Documentar que `builder.ts` es de bajo nivel y `service.ts` es el facade para componentes.

**Opción B: unificar**

- Fusionar `builder.ts` y `service.ts` en `navigation/service/service.ts`.
- `buildNavigationTree` pasa a ser privado o utilidad interna.

**Decisión**: elegir A si se quiere mantener flexibilidad para futuros consumers; elegir B si se prefiere simplicidad.

### Fase 5: Reorganizar reference resolvers

- Mover resolutores con side effects a `src/document/adapters/reference/`:
  - `InternalReferenceResolver`
  - `LocalFileReferenceResolver`
  - `GitHubReferenceResolver` (desde `document/adapters/github-loader/`)
- Dejar en `src/document/reference/` solo:
  - `types.ts`
  - `path-resolver.ts`
  - `resolver.ts` (`CompositeReferenceResolver`)
- Asegurar que los repositorios (`src/document/adapters/repository/`) estén agrupados por separado de los resolutores, la caché y el loader.
- Actualizar `AstroCollectionRepository` para crear el `CompositeReferenceResolver` con los adapters.

### Fase 6: Revisar entry points públicos

- Crear índices (`index.ts`) por dominio para simplificar imports:
  - `src/document/index.ts`
  - `src/navigation/index.ts`
  - `src/rendering/index.ts`
  - `src/edition/index.ts`
- Evitar imports profundos como `../../document/change/snapshot`; preferir `~/document/change` o `~/document`.

### Fase 7: Documentación

- Actualizar `AGENTS.md` con la nueva convención de carpetas.
- Actualizar `src/content/nanobook-project/arquitectura/recompilado-incremental-estado-actual.md` para reflejar las rutas nuevas.
- Añadir diagrama de dependencias entre módulos.

---

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|-----------|
| Imports rotos en muchos archivos | Usar `git mv`, mover por fases pequeñas, ejecutar `pnpm test` y `pnpm build` tras cada fase. |
| Pérdida de historial de git | Usar `git mv` en lugar de borrar/crear archivos. |
| Tests que dependen de rutas de importación | Actualizar imports en tests como parte de cada fase. |
| Astro no resuelve workers si cambia la ruta | Verificar `new URL("./work.ts", import.meta.url)` y ajustar. |
| Tiempo invertido sin funcionalidad nueva | Hacer fases pequeñas y detenerse en cualquier momento sin dejar el proyecto roto. |

---

## Criterios de éxito

- `pnpm test` pasa.
- `pnpm build` pasa.
- `pnpm content:status` y `pnpm content:render` funcionan.
- No queda ninguna carpeta `core/` que sea un cajón de sastre.
- Cada archivo puede ubicarse razonablemente en su módulo sin explicaciones forzadas.

---

## Próximo paso

Revisar este plan, ajustar la estructura objetivo si es necesario, y decidir si se comienza por la **Fase 1** (snapshot persistence) o si se prefiere hacer toda la reestructuración de una sola vez.
