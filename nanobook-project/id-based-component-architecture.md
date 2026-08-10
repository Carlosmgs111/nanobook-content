---
title: "Arquitectura de componentes basada en IDs"
description: "Resumen de la arquitectura de componentes del SidebarNav y TableOfContents de nanobook, basada en IDs del DOM."
date: 2026-08-06
author: "Nanobook"
tags: []
draft: false
index: false
---

# Arquitectura de componentes basada en IDs

## Resumen

Este proyecto adopta un enfoque en el que cada componente Astro encapsula su propia lógica de presentación y comportamiento, y se comunica con otros componentes a través del DOM usando IDs explícitos. El patrón se aplica tanto al `SidebarNav` como al `TableOfContents`.

Este documento describe los principios generales, la estructura de alto nivel, las decisiones arquitectónicas compartidas y los documentos específicos de cada sistema.

## Principios del enfoque

### 1. Atomización de la lógica

Cada componente Astro es responsable de:

- Renderizar su propio markup.
- Adjuntar los listeners que necesita.
- Mantener su estado local mediante atributos `data-*`.

### 2. Comunicación por IDs

Los componentes no reciben callbacks ni estado compartido mediante props complejas. En su lugar, reciben el `id` del elemento del DOM que necesitan manipular y usan `document.getElementById` para leer/escribir atributos.

### 3. Estado declarativo mediante `data-*`

El estado se expresa en el DOM como atributos de datos. El CSS responde a estos atributos, manteniendo separadas las reglas visuales de la lógica de JavaScript.

Atributos comunes:

- `data-sidebar-mode="expanded | collapsed"`
- `data-toc-mode="standard | compact"`
- `data-is-hidden="true | false"`
- `data-is-active="true | false"`
- `data-is-visible="true | false"`

### 4. Eventos custom para desacoplamiento

Cuando un componente necesita notificar a otro sin conocerlo directamente, emite eventos custom sobre un elemento del DOM identificado por ID. Otro componente escucha ese evento y reacciona.

Esto se usa, por ejemplo, entre `TocNav` (emite `toc:headings` y `toc:active` sobre `#toc-root`) y `TocIndicator` (los escucha). Además, el emisor guarda el último estado en el propio elemento DOM, por lo que un receptor que se inicialice tarde puede leer el estado actual sin perder ningún evento.

## Estructura de alto nivel

```text
Layout.astro
├── OpenSidebar.astro
├── SidebarNav/index.astro
│   ├── SidebarHeader.astro
│   │   ├── CloseSidebar.astro
│   │   └── SidebarModeToggle.astro
│   └── SidebarList.astro
│       └── ToolTip.astro
├── main
└── TableOfContents/index.astro
    ├── TocIndicator.astro
    └── TocNav.astro
        └── TocToggle.astro
```

## Decisiones arquitectónicas

### Layout como coordinador

La lógica que comparten el sidebar y la tabla de contenidos (`enforceSingleExpanded`, observers, etc.) permanece en `Layout.astro`. Esto es intencional: `Layout` actúa como contenedor de ambos componentes y como canal de comunicación entre ellos. Ninguno de los dos tiene autoridad directa sobre el otro; ambos leen y escriben atributos `data-*` que `Layout` observa y coordina.

### IDs fijos como contrato

Tanto `SidebarNav` como `TableOfContents` definen IDs fijos en su componente raíz y los propagan a sus hijos. Esto simplifica la depuración y el razonamiento sobre el DOM, a costa de que estos componentes no sean fácilmente reutilizables varias veces en la misma página. Para este proyecto, que usa un único sidebar y un único TOC por página, es una tradeoff aceptable.

## Documentación específica

- [`sidebar-architecture.md`](./sidebar-architecture.md): estructura, contratos de IDs y flujo de datos del `SidebarNav`.
- [`toc-architecture.md`](./toc-architecture.md): estructura, scrollSpy, eventos custom y contratos de IDs del `TableOfContents`.

## Fortalezas

1. **Bajo acoplamiento superficial**: los componentes no dependen de la estructura interna de otros, solo de un contrato de IDs.
2. **Fácil de leer**: cada archivo es pequeño y su responsabilidad está clara.
3. **Estado visible y depurable**: los atributos `data-*` se pueden inspeccionar directamente en el DevTools.
4. **CSS declarativo**: las transiciones y estilos condicionales están centralizados en las hojas de estilo.
5. **Desacoplamiento con eventos**: los componentes pueden reaccionar a cambios sin importarse mutuamente.

## Riesgos y recomendaciones

1. **Acoplamiento por ID**: si cambia un ID, hay que actualizar múltiples lugares. Conviene documentar los contratos en cada componente.
2. **Orden de inicialización**: cuando se usan eventos custom, el receptor debe estar inicializado antes de que el emisor envíe el evento. En la práctica esto funciona porque Astro inyecta los scripts en orden del DOM, pero es un punto a vigilar.
3. **Guardas defensivas**: validar existencia de elementos antes de manipularlos en todos los scripts.
4. **Store global**: si la coordinación crece, evaluar una pequeña utilidad de estado global basada en eventos personalizados en lugar de mutar atributos desde múltiples lugares.
