---
title: Arquitectura del modelo de contenido
description: Decisión de mantener filesystem-first pero storage-agnostic,
  separando Content Model de Storage, Navigation y Astro como capa de
  publicación.
date: 2026-08-23T00:00:00.000Z
author: Nanobook Team
tags:
  - arquitectura
  - contenido
  - storage
  - decision
draft: false
index: false
position: 0
---

## Contexto

Nanobook nació como un generador estático que carga Markdown desde `src/content/` mediante el loader `glob` de Astro. Con el tiempo surgió la necesidad de poder actualizar contenido en producción sin generar un nuevo build. Tras evaluar las alternativas, se decidió consolidar el proyecto como una aplicación SSR pura: una única arquitectura que sirve tanto contenido local como remoto en runtime, sin bifurcaciones de build.

## Decisión

**SSR puro con `ContentRepository` como fuente de verdad.**

- Astro siempre usa `output: "server"`.
- El contenido se carga en runtime a través de implementaciones de `ContentRepository`:
  - `FileSystemRepository`: archivos locales en `src/content/`.
  - `GitHubRepository`: archivos remotos en un repositorio de GitHub.
  - Futuros: `DatabaseRepository`, CMS, S3, etc.
- No hay modos de despliegue mutuamente excluyentes; se elige la fuente con `CONTENT_SOURCE`.
- La edición se hace directamente sobre la fuente de verdad; no hay overrides ni manifest.

> **Nota histórica**: versiones anteriores usaban `OUTPUT_MODE` (`static` / `dynamic`) para elegir entre SSG y SSR. Esa bifurcación fue eliminada en favor de SSR puro. Ver [plan de transición a SSR puro](../plan-transicion-ssr-puro).

## Fuente de contenido

Se elige mediante la variable de entorno `CONTENT_SOURCE`:

```text
CONTENT_SOURCE=filesystem   # default
CONTENT_SOURCE=github
```

### Filesystem (default)

- Fuente de verdad: archivos Markdown en `src/content/`.
- Funciona en desarrollo, build y tests.
- El editor integrado puede escribir directamente sobre estos archivos.

### GitHub

- Fuente de verdad: archivos Markdown en un repositorio de GitHub.
- Requiere `GITHUB_OWNER`, `GITHUB_REPO` y opcionalmente `GITHUB_BRANCH`, `GITHUB_TOKEN` y `GITHUB_PATH`.
- Útil cuando el contenido vive en un repo separado y se actualiza sin redeploy.

## Principio rector

> El dominio de Nanobook nunca debe saber si un documento proviene del filesystem local, GitHub, PostgreSQL o una API.

## Arquitectura objetivo

```text
              Nanobook Core
                    │
         ┌──────────┴──────────┐
         │                     │
    Content Model        Navigation Model
         │                     │
         └──────────┬──────────┘
                    │
             Rendering Layer
                    │
                    ▼
               Astro SSR
                    │
                 Server
                    │
         ┌──────────┴──────────┐
         │                     │
    Filesystem            GitHub
    src/content/          (remoto)
```

Por debajo:

```text
           ContentRepository
                  │
         ┌─────────┴─────────┐
         │                   │
     Filesystem           GitHub
         │                   │
       Markdown          Markdown
         │                   │
         └──────────┬──────────┘
                    │
              Document[]
```

## Conceptos del dominio

- **Document**: unidad mínima de contenido. Tiene `id`, `slug`, `parentId`, `position`, `title`, `description`, `content`, `metadata`, `rawFrontmatter` y opcionalmente `proxyTargetId`. Ver `src/document/model/types.ts`.
- **ContentRepository**: interfaz para listar, obtener, buscar hijos y guardar documentos. Responsabilidad única: persistencia/lectura. Ver `src/document/model/types.ts`.
- **NavigationBuilder**: construye árboles de navegación a partir de una lista de documentos. Ver [`src/navigation/graph/builder.ts`](../../navigation-api).
- **DocumentRenderer**: interfaz para convertir el contenido crudo en HTML. Ver `src/rendering/adapters/markdown/unified-markdown.ts`.

## Flujo de carga y mapeo

El punto de entrada de la carga es `ContentRepository`, no Astro Content Collections. El flujo completo, desde los archivos Markdown hasta el árbol de navegación, es:

```text
src/content/**/*.md  (o GitHub tree)
        │
        ▼
              ContentRepository
              (FileSystemRepository / GitHubRepository)
        │
        ▼
              Document[]
        │
        ▼
    resolveProxies(ref) usando CompositeReferenceResolver
        │
        ▼
              Document[]  (proxies resueltos)
        │
        ▼
              buildNavigationTree(documents)
        │
        ▼
        NavigationTree { roots, nodeMap }
        │
        ▼
   Breadcrumb / Sidebar / Índice
```

### 1. Repositorio: `ContentRepository`

`src/document/adapters/repository/factory.ts` crea la implementación según `CONTENT_SOURCE`:

```typescript
const repository = await createContentRepository();
const documents = await repository.list();
```

Implementaciones actuales:

- `FileSystemRepository`: escanea `src/content/**/*.md` y construye `Document` con `buildDocument()`.
- `GitHubRepository`: obtiene el tree del repo, filtra archivos Markdown y construye `Document` con `buildDocument()`.

### 2. Resolución de proxies

`src/document/parse/proxy.ts` resuelve documentos con `ref` en frontmatter:

```typescript
const resolvedDocuments = await resolveProxies(documents);
```

El resolver es un `CompositeReferenceResolver` con tres plugins:

- `InternalReferenceResolver`: referencias relativas a otros documentos cargados.
- `LocalFileReferenceResolver`: archivos locales fuera de `src/content/` (ej. `/README.md`).
- `GitHubReferenceResolver`: archivos en repositorios de GitHub (`github:owner/repo/path.md`).

### 3. Mapeo a `Document`

`src/document/adapters/repository/document-builder.ts` contiene `buildDocument()`:

```typescript
export function buildDocument(id, data, body, raw): Document {
  return {
    id,
    slug: id === "index" ? "" : id,
    parentId: getParentId(id),
    position: metadata.position,
    title: metadata.title,
    description: metadata.description,
    content: body,
    metadata,
    rawFrontmatter: extractFrontmatter(raw),
  };
}
```

El `parentId` se calcula en `src/document/parse/path.ts`:

```typescript
export function getParentId(id: string): string | null {
  if (id === "index") return null;
  const lastSlash = id.lastIndexOf("/");
  return lastSlash === -1 ? "index" : id.slice(0, lastSlash);
}
```

Esto convierte la ruta del archivo en una jerarquía lógica:

| Archivo | `id` | `parentId` |
| --- | --- | --- |
| `index.md` | `index` | `null` |
| `blog/index.md` | `blog` | `index` |
| `blog/overthinking.md` | `blog/overthinking` | `blog` |
| `nanobook-project/arquitectura/api-de-navegacion.md` | `nanobook-project/arquitectura/api-de-navegacion` | `nanobook-project/arquitectura` |

### 4. Construcción del árbol de navegación

Una vez que la página tiene `Document[]`, construye la navegación con [`buildNavigationTree()`](../../navigation-api):

```typescript
import { buildNavigationTree } from "../navigation/graph/builder";

const { nodeMap } = buildNavigationTree(documents);
```

La construcción del árbol no ocurre dentro del repositorio. `ContentRepository` devuelve datos; `NavigationBuilder` deriva estructuras de navegación. Esta separación mantiene el dominio storage-agnostic y permite cambiar la estrategia de navegación sin tocar la carga de contenido.

Ver también:

- [API de navegación](../api-de-navegacion) para el contrato completo de `NavigationBuilder`.
- [Storage adapters](../storage-adapters) para los adapters disponibles y cómo añadir uno nuevo.

## Estado actual

### Fase 0 completada

- Se creó `src/document/model/types.ts` con los tipos base del dominio.
- Se consolidó `DocumentEntry` en el mismo módulo.
- El build sigue funcionando igual.

### Fase 1 completada

- Se creó `src/document/adapters/repository/astro-collection-repository.ts` con `AstroCollectionRepository`, cacheando la colección a nivel de módulo.
- Se creó `src/navigation/graph/builder.ts` con `NavigationNode`, `NavigationTree`, `buildNavigationTree`, `getBreadcrumbs`, `getImmediateChildren`, `getSidebarEntries` y `getParentEntry`, cacheando el árbol por el array de documentos.
- Se actualizó `src/pages/[...slug]/index.astro` para construir el árbol de navegación y derivar breadcrumb, sidebar e índices del `nodeMap`.
- Los componentes `Layout.astro`, `SidebarNav/index.astro` y `SidebarList.astro` ahora usan `NavigationNode`.

### Fase 2 completada

- Se añadió `position` al schema de Astro (`src/content.config.ts`) y a `DocumentMetadata`.
- `NavigationBuilder` ordena por `position` con fallback por título.
- Se creó `src/rendering/model/types.ts` con la interfaz `DocumentRenderer`.
- Se creó `src/rendering/adapters/markdown/unified-markdown.ts` como renderer principal de Markdown.
- `AstroCollectionRepository` ya no se encarga del renderizado; su responsabilidad es solo la lectura de documentos.
- Se creó `src/document/adapters/cache/astro-cache.ts` para compartir las entradas crudas de Astro entre repository y renderer sin acoplarlos.
- `src/pages/[...slug]/index.astro` crea `repository` y `renderer` como objetos separados.

### Fase 3 completada

- Se creó `src/document/adapters/repository/memory-repository.ts` para tests y desarrollo.
- Se creó `src/document/adapters/repository/database-repository.ts` como stub del adapter de base de datos.
- Se documentó la arquitectura de storage adapters en `src/content/nanobook-project/arquitectura/storage-adapters.md`.
- El build sigue usando `AstroCollectionRepository`; los nuevos adapters demuestran que el dominio es storage-agnostic.

### Transición a SSR puro

- `astro.config.mjs` siempre usa `output: "server"` + `@astrojs/node` (o `@astrojs/vercel` cuando `VERCEL_DEPLOY=true`).
- `src/content.config.ts` carga solo archivos locales de `src/content/`; GitHub ya no pasa por Astro Content Collections.
- Se eliminó `OUTPUT_MODE`; la fuente se elige con `CONTENT_SOURCE`.
- `AstroCollectionRepository` y `astro-cache.ts` fueron eliminados; la resolución de proxies se mudó a `src/document/parse/proxy.ts`.
- El build pasa con `CONTENT_SOURCE=filesystem` y `CONTENT_SOURCE=github`.

### Editor integrado por documento

- Cada documento tiene una vista alterna de edición en `/{slug}/edit` (por ejemplo `/markdown/edit`).
- Se añade un botón "Editar" en el header de las páginas de documento; en la vista de edición se muestra un botón "Ver" para volver.
- La vista de edición reutiliza el layout principal pero oculta el TOC (`hideToc`).
- El editor usa CodeMirror 6 (`codemirror`, `@codemirror/lang-markdown`) encapsulado en `src/components/DocumentEditor/` para permitir cambiar de editor más adelante.
- Lectura y escritura de archivos se hace mediante un plugin de Vite (`src/lib/editor/vite-plugin.ts`) que expone endpoints solo en desarrollo.

## Escalabilidad y SSR

### Árbol completo vs. rama parcial

En el modelo SSR puro, `NavigationBuilder` construye el árbol de navegación completo a partir de todos los documentos devueltos por `ContentRepository`. Esto es correcto porque:

- El contenido se cachea a nivel de repositorio (TTL configurable en `GitHubRepository`).
- El renderizado de página se cachea con `Cache-Control` y/o backends como Redis.
- Un árbol de cientos o miles de nodos es trivial de construir y cachear.

### Cuándo cambiar el enfoque

Si Nanobook escala a **decenas o cientos de miles de documentos**, construir el árbol completo por request se vuelve costoso. En ese escenario, `DatabaseRepository` debería cargar solo la rama necesaria:

```text
/nanobook-project/arquitectura/content-model-architecture/
    ↓
cargar: ancestros + padre + hermanos + hijos directos
```

### Cómo la arquitectura actual lo soporta

La separación entre `ContentRepository` y `NavigationBuilder` permite esta evolución sin reescribir el dominio:

```text
ContentRepository              NavigationBuilder
     │                                │
   get(id)                      buildNavigationTree()
   getAncestors(id)             getBreadcrumbs()
   getSiblings(id)              getSidebarEntries()
   getChildren(id)              getImmediateChildren()
```

`NavigationBuilder` puede evolucionar para aceptar una rama parcial o funciones de consulta en lugar de `Document[]`.

### Regla

No optimizar prematuramente. Hoy el árbol completo es la solución pragmática. Cuando escale, se añadirá una estrategia de rama parcial sobre la misma arquitectura.

## Editor integrado

El editor integrado permite editar documentos Markdown desde el navegador durante el modo desarrollo, sin necesidad de un editor de código.

- Cada documento tiene su propia URL de edición: `/{slug}/edit`.
- La vista de edición reutiliza el mismo layout que la vista de lectura, pero oculta el TOC y muestra el editor en el área de contenido.
- Botón "Editar" en el header de documentos; botón "Ver" en la vista de edición.
- Editor actual: CodeMirror 6, encapsulado en `src/components/DocumentEditor/` para poder cambiarlo sin tocar las páginas.
- Endpoints: plugin de Vite con `GET /api/editor/document?id=...` y `POST /api/editor/document`.
- Solo edita documentos existentes; índices se excluyen de la vista de edición en esta primera versión.
- Los ids se validan para evitar path traversal y se crean directorios padre automáticamente si se guarda en una ruta nueva.

Preguntas abiertas:

- ¿Edición de índices con una experiencia diferente?
- ¿CRUD completo (crear, renombrar, eliminar) o solo edición de contenido existente?
- ¿Activación explícita mediante variable de entorno o siempre en desarrollo?

## Próximos pasos documentados

1. **Editor integrado**: mejorar UX, añadir atajos de teclado y decidir si se amplía a CRUD completo.
2. **Edición en fuente remota**: definir el flujo de escritura de vuelta a GitHub (u otra fuente externa) en producción.
3. **SSR a gran escala**: evolucionar `NavigationBuilder` para soportar ramas parciales cuando haya un `DatabaseRepository` y miles de documentos.
4. **Directivas de contenido**: explorar sintaxis Markdown para insertar componentes reutilizables sin depender de MDX.

## Lo que NO se hará ahora

- No se implementa un adapter de base de datos real.
- No se vuelve a SSG como modo principal.
- No se añaden overrides ni manifest.

Todo eso se documenta y se deja listo para continuar sin reescribir el core.