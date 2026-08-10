---
title: "Comunicación de estado entre componentes: del TOC al layout completo"
description: "Cómo los componentes de nanobook comparten estado mediante data attributes y localStorage, desde el scroll spy del TOC hasta la coordinación entre Sidebar, TOC y el modo de ancho de contenido."
date: 2026-08-05
author: "Nanobook"
tags: ["astro", "javascript", "state", "data-attributes", "localStorage", "toc", "sidebar", "layout"]
draft: false
index: false
---

# Comunicación de estado entre componentes: del TOC al layout completo

## 1. El problema inicial

En el layout de `nanobook` hay tres componentes principales que deben actuar de forma coordinada:

- `SidebarNav`: navegación contextual del directorio actual.
- `TableOfContents`: mapa de encabezados de la página actual.
- `ContentWidthToggle`: alternador entre modo de contenido `constrained` y `full`.

Cada uno de estos componentes tiene su propio script cliente, sus propios estados y sus propios botones. El problema es que las decisiones de un componente afectan a los demás:

- Si el TOC marca un encabezado como activo, el indicador visual y el enlace correspondiente deben actualizarse juntos.
- Si el usuario cambia al modo `full`, el Sidebar y el TOC deben poder expandirse simultáneamente.
- Si el usuario está en modo `constrained`, solo uno de los dos paneles laterales puede estar expandido a la vez.
- El usuario espera que sus preferencias persistan entre navegaciones.

La solución adoptada fue expresar todo el estado de forma **declarativa** en el DOM mediante atributos de datos (`data-*`) y conservar las preferencias en `localStorage`.

---

## 2. Estado declarativo con data attributes

### 2.1 Principio general

Cada componente escribe su estado en el HTML como atributos de datos. Otros componentes o scripts de layout pueden leer esos atributos sin depender de clases CSS internas.

```html
<!-- SidebarNav -->
<aside id="sidebar" data-sidebar-mode="expanded">
  <button id="sidebar-mode-toggle" data-sidebar-mode="expanded" aria-pressed="false">
    ...
  </button>
</aside>

<!-- TableOfContents -->
<div class="toc-root" data-toc-mode="standard" data-is-visible="true">
  <a href="#introduccion" data-is-active="false">Introducción</a>
  <div class="toc-indicator-line" data-is-active="false"></div>
</div>

<!-- Content width -->
<div id="content-wrapper" data-content-mode="constrained">
  <button id="content-width-toggle" data-content-mode="constrained" aria-pressed="false">
    ...
  </button>
</div>
```

### 2.2 Ventajas

1. **Desacoplamiento**: el script no conoce las clases CSS de otro componente; solo lee y escribe atributos.
2. **Inspección directa**: el estado es visible en el DOM, lo que facilita depuración y tests.
3. **CSS como consumidor**: las reglas de estilo pueden reaccionar a los atributos sin JavaScript.
4. **Observabilidad**: `MutationObserver` puede detectar cambios de estado en tiempo real.

---

## 3. Coordenación de múltiples scripts

### 3.1 Tipos de scripts en Astro

El proyecto usa dos tipos de scripts cliente:

- **Scripts inline** (`<script is:inline>`): se ejecutan en el orden en que aparecen en el HTML, durante el parseo del documento.
- **Módulos de componente** (`<script>`): Astro los envuelve como `type="module"`, se ejecutan de forma diferida y en orden, después del parseo.

Esto crea un orden de ejecución crítico:

1. El script inline del layout se ejecuta primero.
2. Los módulos de `SidebarNav`, `ContentWidthToggle` y `TableOfContents` se ejecutan después.

### 3.2 Problema de orden: el sidebar se restauraba después del layout

El layout necesita aplicar restricciones en modo `constrained`. Si el sidebar estaba expandido y el TOC estándar, el layout colapsa el sidebar.

Sin embargo, el módulo de `SidebarNav` restaura el estado desde `localStorage` después del layout. Si el layout persistía el colapso temporal, la preferencia del usuario se veía contaminada.

**Solución**: el layout no escribe en `localStorage` cuando aplica una restricción. Solo colapsa o expande visualmente los atributos. La persistencia sigue siendo responsabilidad del botón de toggle de cada componente.

### 3.3 Problema de orden: el botón de expandir no funcionaba

El `SidebarNav` y el layout escuchan el mismo click en `#sidebar-mode-toggle`. El layout intentaba colapsar el sidebar después de que el `SidebarNav` ya lo había expandido, provocando que el botón pareciera no hacer nada.

**Solución**: el layout registra su listener en la **fase de captura** (`addEventListener("click", handler, true)`), lo que garantiza que se ejecuta antes del listener en fase de burbuja del `SidebarNav`. Así, el layout puede cambiar el TOC a `compact` antes de que el sidebar se expanda, evitando que el `MutationObserver` lo revierta inmediatamente.

---

## 4. Estados del TOC

### 4.1 Estado activo del scroll spy

El scroll spy necesita marcar el encabezado visible como activo. En lugar de mutar clases, el script escribe `data-is-active`:

```js
const activeId = headings[activeIndex].id;

tocLinks.forEach((link) => {
  link.dataset.isActive = link.getAttribute("href") === "#" + activeId;
});

tocLines.forEach((line, i) => {
  line.dataset.isActive = i === activeIndex;
});
```

CSS responde a ese atributo:

```css
.toc-item a[data-is-active="true"] {
  border-left-color: var(--accent);
  color: var(--text);
  background: #ffffff0d;
}

.toc-indicator-line[data-is-active="true"] {
  width: 24px;
  background: var(--accent);
}
```

### 4.2 Estado de modo del TOC

El TOC tiene dos modos:

- `standard`: lista completa de encabezados.
- `compact`: indicador visual reducido.

El modo se guarda en `localStorage` bajo la clave `toc-mode`. Para evitar el flash de cambio de estado al cargar, un script `is:inline` dentro del componente restaura el modo leído **antes** de que el navegador pintee la página, sobrescribiendo el valor por defecto del HTML estático.

```js
const saved = localStorage.getItem("toc-mode");
if (saved === "compact" || saved === "standard") {
  root.setAttribute("data-toc-mode", saved);
}
```

---

## 5. Estados del SidebarNav

### 5.1 Modo de sidebar

El sidebar tiene dos modos:

- `expanded`: ancho completo con texto visible.
- `collapsed`: ancho reducido con iconos y tooltips.

El modo se guarda en `localStorage` bajo la clave `sidebar-mode`. Al igual que en el TOC, un script `is:inline` restaura el modo guardado antes del primer paint para evitar que el usuario vea el estado por defecto (`expanded`) durante un instante y luego el colapso.

```js
const saved = localStorage.getItem("sidebar-mode");
if (saved === "collapsed" || saved === "expanded") {
  root.setAttribute("data-sidebar-mode", saved);
}
```

### 5.2 Tooltips fuera del sidebar

Las tooltips del modo colapsado se renderizan en un único elemento fijo `#sidebar-tooltip` que vive **fuera** del scrollable `<aside>`. Esto evita que se recorten por el `overflow-y-auto` del sidebar.

```html
<div id="sidebar-tooltip" class="fixed z-[60] hidden ..." role="tooltip"></div>
```

---

## 6. Estados del modo de ancho de contenido

### 6.1 Modos disponibles

- `constrained`: el contenedor tiene `max-w-6xl` (72 rem). Aplica la restricción de un solo panel lateral expandido.
- `full`: el contenedor ocupa todo el ancho disponible. Permite ambos paneles expandidos.

### 6.2 Persistencia

El modo se guarda en `localStorage` bajo la clave `content-width-mode`. Un script `is:inline` actualiza el `data-content-mode` del wrapper y del botón antes del primer paint, evitando el flash del modo por defecto.

```js
const saved = localStorage.getItem("content-width-mode");
if (saved === "full" || saved === "constrained") {
  wrapper.setAttribute("data-content-mode", saved);
  button.setAttribute("data-content-mode", saved);
}
```

### 6.3 Botón de ancho en viewports pequeños

El botón de ancho se oculta por debajo de una anchura mínima configurable. El valor por defecto es 1280px:

```css
:root {
  --content-width-min: 1280px;
}

@media (max-width: 1279px) {
  :global(#content-width-toggle) {
    display: none;
  }
}
```

Esto evita ofrecer un modo cuyo botón no sería usable en pantallas muy pequeñas.

> **Nota**: el selector usa `:global(#content-width-toggle)` porque el botón pertenece al componente `ContentWidthToggle`. Astro añade atributos de scope a los estilos de cada componente, por lo que un selector normal del layout no alcanzaría al botón.

### 6.4 Header, breadcrumb, TOC y Sidebar en móvil

- El header se mantiene `sticky` en todos los tamaños de viewport para que el breadcrumb y los toggles siempre estén accesibles.
- El botón de abrir el sidebar vive en el header, junto al toggle de ancho de contenido y al de tema. Antes estaba como botón flotante en la esquina inferior.
- Cada ítem del breadcrumb se trunca con ellipsis si supera `12rem` de ancho, y la lista permite scroll horizontal (`overflow-x-auto`) si el conjunto no cabe.
- El TOC no se muestra en viewports menores de 768px. El contenedor exterior usa `hidden md:block` y las propiedades `sticky`, `flex-shrink-0` y la altura solo se aplican a partir de `md`. La regla `.toc-root[data-is-active="true"]` solo activa `display: block` dentro de `@media (min-width: 768px)`.
- El sidebar móvil se abre como una capa superior (`z-[60]`) sobre un backdrop (`z-50`). El header usa `z-40`, por lo que el breadcrumb queda cubierto al desplegar el sidebar.
- El ancho del sidebar en móvil se limita a `min(80vw, 20rem)` para que nunca supere el ancho de la pantalla.

---

## 7. Reglas del layout: quién puede estar expandido

### 7.1 En modo `constrained`

- Si hay un TOC en la página, el TOC tiene prioridad y permanece expandido (`standard`).
- El Sidebar se colapsa automáticamente si ambos están expandidos.
- El botón de toggle del Sidebar alterna entre dos estados permitidos:
  - Sidebar expandido + TOC compacto.
  - Sidebar colapsado + TOC estándar.

### 7.2 En modo `full`

- No hay restricciones.
- Ambos paneles pueden estar expandidos simultáneamente.
- Al cambiar de `constrained` a `full`, el layout expande automáticamente ambos paneles.
- Al cambiar de `full` a `constrained`, el layout vuelve a aplicar la restricción.

### 7.3 Páginas sin TOC

Si no hay encabezados (`headings` está vacío), no se renderiza el TOC. En ese caso, el Sidebar puede permanecer expandido sin restricciones.

---

## 8. Implementación del layout

El script del layout en `Layout.astro` es inline y se encarga de:

1. Leer los atributos de estado del DOM.
2. Aplicar la restricción de un solo panel en modo `constrained`.
3. Observar cambios de `data-content-mode` para expandir ambos paneles al entrar en modo `full`.
4. Interceptar clicks del toggle del Sidebar en fase de captura para coordinar la alternancia con el TOC.

```js
(function () {
  const wrapper = document.getElementById("content-wrapper");
  const sidebar = document.getElementById("sidebar");
  const tocRoot = document.querySelector(".toc-root");
  const tocToggle = document.getElementById("toc-toggle");
  const sidebarModeToggle = document.getElementById("sidebar-mode-toggle");

  if (!wrapper || !sidebar) return;

  var lastContentMode = wrapper.getAttribute("data-content-mode") || "constrained";

  function setTocMode(mode) { /* ... */ }
  function setSidebarMode(mode) { /* ... */ }
  function expandAll() { /* ... */ }

  function enforceSingleExpanded() {
    if (wrapper.getAttribute("data-content-mode") === "full") return;
    const sidebarExpanded = sidebar.getAttribute("data-sidebar-mode") === "expanded";
    if (!tocRoot) return;
    const tocExpanded = tocRoot.getAttribute("data-toc-mode") === "standard";
    if (sidebarExpanded && tocExpanded) {
      setSidebarMode("collapsed");
    }
  }

  function onContentModeChange() {
    const currentMode = wrapper.getAttribute("data-content-mode") || "constrained";
    if (currentMode === "full" && lastContentMode !== "full") {
      expandAll();
    }
    lastContentMode = currentMode;
    enforceSingleExpanded();
  }

  function init() {
    // Alternancia Sidebar <-> TOC en modo constrained
    if (sidebarModeToggle && tocRoot && tocToggle) {
      sidebarModeToggle.addEventListener(
        "click",
        function () {
          if (wrapper.getAttribute("data-content-mode") === "full") return;
          const sidebarMode = sidebar.getAttribute("data-sidebar-mode") || "expanded";
          const tocMode = tocRoot.getAttribute("data-toc-mode") || "standard";
          if (sidebarMode === "collapsed" && tocMode === "standard") {
            setTocMode("compact");
          } else if (sidebarMode === "expanded" && tocMode === "compact") {
            setTocMode("standard");
          }
        },
        true // fase de captura
      );
    }

    // Observadores
    const wrapperObserver = new MutationObserver(onContentModeChange);
    wrapperObserver.observe(wrapper, { attributes: true, attributeFilter: ["data-content-mode"] });

    const panelObserver = new MutationObserver(enforceSingleExpanded);
    panelObserver.observe(sidebar, { attributes: true, attributeFilter: ["data-sidebar-mode"] });
    if (tocRoot) {
      panelObserver.observe(tocRoot, { attributes: true, attributeFilter: ["data-toc-mode"] });
    }

    enforceSingleExpanded();
  }

  if (document.readyState === "loading") {
    document.addEventListener("DOMContentLoaded", init);
  } else {
    init();
  }
})();
```

---

## 9. Lecciones aprendidas

| Problema | Causa | Solución |
|---|---|---|
| El botón de expandir Sidebar no funcionaba | El layout revertía el cambio después del `SidebarNav` | Registrar el listener del layout en fase de captura |
| La preferencia del usuario se perdía | El layout persistía el colapso temporal en `localStorage` | El layout no escribe en `localStorage`; solo muta atributos |
| El scroll spy no marcaba correctamente | `activeIndex` estaba fuera del closure de `initScrollSpy` | Mantener el estado dentro del closure o usar `findActiveIndex` |
| El indicador del TOC se desalineaba | `margin-top: X%` se resuelve contra el ancho, no el alto | Usar `position: absolute` con `top: X%` |
| Las tooltips se recortaban | Vivían dentro del `aside` con `overflow-y-auto` | Mover el tooltip a un elemento fijo fuera del sidebar |
| El modo `full` no expandía ambos paneles | El layout solo aplicaba restricciones, no expansiones | Añadir `expandAll()` al detectar cambio a `full` |

---

## 10. Referencias

- `src/layouts/Layout.astro` — script de coordinación del layout.
- `src/components/SidebarNav.astro` — estado del sidebar y tooltips.
- `src/components/TableOfContents/index.astro` — estado del TOC y scroll spy.
- `src/components/ContentWidthToggle.astro` — estado del modo de ancho.
- `src/lib/scroll-spy.ts` — helper genérico de scroll spy.
- `src/content/nanobook-project/layout-evolution.md` — evolución del layout de Grid a Flex.

---

## 11. Commits relacionados

- `d75bdb0` — `feat: enforce single expanded panel in constrained mode, allow both in full mode`
- `b505506` — `feat: expand both sidebar and TOC when switching to full mode`
