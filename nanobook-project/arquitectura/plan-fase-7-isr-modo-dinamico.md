---
title: "Plan de implementación — Fase 7: ISR isomórfico con cache abstracto"
description: "Plan de implementación — Fase 7: ISR isomórfico con cache abstracto"
date: 2026-08-23
author: "Nanobook Team"
tags:
  - arquitectura
  - build
  - incremental
  - refactoring
  - roadmap
  - decision
  - isr
  - cache
draft: false
index: false
---

# Plan de implementación — Fase 7: ISR isomórfico con cache abstracto

## Objetivo

Completar la integración del recompilado incremental en el flujo de ejecución real del proyecto de forma isomórfica: el mismo código debe poder desplegarse en Vercel, Netlify, Cloudflare, Node o cualquier entorno serverless con mínimos cambios de configuración.

## Lección aprendida del debate

El primer borrador de esta fase proponía usar ISR nativo de Vercel como mecanismo principal. Se descartó porque crearía una arquitectura no portable: revalidate() solo existe en Vercel, FileSystemRenderedPageCache no persiste en serverless, y cambiar de plataforma requeriría reescribir lógica.

La decisión final es: el cache de cuerpos propio es el mecanismo central, con backends intercambiables. El ISR de plataforma y las cabeceras Cache-Control son una capa opcional de aceleración, no un requisito.

## Principios rectores

1. Isomorfismo: el mismo código funciona en cualquier entorno serverless o tradicional.
2. Contrato central: RenderedPageCache define como se cachean los cuerpos renderizados.
3. Backends intercambiables: filesystem, memoria, Redis, KV sin cambiar el resto del código.
4. Cache de página completa mediante cabeceras HTTP estándar (stale-while-revalidate).
5. Invalidación genérica: endpoint /api/invalidate limpia el cache de cuerpos sin depender de APIs propietarias.

## Arquitectura objetivo

```text
Página Astro SSR (prerender = false)
  └─ Cache-Control: stale-while-revalidate
  └─ PageRenderer
        └─ RenderedPageCache (interfaz)
              ├─ FileSystemRenderedPageCache (local, Node, CI)
              ├─ MemoryRenderedPageCache (serverless efimero)
              └─ RedisRenderedPageCache / KVRenderedPageCache (persistente)
```

## Sub-fases de implementación

### Fase 7.1 — Configurar modo server canónico

Actualizar astro.config.mjs a output: "server". El adapter se selecciona mediante la variable de entorno VERCEL_DEPLOY:

- Local/Node: @astrojs/node con mode standalone (por defecto).
- Vercel: @astrojs/vercel cuando VERCEL_DEPLOY=true.

Esto mantiene el build local funcional en entornos donde Vercel no puede crear symlinks (Windows sin permisos de desarrollador), mientras que el despliegue en Vercel usa su adapter nativo.

> Nota: `OUTPUT_MODE` fue eliminado posteriormente; `output: "server"` es el único modo. Ver [plan de transición a SSR puro](../plan-transicion-ssr-puro).

### Fase 7.2 — Refactorizar RenderedPageCache como interfaz

Definir el contrato RenderedPageCache con get, set e invalidate. Mover FileSystemRenderedPageCache a ser una implementación del contrato. Crear MemoryRenderedPageCache para entornos serverless donde el filesystem no persiste.

### Fase 7.3 — Factory de ContentRepository

Crear src/document/adapters/repository/factory.ts que devuelva FileSystemRepository o GitHubRepository según CONTENT_SOURCE. Reemplazar instanciaciones directas de AstroCollectionRepository en páginas Astro.

> Nota: `AstroCollectionRepository` fue eliminado en la transición a SSR puro; `ContentRepository` es ahora la única fuente de verdad.

### Fase 7.4 — Factory de RenderedPageCache

Crear src/rendering/adapters/cache/factory.ts que seleccione la implementación según el entorno:

- local/dev → FileSystemRenderedPageCache
- serverless sin Redis → MemoryRenderedPageCache
- con Redis → RedisRenderedPageCache (funciona en Vercel con Upstash Redis, Netlify, Cloudflare, Node, etc.)

### Fase 7.5 — GitHubRepository

Crear src/document/adapters/repository/github-repository.ts que implemente ContentRepository usando la API de GitHub. Reutilizar github-loader/api.ts y parser.ts. Implementar list, get y listChildren. Cachear lista de documentos en memoria con TTL.

### Fase 7.6 — Prerender condicional

Marcar prerender false en src/pages/[...slug]/index.astro, edit.astro y preview.astro. Dado que el home comparte la ruta dinámica [...slug]/index.astro, también se sirve bajo demanda. En una fase posterior se puede separar el home en src/pages/index.astro si se necesita que sea estático.

### Fase 7.7 — SSR de páginas de contenido

Refactorizar src/pages/[...slug]/index.astro para obtener slug de Astro.params, crear ContentRepository y RenderedPageCache mediante factories, obtener documento por ID, usar PageRenderer para cuerpo y navegación, y pasar todo a Layout y DocumentPage. Inyectar cabecera Cache-Control: public, max-age=60, stale-while-revalidate=600.

### Fase 7.8 — Endpoint de invalidación genérico

Crear src/pages/api/invalidate.ts que reciba un ChangeSet, calcule invalidatedIds con ContentChangeService, construya las claves de cache y llame a RenderedPageCache.invalidate(). No depende de revalidate() de Vercel. Requerir INVALIDATE_TOKEN en header Authorization.

### Fase 7.9 — Webhook de GitHub

Crear src/pages/api/webhook/github.ts que verifique firma con GITHUB_WEBHOOK_SECRET, parse el payload de push, mapee archivos a IDs y llame al endpoint de invalidación internamente.

### Fase 7.10 — Adaptadores de despliegue

Configurar el adapter específico según la plataforma elegida. Implementado para Vercel y Node:

- Vercel: @astrojs/vercel con output server. Activar con VERCEL_DEPLOY=true.
- Node propio: @astrojs/node con mode standalone (por defecto).

La lógica de negocio no cambia entre plataformas. Para Netlify o Cloudflare bastaría con instalar su adapter y añadir una rama más en el selector de astro.config.mjs.

## Variables de entorno

- VERCEL_DEPLOY: true para usar @astrojs/vercel, omitir para @astrojs/node
- CONTENT_SOURCE: filesystem o github
- GITHUB_OWNER, GITHUB_REPO, GITHUB_BRANCH, GITHUB_TOKEN
- GITHUB_PATH: ruta base del contenido en el repo de GitHub
- INVALIDATE_TOKEN
- GITHUB_WEBHOOK_SECRET
- CACHE_BACKEND: filesystem | memory | redis
- REDIS_URL: URL de conexión Redis (requerida cuando CACHE_BACKEND=redis)

## Criterios de éxito

- output server es el modo canónico.
- RenderedPageCache es una abstracción con múltiples implementaciones.
- Las páginas de contenido se sirven bajo demanda con Cache-Control stale-while-revalidate.
- El endpoint /api/invalidate funciona sin APIs propietarias.
- pnpm test y pnpm build pasan en modo server.
- Migrar de Vercel a Netlify, Cloudflare o Node solo requiere cambiar adapter y CACHE_BACKEND.
