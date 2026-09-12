---
title: "Thundering herd contra la API de GitHub"
description: "Análisis del error 503 Backend.max_conn reached tras la refactorización modular y la solución mediante in-flight promises en GitHubRepository."
date: 2026-09-11
author: "Nanobook Team"
tags:
  - arquitectura
  - github
  - cache
  - performance
  - bugfix
draft: false
index: false
---

# Thundering herd contra la API de GitHub

## Síntoma

Tras la refactorización a composición raíz modular, al usar `CONTENT_SOURCE=github` apareció el error:

```text
503 Backend.max_conn reached
```

Esto ocurría al renderizar múltiples páginas concurrentemente.

## Diagnóstico

### Cómo renderiza Astro

Astro ejecuta el frontmatter de las páginas en **paralelo**, tanto en `astro build` como en `astro dev` bajo carga. Si hay 20 páginas pendientes, Astro puede iniciar 20 renders simultáneos.

### Qué hace cada página

Cada página importa `getApp()`:

```ts
const app = await getApp();
const result = await app.renderPage(slug);
```

`renderPage` necesita los documentos, así que llega a:

```ts
githubRepository.list()
```

### El problema: falta de deduplicación

Antes del arreglo, `GitHubRepository.list()` se veía así:

```ts
async list(): Promise<Document[]> {
  const cached = globalCache.get(this.cacheKey);
  if (cached && cached.expiresAt > now) {
    return cached.documents;
  }

  const documents = await this.fetchDocuments(); // cada llamada hace su propio fetch
  globalCache.set(this.cacheKey, { documents, expiresAt: ... });
  return documents;
}
```

Si 20 páginas llamaban `list()` al mismo tiempo y el caché aún estaba vacío:

- Cada una iniciaba `fetchDocuments()`.
- Cada `fetchDocuments()` llamaba `fetchTree()` (una llamada a GitHub API).
- Cada `fetchDocuments()` llamaba `fetchFileContent()` para cada archivo.

Con 20 páginas y 30 archivos, se generaban cientos de llamadas concurrentes a GitHub, superando el límite de conexiones salientes.

## ¿Por qué antes no pasaba?

Antes de la refactorización, `src/document/index.ts` exportaba el repositorio como un **singleton de módulo**:

```ts
export const contentRepository = await createContentRepository(...);
```

Eso tenía dos efectos:

1. **Una sola instancia por worker**: todas las páginas dentro del mismo proceso compartían el mismo `GitHubRepository`.
2. **Carga más temprana**: el singleton se evaluaba al importar el módulo, probablemente antes de que Astro empezara a renderizar páginas en paralelo. Para cuando las páginas llamaban `list()`, el caché ya estaba poblado.

La refactorización cambió el momento de la carga. Ahora `Application.create()` se ejecuta desde el frontmatter de la primera página, coincidiendo con el renderizado paralelo. Eso expuso un bug latente: `list()` no deduplicaba llamadas concurrentes.

## Solución

Agregar **in-flight promises** en `GitHubRepository`:

```ts
private listPromise: Promise<Document[]> | null = null;

async list(): Promise<Document[]> {
  const cached = globalCache.get(this.cacheKey);
  if (cached && cached.expiresAt > now) {
    return cached.documents;
  }

  if (!this.listPromise) {
    this.listPromise = this.fetchDocuments()
      .then((documents) => {
        globalCache.set(this.cacheKey, {
          documents,
          expiresAt: Date.now() + this.cacheTtl,
        });
        return documents;
      })
      .finally(() => {
        this.listPromise = null;
      });
  }

  return this.listPromise;
}
```

Se aplicó el mismo patrón a `fetchTree()`.

### Resultado

Ahora, dentro de un mismo proceso/worker:

- La primera página que llama `list()` inicia la carga.
- Las páginas concurrentes esperan la misma promesa.
- Cuando la carga termina, todas reciben el mismo resultado.

De 20 cargas simultáneas se pasa a 1.

## Limitaciones

La deduplicación in-flight funciona **dentro de un mismo proceso/worker**. Si Astro crea muchos workers separados, cada uno hará su propia carga. En ese caso habría que agregar un caché compartido entre workers (por ejemplo, en filesystem o Redis) durante el build.

Por ahora, el in-flight redujo la presión suficientemente como para eliminar el error.

## Estado

Resuelto. El patrón in-flight está implementado en `src/document/infraestructure/repository/GitHubRepository.ts`.

## Véase también

- [Composición raíz modular](./composicion-raiz-modular)
- [Refactorización del grafo de dependencias](./refactorizacion-grafo-dependencias)
- [Webhook de GitHub: ubicación y dependencias](./webhook-github-arquitectura)
