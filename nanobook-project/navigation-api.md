---
title: API de navegación
description: "Punto de entrada a la documentación de NavigationBuilder:
  construcción del árbol de navegación, breadcrumbs, sidebar e índices."
date: 2026-08-23T00:00:00.000Z
author: Nanobook Team
tags:
  - arquitectura
  - navegación
  - api
  - navigation-builder
draft: false
index: false
position: 0
---

La documentación completa de la API de navegación vive en:

## [→ API de navegación](../arquitectura/api-de-navegacion)

Allí se describe:

- La ubicación de `NavigationBuilder` (`src/navigation/graph/builder.ts`).
- Los tipos `NavigationNode`, `NavigationTree` y `Crumb`.
- Las funciones `buildNavigationTree`, `getBreadcrumbs`, `getSidebarEntries`, `getImmediateChildren` y `getParentEntry`.
- Por qué la construcción del árbol no vive en el repositorio de contenido.
- Cómo se relaciona con la [arquitectura del modelo de contenido](./arquitectura/content-model-architecture) y los [storage adapters](./arquitectura/storage-adapters).