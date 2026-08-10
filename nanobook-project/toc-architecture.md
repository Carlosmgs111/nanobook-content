---
title: "Arquitectura del TableOfContents (TOC)"
description: "Resumen del TableOfContents (TOC) de nanobook"
date: 2026-08-06
author: "Nanobook"
tags: []
draft: false
index: false
---

# Arquitectura del TableOfContents (TOC)

## Resumen

El `TableOfContents` muestra los headings del artículo actual, resalta la sección visible mediante scrollSpy y ofrece un modo compacto para ahorrar espacio en escritorio. Está compuesto por componentes atómicos que se comunican mediante IDs del DOM, atributos `data-*` y eventos custom.

## Estructura de componentes

```text
TableOfContents/index.astro
├── TocIndicator.astro
└── TocNav.astro
    └── TocToggle.astro
```

A nivel de `Layout.astro`:

```text
Layout.astro
├── main
└── TableOfContents/index.astro
```

## Contratos de IDs

| ID del DOM           | Propósito                              | Definido en                   |
| -------------------- | -------------------------------------- | ----------------------------- |
| `toc-root`           | Contenedor raíz del TOC.               | `TableOfContents/index.astro` |
| `toc-nav`            | Navegación con los enlaces a headings. | `TocNav.astro`                |
| `toc-toggle`         | Botón para alternar standard/compact.  | `TocToggle.astro` (default)   |
| `toc-root-indicator` | Contenedor de las líneas indicadoras.  | `TocIndicator.astro`          |

## Estados y atributos `data-*`

### `data-toc-mode`

Valores: `standard` | `compact`

- En modo `standard` se muestra el nav completo con enlaces.
- En modo `compact` se muestra solo el indicador visual; el nav aparece al hacer hover o focus.
- Se persiste en `localStorage` con la clave `toc-mode`.

### `data-is-active`

Valores: `true` | `false`

- Aplicado a los enlaces del TOC para resaltar el heading actual.
- Aplicado a las líneas del indicador para marcar la sección activa.

### `data-is-visible`

Valores: `true` | `false`

- Determina si el TOC debe mostrarse en escritorio.
- Depende de si hay headings disponibles.

## Flujo de datos

### Inicialización del modo

1. `TocToggle` usa un script `is:inline` para leer `localStorage` y aplicar el modo antes de la primera pintura.
2. Esto evita un flash visual entre `standard` y `compact`.

### Cambio de modo

1. El usuario hace click en `TocToggle`.
2. `TocToggle` actualiza `data-toc-mode` en `#toc-root` y en sí mismo.
3. El CSS de `TocNav` y `TocIndicator` responde al nuevo modo.
4. `Layout.astro` observa el atributo mediante `MutationObserver` y ejecuta `enforceSingleExpanded`.

### ScrollSpy

1. `TocNav` recopila los headings del artículo a partir de los `href` de los enlaces.
2. Guarda las referencias en `#toc-root._tocHeadings` y emite el evento custom `toc:headings` sobre `#toc-root`.
3. En cada scroll (throttled con `requestAnimationFrame`), calcula el heading activo usando `findActiveIndex`.
4. Actualiza `data-is-active` en el enlace activo.
5. Guarda el `id` activo en `#toc-root._tocActiveId` y emite el evento custom `toc:active` sobre `#toc-root`.
6. `TocIndicator` lee el estado actual al inicializarse y escucha `toc:headings` y `toc:active` para actualizaciones futuras.

### Robustez ante el orden de carga

Los eventos custom se emiten sobre `#toc-root`, que existe en el HTML estático desde el principio. Además, `TocNav` guarda el último estado en propiedades del elemento root (`_tocHeadings`, `_tocActiveId`). De este modo, `TocIndicator` puede leer el estado actual incluso si se inicializó después de que `TocNav` emitió los eventos.

## Componentes

### `TableOfContents/index.astro`

- Define los IDs fijos (`toc-root`, `toc-nav`, `toc-toggle`).
- Renderiza `TocIndicator` y `TocNav`, pasándoles los IDs como props.
- Aplica `data-is-visible` según si hay headings.

### `TocNav.astro`

- Renderiza la lista de enlaces a los headings.
- Recibe `tocRootId`, `tocNavId` y `toggleId`.
- Contiene la lógica del scrollSpy.
- Guarda el estado en `#toc-root` y emite eventos custom `toc:headings` y `toc:active` sobre `#toc-root`.

### `TocIndicator.astro`

- Renderiza líneas visuales que representan la posición de cada heading en el documento.
- Recibe `tocRootId`.
- Al inicializarse lee `#toc-root._tocHeadings` y `#toc-root._tocActiveId`.
- Escucha los eventos `toc:headings` y `toc:active` de `#toc-root`.
- Incluye guarda defensiva: `if (!tocIndicator || !root) return;`.

### `TocToggle.astro`

- Renderiza el botón de cambio de modo.
- Recibe `toggleId` y `tocRootId`.
- Usa `is:inline` para inicializar el modo desde `localStorage` temprano.
- Alterna entre `standard` y `compact` al hacer click.

## Decisiones clave

- **Eventos custom + estado en el DOM**: `TocNav` no conoce a `TocIndicator`. Emite eventos sobre `#toc-root` y, además, guarda el último estado en propiedades del elemento. Esto desacopla los componentes y elimina la fragilidad del orden de carga.
- **ScrollSpy en `TocNav`**: la responsabilidad de decidir qué heading está activo vive junto a los enlaces que lo representan.
- **`is:inline` en `TocToggle`**: al no depender de imports externos, el toggle puede ejecutarse antes de la pintura, evitando flashes de modo.
- **IDs fijos**: simplifican el razonamiento sobre el DOM, asumiendo un único TOC por página.

## Consideraciones

- El TOC solo se muestra en escritorio (`md:block`).
- En modo `compact`, el nav se posiciona absolutamente a la izquierda del indicador y aparece en hover/focus.
- El scrollSpy usa `requestAnimationFrame` para no saturar el thread principal.
- Los offsets de los headings se recalculan en `resize` y `orientationchange`.
