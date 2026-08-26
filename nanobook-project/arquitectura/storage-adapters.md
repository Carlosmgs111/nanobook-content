---
title: "Storage adapters"
description: "Adapters disponibles para ContentRepository y cómo el dominio de Nanobook es agnóstico al almacenamiento."
date: 2026-08-23
author: "Nanobook Team"
tags: ["arquitectura", "storage", "repository", "adapter"]
draft: false
index: false
---

## Principio

El dominio de Nanobook no sabe si los documentos vienen de Markdown, PostgreSQL, GitHub o memoria. Esa decisión queda en los **adapters** que implementan `ContentRepository`.

```text
                ContentRepository
                       │
        ┌──────────────┼──────────────┐
        │              │              │
 AstroCollection   MemoryRepository  DatabaseRepository
      │                   │                  │
   Astro glob           Document[]         PostgreSQL
   + GitHub loader                        (futuro)
```

## Adapters actuales

### AstroCollectionRepository

Ubicación: `src/document/adapters/repository/astro-collection-repository.ts`

Implementación actual. Lee documentos desde la colección de Astro, que a su vez usa `glob` (filesystem local) o el loader `github` (contenido remoto).

```typescript
const repository = new AstroCollectionRepository();
const documents = await repository.list();
```

Internamente:

1. Llama a `getAstroEntries()` (`src/document/adapters/cache/astro-cache.ts`) para obtener un `Map<id, Astro entry>`.
2. Construye un `CompositeReferenceResolver` para soportar referencias `ref`.
3. Filtra `draft: true`.
4. Mapea cada entrada a `Document` mediante `toDocument()` o `resolveProxy()`.

Ver [Arquitectura del modelo de contenido](./content-model-architecture#flujo-de-carga-y-mapeo) para el flujo completo.

### MemoryRepository

Ubicación: `src/document/adapters/repository/memory-repository.ts`

Trabaja con un array de `Document` en memoria. Útil para tests, prototipado y contenido generado dinámicamente.

```typescript
import { MemoryRepository } from "../document/adapters/repository/memory-repository";

const documents: Document[] = [
  {
    id: "hello",
    slug: "hello",
    parentId: null,
    position: 0,
    title: "Hello",
    description: "...",
    content: "# Hello",
    metadata: { /* ... */ },
    rawFrontmatter: "---\n...\n---\n\n",
  },
];

const repository = new MemoryRepository(documents);
```

### DatabaseRepository

Ubicación: `src/document/adapters/repository/database-repository.ts`

Stub sin implementar. Define el contrato futuro para cuando Nanobook necesite PostgreSQL u otra base de datos (por ejemplo, en el SaaS).

## Cómo añadir un nuevo adapter

1. Crear una clase que implemente `ContentRepository` (`src/document/model/types.ts`).
2. Mapear la fuente de datos a objetos `Document`.
3. Filtrar `draft: true` en `list()` y `get()`.
4. Calcular `parentId` a partir del `id` o de la estructura de la fuente.

Ejemplo mínimo:

```typescript
import type { ContentRepository, Document } from "../document/types";

export class MyAdapter implements ContentRepository {
  async list(): Promise<Document[]> { /* ... */ }
  async get(id: string): Promise<Document | null> { /* ... */ }
  async listChildren(parentId: string | null): Promise<Document[]> { /* ... */ }
  async save(document: Document): Promise<void> { /* ... */ }
}
```

## Relación con NavigationBuilder

`NavigationBuilder` recibe `Document[]` y construye el árbol de navegación. No importa de dónde vengan esos documentos.

```text
ContentRepository → Document[] → NavigationBuilder → NavigationTree
```

Esto es clave para mantener el dominio storage-agnostic. La construcción del árbol vive en `src/navigation/graph/builder.ts`, no en los adapters. Ver [API de navegación](./api-de-navegacion).

## Estado

- ✅ `AstroCollectionRepository` implementado y en uso.
- ✅ `MemoryRepository` implementado para tests/desarrollo.
- ⏳ `DatabaseRepository` como stub; se implementará cuando se añada el SaaS.

## Documentación relacionada

- [Arquitectura del modelo de contenido](./content-model-architecture)
- [API de navegación](./api-de-navegacion)
