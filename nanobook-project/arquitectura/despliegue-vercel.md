---
title: "Despliegue en Vercel"
description: "Guía de despliegue de Nanobook en Vercel con SSR puro y cache isomórfico"
date: 2026-08-25
author: "Nanobook Team"
tags:
  - arquitectura
  - deploy
  - vercel
  - isr
draft: false
index: false
---

# Despliegue en Vercel

## Requisitos

- Cuenta en Vercel.
- Repo de Nanobook conectado a Vercel.
- Si el contenido vive en un repo separado, un token de GitHub con acceso de lectura.

## Configuración del adapter

El proyecto usa `@astrojs/vercel` cuando la variable `VERCEL_DEPLOY` es `true`. Vercel la puede configurar en el dashboard:

```text
VERCEL_DEPLOY=true
```

Localmente el build usa `@astrojs/node` para evitar problemas de symlinks en Windows.

## Variables de entorno en Vercel

| Variable | Valor ejemplo | Descripción |
|----------|---------------|-------------|
| `VERCEL_DEPLOY` | `true` | Activa el adapter de Vercel. |
| `CONTENT_SOURCE` | `filesystem` / `github` | Fuente de contenido. |
| `GITHUB_OWNER` | `usuario` | Owner del repo de contenido (solo si CONTENT_SOURCE=github). |
| `GITHUB_REPO` | `nanobook-content` | Repo de contenido. |
| `GITHUB_BRANCH` | `main` | Rama del contenido. |
| `GITHUB_TOKEN` | `ghp_...` | Token de GitHub. |
| `GITHUB_PATH` | `src/content` | Ruta base del contenido en el repo. |
| `INVALIDATE_TOKEN` | `secreto` | Token para /api/invalidate. |
| `GITHUB_WEBHOOK_SECRET` | `secreto` | Secreto del webhook de GitHub. |
| `CACHE_BACKEND` | `memory` / `redis` | Cache de cuerpos. En producción se recomienda `redis`. |
| `REDIS_URL` | `redis://...` o `rediss://...` | URL de conexión Redis (requerida si `CACHE_BACKEND=redis`). |

## Cache de cuerpos en producción

En Vercel, `CACHE_BACKEND=memory` es efímero: cada invocación serverless tiene su propia memoria, por lo que el cache no se comparte y se pierde al terminar la request. Para que el cache de cuerpos persista y la invalidación por webhook funcione correctamente, se recomienda usar **Redis**.

Vercel KV está deprecado. La opción recomendada es **Upstash Redis** desde el Vercel Marketplace:

1. Ve a Vercel Dashboard → Marketplace → Redis → Upstash.
2. Crea una base de datos y conecta el proyecto.
3. Vercel inyectará automáticamente `REDIS_URL`, `REDIS_REST_URL`, etc.
4. Configura `CACHE_BACKEND=redis`.

También funciona con cualquier otro proveedor Redis (Redis Cloud, etc.) añadiendo `REDIS_URL` manualmente.

## Webhook de GitHub

En el repo de contenido, configurar un webhook:

- **Payload URL**: `https://tu-dominio.com/api/webhook/github`
- **Content type**: `application/json`
- **Secret**: el mismo valor de `GITHUB_WEBHOOK_SECRET`
- **Eventos**: **Just the push event.**

Cuando se hace push, el webhook invalida automáticamente el cache de cuerpos de los documentos afectados.

## Invalidación manual

Si no se usa webhook, se puede llamar al endpoint manualmente:

```bash
curl -X POST https://tu-dominio.com/api/invalidate \
  -H "Authorization: Bearer $INVALIDATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"changes": [{"id": "blog/post", "kind": "modified", "scope": "content"}]}'
```

## Comportamiento de cache

Las páginas de contenido responden con:

```http
Cache-Control: public, max-age=60, stale-while-revalidate=600
```

Esto significa:

- La CDN de Vercel cachea la página durante 60 segundos.
- Después de esos 60 segundos, sigue sirviendo la versión cacheada mientras regenera en background durante 600 segundos.
- El cache de cuerpos (RenderedPageCache) acelera la regeneración.

## Pruebas locales con `vercel dev`

Para probar el comportamiento de Vercel localmente sin hacer deploy:

### 1. Levantar Redis local con Docker

```bash
docker run -d --name nanobook-redis -p 6379:6379 redis:7-alpine
```

### 2. Configurar variables de entorno

Asegúrate de tener `.env.local` con:

```text
CACHE_BACKEND=redis
REDIS_URL=redis://localhost:6379
CONTENT_SOURCE=github
GITHUB_OWNER=Carlosmgs111
GITHUB_REPO=nanobook-content
GITHUB_BRANCH=main
GITHUB_PATH=
GITHUB_TOKEN=ghp_...
INVALIDATE_TOKEN=...
GITHUB_WEBHOOK_SECRET=...
VERCEL_DEPLOY=true
```

Este archivo está en `.gitignore` y no debe commitearse.

### 3. Iniciar `vercel dev`

```bash
vercel dev --listen 3000 --yes
```

### 4. Verificar cache en Redis

```bash
docker exec nanobook-redis redis-cli keys '*nanobook*'
```

### 5. Probar invalidación manual

```bash
curl -X POST http://localhost:3000/api/invalidate \
  -H "Authorization: Bearer $INVALIDATE_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"changes": [{"id": "index", "kind": "modified", "scope": "content"}]}'
```

### 6. Probar webhook de GitHub

Generar firma HMAC-SHA256 y enviar:

```bash
PAYLOAD='{"ref":"refs/heads/main","commits":[{"added":[],"removed":[],"modified":["markdown.md"]}]}'
SIGNATURE=$(echo -n "$PAYLOAD" | openssl dgst -sha256 -hmac "$GITHUB_WEBHOOK_SECRET" | sed 's/^.* //')
curl -X POST http://localhost:3000/api/webhook/github \
  -H "X-Hub-Signature-256: sha256=$SIGNATURE" \
  -H "Content-Type: application/json" \
  -d "$PAYLOAD"
```

## Limitaciones conocidas

- El build local con `@astrojs/vercel` puede fallar en Windows sin permisos de desarrollador por symlinks. Por eso el adapter de Vercel solo se activa con `VERCEL_DEPLOY=true`.
- `GitHubRepository` no soporta guardar documentos; es de solo lectura.
