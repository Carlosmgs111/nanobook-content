---
title: "Webhook de GitHub: ubicación y dependencias"
description: "Decisión de ubicar el caso de uso del webhook de GitHub en publishing y orquestarlo desde Application, con dependencia directa hacia navigation."
date: 2026-09-11
author: "Nanobook Team"
tags:
  - arquitectura
  - webhook
  - github
  - publishing
  - decision
draft: false
index: false
---

# Webhook de GitHub: ubicación y dependencias

## Contexto

El webhook de GitHub (`POST /api/webhook/github`) es un caso de uso que cruza tres dominios:

1. **Documento**: parsea el payload de GitHub y genera `DocumentChange[]`.
2. **Navegación**: calcula qué documentos dependen de los cambios (`invalidatedIds`).
3. **Publicación**: invalida el caché de páginas renderizadas.

La pregunta arquitectónica fue: ¿en qué dominio vive este caso de uso?

## Alternativas evaluadas

### Alternativa A: `document`

Colocar el webhook en `document` porque recibe cambios de documentos.

**Desventaja**: `document` terminaría dependiendo de `navigation` (para calcular dependencias) y de `publishing` (para invalidar caché). Rompe la separación de dominios.

### Alternativa B: `shared`

Colocar el orquestador en `src/shared`.

**Desventaja**: `shared` debe contener utilidades transversales puras, no lógica de negocio. Convertiría a `shared` en un cajón de sastre.

### Alternativa C: puerto semántico en `document/ports`

Definir un puerto `InvalidatedIdsProvider` en `document/ports/` e implementarlo en `navigation`.

**Ventaja**: desacopla completamente `publishing` de `navigation`. Ambos dependen solo de `document`.

**Desventaja**: añade una capa de indirección que, en el estado actual del proyecto, aumentaba la carga cognitiva sin un beneficio inmediato claro.

### Alternativa D: `publishing` con dependencia directa a `navigation`

Mantener el caso de uso en `publishing/application/GithubWebhookHandler.ts` e inyectarle `NavigationService` directamente. La composición se hace en `Application.ts`.

**Ventaja**: el efecto final del webhook es publicación, así que conceptualmente pertenece a `publishing`. La orquestación queda en `Application`, que ya conoce los tres módulos.

**Desventaja**: crea una dependencia directa `publishing → navigation`.

## Decisión

Aplicar la **Alternativa D** como solución pragmática:

- `GithubWebhookHandler` vive en `publishing` (idealmente en `application/`, no en `infraestructure/`).
- Recibe `RenderedPageCache` y `NavigationService` por constructor.
- `Application.ts` crea el handler con las dependencias de ambos módulos.
- La ruta Astro (`src/pages/api/webhook/github.ts`) solo delega en `app.webhookController.handle`.

```text
src/Application.ts
  ├── PublishingModule.renderedPageCache
  ├── NavigationModule.navigationService
  └── GitHubWebhookHandler(cache, navigationService)
        └── WebhookController(handler)
```

## Justificación

### 1. El efecto final es publicación

El webhook no crea ni edita documentos: invalida caché de páginas publicadas. Eso es responsabilidad de `publishing`.

### 2. La orquestación pertenece a Application

`Application` ya conoce los tres módulos, así que es el lugar natural para conectar `publishing` con `navigation`.

### 3. Pragmatismo

Añadir un puerto `InvalidatedIdsProvider` era más limpio arquitectónicamente, pero en el estado transitorio del proyecto aumentaba la complejidad. Se deja como deuda técnica documentada para cuando el sistema esté más estable.

## Consecuencias

- `publishing/application/GithubWebhookHandler.ts` depende de `navigation`.
- `Application.ts` es el único lugar que ensambla el webhook.
- La ruta Astro es un adaptador HTTP de una sola línea.
- En el futuro se puede extraer un puerto `InvalidatedIdsProvider` para desacoplar sin cambiar la API pública.

## Estado

Aceptada como solución intermedia. El handler se compone en `Application.ts` y se expone como `app.webhookController`.

## Véase también

- [Composición raíz modular](./composicion-raiz-modular)
- [Refactorización del grafo de dependencias](./refactorizacion-grafo-dependencias)
- [Thundering herd contra la API de GitHub](./thundering-herd-github)
