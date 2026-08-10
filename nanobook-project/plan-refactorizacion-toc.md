---
title: "Plan de refactorización del TableOfContents"
description: "Propuesta para reorganizar la lógica JavaScript del TableOfContents de nanobook usando componentes autocontenidos con archivos .logic.ts, siguiendo el mismo enfoque aplicado al SidebarNav."
date: 2026-08-06
author: "Nanobook"
tags: ["astro", "toc", "refactor", "componentes", "arquitectura", "solid"]
draft: false
index: false
---

# Plan de refactorización del `TableOfContents`

## 1. Diagnóstico actual

El componente `src/components/TableOfContents/index.astro` ya está dividido en subcomponentes de presentación (`TocNav`, `TocIndicator`, `TocToggle`), pero aún acumula toda la lógica de comportamiento en un único archivo de ~200 líneas:

1. **Renderizado del contenedor**: `.toc-root`, visibilidad y modo por defecto.
2. **Script inline anti-flash**: restaurar `data-toc-mode` desde `localStorage` antes del primer paint.
3. **Gestión del modo**: toggle entre `standard` y `compact`, persistencia en `localStorage`.
4. **Scroll spy**: detectar el heading activo según el scroll.
5. **Posicionamiento de líneas del indicador**: traducir offsets de headings a posición porcentual.
6. **Coordinación de estados**: actualizar tanto links de `TocNav` como líneas de `TocIndicator`.

Además, la lógica de comportamiento no es testeable de forma aislada porque vive dentro de la etiqueta `<script>` del orquestador.

## 2. Objetivo de la refactorización

Aplicar el mismo enfoque usado en `SidebarNav`: **componentes autocontenidos con lógica pura en archivos `.logic.ts`**.

- Cada componente es responsable de su propio comportamiento.
- La lógica pura se extrae a archivos `.ts` testeables.
- El orquestador solo conserva el anti-flash inline y la coordinación mínima necesaria.
- Se respetan los principios SOLID y el patrón `data-*` + CSS existente.

## 3. Propuesta de estructura

```
src/
  components/
    TableOfContents/
      index.astro                      # Orquestador + anti-flash inline
      TocToggle.astro                  # Botón + script de modo
      TocToggle.logic.ts               # Lógica pura de modo standard/compact
      TocNav.astro                     # Lista de links + script de scroll spy
      TocNav.logic.ts                  # Lógica pura de scroll spy
      TocIndicator.astro               # Indicador compacto + script de posicionamiento
      TocIndicator.logic.ts            # Lógica pura de posicionamiento de líneas
      TableOfContents.logic.ts         # Funciones compartidas (collectHeadings)
```

### 3.1 ¿Por qué un archivo `TableOfContents.logic.ts` compartido?

Tanto el scroll spy (`TocNav`) como el indicador compacto (`TocIndicator`) necesitan resolver los headings del documento a partir de los links del TOC. Para evitar duplicar `collectHeadings`, se extrae a un archivo compartido dentro de la misma carpeta.

Este archivo no ejecuta side effects; solo exporta funciones puras y deterministas.

## 4. División de responsabilidades

### 4.1 `TableOfContents/index.astro` (orquestador)

Responsabilidades:

- Recibir `headings` como prop.
- Renderizar el contenedor `.toc-root` con `data-toc-mode` y `data-is-visible`.
- Importar y componer `TocIndicator`, `TocNav` y `TocToggle`.
- Contener el `<script is:inline>` mínimo anti-flash.
- No contener lógica de scroll spy ni de modo; solo orquestación.

### 4.2 `TableOfContents/TocToggle.astro`

Responsabilidades:

- Renderizar el botón `#toc-toggle` con ambos iconos.
- Contener un `<script>` que importa `initTocMode` desde `./TocToggle.logic.ts`.
- Inicializar el modo sobre `.toc-root` y `#toc-toggle`.
- No contener lógica de negocio; solo inicialización.

### 4.3 `TableOfContents/TocToggle.logic.ts`

Responsabilidades:

- Exportar funciones puras para leer/escribir el modo del TOC.
- No tener side effects ni dependencias de DOM globales.
- Exportar `getSavedTocMode`, `setTocMode` e `initTocMode`.

```ts
export type TocMode = "standard" | "compact";

export function getSavedTocMode(): TocMode {
  if (typeof localStorage === "undefined") return "standard";
  const saved = localStorage.getItem("toc-mode");
  return saved === "compact" ? "compact" : "standard";
}

export function setTocMode(
  root: HTMLElement,
  button: HTMLElement,
  mode: TocMode
): void {
  root.setAttribute("data-toc-mode", mode);
  button.setAttribute("data-toc-mode", mode);
  button.setAttribute("aria-pressed", String(mode === "compact"));
  if (typeof localStorage !== "undefined") {
    localStorage.setItem("toc-mode", mode);
  }
}

export function initTocMode(
  root: HTMLElement,
  button: HTMLElement
): void {
  setTocMode(root, button, getSavedTocMode());

  button.addEventListener("click", () => {
    const current =
      (root.getAttribute("data-toc-mode") as TocMode) || "standard";
    const next = current === "standard" ? "compact" : "standard";
    setTocMode(root, button, next);
  });
}
```

### 4.4 `TableOfContents/TocNav.astro`

Responsabilidades:

- Recibir `headings`.
- Renderizar `<nav class="toc-nav">` con la lista de links.
- Contener un `<script>` que importa `initScrollSpy` desde `./TocNav.logic.ts`.
- Inicializar el scroll spy sobre los links `.toc-item a` y las líneas `.toc-indicator-line`.

### 4.5 `TableOfContents/TocNav.logic.ts`

Responsabilidades:

- Exportar funciones puras para el scroll spy.
- Importar `findActiveIndex` desde `src/lib/scroll-spy.ts`.
- Importar `collectHeadings` desde `./TableOfContents.logic.ts`.
- No tener side effects fuera de los listeners registrados por `initScrollSpy`.
- Exportar `initScrollSpy` y `updateActiveState`.

```ts
import { findActiveIndex } from "../../lib/scroll-spy.ts";
import { collectHeadings } from "./TableOfContents.logic.ts";

export interface HeadingRef {
  id: string;
  element: HTMLElement;
}

export function updateActiveState(
  links: NodeListOf<HTMLElement>,
  lines: NodeListOf<HTMLElement>,
  headings: HeadingRef[],
  offsets: number[],
  offset: number,
  activeIndexRef: { current: number }
): void {
  const newIndex = findActiveIndex(window.scrollY, offsets, offset);
  if (newIndex === activeIndexRef.current) return;
  activeIndexRef.current = newIndex;

  const activeId = headings[newIndex].id;

  links.forEach((link) => {
    link.dataset.isActive = String(link.getAttribute("href") === "#" + activeId);
  });

  lines.forEach((line, i) => {
    line.dataset.isActive = String(i === newIndex);
  });
}

export function initScrollSpy(
  root: HTMLElement,
  links: NodeListOf<HTMLElement>,
  lines: NodeListOf<HTMLElement>,
  offset: number = 120
): void {
  if (links.length === 0) return;

  const headings = collectHeadings(links);
  if (headings.length === 0) return;

  let offsets: number[] = [];
  const activeIndex = { current: -1 };

  function measure() {
    offsets = headings.map((heading) => heading.element.offsetTop);
    updateActiveState(links, lines, headings, offsets, offset, activeIndex);
  }

  function onScroll() {
    updateActiveState(links, lines, headings, offsets, offset, activeIndex);
  }

  let ticking = false;
  window.addEventListener("scroll", () => {
    if (!ticking) {
      window.requestAnimationFrame(() => {
        onScroll();
        ticking = false;
      });
      ticking = true;
    }
  });

  window.addEventListener("resize", measure);
  window.addEventListener("orientationchange", measure);

  if (document.readyState === "complete") {
    measure();
  } else {
    window.addEventListener("load", measure);
  }
}
```

### 4.6 `TableOfContents/TocIndicator.astro`

Responsabilidades:

- Recibir `headings`.
- Renderizar el contenedor `.toc-indicator` con una línea por heading.
- Contener un `<script>` que importa `initIndicatorLines` desde `./TocIndicator.logic.ts`.
- Inicializar el posicionamiento de las líneas al cargar y al redimensionar.

### 4.7 `TableOfContents/TocIndicator.logic.ts`

Responsabilidades:

- Exportar funciones puras para posicionar las líneas del indicador.
- Importar `collectHeadings` desde `./TableOfContents.logic.ts`.
- No tener side effects fuera de los listeners registrados por `initIndicatorLines`.
- Exportar `positionIndicatorLines` e `initIndicatorLines`.

```ts
import { collectHeadings } from "./TableOfContents.logic.ts";

export function positionIndicatorLines(
  lines: NodeListOf<HTMLElement>,
  offsets: number[]
): void {
  const total = document.documentElement.scrollHeight - window.innerHeight;
  lines.forEach((line, i) => {
    const percent =
      total > 0 ? (offsets[i] / total) * 100 : i === 0 ? 0 : 100;
    line.style.top = `${Math.min(100, Math.max(0, percent))}%`;
  });
}

export function initIndicatorLines(
  links: NodeListOf<HTMLElement>,
  lines: NodeListOf<HTMLElement>
): void {
  if (links.length === 0 || lines.length === 0) return;

  const headings = collectHeadings(links);
  if (headings.length === 0) return;

  function measure() {
    const offsets = headings.map((heading) => heading.element.offsetTop);
    positionIndicatorLines(lines, offsets);
  }

  window.addEventListener("resize", measure);
  window.addEventListener("orientationchange", measure);

  if (document.readyState === "complete") {
    measure();
  } else {
    window.addEventListener("load", measure);
  }
}
```

### 4.8 `TableOfContents/TableOfContents.logic.ts`

Responsabilidades:

- Exportar funciones compartidas entre `TocNav` y `TocIndicator`.
- No tener side effects.
- Exportar `collectHeadings`.

```ts
export interface HeadingRef {
  id: string;
  element: HTMLElement;
}

export function collectHeadings(
  links: NodeListOf<HTMLElement>
): HeadingRef[] {
  const headings: HeadingRef[] = [];
  links.forEach((link) => {
    const href = link.getAttribute("href");
    if (!href || !href.startsWith("#")) return;
    const heading = document.getElementById(href.slice(1));
    if (heading) {
      headings.push({ id: heading.id, element: heading });
    }
  });
  return headings;
}
```

## 5. Distribución de scripts tras la refactorización

| Componente | Script que contiene |
|---|---|
| `TableOfContents/index.astro` | Anti-flash inline |
| `TocToggle.astro` | Inicialización del modo standard/compact |
| `TocNav.astro` | Inicialización del scroll spy |
| `TocIndicator.astro` | Inicialización del posicionamiento de líneas |

### Script inline anti-flash

El script `is:inline` debe permanecer en el orquestador porque Astro no empaqueta imports dentro de scripts inline. Su única responsabilidad será restaurar `data-toc-mode` antes del primer paint:

```astro
<script is:inline>
  (function () {
    const saved =
      typeof localStorage !== "undefined"
        ? localStorage.getItem("toc-mode")
        : null;
    if (saved !== "compact" && saved !== "standard") return;

    const root = document.querySelector(".toc-root");
    const toggle = document.getElementById("toc-toggle");
    if (!root) return;

    root.setAttribute("data-toc-mode", saved);
    if (toggle) {
      toggle.setAttribute("data-toc-mode", saved);
      toggle.setAttribute("aria-pressed", String(saved === "compact"));
    }
  })();
</script>
```

## 6. Consideraciones de integración con `Layout.astro`

`Layout.astro` contiene un script inline de coordinación entre sidebar, TOC y modo de ancho de contenido. Este script:

- Lee y escribe `data-toc-mode`.
- Llama a `setTocMode` localmente.
- Escucha cambios en `data-toc-mode` mediante `MutationObserver`.

Para mantener la compatibilidad:

1. **No romper el contrato de `data-toc-mode`**: la lógica debe usar el mismo atributo y los mismos valores (`standard`/`compact`).
2. **Mantener el `localStorage` como fuente de verdad**: tanto el orquestador como `Layout.astro` leen/escriben la misma clave `toc-mode`.
3. **Evitar doble persistencia**: el listener del `Layout.astro` no debe llamar a `localStorage.setItem` cuando aplica restricciones; solo debe cambiar atributos.

## 7. Manejo de estilos

Se mantienen los estilos co-localizados en cada subcomponente. El orquestador solo conserva los estilos de `.toc-root`, visibilidad y modo:

- `TableOfContents/index.astro`: `.toc-root`, `data-toc-mode`, `data-is-visible`.
- `TocNav.astro`: `.toc-nav`, `.toc-item`, estados activos.
- `TocIndicator.astro`: `.toc-indicator`, `.toc-indicator-line`.
- `TocToggle.astro`: `#toc-toggle` y visibilidad de iconos.

> **Nota**: los estilos que reaccionan al modo del TOC (por ejemplo, ocultar `.toc-indicator` en modo `standard`) ya usan `:global(.toc-root[data-toc-mode="standard"])`, lo cual sigue siendo válido.

## 8. Pasos de implementación sugeridos

1. **Crear** `TableOfContents/TableOfContents.logic.ts` con `collectHeadings`.
2. **Crear** `TableOfContents/TocToggle.logic.ts` con funciones puras de modo.
3. **Crear** `TableOfContents/TocNav.logic.ts` con funciones puras de scroll spy.
4. **Crear** `TableOfContents/TocIndicator.logic.ts` con funciones puras de posicionamiento.
5. **Actualizar** `TocToggle.astro` para importar e inicializar `initTocMode`.
6. **Actualizar** `TocNav.astro` para importar e inicializar `initScrollSpy`.
7. **Actualizar** `TocIndicator.astro` para importar e inicializar `initIndicatorLines`.
8. **Refactorizar** `TableOfContents/index.astro` como orquestador: solo anti-flash inline.
9. **Mover** los estilos correspondientes si es necesario.
10. **Ejecutar** `npm run build` y `npm run dev`.
11. **Probar manualmente**:
    - Toggle standard/compact en desktop.
    - Persistencia del modo tras recargar.
    - Scroll spy al navegar por la página.
    - Indicador compacto mostrando la posición correcta.
    - Coordinación con Sidebar en modo `constrained`.

## 9. Qué NO cambiar

Para mantener la estabilidad del sistema:

- El contrato de `data-toc-mode` (`standard`/`compact`).
- La clave de `localStorage`: `toc-mode`.
- El uso de `data-is-active` para marcar links y líneas activas.
- La función `findActiveIndex` en `src/lib/scroll-spy.ts`.
- El renderizado de ambos iconos en el toggle y la ocultación por CSS.
- El script de coordinación del `Layout.astro` se mantiene en esta iteración.

## 10. Conclusión

La refactorización del `TableOfContents` sigue el mismo enfoque aplicado al `SidebarNav`: **componentes autocontenidos con lógica pura en archivos `.logic.ts`**. El orquestador Astro solo conserva el anti-flash inline, mientras que cada subcomponente (`TocToggle`, `TocNav`, `TocIndicator`) es responsable de importar e inicializar su propio comportamiento.

Esto mejora la testeabilidad, reduce el acoplamiento y mantiene la arquitectura declarativa basada en `data-*` y CSS que ya funciona en el resto del proyecto.
