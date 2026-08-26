---
title: "Plan de transición a SSR puro"
description: "Plan por fases para eliminar el modo híbrido estático/dinámico y consolidar Nanobook como aplicación server-first"
date: 2026-08-25
author: "Nanobook Team"
tags:
  - arquitectura
  - ssr
  - roadmap
  - refactor
draft: false
index: false
---

# Plan de transición a SSR puro

## Estado de ánimo

El proyecto comenzó como un sitio estático con Astro Content Collections y luego creció hacia un modelo dinámico con `ContentRepository`, SSR, cache isomórfico y webhooks. Hoy tenemos dos sistemas de carga de contenido conviviendo:

1. **Astro Content Loader** (`github-loader` en `content.config.ts`) — usado en build/dev.
2. **`ContentRepository`** (`GitHubRepository`, `FileSystemRepository`) — usado en runtime.

Esta duplicación genera:
- Doble fetch de GitHub (loader + repository).
- Bifurcaciones `OUTPUT_MODE` / `CONTENT_SOURCE` difíciles de razonar.
- Dependencias de `astro:content` en el dominio puro del documento.
- Build local de Vercel que falla por symlinks y `vercel dev` que carga el loader innecesariamente.

La transición a **SSR puro** consolida el proyecto como aplicación server-first: una sola fuente de verdad (`ContentRepository`), sin loaders de Astro en runtime, y despliegue en Vercel/Node/serverless.

## Principios rectores

1. **No código nuevo innecesario**: solo mover, eliminar o simplificar.
2. **Una sola fuente de verdad**: `ContentRepository` es el único punto de lectura/escritura de documentos.
3. **Astro Content Collections como detalle interno**: `content.config.ts` seguirá existiendo para los archivos locales de `src/content/`, pero no cargará contenido de GitHub.
4. **Mínima ruptura**: cada fase debe dejar tests y build verdes.
5. **Documentación actualizada**: cada decisión se refleja en los documentos del proyecto.

## Fases

### ✅ Fase 1 — Preparación y auditoría

**Objetivo**: tener una foto clara de todo lo que depende del modo híbrido.

**Tareas**:
- Listar todos los archivos que importan `astro:content`.
- Listar todos los usos de `OUTPUT_MODE`, `outputMode`, `isDynamicMode`, `isStaticMode`.
- Identificar páginas Astro que usen `getCollection`, `AstroCollectionRepository`, `astro-cache`.
- Revisar documentación obsoleta que mencione `OUTPUT_MODE` o modos estático/dinámico.
- Ejecutar tests y build como línea base.

**Criterios de éxito**:
- Inventario completo de dependencias.
- Tests y build pasan antes de tocar código.

**Inventario confirmado (línea base)**: ver historial del commit `033bca9`.

### ✅ Fase 2 — Eliminar `OUTPUT_MODE`

**Objetivo**: quitar la variable `OUTPUT_MODE` y asumir `output: "server"` siempre.

**Tareas**:
- En `astro.config.mjs`: eliminar cualquier lógica condicional basada en `OUTPUT_MODE`.
- En `content.config.ts`: eliminar `outputMode`, `isDynamicMode`, `dynamicLoader`.
- En `src/document/api/document.ts`: cambiar `prerender` a `false` directamente.
- Revisar scripts (`package.json`) y variables de entorno para eliminar `OUTPUT_MODE`.
- Actualizar `.env` y `.env.local` de ejemplo.

**Criterios de éxito**:
- No quedan referencias a `OUTPUT_MODE` en código fuente.
- `pnpm build` y `pnpm test` pasan.

### ✅ Fase 3 — Simplificar `content.config.ts`

**Objetivo**: que `content.config.ts` solo gestione la colección local de `src/content/`, sin fetch de GitHub.

**Tareas realizadas**:
- `content.config.ts` usa solo un loader `glob` sobre `./src/content/`.
- Eliminado el loader `github` de Astro.
- `astro dev` y `astro build` ya no hacen fetch a GitHub por el loader.

### ✅ Fase 4 — Migrar dependencias de `astro:content`

**Objetivo**: eliminar imports de `astro:content` del dominio del documento.

**Tareas realizadas**:
- Eliminados `astro-collection-repository.ts` y `astro-cache.ts`.
- Eliminado `toDocument` basado en `CollectionEntry` de `src/document/parse/document.ts`.
- `src/document/parse/proxy.ts` trabaja con `Document` y exporta `createReferenceResolver` y `resolveProxies`.
- `src/document/reference/types.ts`, `resolver.ts` e `internal-resolver.ts` usan `Document` en lugar de `CollectionEntry`.
- `factory.ts` ya no ofrece la opción `astro`; el default es `filesystem`.
- `FileSystemRepository` y `GitHubRepository` aplican `resolveProxies()` en `list()`.

### ✅ Fase 5 — Limpiar páginas Astro

**Objetivo**: asegurar que todas las páginas usan `createContentRepository()` y no `getCollection` ni `AstroCollectionRepository`.

**Tareas realizadas**:
- Revisadas todas las páginas en `src/pages/`.
- Ninguna usa `getCollection` ni `CollectionEntry`.
- `src/document/api/document.ts` se mantiene como endpoint PATCH con `prerender = false`.

### ✅ Fase 6 — Actualizar documentación

**Tareas realizadas**:
- Actualizado `content-model-architecture.md`.
- Actualizado `informe-factibilidad-recompilado-incremental.md`.
- Actualizado `recompilado-incremental-estado-actual.md`.
- Actualizado `plan-fase-7-isr-modo-dinamico.md`.
- Actualizado `despliegue-vercel.md`.
- Actualizado `AGENTS.md`.
- Actualizado `CHANGELOG.md`.
- Actualizado `README.md`.

### ✅ Fase 7 — Verificación final

**Tareas realizadas**:
- `pnpm test`: 78 passed.
- `pnpm build`: OK.
- `vercel dev` con `CONTENT_SOURCE=filesystem`: OK (HTTP 200 en `/`).
- `vercel dev` con `CONTENT_SOURCE=github`: OK (HTTP 200 en `/`).
- Eliminados imports muertos (`stringify`, `DocumentMetadata`, `Document` en `parse/document.ts`).
- Versión actualizada a `0.4.0` en `package.json` y `CHANGELOG.md`.

**Criterios de éxito**:
- 0 imports de `astro:content` fuera de `content.config.ts`.
- 0 referencias a `OUTPUT_MODE` en código fuente.
- Tests y build verdes.
- `vercel dev` funciona en ambos modos de contenido.

## Riesgos y mitigaciones

| Riesgo | Estado | Mitigación |
|--------|--------|------------|
| Se rompe resolución de proxies | Resuelto | Migrada a `parse/proxy.ts` con `Document`; `FileSystemRepository` y `GitHubRepository` la aplican en `list()`. |
| Se pierde funcionalidad de edición | Resuelto | `FileSystemRepository` sigue disponible para `CONTENT_SOURCE=local`. |
| Build de Vercel en Windows | Vigente | El build de Vercel requiere symlinks; usar `VERCEL_DEPLOY=true` solo en CI/Linux o Vercel. |
| `src/content/` deja de funcionar | Resuelto | `content.config.ts` sigue cargando archivos locales de `src/content/`. |

## Próximos pasos

1. Completar Fase 6 (documentación).
2. Ejecutar Fase 7 (verificación final).
3. Decidir si se publica como `v0.4.0`.
