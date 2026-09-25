---
id: "eb9f6c0c-a4eb-4bcd-95dd-c7787e2e2703"
title: "Arquitectura"
description: "Decisiones y propuestas arquitectónicas del proyecto Nanobook."
date: 2026-08-23
author: "Nanobook"
tags: ["nanobook", "arquitectura", "decisiones"]
draft: false
index: true
---

## Decisiones recientes

- [Composición raíz modular](./composicion-raiz-modular): por qué se introdujo `src/Application.ts` y los módulos `DocumentModule`, `NavigationModule` y `PublishingModule`.
- [Refactorización del grafo de dependencias](./refactorizacion-grafo-dependencias): por qué `DocumentsGraph` se movió a `navigation` y cómo se calculan los `invalidatedIds`.
- [Webhook de GitHub: ubicación y dependencias](./webhook-github-arquitectura): dónde se ubicó el caso de uso del webhook y qué alternativas se evaluaron.
- [Thundering herd contra la API de GitHub](./thundering-herd-github): el error `503 Backend.max_conn reached` tras la refactorización y la solución con in-flight promises.
- [Cardinalidad de la notificación document/publishing](./notificacion-cambio-document): por qué la fuente del cambio siempre debe activar el flujo de notificación y cómo mantener el desacoplamiento si la necesidad la declara `publishing`.
- [Tipos de buses de eventos](./tipos-de-buses-de-eventos): un bus de eventos no es una sola categoría; diferencias entre intra-proceso, inter-proceso y distribuido.
