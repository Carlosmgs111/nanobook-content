---
title: Plan de preparación para el grafo de dependencias
description: Roadmap de refactorización incremental para desacoplar el árbol de navegación global, modelar dependencias entre documentos y habilitar recompilado incremental en Nanobook.
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

## Objetivo

Este plan describe los pasos para preparar a Nanobook a un modelo de **recompilado incremental** sin reescribir el build de golpe. El objetivo intermedio no es construir todavía un motor incremental, sino **hacer que el árbol de documentos sea explícito, observable y desacoplado**, de forma que en el futuro podamos invalidar y regenerar solo las páginas afectadas por un cambio.

## Visión de la arquitectura objetivo

```text
┌─────────────────────────────────────────────────────────────┐
│                        Content Source                        │
│              (filesystem / GitHub / database)               │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   ContentRepository                          │
│    Carga documentos, resuelve proxies, expone Document[]    │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                    DocumentGraph                             │
│  Nodos = Documentos                                          │
│  Aristas = padre-hijo, hermano, proxy-target, link-interno   │
│  Capacidades:                                                │
│    - getAncestors(id)                                        │
│    - getChildren(id)                                         │
│    - getSiblings(id)                                         │
│    - getDependents(id)  ← qué páginas invalida este doc     │
│    - getInvalidatedIds(changeSet)                            │
└──────────────────────┬──────────────────────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                 Navigation + Rendering                       │
│   Cada página consulta solo las relaciones que necesita,    │
│   no todo el árbol global.                                   │
└─────────────────────────────────────────────────────────────┘
```

## Principios guía

1. **No cambiar el comportamiento visible**: cada refactor debe producir el mismo HTML y el mismo árbol de navegación.
2. **Un paso a la vez**: cada fase es independently deployable y verificable con `npm run build`.
3. **Tests antes que refactor**: estabilizar el comportamiento con tests de contrato antes de mover código.
4. **Astro sigue siendo el build pipeline**: no luchamos contra el framework; solo preparamos el dominio.
5. **Backward compatibility**: los componentes Astro pueden seguir usando abstracciones familiares mientras el interior evoluciona.

## Fases del plan

### Fase 0 — Cimientos: tests de contrato y snapshot del árbol

**Objetivo**: poder refactorizar `src/navigation/graph/builder.ts` con seguridad.

**Tareas**:

1. Añadir un runner de tests al proyecto (por ejemplo, `vitest` o `node:test`).
2. Crear tests para `buildNavigationTree` usando `MemoryRepository`:
   - Dado un set fijo de `Document[]`, el árbol resultante debe coincidir con un snapshot aprobado.
   - Verificar ordenamiento por `position` y desempate por título.
   - Verificar que un documento sin padre visible se convierte en raíz.
   - Verificar que borradores se excluyen (responsabilidad del repository).
3. Crear tests para helpers derivados:
   - `getBreadcrumbs`
   - `getSidebarEntries`
   - `getImmediateChildren`
   - `getParentEntry`
4. Añadir un test de integración que compare el árbol generado desde `src/content/` contra un snapshot JSON versionado.

**Criterios de aceptación**:

- `npm test` existe y pasa.
- Los tests cubren al menos el 80 % de `src/navigation/graph/builder.ts`.
- Un cambio que altere el árbol de navegación falla explícitamente en CI.

**Archivos afectados**:

- `package.json` (nuevo script `test` y dependencias).
- Nuevos archivos bajo `src/navigation/graph/__tests__/`.
- Posiblemente `tsconfig.json` para incluir paths de test.

### Fase 1 — Introducir `DocumentGraph` como abstracción

**Objetivo**: envolver el árbol de navegación en un objeto explícito sin cambiar la API pública de `builder.ts`.

**Tareas**:

1. Crear el tipo `DocumentGraph` en `src/navigation/graph/graph.ts`:
   ```ts
   export interface DocumentGraph {
     getRoots(): NavigationNode[];
     getNode(id: string): NavigationNode | undefined;
     getParent(id: string): NavigationNode | undefined;
     getChildren(id: string): NavigationNode[];
     getSiblings(id: string): NavigationNode[];
     getAncestors(id: string): NavigationNode[];
     getAllNodes(): NavigationNode[];
   }
   ```
2. Implementar `buildDocumentGraph(documents: Document[]): DocumentGraph`.
   - Internamente construye el mismo árbol que `buildNavigationTree`.
   - Cachea el resultado en un `WeakMap<Document[], DocumentGraph>`.
3. Refactorizar `buildNavigationTree` para que sea un alias de:
   ```ts
   export function buildNavigationTree(documents: Document[]): NavigationTree {
     const graph = buildDocumentGraph(documents);
     return {
       roots: graph.getRoots(),
       nodeMap: new Map(graph.getAllNodes().map(n => [n.id, n])),
     };
   }
   ```
4. Asegurar que `getBreadcrumbs`, `getSidebarEntries`, `getImmediateChildren` y `getParentEntry` puedan aceptar un `DocumentGraph` además del `Map` actual (sobrecarga).

**Criterios de aceptación**:

- Todos los tests de la Fase 0 siguen pasando sin modificaciones.
- `npm run build` genera el mismo `dist/` (comparando hashes de HTML si es posible).
- No hay referencias directas a `nodeMap` fuera de `builder.ts` que no puedan usar `DocumentGraph`.

**Archivos afectados**:

- `src/navigation/graph/graph.ts` (nuevo).
- `src/navigation/graph/builder.ts` (refactorizado).
- `src/pages/[...slug]/*.astro` (posiblemente adaptados para recibir `graph` en lugar de `nodeMap`).

### Fase 2 — Desacoplar las páginas del árbol global

**Objetivo**: que cada página de Astro no necesite `allDocuments` completo para renderizarse.

**Tareas**:

1. Introducir `NavigationService` en `src/navigation/service/service.ts`:
   ```ts
   export interface NavigationService {
     getBreadcrumbs(documentId: string): Crumb[];
     getSidebarEntries(documentId: string): NavigationNode[];
     getParentEntry(documentId: string): ParentEntry | null;
     getImmediateChildren(documentId: string): NavigationNode[];
   }
   ```
2. Implementar `NavigationService` a partir de `DocumentGraph`.
3. Refactorizar `src/pages/[...slug]/index.astro`, `edit.astro` y `preview.astro` para que:
   - Sigan obteniendo `Document[]` en `getStaticPaths` (Astro lo requiere).
   - En runtime usen `NavigationService` en lugar de construir el árbol manualmente.
4. Extraer una función `createNavigationService(documents: Document[]): NavigationService` que cachee el grafo.

**Criterios de aceptación**:

- Los archivos Astro son más cortos y no llaman `buildNavigationTree` directamente.
- `npm run build` sigue generando el mismo output.
- Los tests de la Fase 0 pasan.

**Archivos afectados**:

- `src/navigation/service/service.ts` (nuevo).
- `src/pages/[...slug]/index.astro`.
- `src/pages/[...slug]/edit.astro`.
- `src/pages/[...slug]/preview.astro`.

### Fase 3 — Modelar aristas de dependencia

**Objetivo**: que el grafo sepa no solo quién es padre de quién, sino **quién depende de quién para invalidación**.

**Tareas**:

1. Extender `DocumentGraph` con:
   ```ts
   export interface DocumentGraph {
     // ...métodos existentes...
     getDependencies(id: string): DependencyEdge[];
     getDependents(id: string): string[];  // documentos que se invalidan si cambia `id`
   }
   ```
2. Definir tipos de arista:
   ```ts
   export type DependencyKind =
     | "parent-child"
     | "sibling-order"
     | "proxy-target"
     | "internal-link";

   export interface DependencyEdge {
     sourceId: string;
     targetId: string;
     kind: DependencyKind;
   }
   ```
3. Implementar generación de aristas:
   - `parent-child`: padre → hijo.
   - `sibling-order`: hermanos bajo el mismo padre (cualquier cambio de posición/título afecta el orden).
   - `proxy-target`: documento proxy → documento destino.
   - `internal-link`: parsear links internos del cuerpo Markdown (`./foo`, `../bar`, `/baz`).
4. Añadir tests que verifiquen las aristas para el contenido actual.

**Criterios de aceptación**:

- Para cada documento real, `getDependents(id)` devuelve el conjunto esperado.
- Los documentos proxy aparecen con arista `proxy-target`.
- Los links internos generan aristas `internal-link`.
- No se rompe la navegación existente.

**Archivos afectados**:

- `src/navigation/graph/graph.ts`.
- `src/navigation/model/types.ts` (nuevo archivo de tipos compartidos).
- `src/document/parse/document.ts` (posiblemente reutilizar `parseDocument` para extraer links).
- Tests correspondientes.

### Fase 4 — Sistema de invalidación por cambio

**Objetivo**: dado un conjunto de documentos modificados, determinar el conjunto mínimo de IDs a regenerar.

**Tareas**:

1. Introducir `ChangeSet` y `InvalidationEngine`:
   ```ts
   export interface DocumentChange {
     id: string;
     kind: "added" | "modified" | "removed" | "renamed";
     previousId?: string;  // para renombramientos
   }

   export interface InvalidationResult {
     invalidatedIds: string[];
     removedIds: string[];
     addedIds: string[];
   }
   ```
2. Implementar `computeInvalidatedIds(graph: DocumentGraph, changes: DocumentChange[]): InvalidationResult`:
   - Si un documento cambia, se invalida a sí mismo.
   - Se propagan invalidaciones por aristas de dependencia.
   - Un renombramiento se trata como `removed(oldId) + added(newId)`.
   - Se detecta cambio de `draft`, `index`, `position`, `title`, `ref` mediante hashes de frontmatter.
3. Añadir tests con escenarios:
   - Editar contenido de un documento → solo ese documento.
   - Cambiar `position` de un hijo → hijo + padre + hermanos.
   - Renombrar un documento → todos los dependientes.
   - Cambiar target de un proxy → proxy + padre/hermanos del proxy.

**Criterios de aceptación**:

- Tests de invalidación cubren los 4 escenarios anteriores.
- El motor nunca produce un conjunto vacío si hay cambios.
- El motor no incluye documentos no afectados en escenarios controlados.

**Archivos afectados**:

- `src/navigation/graph/invalidation.ts` (nuevo).
- Tests correspondientes.

### Fase 5 — Integrar con `ContentRepository`

**Objetivo**: que el repositorio pueda exponer el grafo y el motor de invalidación.

**Tareas**:

1. Extender `ContentRepository`:
   ```ts
   export interface ContentRepository {
     list(): Promise<Document[]>;
     get(id: string): Promise<Document | null>;
     listChildren(parentId: string | null): Promise<Document[]>;
     save(document: Document): Promise<void>;
     // Nuevos métodos:
     getGraph(): Promise<DocumentGraph>;
     getInvalidatedIds(changes: DocumentChange[]): Promise<InvalidationResult>;
   }
   ```
2. Implementar en `AstroCollectionRepository`, `MemoryRepository` y stub en `DatabaseRepository`.
3. Introducir `DocumentSnapshot` para comparar estados:
   - Guardar un snapshot JSON con hashes de cada documento.
   - Comparar snapshot anterior vs actual para detectar `DocumentChange[]`.
4. Añadir utilidad CLI (opcional en esta fase):
   - `pnpm content:status` (`scripts/content.ts status`) que imprima el `ChangeSet` y las páginas invalidadas.

**Criterios de aceptación**:

- `AstroCollectionRepository.getGraph()` devuelve un grafo consistente con `repository.list()`.
- El script de detección de cambios funciona en local y es útil para debugging.
- Build pasa.

**Archivos afectados**:

- `src/document/model/types.ts`.
- `src/document/adapters/repository/astro-collection-repository.ts`.
- `src/document/adapters/repository/memory-repository.ts`.
- `src/document/adapters/repository/database-repository.ts`.
- `scripts/content.ts` con subcomandos `status` y `render` (nuevo).

### Fase 6 — Cachear renders por dependencias (preparación)

**Objetivo**: que el renderizado de una página pueda invalidarse selectivamente, incluso si todavía usamos build completo.

**Tareas**:

1. Introducir `RenderedPageCache` con clave compuesta:
   ```ts
   export interface RenderedPageCache {
     get(pageId: string, contentHash: string): Promise<string | null>;
     set(pageId: string, contentHash: string, html: string): Promise<void>;
     invalidate(pageIds: string[]): Promise<void>;
   }
   ```
2. Implementar una versión en filesystem bajo `.nanobook/cache/`.
3. Crear un renderizador de página aislado (fuera de Astro) que pueda usar `DocumentGraph`, `NavigationService` y `UnifiedMarkdownRenderer` para generar HTML dado un `documentId`.
4. En modo desarrollo, permitir regenerar una sola página mediante un endpoint o comando.

**Criterios de aceptación**:

- La cache en filesystem persiste entre builds.
- Una página cuyo hash de dependencias no cambia se sirve desde cache.
- La invalidación manual funciona.

**Archivos afectados**:

- `src/rendering/page/page-cache.ts` (nuevo).
- `src/rendering/page/page-renderer.ts` (nuevo).
- Posiblemente un nuevo adapter de `ContentRepository` orientado a cache.

### Fase 7 — SSR puro con cache isomórfico (completado)

**Objetivo**: habilitar actualizaciones en producción sin build completo.

**Tareas realizadas**:

1. Consolidar `output: "server"` como único modo; eliminar `OUTPUT_MODE`.
2. Implementar `ContentRepository` que lea contenido en runtime (`FileSystemRepository`, `GitHubRepository`).
3. Usar `NavigationService` y `DocumentGraph` en runtime.
4. Añadir capa de cacheo HTTP (`Cache-Control`) y cache de cuerpos (`RenderedPageCache`) con backends intercambiables (memory, filesystem, Redis).
5. Exponer endpoint de invalidación (`/api/invalidate`) y webhook de GitHub (`/api/webhook/github`).

**Criterios de aceptación**:

- Una petición a cualquier URL renderiza correctamente bajo demanda.
- El cache se invalida cuando cambia el contenido fuente.
- `pnpm test` y `pnpm build` pasan.

Ver [plan de transición a SSR puro](../plan-transicion-ssr-puro) para el detalle completo.

## Roadmap tentativo

| Fase | Duración estimada | Entregable clave |
|------|-------------------|------------------|
| Fase 0 | 1 sesión | Tests de navegación pasando |
| Fase 1 | 1 sesión | `DocumentGraph` funcional sin cambios visibles |
| Fase 2 | 1-2 sesiones | Páginas Astro usan `NavigationService` |
| Fase 3 | 2 sesiones | Grafo con aristas de dependencia |
| Fase 4 | 2 sesiones | Motor de invalidación con tests |
| Fase 5 | 1-2 sesiones | Repositorio expone grafo e invalidación |
| Fase 6 | 2-3 sesiones | Cache de renders y renderizador aislado |
| Fase 7 | 3+ sesiones | Modo dinámico con ISR |

## Criterios para activar el recompilado incremental real

No avanzar a la Fase 6 hasta que se cumplan:

1. Todas las fases 0-5 están mergeadas y testeadas.
2. El árbol de navegación actual es idéntico al producido por la nueva arquitectura (validado por snapshot).
3. El motor de invalidación tiene cobertura de tests > 90 %.
4. Se documentó el modelo de dependencias y sus límites.

## Riesgos y mitigaciones

| Riesgo | Mitigación |
|--------|------------|
| Refactor muy grande | Dividir en fases; cada una mergeable por separado. |
| Regresión en navegación | Tests de snapshot antes de tocar `builder.ts`. |
| Proxies complejos | Modelar arista `proxy-target` desde la Fase 3; no postergar. |
| Links internos rotos al renombrar | Validar aristas `internal-link` y documentar que el sistema detectará dependencias, pero no reescribirá URLs automáticamente. |
| Astro cambia de versión | No depender de internals de Astro en el dominio; mantener capa de adapter. |

## Próximo paso inmediato

Empezar por la **Fase 0**: añadir tests de contrato sobre `buildNavigationTree` usando `MemoryRepository`. Esto desbloquea todas las refactorizaciones siguientes y establece una línea base segura.
