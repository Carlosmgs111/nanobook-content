---
title: "API de navegación"
description: "Documentación de la API de NavigationBuilder: funciones disponibles, contratos, ejemplos de uso y por qué vive separada del repositorio de contenido."
date: 2026-08-23
author: "Nanobook Team"
tags: ["arquitectura", "navegación", "api", "navigation-builder"]
draft: false
index: false
---

## Propósito

La navegación de Nanobook se construye a partir del modelo de contenido, no del filesystem. `NavigationBuilder` es el módulo del dominio que transforma una lista de `Document` en estructuras de navegación: árboles, breadcrumbs, sidebars e índices.

El flujo es:

```text
Document[]
    ↓
buildNavigationTree()
    ↓
NavigationTree { roots, nodeMap }
    ↓
getBreadcrumbs() / getSidebarEntries() / getImmediateChildren() / getParentEntry()
```

## Ubicación

```text
src/navigation/graph/builder.ts
```

## ¿Por qué no está en el repositorio?

El repositorio (`ContentRepository`) se encarga de obtener y persistir documentos. Su contrato es `Document[]`. La construcción del árbol de navegación es una **proyección derivada** de esos documentos: ordenarlos, agruparlos por `parentId` y exponer consultas como breadcrumb o sidebar.

Mantener `buildNavigationTree` fuera del repositorio preserva la separación de responsabilidades:

```text
ContentRepository          NavigationBuilder           Páginas / UI
      │                            │                         │
  Document[] ──────────────► NavigationTree ────────► breadcrumbs, sidebar, índices
```

Esto permite, por ejemplo, cambiar la estrategia de navegación o construir solo una rama parcial en SSR sin tocar la capa de almacenamiento. Ver [Arquitectura del modelo de contenido](./content-model-architecture#escalabilidad-y-ssr) para la evolución futura.

## Tipos

### NavigationNode

Nodo de un árbol de navegación. Es una proyección ligera de `Document`.

```typescript
interface NavigationNode {
  id: string;
  slug: string;
  title: string;
  description: string;
  isIndex: boolean;
  parentId: string | null;
  position: number;
  metadata: DocumentMetadata;
  children: NavigationNode[];
  current?: boolean;
}
```

### Crumb

Elemento de un breadcrumb.

```typescript
interface Crumb {
  id: string;
  title: string;
  href: string;
  current: boolean;
}
```

### NavigationTree

Resultado de construir el árbol. Contiene las raíces y un mapa para consultas directas.

```typescript
interface NavigationTree {
  roots: NavigationNode[];
  nodeMap: Map<string, NavigationNode>;
}
```

## Funciones

### buildNavigationTree

Construye el árbol de navegación completo a partir de todos los documentos.

```typescript
buildNavigationTree(documents: Document[]): NavigationTree
```

- Ordena primero por `position`, luego por título.
- Agrupa los nodos bajo su `parentId`.
- Los documentos sin padre visible aparecen como raíces.
- Cachea el resultado por el array de documentos para no reconstruirlo en cada página durante el build.

Devuelve `{ roots, nodeMap }`. `roots` sirve para menús globales; `nodeMap` para resolver breadcrumb, sidebar e hijos en O(1).

> Nota: `buildNavigationTree` asume que los documentos ya están filtrados (por ejemplo, sin `draft: true`). En la práctica, `AstroCollectionRepository.list()` filtra los borradores antes de devolver `Document[]`.

### getBreadcrumbs

Devuelve el breadcrumb de un documento dado.

```typescript
getBreadcrumbs(
  nodeMap: Map<string, NavigationNode>,
  entryId: string,
  homeTitle?: string
): Crumb[]
```

- `homeTitle` por defecto es `"Inicio"`.
- Cada segmento del `id` se resuelve contra `nodeMap` para obtener su título.

### getImmediateChildren

Devuelve los nodos hijos directos de una carpeta.

```typescript
getImmediateChildren(
  nodeMap: Map<string, NavigationNode>,
  folderPath: string,
  excludeId?: string
): NavigationNode[]
```

- `folderPath = ""` representa la raíz.
- Respeta la regla de carpetas: una subcarpeta solo aparece si tiene un `index.md` con `index: true` (esto ya está resuelto al construir el árbol).
- `excludeId` permite omitir el documento actual (útil para índices).

### getSidebarEntries

Devuelve los hermanos de un documento dentro de su carpeta padre. Es lo que se muestra en el sidebar.

```typescript
getSidebarEntries(
  nodeMap: Map<string, NavigationNode>,
  entryId: string
): NavigationNode[]
```

- Marca el documento actual con `current: true`.
- Si el documento es la raíz (`index`), devuelve un array vacío.

### getParentEntry

Devuelve el nodo padre de un documento dado.

```typescript
getParentEntry(
  nodeMap: Map<string, NavigationNode>,
  entryId: string
): NavigationNode | null
```

- La raíz (`index`) no tiene padre.
- Los documentos de primer nivel tienen como padre el `index` raíz.

## Ejemplo de uso

```typescript
import { AstroCollectionRepository } from "../../document/adapters/repository/astro-collection-repository";
import {
  buildNavigationTree,
  getBreadcrumbs,
  getSidebarEntries,
  getImmediateChildren,
} from "../../navigation/graph/builder";

const repository = new AstroCollectionRepository();
const documents = await repository.list();
const { nodeMap } = buildNavigationTree(documents);

const currentId = "nanobook-project/arquitectura/content-model-architecture";
const breadcrumbs = getBreadcrumbs(nodeMap, currentId);
const sidebar = getSidebarEntries(nodeMap, currentId);
const children = getImmediateChildren(nodeMap, "nanobook-project/arquitectura");
```

## Ordenación de documentos

Los documentos se ordenan usando el campo `position` del frontmatter. Si dos documentos tienen el mismo `position`, se ordenan alfabéticamente por título.

```md
---
title: "Primer documento"
description: "..."
position: 1
---
```

```md
---
title: "Segundo documento"
description: "..."
position: 2
---
```

- `position` menor → aparece primero.
- Sin `position` → se asume `0`.
- Mismo `position` → orden alfabético por `title`.

## Reglas semánticas

- `folder = hierarchy`: la estructura física de carpetas se proyecta en la jerarquía de navegación.
- `index.md` con `index: true` representa una carpeta.
- Los documentos se listan en el índice de su carpeta padre inmediata.
- Los documentos marcados como `draft: true` se excluyen de navegación e índices (filtrados por el repositorio).
- El orden de los documentos se controla mediante `position`.

## Relación con otros módulos

- Recibe `Document[]` desde `ContentRepository`.
- Devuelve `NavigationTree`, `NavigationNode[]` y `Crumb[]` a los componentes de UI (`Layout`, `SidebarNav`, `Breadcrumb`, `IndexList`).
- No depende de Astro ni de ningún storage concreto.
- Los `id` de los documentos, generados por Astro a partir de la ruta del archivo, son la base para relacionar padres e hijos. Ver [Arquitectura del modelo de contenido](./content-model-architecture#flujo-de-carga-y-mapeo).

## Documentación relacionada

- [Arquitectura del modelo de contenido](./content-model-architecture)
- [Storage adapters](./storage-adapters)
