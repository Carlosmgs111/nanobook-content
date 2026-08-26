---
title: "Arquitectura"
description: "Decisiones y propuestas arquitectónicas del proyecto Nanobook."
date: 2026-08-23
author: "Nanobook"
tags: ["nanobook", "arquitectura", "decisiones"]
draft: false
index: true
---

## Decisiones arquitectónicas

- [Arquitectura del modelo de contenido](./content-model-architecture): cómo se carga el contenido, se mapea a `Document` y se separa de la capa de navegación.
- [API de navegación](./api-de-navegacion): construcción del árbol de navegación, breadcrumbs, sidebar e índices.
- [Storage adapters](./storage-adapters): adapters disponibles para `ContentRepository` y cómo añadir uno nuevo.
- [Documentos proxy](./proxy-documents): sistema de referencias `ref` para alias, READMEs y contenido de proyectos ajenos.
- [Arquitectura basada en IDs](./id-based-component-architecture): decisiones sobre identificadores y referencias entre documentos.
- [Informe de estado: IDs y referencias](./informe-estado-ids-referencias): estado actual de los identificadores y referencias del proyecto.
