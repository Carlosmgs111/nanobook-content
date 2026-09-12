---
title: "Composición raíz modular"
description: "Decisión de introducir src/Application.ts como composition root y convertir los dominios en módulos con ciclo de vida explícito."
date: 2026-09-11
author: "Nanobook Team"
tags:
  - arquitectura
  - composition-root
  - modularidad
  - decision
draft: false
index: false
---

# Composición raíz modular

## Contexto

Antes de la refactorización, la inicialización de la aplicación estaba dispersa en varios módulos:

- `src/document/index.ts` exportaba `contentRepository` y `documentService` como singletones de módulo.
- `src/publishing/index.ts` creaba `pagePublisher`, `renderedPageCache` e `invalidatePagesController` como constantes globales.
- Las páginas Astro importaban directamente estos singletones y ensamblaban lo que necesitaban.

Ese enfoque funcionaba a corto plazo, pero tenía problemas:

1. **Ciclo de vida implícito**: no quedaba claro cuándo ni en qué orden se creaban las dependencias.
2. **Difícil de testear**: para testear un caso de uso había que mockear imports estáticos o rearrancar el módulo.
3. **Acoplamiento accidental**: las páginas conocían detalles de construcción de repositorios, cachés y servicios.
4. **Riesgo de instancias duplicadas**: si dos partes del sistema creaban el mismo adaptador por separado, podían usar cachés distintos.

## Decisión

Introducir una **composición raíz** en `src/Application.ts` y convertir cada dominio en un **módulo con ciclo de vida explícito**:

- `DocumentModule` (`src/document/index.ts`)
- `NavigationModule` (`src/navigation/index.ts`)
- `PublishingModule` (`src/publishing/index.ts`)

Cada módulo expone un `static async create(...)` que crea sus adaptadores y casos de uso. `Application.create()` instancia los tres módulos en el orden correcto y los conecta.

```text
src/Application.ts
  ├── DocumentModule.create()
  │     ├── ContentRepository
  │     ├── GetAllDocuments
  │     ├── GetDocument
  │     ├── CreateDocument / UpdateDocument
  │     └── Controllers
  ├── PublishingModule.create()
  │     ├── RenderedPageCache
  │     ├── PagePublisher
  │     └── InvalidatePagesController
  └── NavigationModule.create(documents)
        ├── DocumentsGraph
        └── NavigationService
```

Las páginas Astro y los endpoints de API solo importan `getApp()` y usan la aplicación ya compuesta:

```ts
const app = await getApp();
const result = await app.renderPage(slug);
```

## Justificación

### 1. Ciclo de vida explícito

Con `Application.create()` queda claro qué se crea, en qué orden y con qué dependencias. No hay singletones ocultos que se evalúen al importar un módulo.

### 2. Testabilidad

Para testear un caso de uso se puede crear un módulo con dependencias dobles, sin mockear imports estáticos ni depender de variables de entorno.

### 3. Un solo punto de verdad

Todas las rutas obtienen la misma instancia de `Application` a través de `getApp()`, que cachea la promesa. Esto evita que dos rutas creen repositorios o cachés distintos dentro del mismo proceso.

### 4. Límites de dominio claros

- `DocumentModule` expone controllers y queries.
- `NavigationModule` expone `NavigationService`.
- `PublishingModule` expone `pagePublisher`, `renderedPageCache` e `invalidatePagesController`.

`Application` es el único lugar que sabe cómo conectarlos.

## Alternativas evaluadas

### Alternativa A: mantener singletones de módulo

Mantener `export const contentRepository = await createContentRepository(...)` en `src/document/index.ts`.

**Desventaja**: los singletones se evalúan al importar el módulo, lo que dificulta controlar el orden de inicialización y complica los tests.

### Alternativa B: módulo `document` como core

Hacer que `document` sea el centro de la aplicación y que `navigation` y `publishing` importen directamente desde él.

**Desventaja**: convertiría a `document` en un cajón de sastre y acoplaría los tres dominios. Va en contra del principio de evitar `core` como aglutinador.

### Alternativa C: orquestadores en `shared`

Colocar el webhook u otros orquestadores en `src/shared`.

**Desventaja**: `shared` debe contener utilidades transversales puras, no lógica de negocio ni orquestación entre dominios.

## Consecuencias

- `src/Application.ts` es el único lugar que conoce las tres capas.
- Los módulos de dominio no dependen entre sí; solo se conectan en `Application`.
- Las rutas Astro son adaptadores delgados que delegan en `Application`.
- Es fácil reemplazar una implementación (por ejemplo, cambiar el backend de caché) sin tocar los dominios.

## Estado

Aceptada. La estructura actual sigue este modelo: `Application.create()` ensambla `DocumentModule`, `PublishingModule` y `NavigationModule`.

## Véase también

- [Refactorización del grafo de dependencias](./refactorizacion-grafo-dependencias)
- [Webhook de GitHub: ubicación y dependencias](./webhook-github-arquitectura)
- [Thundering herd contra la API de GitHub](./thundering-herd-github)
