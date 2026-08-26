---
title: "Plan de componetización del SidebarNav"
description: "Propuesta para aplicar SOLID al componente SidebarNav de nanobook usando componentes autocontenidos con lógica co-localizada en archivos .logic.ts."
date: 2026-08-06
author: "Nanobook"
tags: ["astro", "sidebar", "refactor", "componentes", "arquitectura", "solid"]
draft: false
index: false
---

# Plan de componetización del `SidebarNav`

## 1. Diagnóstico actual

El componente `src/components/SidebarNav.astro` acumula varias responsabilidades en un único archivo de ~370 líneas:

1. **Renderizado del layout del sidebar**: `aside`, backdrop, tooltip flotante, header y lista.
2. **Transformación de datos server-side**: filtrar entradas no borrador, ordenar por título, resolver link del padre y clases de link.
3. **Script inline anti-flash**: restaurar `data-sidebar-mode` desde `localStorage` antes del primer paint.
4. **Control del sidebar móvil**: abrir/cerrar el panel desde el botón `#sidebar-toggle` que vive en `Layout.astro`.
5. **Gestión del modo expandido/colapsado**: toggle, persistencia en `localStorage` y atributos ARIA.
6. **Tooltips del modo colapsado**: posicionamiento, visibilidad y ajuste al viewport.
7. **Estilos CSS**: todo el CSS del sidebar en un único bloque `<style>`.

Esto dificulta el testing, la reutilización y el mantenimiento: cualquier cambio en los tooltips puede afectar al modo, y la lógica móvil depende de un elemento que no está en el componente.

## 2. Objetivo de la refactorización

Aplicar los principios SOLID para convertir `SidebarNav` en un conjunto de piezas pequeñas, cohesivas y desacoplables:

- **Single Responsibility**: cada archivo (componente o módulo) hace una sola cosa.
- **Open/Closed**: se podrá añadir un nuevo comportamiento sin tocar los módulos existentes.
- **Liskov Substitution**: los subcomponentes de presentación serán intercambiables si respetan sus props.
- **Interface Segregation**: las props de cada componente serán mínimas y específicas.
- **Dependency Inversion**: los componentes dependen de funciones puras locales en lugar de contener toda la lógica mezclada.

## 3. Propuesta de estructura

Se adopta el enfoque de **componentes autocontenidos**: cada pieza de presentación lleva su propia lógica en un archivo `.logic.ts` dentro de la misma carpeta.

```
src/
  components/
    SidebarNav/
      index.astro                      # Orquestador: contenedor, anti-flash y lógica móvil
      SidebarNavMobile.logic.ts        # Lógica pura de apertura/cierre móvil
      SidebarHeader.astro              # Cabecera: link padre y botón cerrar
      SidebarList.astro                # Lista de entradas + script de tooltips
      SidebarList.logic.ts             # Lógica pura de tooltips
      SidebarModeToggle.astro          # Botón + script de modo
      SidebarModeToggle.logic.ts       # Lógica pura de modo expandido/colapsado
      SidebarTooltip.astro             # Markup del tooltip flotante
      SidebarBackdrop.astro            # Markup del backdrop móvil
    TableOfContents/                   # Patrón existente
```

### 3.1 ¿Por qué archivos `.logic.ts` dentro de la carpeta del componente?

Esta es la **Opción B**: mantener la lógica cerca del componente que la usa, pero fuera de la etiqueta `<script>`, para que sea testeable y potencialmente reutilizable.

Ventajas:

- **Co-localización**: el comportamiento vive junto al markup que lo usa.
- **Testeabilidad**: un archivo `.ts` puro se puede importar y testear sin Astro.
- **Reutilización**: si la lógica es agnóstica del DOM, se puede reutilizar en otros componentes.
- **Sin carpetas adicionales en `src/lib/`**: no se introduce una nueva convención de organización.

La única excepción necesaria es el **script anti-flash** (`is:inline`), que debe quedar en el orquestador porque Astro no permite imports dentro de scripts inline.

## 4. División de responsabilidades

### 4.1 `SidebarNav/index.astro` (orquestador)

- Recibir `entries` y `parentEntry` como props.
- Renderizar el contenedor raíz (`aside#sidebar`), el backdrop y el tooltip.
- Importar y componer `SidebarHeader`, `SidebarList`, `SidebarModeToggle`, `SidebarTooltip` y `SidebarBackdrop`.
- Contener el `<script is:inline>` mínimo anti-flash.
- Contener el `<script>` que inicializa la lógica móvil importando `SidebarNavMobile.logic.ts`, porque el botón de apertura (`#sidebar-toggle`) vive fuera del componente, en `Layout.astro`.

### 4.2 `SidebarNav/SidebarNavMobile.logic.ts`

- Exportar funciones puras para abrir/cerrar el sidebar móvil.
- No tener side effects ni dependencias de DOM globales.
- Exportar `openSidebar`, `closeSidebar` e `initSidebarMobile`.

### 4.3 `SidebarNav/SidebarHeader.astro`

- Renderizar el contenedor `#sidebar-header`.
- Incluir el link al padre (`#sidebar-parent`) cuando existe `parentEntry`.
- Incluir el botón cerrar (`#sidebar-close`) para móvil.
- Incluir `SidebarModeToggle`.

### 4.4 `SidebarNav/SidebarModeToggle.astro`

- Renderizar el botón `#sidebar-mode-toggle` con ambos iconos.
- Contener un `<script>` que importa `initSidebarMode` desde `./SidebarModeToggle.logic.ts` y lo ejecuta sobre `#sidebar` y `#sidebar-mode-toggle`.
- No contener lógica de negocio; solo la inicialización específica de este componente.

### 4.5 `SidebarNav/SidebarModeToggle.logic.ts`

- Exportar funciones puras para leer/escribir el modo del sidebar.
- No tener side effects ni dependencias de DOM globales.
- Exportar `getSavedSidebarMode`, `setSidebarMode` e `initSidebarMode`.

### 4.6 `SidebarNav/SidebarList.astro`

- Recibir las entradas ya filtradas y ordenadas.
- Renderizar `<ul class="sidebar-list">` con los links.
- Mostrar el mensaje de lista vacía si corresponde.
- Contener un `<script>` que importa `initSidebarTooltips` desde `./SidebarList.logic.ts` y lo ejecuta sobre los links y `#sidebar-tooltip`.

### 4.7 `SidebarNav/SidebarList.logic.ts`

- Exportar funciones puras para mostrar/ocultar y posicionar tooltips.
- No tener side effects ni dependencias de DOM globales.
- Exportar `showTooltip`, `hideTooltip` e `initSidebarTooltips`.

### 4.8 `SidebarNav/SidebarTooltip.astro`

- Renderizar el contenedor fijo `#sidebar-tooltip` fuera del `aside`.
- No contiene lógica; solo markup y estilos base.

### 4.9 `SidebarNav/SidebarBackdrop.astro`

- Renderizar el backdrop móvil `#sidebar-backdrop`.
- No contiene lógica de cierre (la maneja el orquestador a través de `SidebarNavMobile.logic.ts`).

## 5. Archivos `.logic.ts`

### 5.1 `SidebarNavMobile.logic.ts`

```ts
export function openSidebar(
  root: HTMLElement,
  backdrop: HTMLElement
): void {
  root.classList.remove("hidden", "-translate-x-full");
  root.classList.add("translate-x-0");
  backdrop.classList.remove("hidden");
  backdrop.setAttribute("aria-hidden", "false");
}

export function closeSidebar(
  root: HTMLElement,
  backdrop: HTMLElement
): void {
  root.classList.add("hidden", "-translate-x-full");
  root.classList.remove("translate-x-0");
  backdrop.classList.add("hidden");
  backdrop.setAttribute("aria-hidden", "true");
}

export function initSidebarMobile(
  root: HTMLElement,
  openButton: HTMLElement | null,
  closeButton: HTMLElement | null,
  backdrop: HTMLElement
): void {
  if (!openButton || !closeButton) return;

  openButton.addEventListener("click", () => openSidebar(root, backdrop));
  closeButton.addEventListener("click", () => closeSidebar(root, backdrop));
  backdrop.addEventListener("click", () => closeSidebar(root, backdrop));

  root.querySelectorAll("a").forEach((link) => {
    link.addEventListener("click", () => closeSidebar(root, backdrop));
  });
}
```

### 5.2 `SidebarModeToggle.logic.ts`

```ts
export type SidebarMode = "expanded" | "collapsed";

export function getSavedSidebarMode(): SidebarMode {
  if (typeof localStorage === "undefined") return "expanded";
  const saved = localStorage.getItem("sidebar-mode");
  return saved === "collapsed" ? "collapsed" : "expanded";
}

export function setSidebarMode(
  root: HTMLElement,
  button: HTMLElement,
  mode: SidebarMode
): void {
  root.setAttribute("data-sidebar-mode", mode);
  button.setAttribute("data-sidebar-mode", mode);
  button.setAttribute("aria-pressed", String(mode === "collapsed"));
  button.setAttribute(
    "aria-label",
    mode === "expanded" ? "Contraer navegación" : "Expandir navegación"
  );
  if (typeof localStorage !== "undefined") {
    localStorage.setItem("sidebar-mode", mode);
  }
}

export function initSidebarMode(
  root: HTMLElement,
  button: HTMLElement
): void {
  setSidebarMode(root, button, getSavedSidebarMode());

  button.addEventListener("click", () => {
    const current =
      (root.getAttribute("data-sidebar-mode") as SidebarMode) || "expanded";
    const next = current === "expanded" ? "collapsed" : "expanded";
    setSidebarMode(root, button, next);
  });
}
```

### 5.3 `SidebarList.logic.ts`

```ts
export function showTooltip(
  root: HTMLElement,
  target: Element,
  tooltip: HTMLElement
): void {
  if (root.getAttribute("data-sidebar-mode") !== "collapsed") {
    hideTooltip(tooltip);
    return;
  }

  const title = target.getAttribute("data-tooltip");
  if (!title) return;

  const rect = target.getBoundingClientRect();
  const gap = 12;
  let left = rect.right + gap;
  let top = rect.top + rect.height / 2;

  tooltip.textContent = title;
  tooltip.classList.remove("hidden");

  const tipRect = tooltip.getBoundingClientRect();
  top = top - tipRect.height / 2;

  const maxLeft = window.innerWidth - tipRect.width - gap;
  if (left > maxLeft) {
    left = rect.left - gap - tipRect.width;
  }

  tooltip.style.left = `${left}px`;
  tooltip.style.top = `${top}px`;
}

export function hideTooltip(tooltip: HTMLElement): void {
  tooltip.classList.add("hidden");
}

export function initSidebarTooltips(
  root: HTMLElement,
  tooltip: HTMLElement
): void {
  if (window.innerWidth < 768) return;

  const targets = root.querySelectorAll(".sidebar-link, .sidebar-parent");
  targets.forEach((target) => {
    target.addEventListener("mouseenter", () =>
      showTooltip(root, target, tooltip)
    );
    target.addEventListener("mouseleave", () => hideTooltip(tooltip));
    target.addEventListener("focus", () => showTooltip(root, target, tooltip));
    target.addEventListener("blur", () => hideTooltip(tooltip));
  });

  window.addEventListener("resize", () => hideTooltip(tooltip));
  window.addEventListener("scroll", () => hideTooltip(tooltip), true);
}
```

## 6. Distribución de scripts tras la refactorización

| Componente | Script que contiene |
|---|---|
| `SidebarNav/index.astro` | Anti-flash inline + inicialización móvil |
| `SidebarModeToggle.astro` | Inicialización del modo expandido/colapsado |
| `SidebarList.astro` | Inicialización de tooltips |
| `SidebarHeader.astro` | Ninguno (solo presentación) |
| `SidebarTooltip.astro` | Ninguno (solo presentación) |
| `SidebarBackdrop.astro` | Ninguno (solo presentación) |

### Script inline anti-flash

El script `is:inline` debe permanecer en el orquestador porque Astro no empaqueta imports dentro de scripts inline. Su única responsabilidad será restaurar `data-sidebar-mode` antes del primer paint:

```astro
<script is:inline>
  (function () {
    const saved =
      typeof localStorage !== "undefined"
        ? localStorage.getItem("sidebar-mode")
        : null;
    if (saved !== "collapsed" && saved !== "expanded") return;

    const sidebar = document.getElementById("sidebar");
    const toggle = document.getElementById("sidebar-mode-toggle");
    if (!sidebar) return;

    sidebar.setAttribute("data-sidebar-mode", saved);
    if (toggle) {
      toggle.setAttribute("data-sidebar-mode", saved);
      toggle.setAttribute("aria-pressed", String(saved === "collapsed"));
      toggle.setAttribute(
        "aria-label",
        saved === "expanded" ? "Contraer navegación" : "Expandir navegación"
      );
    }
  })();
</script>
```

## 7. Consideraciones de integración con `Layout.astro`

`Layout.astro` contiene un script inline de coordinación entre sidebar, TOC y modo de ancho de contenido. Este script:

- Lee y escribe `data-sidebar-mode`.
- Llama a `setSidebarMode` localmente.
- Escucha clicks en `#sidebar-mode-toggle` en fase de captura.

Para mantener la compatibilidad:

1. **No romper el contrato de `data-sidebar-mode`**: la lógica debe usar el mismo atributo y los mismos valores (`expanded`/`collapsed`).
2. **Mantener el `localStorage` como fuente de verdad**: tanto el orquestador como `Layout.astro` leen/escriben la misma clave `sidebar-mode`.
3. **Evitar doble persistencia en la fase de captura**: el listener del `Layout.astro` no debe llamar a `localStorage.setItem` cuando aplica restricciones; solo debe cambiar atributos. La persistencia sigue siendo responsabilidad del botón de toggle.
4. **Opcional: extraer la coordinación del layout**: en una iteración futura se puede mover el script de coordinación a `src/lib/layout-coordinator.ts`, pero eso queda fuera del alcance inicial.

## 8. Manejo de estilos

Se recomienda **co-localizar** los estilos en cada subcomponente, pero tener en cuenta que Astro scopea los estilos por componente. Los estilos que afectan a elementos de varios subcomponentes (como `.sidebar-text` o `#sidebar-header` en modo colapsado) deben vivir en el orquestador usando `:global()`:

- `SidebarNav/index.astro`:
  - `.sidebar-root`, estados `data-sidebar-mode`, media queries de ancho.
  - `.sidebar-root[data-sidebar-mode="collapsed"] :global(.sidebar-text)` para ocultar textos en modo colapsado.
  - `:global(#sidebar-header)` y `.sidebar-root[data-sidebar-mode="collapsed"] :global(#sidebar-header)` para el layout del header.
- `SidebarModeToggle.astro`: visibilidad de iconos según modo.
- `SidebarTooltip.astro`: posicionamiento y flecha del tooltip.
- `SidebarBackdrop.astro`: opacidad y blur.
- `SidebarList.astro`: estilos de `.sidebar-link`, `.sidebar-icon` y `.sidebar-text`.

> **Nota importante**: Astro no atraviesa automáticamente los scopes de componentes hijos. Si un estilo del orquestador necesita afectar a un elemento de un subcomponente, debe usar `:global(selector)`.

## 9. Pasos de implementación sugeridos

1. **Crear la carpeta** `src/components/SidebarNav/`.
2. **Crear** `SidebarNavMobile.logic.ts` con funciones puras de apertura/cierre móvil.
3. **Crear** `SidebarModeToggle.logic.ts` con funciones puras de modo.
4. **Crear** `SidebarList.logic.ts` con funciones puras de tooltips.
5. **Crear** `SidebarModeToggle.astro` con markup + `<script>` que importe e inicialice `initSidebarMode`.
6. **Crear** `SidebarList.astro` con markup + `<script>` que importe e inicialice `initSidebarTooltips`.
7. **Crear** `SidebarBackdrop.astro`, `SidebarHeader.astro` y `SidebarTooltip.astro` como componentes de solo presentación.
8. **Mover** `SidebarNav.astro` a `SidebarNav/index.astro` y refactorizarlo como orquestador: anti-flash inline + inicialización móvil.
9. **Mover** los estilos correspondientes a cada subcomponente.
10. **Actualizar** `Layout.astro` para importar `SidebarNav` desde su nueva ruta (`../components/SidebarNav/index.astro` o el barrel Astro `../components/SidebarNav`).
11. **Verificar** que el script de coordinación de `Layout.astro` sigue funcionando sin modificar el contrato de `data-sidebar-mode`.
12. **Ejecutar** `npm run build` y `npm run dev`.
13. **Probar manualmente**:
    - Apertura/cierre del sidebar en móvil.
    - Toggle expandir/colapsar en desktop.
    - Persistencia del modo tras recargar.
    - Tooltips en modo colapsado.
    - Coordinación con TOC en modo `constrained`.

## 10. Qué NO cambiar

Para mantener la estabilidad del sistema:

- El contrato de `data-sidebar-mode` (`expanded`/`collapsed`).
- La clave de `localStorage`: `sidebar-mode`.
- El estado del modo expandido/colapsado controlado por atributos, no por clases CSS.
- El tooltip renderizado fuera del `aside` para evitar recorte por `overflow-y-auto`.
- El botón `#sidebar-toggle` permanece en `Layout.astro` (se inicializa desde el orquestador).
- El script de coordinación del `Layout.astro` se mantiene en esta iteración.

## 11. Conclusión

La refactorización de `SidebarNav` no requiere una reescritura profunda. Se divide en **subcomponentes de presentación autocontenidos**, cada uno con su lógica en un archivo `.logic.ts` cercano. El **orquestador Astro** mantiene solo el anti-flash inline y la lógica que depende de elementos externos.

Este enfoque respeta los principios SOLID, mejora la mantenibilidad, mantiene la lógica testeable y conserva la arquitectura declarativa basada en `data-*` y CSS que ya funciona en el resto del proyecto.
