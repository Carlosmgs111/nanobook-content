---
title: "Sistema de cacheo"
description: "Flujo de ejecucion y capas de cacheo de Nanobook: CDN, Redis, memoria y GitHub."
date: 2026-08-26
author: "Nanobook Team"
tags:
  - arquitectura
  - cache
  - redis
  - performance
  - deploy
draft: false
index: false
position: 0
---

# Sistema de cacheo

Nanobook usa cuatro capas de cacheo superpuestas. Cada una resuelve un problema diferente y tiene su propio tiempo de vida, clave y mecanismo de invalidacion. Entender el flujo completo permite diagnosticar por que un cambio tarda en verse o por que una peticion falla con rate limit de GitHub.

## Capas de cacheo

```text
Navegador / CDN de Vercel
        │
        │ Cache-Control: public, max-age=60, stale-while-revalidate=600
        │
        ▼
Servidor Astro SSR
        │
        ├─► RenderedPageCache (Redis / memory / filesystem)
        │      Clave: nanobook:page:{pageId}
        │
        ├─► GitHubRepository tree cache (Redis)
        │      Clave: nanobook:github:tree:{...}
        │
        └─► GitHubRepository document cache (memoria de la funcion)
               Map<string, GitHubRepositoryCache>
```

## 1. CDN / HTTP Cache-Control

**Archivo**: `src/pages/[...slug]/index.astro` (lineas 32-35).

```astro
Astro.response.headers.set(
  "Cache-Control",
  "public, max-age=60, stale-while-revalidate=600"
);
```

**Comportamiento**:

- `max-age=60`: durante 60 segundos, la CDN de Vercel sirve la pagina **sin consultar el servidor**.
- `stale-while-revalidate=600`: entre 60 y 600 segundos, la CDN sigue sirviendo la version cacheada mientras pide una version fresca al servidor en segundo plano. La siguiente peticion recibe la version fresca.
- Despues de 600 segundos, la CDN siempre consulta el servidor.

**Invalidacion**: no hay invalidacion manual de la CDN. Los headers controlan el tiempo. Para forzar una peticion fresca desde el navegador se puede usar `curl -H "Cache-Control: no-cache"` o esperar a que pase el tiempo.

## 2. RenderedPageCache (cuerpos renderizados)

**Contrato**: `src/rendering/model/types.ts`.

**Implementaciones**:
- `MemoryRenderedPageCache` — `src/rendering/adapters/cache/memory-page-cache.ts`
- `FileSystemRenderedPageCache` — `src/rendering/adapters/cache/file-system-page-cache.ts`
- `RedisRenderedPageCache` — `src/rendering/adapters/cache/redis-page-cache.ts`

**Seleccion**: `src/rendering/adapters/cache/factory.ts` segun `CACHE_BACKEND`.

**Clave en Redis**:
```text
nanobook:page:{pageId}
```

**Valor almacenado**:
```typescript
{
  pageId: string;
  contentHash: string;
  html: string;
  renderedAt: string; // ISO 8601
}
```

**TTL**: actualmente no tiene TTL. El cuerpo se mantiene en Redis hasta que se invalide explicitamente o el `contentHash` cambie (en cuyo caso se ignora y se vuelve a renderizar).

**Invalidacion**:
- Endpoint `POST /api/invalidate` con `Authorization: Bearer <INVALIDATE_TOKEN>`.
- Webhook `POST /api/webhook/github` tras un push en el repo de contenido.
- Ambos usan `ContentChangeService` para calcular los `pageId` afectados y luego llaman `RenderedPageCache.invalidate(pageIds)`.

## 3. GitHubRepository tree cache (Redis)

**Archivo**: `src/document/adapters/repository/github-repository.ts`.

**Proposito**: evitar llamar a la API de GitHub (`GET /repos/{owner}/{repo}/git/trees/{branch}?recursive=1`) en cada request. El tree es la lista recursiva de archivos del repo de contenido.

**Clave en Redis**:
```text
nanobook:github:tree:{owner}:{repo}:{branch}:{path}:{pattern}
```

**Mecanismo**: Una funcion auxiliar compartida, `withRedisClient` que invoca una instancia del cliente `redis`, ubicada en:
```text
src/shared/utils/redis.ts
```

**TTL**: 5 minutos (300.000 ms), configurable por `treeCacheTtl`.

**Flujo**:
1. `GitHubRepository.list()` llama a `fetchDocuments()`.
2. `fetchDocuments()` llama a `fetchTree()`.
3. `fetchTree()` primero consulta Redis.
   - Si existe: lo devuelve.
   - Si no existe: llama a `fetchGitHubTree()`, guarda el resultado en Redis y lo devuelve.

**Invalidacion**: expira por TTL. No hay invalidacion manual en `v0.4.2`.

## 4. GitHubRepository document cache (memoria)

**Archivo**: `src/document/adapters/repository/github-repository.ts`.

**Proposito**: evitar reconstruir el array de `Document[]` dentro de la misma invocacion serverless. El `Map` `globalCache` vive en el modulo y se comparte entre instancias del mismo proceso.

**Clave**:
```text
{owner}:{repo}:{branch}:{path}:{pattern}
```

**TTL**: 5 minutos (300.000 ms), configurable por `cacheTtl`. Antes de `v0.4.2` era 60 segundos.

**Limitacion**: en entornos serverless como Vercel, cada invocacion puede ser un proceso diferente. Ese cache no se comparte entre invocaciones, pero si se comparte Redis (capas 2 y 3).

## Flujos de ejecucion

### Flujo A: primera visita a una pagina

```text
Usuario -> GET /markdown

1. CDN de Vercel no tiene cache (pasaron mas de 600s o es la primera vez).
   -> Pasa la peticion al servidor.

2. Servidor Astro ejecuta src/pages/[...slug]/index.astro.

3. renderDocumentPage("markdown") en src/rendering/page/render-document-page.ts.

4. Crea ContentRepository via createContentRepository():
   -> Como CONTENT_SOURCE=github, devuelve GitHubRepository.

5. GitHubRepository.get("markdown") llama a list().

6. list() no encuentra cache en memoria.

7. fetchDocuments() -> fetchTree() consulta Redis:
   -> Tree no existe.
   -> fetchGitHubTree() llama a api.github.com/repos/.../git/trees/... (1 llamada API).
   -> Guarda el tree en Redis con TTL 5 min.

8. filterContentFiles(tree) aplica el pattern y excluye README.md.
   -> Quedan ~N archivos Markdown.

9. Para cada archivo:
   -> fetchFileContent() descarga desde raw.githubusercontent.com (sin rate limit de API).
   -> createParsedEntry() parsea frontmatter.
   -> buildDocument() construye Document.

10. resolveProxies(documents) resuelve referencias ref si las hay.

11. GitHubRepository guarda Document[] en memoria (TTL 5 min).

12. PageRenderer.render("markdown", contentHash):
    -> Consulta RenderedPageCache (Redis):
       -> No existe.
    -> Renderiza Markdown a HTML.
    -> Guarda el cuerpo en Redis.

13. NavigationService construye breadcrumb, sidebar, etc. Usa cache WeakMap interno.

14. Servidor responde con HTML + header Cache-Control.

15. CDN de Vercel cachea la respuesta por 60s.
```

### Flujo B: recarga dentro de 60 segundos

```text
Usuario -> GET /markdown (segunda vez, < 60s)

1. CDN tiene la respuesta en cache (max-age=60).
   -> Sirve directamente. No toca servidor, Redis ni GitHub.
```

### Flujo C: recarga entre 60 y 600 segundos

```text
Usuario -> GET /markdown (60s < t < 600s)

1. CDN sirve la version cacheada inmediatamente.
2. En paralelo, la CDN envia una peticion de revalidacion al servidor.
3. El servidor procesa la peticion:
   -> GitHubRepository.list() usa Document[] de memoria (aun dentro de 5 min).
   -> RenderedPageCache (Redis) tiene el cuerpo con el mismo contentHash.
   -> No renderiza de nuevo.
4. La CDN actualiza su cache con la version fresca (que probablemente es igual).
```

### Flujo D: recarga despues de 600 segundos sin cambios de contenido

```text
Usuario -> GET /markdown (t > 600s, contenido sin cambiar)

1. CDN no sirve cache. Pasa al servidor.

2. Servidor ejecuta src/pages/[...slug]/index.astro.

3. GitHubRepository.list():
   -> Memoria: Document[] aun valido (dentro de 5 min).
   -> O si expiro, consulta Redis por el tree:
      -> Tree existe (dentro de 5 min).
      -> No llama a la API de GitHub.
   -> Descarga archivos desde raw.githubusercontent.com.

4. contentHash del documento no cambio.

5. PageRenderer.render() consulta Redis:
   -> Encuentra el cuerpo con el mismo contentHash.
   -> Devuelve el HTML cacheado. No renderiza.

6. Servidor responde. CDN cachea por 60s.
```

### Flujo E: cambio de contenido con invalidacion manual

```text
Paso 1: editor modifica markdown.md en nanobook-content y hace push.

Paso 2: llamada manual a /api/invalidate:

POST /api/invalidate
Authorization: Bearer <INVALIDATE_TOKEN>
Content-Type: application/json

{"changes": [{"id": "markdown", "kind": "modified", "scope": "content"}]}

Flujo del endpoint:
1. invalidate-handler.ts valida el token.
2. Construye DocumentChange[] y llama a ContentChangeService.invalidate().
3. ContentChangeService calcula los ids afectados:
   -> "markdown" + sus dependientes (padres, hijos, links internos, etc.).
4. Llama a RenderedPageCache.invalidate([...pageIds]).
5. Redis borra las claves nanobook:page:{pageId} afectadas.
6. GitHubRepository tree cache en Redis NO se borra.
7. Memoria de GitHubRepository NO se borra.

Paso 3: usuario recarga /markdown.

1. CDN puede servir cache si aun esta dentro de 60-600s.
   -> Para ver el cambio inmediato, usar modo incognito o curl con no-cache.

2. Servidor procesa la peticion.

3. GitHubRepository.list() usa Document[] de memoria (puede tener el contenido antiguo).
   -> Pero el archivo markdown.md se descarga de nuevo desde raw.githubusercontent.com.
   -> El contentHash sera diferente.

4. PageRenderer.render("markdown", nuevoContentHash):
   -> Redis no tiene cuerpo con ese hash.
   -> Renderiza de nuevo con el contenido actualizado.
   -> Guarda en Redis.

5. Usuario ve el cambio.
```

### Flujo F: cambio de contenido sin invalidacion

```text
Paso 1: editor modifica markdown.md y hace push.
Paso 2: no llama a /api/invalidate ni webhook.
Paso 3: usuario recarga /markdown.

1. Si la CDN aun tiene cache (< 600s), sirve la version antigua.
2. Si la CDN revalida o expira:
   -> Servidor descarga el archivo nuevo de raw.githubusercontent.com.
   -> contentHash cambia.
   -> RenderedPageCache no tiene cuerpo con ese hash.
   -> Renderiza y guarda en Redis.
   -> Usuario ve el cambio.
```

Conclusion: para cambios de contenido, la invalidacion no es estrictamente necesaria porque el contentHash detecta el cambio. Pero acelera el proceso y evita que la CDN sirva la version antigua durante los primeros 60s.

### Flujo G: cambio de estructura (nuevo documento)

```text
Paso 1: se crea blog/nuevo.md en nanobook-content y se hace push.
Paso 2: se llama a /api/invalidate con el id del padre o del nuevo documento.

Paso 3: usuario visita /blog/nuevo.

Problema:
- GitHubRepository tree cache en Redis aun tiene el tree antiguo (hasta 5 min).
- Memoria de GitHubRepository tambien tiene Document[] antiguo (hasta 5 min).
- El nuevo documento no aparece en ninguno de esos caches.

Resultado:
- /blog/nuevo devuelve 404 hasta que expire algun cache (maximo 5 min).
- Si se fuerza un fetch del tree (por ejemplo, reiniciando la funcion), el nuevo documento aparece.
```

**Recomendacion operativa**: para cambios de estructura, esperar 5 minutos o invalidar el tree manualmente. En versiones futuras se puede anadir invalidacion del tree al endpoint `/api/invalidate`.

### Flujo H: deploy nuevo en Vercel

```text
1. Vercel construye la aplicacion.
2. No hay memoria de ejecuciones anteriores.
3. No hay cache de Redis a menos que ya exista de deploys previos.
4. Primer request:
   -> fetchTree() no encuentra tree en Redis.
   -> Llama a fetchGitHubTree() (1 llamada API).
   -> Guarda tree en Redis.
   -> Descarga archivos desde raw.githubusercontent.com.
   -> Renderiza y guarda cuerpos en Redis.
```

Si el rate limit de GitHub API esta agotado en el momento del deploy, el primer request fallara hasta que se resetee la cuota (cada hora).

## Variables de entorno relevantes

| Variable | Efecto |
|----------|--------|
| `CACHE_BACKEND` | Selecciona el backend de RenderedPageCache: `memory`, `filesystem`, `redis`. |
| `REDIS_URL` | URL de conexion Redis para `RedisRenderedPageCache` y tree cache de GitHubRepository. |
| `CONTENT_SOURCE` | `filesystem` o `github`. La capa 3 y 4 solo aplican con `github`. |
| `GITHUB_TOKEN` | Token para autenticar la llamada al tree. Sin el, el rate limit es 60 requests/hora. |

## Limitaciones conocidas en v0.4.2

1. **Tree cache no se invalida manualmente**: expira solo por TTL (5 min).
2. **Memoria de GitHubRepository no se invalida**: expira solo por TTL (5 min).
3. **RenderedPageCache no tiene TTL**: crece indefinidamente en Redis hasta invalidacion explicita o cambio de contentHash.
4. **CDN no se invalida manualmente**: depende de Cache-Control.
5. **Primer request tras deploy siempre llama a la API de GitHub para el tree**: si la cuota esta agotada, falla.

## Proximas mejoras posibles

- Extender `/api/invalidate` para tambien borrar el tree cache de GitHub en Redis.
- Anadir TTL al RenderedPageCache para evitar crecimiento infinito.
- Generar un `manifest.json` estatico en el repo de contenido para eliminar la llamada al tree de la API de GitHub.
