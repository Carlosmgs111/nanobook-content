---
title: "Recompilado incremental - Estado actual y flujo de trabajo"
description: "Documentación de las fases completadas del plan de preparación para el grafo de dependencias, arquitectura resultante y flujo de principio a fin para detectar y regenerar contenido de forma incremental."
date: 2026-08-23
author: "Nanobook Team"
tags: ["arquitectura", "build", "incremental", "dependencias", "documentation"]
draft: false
index: false
---

## Resumen

Este documento describe el estado actual del trabajo de preparación para el **recompilado incremental** en Nanobook. Se han completado las fases 0 a 6 del plan de preparación para el grafo de dependencias, lo que significa que el proyecto ya puede:

1. Modelar el árbol de documentos como un grafo explícito.
2. Detectar dependencias entre documentos (`parent-child`, `sibling-order`, `proxy-target`, `internal-link`).
3. Calcular el conjunto mínimo de documentos a regenerar ante un cambio.
4. Renderizar y cachear cuerpos de documentos de forma selectiva.

**No se ha implementado todavía** el recompilado incremental real de HTML completo en producción. El objetivo intermedio era desacoplar el dominio y preparar las abstracciones necesarias.

## Fases completadas

### Fase 0 — Tests de contrato sobre navegación

- Se añadió **Vitest** como runner de tests.
- Se crearon tests de contrato para `buildNavigationTree` y sus helpers (`getBreadcrumbs`, `getSidebarEntries`, `getImmediateChildren`, `getParentEntry`).
- Esto estableció una línea base segura para refactorizar la navegación sin romper el árbol existente.

### Fase 1 — `DocumentGraph`

- Se introdujo la abstracción `DocumentGraph` en `src/navigation/graph/graph.ts`.
- `buildNavigationTree` se refactorizó para delegar en `buildDocumentGraph`, manteniendo la misma API pública.
- El grafo expone métodos de relación: `getRoots`, `getNode`, `getParent`, `getChildren`, `getSiblings`, `getAncestors`, `getAllNodes`.

### Fase 2 — `NavigationService`

- Se creó `NavigationService` en `src/navigation/service/service.ts`.
- Las páginas Astro (`src/pages/[...slug]/index.astro` y `edit.astro`) dejaron de construir el árbol global manualmente y ahora consumen el servicio.
- Esto redujo el acoplamiento global de cada página al árbol completo.

### Fase 3 — Aristas de dependencia

- Se extendió `DocumentGraph` para modelar aristas de dependencia:
  - `parent-child`: padre ↔ hijo.
  - `sibling-order`: entre hermanos (afecta orden en índices y sidebar).
  - `proxy-target`: proxy → documento destino.
  - `internal-link`: documento origen → documento destino por link Markdown.
- Se añadieron `getDependencies(id)` y `getDependents(id)`.
- Se creó `src/document/reference/link-extractor.ts` para extraer links internos del cuerpo Markdown.
- Se extrajo la lógica de resolución de referencias a `src/document/reference/path-resolver.ts` para reutilizarla.

### Fase 4 — Motor de invalidación

- Se implementó `computeInvalidatedIds()` en `src/navigation/graph/invalidation.ts`.
- Soporta `DocumentChange` con `kind` (`added`, `modified`, `removed`, `renamed`) y `scope` (`content`, `metadata`, `all`).
- Propaga invalidaciones por BFS sobre el grafo, evitando ciclos.
- Un cambio de contenido solo invalida el documento y sus proxies; un cambio estructural invalida también dependientes.

### Fase 5 — Detección de cambios por snapshot

- Se creó `src/document/change/snapshot.ts` con `hashDocument()` y `computeDocumentChanges()`.
- Cada documento tiene dos hashes: `contentHash` y `metadataHash`.
- Se creó `ContentChangeService` en `src/document/change/change-service.ts` para orquestar detección, grafo e invalidación.
- Se creó `FileSystemRepository` en `src/document/adapters/repository/file-system-repository.ts` para leer contenido sin depender de Astro.
- Se añadió el subcomando `pnpm content:status` (`scripts/content.ts status`) que compara contra `.nanobook/content-snapshot.json`.

### Fase 6 — Cache de renders y renderizador aislado

- Se creó `RenderedPageCache` en `src/rendering/page/page-cache.ts`.
- Se implementó cache en filesystem en `src/rendering/adapters/cache/file-system-page-cache.ts`.
- Se creó `PageRenderer` en `src/rendering/page/page-renderer.ts` para renderizar cuerpos de documentos fuera del pipeline de Astro.
- Se añadió el subcomando `pnpm content:render` (`scripts/content.ts render`) que detecta cambios y regenera solo los cuerpos invalidados.

## Arquitectura resultante

```text
┌─────────────────────────────────────────────────────────────────────┐
│                          Content Source                              │
│                  (src/content/ / GitHub / database)                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     ContentRepository                                │
│   Carga documentos y expone Document[]                                │
│   Implementaciones: AstroCollectionRepository, FileSystemRepository   │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    ContentChangeService                              │
│   - detectChanges(previousHashes) → DocumentChange[]                 │
│   - getGraph() → DocumentGraph                                       │
│   - getInvalidatedIds(changes) → InvalidationResult                  │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                       DocumentGraph                                  │
│   Nodos = Documentos                                                  │
│   Aristas = parent-child | sibling-order | proxy-target | link       │
│   - getDependencies(id)                                               │
│   - getIncomingEdges(id)  ← usado para propagar invalidaciones       │
│   - getDependents(id)                                                 │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                    NavigationService                                 │
│   - getBreadcrumbs(id)                                                │
│   - getSidebarEntries(id)                                             │
│   - getParentEntry(id)                                                │
│   - getImmediateChildren(id)                                          │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      PageRenderer                                    │
│   Renderiza el cuerpo HTML de un documento + metadatos de navegación │
└──────────────────────────┬──────────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                  FileSystemRenderedPageCache                         │
│   Almacena cuerpos renderizados en .nanobook/cache/pages/            │
│   Clave de validez: contentHash del documento                        │
└─────────────────────────────────────────────────────────────────────┘
```

## Flujo de principio a fin

El siguiente flujo describe cómo funciona el nuevo enfoque desde que un autor modifica un documento hasta que se calculan las páginas a regenerar.

### 1. Modificación de un documento

Un autor edita un archivo Markdown en `src/content/`:

```text
src/content/nanobook-project/arquitectura/proxy-documents.md
```

El cambio puede ser:

- **Contenido**: modificar el cuerpo Markdown sin tocar frontmatter.
- **Metadata**: cambiar `title`, `description`, `position`, `draft`, `index` o `ref`.
- **Estructural**: agregar, renombrar o eliminar archivos/directorios.

### 2. Detección de cambios

El script `pnpm content:status` lee el snapshot anterior y compara hashes:

```text
$ pnpm content:status
Changes detected:
  [modified] nanobook-project/arquitectura/proxy-documents (content)

Invalidated documents:
  - nanobook-project/arquitectura/proxy-documents
  - proyectos/generador-demos-alias
```

Si el cambio es solo de contenido, solo se invalida el documento editado y los proxies que lo referencian. Si es de metadata, también se invalidan padres, hermanos y documentos con links internos.

### 3. Cálculo de dependientes

`ContentChangeService` construye el `DocumentGraph` y llama a `computeInvalidatedIds()`:

```text
DocumentChange[]
        │
        ▼
computeInvalidatedIds(graph, changes)
        │
        ▼
BFS sobre getIncomingEdges()
        │
        ▼
InvalidationResult { invalidatedIds, addedIds, removedIds }
```

Paralelamente, `shouldRebuildNavigationTree(changes)` indica si el cambio afecta la estructura del árbol de navegación:

```text
DocumentChange[]
        │
        ▼
shouldRebuildNavigationTree(changes)
        │
        ├─ true  → recalcular árbol (cambio estructural o de metadata)
        └─ false → el árbol es el mismo, solo renderizar contenido
```

### 4. Renderizado selectivo

El script `pnpm content:render` regenera solo los cuerpos de los documentos invalidados:

```text
$ pnpm content:render
Rendering 5 invalidated page(s)...
  - rendered: nanobook-project/arquitectura/proxy-documents
  - rendered: proyectos/generador-demos-alias
  - rendered: nanobook-project/arquitectura
  - rendered: nanobook-project/proyectos
  - rendered: nanobook-project/arquitectura/index
Snapshot updated: .nanobook/content-snapshot.json
```

Cada cuerpo renderizado se guarda en `.nanobook/cache/pages/<hash>.json` junto con el `contentHash` del documento.

### 5. Cacheo

`FileSystemRenderedPageCache` verifica si ya existe un render válido antes de regenerar:

```text
get(pageId, contentHash)
  │
  ├─ cache no existe → null → renderizar
  ├─ contentHash no coincide → null → renderizar
  └─ cache válido → devolver HTML almacenado
```

### 6. Integración futura con Astro / ISR

En la Fase 7, el flujo se completará:

```text
Petición a /nanobook-project/arquitectura/proxy-documents/
        │
        ▼
SSR en modo dinámico
        │
        ▼
¿Hay cache válido en filesystem/redis? → servir
        │
        ▼
Renderizar página completa con Astro + cachear
```

Hasta ahora, el flujo se detiene en el cuerpo renderizado. El layout completo sigue generándose con `pnpm build`.

## Herramientas disponibles

### Tests

```bash
pnpm test
```

Actualmente hay **49 tests** distribuidos en:

- `src/navigation/graph/__tests__/builder.test.ts` — 9 tests.
- `src/navigation/graph/__tests__/graph.test.ts` — 8 tests.
- `src/navigation/graph/__tests__/invalidation.test.ts` — 11 tests.
- `src/navigation/service/__tests__/service.test.ts` — 5 tests.
- `src/document/change/__tests__/snapshot.test.ts` — 8 tests.
- `src/document/change/__tests__/change-service.test.ts` — 2 tests.
- `src/rendering/page/__tests__/page-cache.test.ts` — 4 tests.
- `src/rendering/page/__tests__/page-renderer.test.ts` — 2 tests.

### Scripts de utilidad

```bash
# Detectar cambios e imprimir invalidaciones
pnpm content:status

# Detectar cambios, renderizar cuerpos invalidados y actualizar cache
pnpm content:render
```

Ambos comandos comparten la misma lógica de detección; la diferencia es que `render` ejecuta la fase de renderizado y actualiza `.nanobook/content-snapshot.json`.

### APIs principales

```ts
// Grafo de dependencias
const graph = buildDocumentGraph(documents);
graph.getDependents("doc-id");
graph.getDependencies("doc-id");

// Servicio de navegación
const navigation = createNavigationService(documents);
navigation.getBreadcrumbs("doc-id");
navigation.getSidebarEntries("doc-id");

// Detección de cambios e invalidación
const service = new ContentChangeService(repository);
const changes = await service.detectChanges(previousHashes);
const result = await service.getInvalidatedIds(changes);

// Decidir si es necesario reconstruir el árbol de navegación
const needsTreeRebuild = shouldRebuildNavigationTree(changes);

// Renderizado y cache
const renderer = new PageRenderer(repository, new UnifiedMarkdownRenderer());
const cache = new FileSystemRenderedPageCache();
```

## Limitaciones conocidas

1. **No se renderiza el layout Astro completo**: `PageRenderer` solo genera el cuerpo del documento y los metadatos de navegación. El layout sigue siendo responsabilidad del build de Astro.
2. **Los renombramientos se detectan como removed + added**: no se reconocen automáticamente como `renamed` con `previousId`.
3. **FileSystemRepository no resuelve proxies**: en esta etapa lee documentos Markdown tal cual; no aplica la lógica de `ref`/`proxyTargetId`.
   - *Actualización*: en SSR puro ambos repositorios (`FileSystemRepository` y `GitHubRepository`) aplican `resolveProxies()` en `list()`.
4. **Modo dinámico incompleto**: `OUTPUT_MODE=dynamic` aún no permite SSR real de las páginas de contenido.
   - *Actualización*: `OUTPUT_MODE` fue eliminado; `output: "server"` es el modo canónico. Ver [plan de transición a SSR puro](../plan-transicion-ssr-puro).
5. **La invalidación es conservadora**: algunos cambios de metadata podrían propagarse a más documentos de lo estrictamente necesario; es seguro pero no óptimo.
6. **El árbol de navegación sigue reconstruyéndose en build time**: aunque está cacheado en memoria por `WeakMap`, Astro aún genera todas las páginas. `shouldRebuildNavigationTree()` es la utilidad que en el futuro permitirá decidir si es necesario recalcular el árbol o solo re-renderizar contenido.
   - *Actualización*: con SSR puro el árbol se reconstruye en runtime por request, pero el repositorio y el renderizado se cachean.

## Próximos pasos

La Fase 7 (modo dinámico con ISR) fue completada y posteriormente consolidada como **SSR puro**. Ver:

- [Plan de implementación — Fase 7: ISR isomórfico con cache abstracto](../plan-fase-7-isr-modo-dinamico)
- [Plan de transición a SSR puro](../plan-transicion-ssr-puro)

El build completo ya no es el mecanismo de producción; el sitio se sirve bajo demanda con cache.
