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
 FileSystemRepository MemoryRepository DatabaseRepository
      │                   │                  │
   src/content/        Document[]         PostgreSQL
                                                 
 GitHubRepository
      │
 GitHub API (read + write)
```

## Adapters actuales

### FileSystemRepository

Ubicación: `src/document/adapters/repository/file-system-repository.ts`

Lee documentos directamente desde `src/content/` sin depender de Astro. Es el adapter por defecto cuando `CONTENT_SOURCE` no está definida o vale `filesystem`.

```typescript
const repository = new FileSystemRepository();
const documents = await repository.list();
```

Internamente escanea archivos `.md`, parsea el frontmatter y construye objetos `Document`.

### GitHubRepository

Ubicación: `src/document/adapters/repository/github-repository.ts`

Lee documentos Markdown desde un repositorio de GitHub remoto. Se activa con `CONTENT_SOURCE=github` y las variables de entorno `GITHUB_OWNER`, `GITHUB_REPO`, `GITHUB_BRANCH`, `GITHUB_TOKEN` y `GITHUB_PATH`.

Además de `list()`, `get()` y `listChildren()`, implementa `save(document)` usando la [GitHub Contents API](https://docs.github.com/en/rest/repos/contents?apiVersion=2022-11-28#create-or-update-file-contents):

1. Calcula la ruta del archivo dentro del repo a partir del `id` del documento y del `path` base configurado.
2. Obtiene el `sha` actual del archivo mediante `GET /repos/{owner}/{repo}/contents/{path}`.
3. Envía un `PUT` con el contenido codificado en base64, el mensaje de commit y el `sha` cuando el archivo ya existe (omite el `sha` para crear uno nuevo).
4. Invalida el cache de documentos para que la siguiente lectura refleje el cambio.

```typescript
const repository = new GitHubRepository({
  owner: "usuario",
  repo: "nanobook-content",
  branch: "main",
  token: process.env.GITHUB_TOKEN,
  path: "docs",
});

await repository.save(document);
```

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

- ✅ `FileSystemRepository` implementado y en uso como adapter por defecto.
- ✅ `GitHubRepository` implementado para lectura y escritura desde GitHub.
- ✅ `MemoryRepository` implementado para tests/desarrollo.
- ⏳ `DatabaseRepository` como stub; se implementará cuando se añada el SaaS.

## Documentación relacionada

- [Arquitectura del modelo de contenido](./content-model-architecture)
- [API de navegación](./api-de-navegacion)
