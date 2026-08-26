---
title: "Evolución del layout principal: de Grid a Flex y mejora del TOC"
description: "Análisis de la migración del layout principal de nanobook de CSS Grid a Flexbox, junto con la evolución del componente TableOfContents y la gestión de estado mediante data attributes."
date: 2026-08-04
author: "Nanobook"
tags: ["astro", "tailwind", "css", "flexbox", "grid", "toc", "layout"]
draft: false
index: false
---

# Evolución del layout principal: de Grid a Flex y mejora del TOC

## 1. Introducción

La página principal de `nanobook` se compone de tres elementos fundamentales:

1. **SidebarNav** (barra lateral izquierda): navegación contextual del directorio actual.
2. **Main content** (contenido central): el artículo o índice propiamente dicho.
3. **TableOfContents** (panel derecho): mapa de encabezados de la página actual.

En las últimas iteraciones, la arquitectura de estos tres componentes cambió de un enfoque basado en **CSS Grid** a uno basado en **Flexbox**, y el TOC pasó de ser una simple lista de enlaces a un componente interactivo con dos modos de visualización. Este documento describe esos cambios, el porqué de las reglas de Flex que los hacen posibles y cómo se mejoró la experiencia del TOC.

---

## 2. Layout anterior: Grid de tres columnas

El layout anterior en `Layout.astro` usaba una rejilla fija:

```astro
<div class="relative z-10 mx-auto grid max-w-6xl grid-cols-1 gap-6 px-6 py-12 md:grid-cols-[16rem_1fr_16rem]">
  {sidebarEntries && <SidebarNav ... />}
  <main class="border-x border-[var(--grid-line)] px-6">
    ...
  </main>
  {headings && <TableOfContents ... />}
</div>
```

### Características

- En escritorio se reservaban **16 rem** para la barra lateral, **1 fr** para el contenido y **16 rem** para el TOC.
- En móvil se colapsaba a una sola columna (`grid-cols-1`).

### Limitaciones

1. **Rigidez dimensional**: las columnas laterales tenían un ancho fijo de `16rem`. Si en alguna página no había `sidebarEntries` o `headings`, la cuadrícula seguía reservando ese espacio, dejando huecos o forzando a que los componentes llenaran celdas vacías.
2. **Condicionalidad incómoda**: cuando un componente no se renderizaba, Grid seguía generando la celda correspondiente. Había que compensar con clases condicionales o dejar que el contenido quedara desbalanceado.
3. **Sidebar en móvil**: el `SidebarNav` usaba `fixed` para el panel móvil, lo que lo sacaba del flujo normal y complicaba la alineación con el resto del layout.
4. **TOC estático**: el TOC solo ofrecía una lista de enlaces; no había una versión reducida o un indicador visual de progreso de lectura.

---

## 3. Layout actual: Flexbox

El nuevo layout utiliza Flexbox:

```astro
<div class="z-10 mx-auto flex max-w-6xl gap-6 px-6 py-12">
  {sidebarEntries && <SidebarNav ... />}
  <main class="border-x border-[var(--grid-line)] px-6 flex-1 min-w-0">
    ...
  </main>
  {headings && <TableOfContents ... />}
</div>
```

### Ventajas principales

1. **Columnas condicionales sin huecos**: si no hay entradas para el sidebar o no hay encabezados para el TOC, esos elementos simplemente no se renderizan. El contenido central, gracias a `flex-1`, ocupa todo el espacio disponible sin dejar columnas vacías.
2. **Anchos adaptativos**: los componentes laterales controlan su propio ancho (`max-w-64`, `width: 18rem`, etc.), mientras que el contenido central crece para llenar el espacio restante.
3. **Flujo coherente en móvil**: en pantallas pequeñas, los elementos se apilan o se muestran como paneles superpuestos sin romper la estructura de contenedor principal.

---

## 4. Reglas de Flex clave para el comportamiento esperado

### 4.1 `flex-shrink-0` en los componentes laterales

Tanto `SidebarNav` como `TableOfContents` declaran `flex-shrink-0` (directamente o a través de sus contenedores).

```astro
<!-- SidebarNav -->
<aside class="sticky flex-shrink-0 max-w-64 ...">

<!-- TableOfContents -->
<div class="h-[calc(100vh-10rem)] sticky top-32 flex-shrink-0">
```

**Por qué importa**: sin `flex-shrink-0`, Flexbox puede encoger esos elementos cuando el contenido central necesita más espacio. Esto haría que el sidebar o el TOC se volvieran demasiado estrechos o incluso invisibles. Con `flex-shrink-0`, sus anchos mínimos se respetan.

### 4.2 `flex-1` y `min-w-0` en el contenido principal

```astro
<main class="border-x border-[var(--grid-line)] px-6 flex-1 min-w-0">
```

- **`flex-1`**: le dice al `main` que ocupe todo el espacio restante del contenedor flex.
- **`min-w-0`**: es crítico. Por defecto, un flex item no puede ser más estrecho que su contenido natural (`min-width: auto`). Si el contenido principal incluye bloques de código, tablas o palabras largas, el `main` forzaría al layout a crecer más allá del viewport, provocando scroll horizontal. `min-w-0` permite que el `main` se encoja y respete los límites del contenedor.

### 4.3 `sticky` en lugar de `fixed` para la barra lateral y el TOC

- **`fixed`** saca al elemento del flujo normal y lo posiciona respecto al viewport. Eso obliga a calcular manualmente márgenes y z-index, y puede tapar otros elementos.
- **`sticky`** mantiene al elemento en el flujo del documento pero lo "pega" al viewport cuando se alcanza su punto de anclaje (`top-32`). Así, el layout respeta su ancho y posición natural, y no es necesario reservar espacio artificialmente.

### 4.4 `max-w-64` y `max-w-6xl`

- `max-w-64` en el sidebar limita su ancho máximo para que no desproporcione el layout.
- `max-w-6xl` en el contenedor principal evita que la fila de tres componentes se extienda demasiado en pantallas muy grandes, manteniendo la legibilidad.

---

## 5. Evolución del TableOfContents (TOC)

### 5.1 Estado anterior

El TOC era simplemente una lista de enlaces dentro de un `nav` sticky:

```astro
<nav class="toc-nav hidden md:block" aria-label="En esta página">
  <div class="sticky top-32 max-h-[calc(100vh-9rem)] overflow-y-auto border border-[var(--grid-line-strong)] bg-[var(--layer-1)] p-4">
    <ul ...>
      <li ...><a href="#slug">...</a></li>
    </ul>
  </div>
</nav>
```

### 5.2 Estado actual: TOC dual

Ahora el TOC está envuelto en un contenedor con dos modos controlados por `data-toc-mode`:

- **Modo `standard`**: lista completa de encabezados, igual que antes.
- **Modo `compact`**: indicador visual reducido compuesto por pequeñas líneas horizontales que representan la posición de cada encabezado en el documento.

```astro
<div class="h-[calc(100vh-10rem)] sticky top-32 flex-shrink-0">
  <div class="toc-root md:block" data-toc-mode="standard">
    <div class="toc-indicator" aria-hidden="true">
      {headings.map((heading) => <div class="toc-indicator-line" data-slug={heading.slug} />)}
    </div>

    <nav class="toc-nav" aria-label="En esta página">
      ...
    </nav>
  </div>
</div>
```

El cambio de modo se controla desde el header mediante un botón y se persiste en `localStorage`:

```js
const saved = localStorage.getItem("toc-mode");
const mode = saved === "compact" ? "compact" : "standard";
root.setAttribute("data-toc-mode", mode);
```

### 5.3 Mejora de la experiencia

1. **Progreso visual compacto**: en modo compacto, el usuario ve de un vistazo dónde está y cuánto le queda por leer.
2. **Acceso rápido al menú**: al pasar el mouse o enfocar el indicador, aparece el menú completo (`:hover` / `:focus-within`).
3. **Persistencia**: el modo elegido se guarda entre navegaciones.
4. **Accesibilidad**: se actualizan atributos como `aria-pressed` y `aria-label` en el botón de cambio de modo.

### 5.4 Posicionamiento correcto de las líneas indicadoras

Inicialmente se intentó posicionar las líneas usando `margin` porcentual:

```js
line.style.margin = `${percent}%`;
```

Esto fue incorrecto porque **CSS resuelve los márgenes verticales en porcentaje contra el ancho del contenedor**, no contra su alto. La solución fue usar `position: absolute` con `top: X%`:

```css
.toc-indicator {
  position: relative;
  height: 100%;
}

.toc-indicator-line {
  position: absolute;
  top: 0;
  left: 50%;
  transform: translateX(-50%);
}
```

```js
line.style.top = `${percent}%`;
```

Así, `top: X%` se resuelve correctamente contra la altura del contenedor `.toc-indicator`, logrando la correspondencia visual deseada entre la posición del encabezado en el documento y la posición de la línea en el indicador.

### 5.5 Gestión de estado: de clases dinámicas a `data-is-active`

Uno de los cambios más significativos en el TOC fue **dejar de manipular clases CSS desde JavaScript** para expresar estados y, en su lugar, **establecer atributos de datos** que CSS interpreta directamente. El script ya no conoce los nombres de las clases de presentación; solo escribe un booleano en el dataset. La apariencia queda enteramente delegada a las reglas de estilo.

#### Antes: administración imperativa de clases

El código anterior alternaba clases de Tailwind y CSS según el estado:

```js
const isActive = link.getAttribute("href") === "#" + activeId;
if (isActive) {
  link.classList.remove("border-transparent", "text-[var(--muted)]");
  link.classList.add("border-[var(--accent)]", "text-[var(--text)]", "bg-white/5");
} else {
  link.classList.add("border-transparent", "text-[var(--muted)]");
  link.classList.remove("border-[var(--accent)]", "text-[var(--text)]", "bg-white/5");
}

// Y para las líneas del indicador:
line.classList.toggle("active", i === activeIndex);
```

**Problemas de este enfoque:**

1. **Acoplamiento entre lógica y presentación**: el script debe conocer los nombres exactos de las clases y sus combinaciones.
2. **Riesgo de inconsistencia**: si se renombra una clase o se cambia una variable, el script deja de aplicar el estilo correcto.
3. **Difícil de escalar**: cada nuevo estado implica más ramas de `classList.add` / `remove`.
4. **Flash visual inicial**: al renderizar primero con el estilo "activo" por defecto, todos los enlaces se veían resaltados hasta que el script corregía el estado.

#### Ahora: estados declarativos con `data-is-active`

El script solo asigna un atributo de datos:

```js
const activeId = headings[activeIndex].id;

tocLinks.forEach((link) => {
  link.dataset.isActive = link.getAttribute("href") === "#" + activeId;
});

tocLines.forEach((line, i) => {
  line.dataset.isActive = i === activeIndex;
});
```

Y CSS define la apariencia de cada estado:

```css
/* Estado base: inactivo */
.toc-item a {
  border-left: 2px solid transparent;
  color: var(--muted);
  background: unset;
}

/* Interacción */
.toc-item a:hover {
  border-left-color: var(--grid-line-strong);
  color: var(--text);
}

/* Estado activo */
.toc-item a[data-is-active="true"] {
  border-left: 2px solid var(--accent);
  color: var(--text);
  background: #ffffff0d;
}

/* El hover no debe anular el borde activo */
.toc-item a[data-is-active="true"]:hover {
  border-left-color: var(--accent);
}
```

```css
/* Indicador compacto */
.toc-indicator-line {
  width: 12px;
  background: var(--muted);
}

.toc-indicator-line[data-is-active="true"] {
  width: 24px;
  background: var(--accent);
}
```

#### Ventajas de este enfoque

| Aspecto | Con clases dinámicas | Con `data-is-active` |
|---|---|---|
| **Responsabilidad del script** | Conocer y mutar clases CSS | Solo escribir un booleano en el dataset |
| **Mantenimiento del estilo** | Disperso entre JS y CSS | Centralizado en CSS |
| **Flash visual inicial** | Todos los enlaces se veían activos | El estado base es inactivo por defecto |
| **Escalabilidad** | Cada estado nuevo requiere más `classList` | Solo se añade un selector `[data-*]` |
| **Consistencia** | Links y líneas usaban mecanismos distintos | Ambos usan `data-is-active` |
| **Colisiones con hover** | Tailwind `hover:` podía anular el estado activo | CSS controla explícitamente ambos estados |

#### Por qué dataset y no clases

La elección de `dataset` sobre clases dinámicas refuerza la separación de responsabilidades:

- **JavaScript** responde a eventos y calcula estado: "este enlace corresponde al heading activo".
- **HTML** expresa ese estado: `data-is-active="true"`.
- **CSS** decide cómo se ve: selectores por atributo.

Este modelo hace que el componente sea más predecible, más fácil de testear y más sencillo de rediseñar en el futuro, porque el estado es visible en el DOM y la presentación vive en un solo lugar.

---

## 6. Cambios en SidebarNav

El componente también evolucionó para integrarse mejor con Flexbox:

### Breadcrumb con truncamiento

Cada ítem del breadcrumb tiene un ancho máximo de `12rem`. Si el título es más largo, se trunca con ellipsis y se muestra completo en el atributo `title`:

```astro
<li class="flex min-w-0 max-w-[12rem] flex-shrink items-center gap-2">
  <a href={crumb.href} class="block truncate" title={crumb.title}>
    {crumb.title}
  </a>
</li>
```

La lista usa `flex-nowrap` y `overflow-x-auto`, por lo que si el conjunto de ítems no cabe, el usuario puede desplazarse horizontalmente en lugar de romper el layout.

### Header unificado

El botón de abrir el sidebar se movió del borde inferior fijo al header, junto a los demás toggles:

```astro
<header class="sticky top-0 z-40 flex items-center justify-between gap-4 px-6 py-5 ...">
  <div class="min-w-0 flex-1">
    <Breadcrumb ... />
  </div>
  <div class="flex flex-shrink-0 items-center gap-2">
    <button id="sidebar-toggle" ... md:hidden>
      <MenuSVG />
    </button>
    <ContentWidthToggle />
    <ThemeToggle />
  </div>
</header>
```

Esto mantiene los controles siempre accesibles y evita que el botón de ancho de contenido parezca pertenecer al breadcrumb.

### De `fixed` a `sticky`

```astro
<!-- Antes -->
<aside class="fixed inset-y-0 left-0 z-50 hidden w-64 -translate-x-full ...">

<!-- Ahora -->
<aside class="sticky flex-shrink-0 max-w-64 inset-y-0 left-0 z-50 hidden -translate-x-full ...">
```

- En escritorio, el sidebar es `sticky` y ocupa su lugar natural dentro del flujo flex.
- En móvil, el mismo elemento se convierte en un panel deslizable controlado por JavaScript, manteniendo una única estructura de markup.

### Control del ancho

- `max-w-64` limita el crecimiento del sidebar.
- `md:w-full` permite que ocupe el ancho disponible dentro de su celda flex en escritorio.

---

## 7. Modo de ancho de contenido: `constrained` y `full`

Después de estabilizar el layout en Flexbox, se añadió un control de ancho de contenido en el header. El objetivo era permitir dos experiencias distintas:

- **`constrained`**: el contenido está limitado a `max-w-6xl` (72 rem). Es el modo por defecto y está pensado para lectura.
- **`full`**: el contenedor principal ocupa todo el ancho disponible. Permite tener ambos paneles laterales expandidos simultáneamente.

### 7.1 Implementación del toggle

El toggle se implementa en `ContentWidthToggle.astro` y persiste la preferencia en `localStorage` bajo la clave `content-width-mode`:

```js
const saved = localStorage.getItem("content-width-mode");
const mode = saved === "full" ? "full" : "constrained";
```

Ambos elementos, el wrapper del contenido y el botón, comparten el mismo estado mediante `data-content-mode`:

```html
<div id="content-wrapper" data-content-mode="constrained">...</div>
<button id="content-width-toggle" data-content-mode="constrained">...</button>
```

### 7.2 Por qué ocultar el toggle en viewports pequeños

El botón de ancho se oculta por debajo de una anchura mínima configurable mediante CSS. El valor por defecto es 1280px:

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

La razón es pragmática: en pantallas muy pequeñas el modo `full` no ofrece ventaja real y el toggle solo generaría confusión. Los paneles laterales ya tienen su propio comportamiento responsive en móvil.

> **Nota**: el selector usa `:global(#content-width-toggle)` porque el botón pertenece a otro componente Astro. Sin `:global(...)`, el estilo scoped del layout no llegaría al botón.

---

## 8. Reglas de convivencia entre Sidebar y TOC

### 8.1 Filosofía de los dos modos

- **Modo `full`**: sin restricciones. Ambos paneles pueden expandirse. El usuario tiene el control total.
- **Modo `constrained`**: el contenido tiene ancho limitado, por lo que no debería competir con dos paneles anchos a la vez. Solo uno de los dos paneles laterales puede estar expandido en su forma completa.

### 8.2 Prioridad del TOC

En documentos con encabezados, el TOC tiene prioridad. Si al cargar la página ambos paneles están expandidos en modo `constrained`, el layout colapsa automáticamente el Sidebar:

```js
if (sidebarExpanded && tocExpanded) {
  setSidebarMode("collapsed");
}
```

### 8.3 Alternancia mediante el botón de Sidebar

En modo `constrained`, el botón de toggle del Sidebar no se queda inerte. Alterna entre dos estados válidos:

1. **Sidebar expandido + TOC compacto**.
2. **Sidebar colapsado + TOC estándar**.

Para que esto funcione, el layout intercepta el click del botón en **fase de captura** (`addEventListener(..., true)`), antes de que el módulo del Sidebar reaccione. Así, el layout puede cambiar el TOC a `compact` antes de que el Sidebar se expanda, evitando que el `MutationObserver` lo revierta inmediatamente.

### 8.4 Transición automática a `full`

Cuando el usuario cambia de `constrained` a `full`, el layout expande ambos paneles automáticamente para aprovechar el ancho extra:

```js
function onContentModeChange() {
  const currentMode = wrapper.getAttribute("data-content-mode") || "constrained";
  if (currentMode === "full" && lastContentMode !== "full") {
    expandAll();
  }
  lastContentMode = currentMode;
  enforceSingleExpanded();
}
```

Al volver a `constrained`, se vuelve a aplicar la restricción de un solo panel expandido.

### 8.5 Páginas sin TOC

Si la página no tiene encabezados, el TOC no se renderiza. En ese caso, el Sidebar puede permanecer expandido sin restricciones incluso en modo `constrained`.

---

## 9. Persistencia y estado temporal

Una decisión importante fue separar el **estado de preferencia** del **estado temporal impuesto por el layout**:

- Los botones de toggle persisten en `localStorage` cuando el usuario interactúa directamente.
- El layout nunca escribe en `localStorage` cuando aplica una restricción.

Esto evita que una regla de layout contamine la preferencia del usuario. Si el usuario vuelve a modo `full`, el Sidebar recupera el estado que tenía antes de la restricción.

Además, para evitar el *flash* de cambio de estado al cargar una página, cada componente con estado persistente incluye un pequeño script `is:inline` que lee `localStorage` y ajusta los atributos `data-*` antes de que el navegador pintee el contenido. Así el usuario nunca ve el estado por defecto del HTML estático durante un instante.

---

## 10. Conclusión

La migración de Grid a Flexbox resolvió dos problemas principales:

1. **Layout condicional limpio**: los tres componentes principales ahora se muestran solo cuando son necesarios, y el contenido central se adapta automáticamente al espacio disponible.
2. **Control refinado del dimensionamiento**: combinando `flex-shrink-0`, `flex-1` y `min-w-0`, cada pieza respeta su rol sin provocar desbordamientos ni encogimientos indebidos.

Además, se ajustaron dos comportamientos responsive:

- **Header sticky universal**: el header pasa de `md:sticky` a `sticky` para que el breadcrumb permanezca fijo también en móvil. El z-index del header se redujo a `z-40` para que el sidebar móvil, al abrirse, pueda desplegarse por encima y ocultar el breadcrumb.
- **TOC oculto en móvil**: el contenedor del TOC solo se muestra a partir de 768px. El contenedor exterior usa `hidden md:block` y las propiedades dimensionales (`sticky`, `flex-shrink-0`, altura) se aplican solo a partir de `md`. El atributo `data-is-visible="true"` activa `display: block` dentro de la misma media query.
- **Sidebar móvil como capa superior**: el sidebar móvil usa `z-[60]`, el backdrop `z-50`, y el header `z-40`. Al abrirse, el sidebar cubre el contenido y el breadcrumb. El ancho en móvil se limita a `min(80vw, 20rem)` para no desbordar la pantalla.

Paralelamente, el TOC evolucionó de un simple listado a una herramienta de navegación dual, mejorando la densidad de información y la experiencia de lectura. El cambio más notable en la implementación del TOC fue **prescindir por completo de la manipulación de clases CSS desde JavaScript** para expresar estados. En lugar de usar `classList.add`, `classList.remove` o `classList.toggle`, el script ahora escribe atributos de datos (`data-is-active`) y deja que CSS decida la apariencia mediante selectores por atributo.

Finalmente, la introducción de los modos `constrained` y `full` añadió una capa de coordinación entre componentes. Mediante `MutationObserver`, `localStorage` y atributos `data-*`, el layout puede aplicar restricciones inteligentes sin acoplar la lógica de los componentes individuales.

### Reglas y técnicas clave utilizadas

| Regla / técnica | Función en el layout |
|---|---|---|
| `display: flex` | Distribuye los tres componentes en una fila adaptable. |
| `flex-shrink-0` | Impide que el sidebar y el TOC se encojan. |
| `flex-1` | Hace que el contenido principal ocupe el espacio sobrante. |
| `min-w-0` | Permite que el contenido principal se ajuste sin desbordar el viewport. |
| `sticky` | Mantiene sidebar y TOC visibles durante el scroll sin sacarlos del flujo. |
| `position: absolute` + `top: X%` | Posiciona las líneas del indicador del TOC proporcionalmente a la altura del contenedor. |
| `data-toc-mode` + `localStorage` | Alterna entre modo estándar y compacto de forma persistente. |
| `data-is-active` + CSS | Separa el estado activo de la presentación en los enlaces y líneas del TOC. |
| `data-content-mode` | Controla el modo de ancho de contenido (`constrained` / `full`). |
| `data-sidebar-mode` + `MutationObserver` | Coordina el Sidebar con el TOC según el modo de ancho. |
| `addEventListener(..., true)` | Permite al layout actuar antes del listener del componente en el toggle del Sidebar. |

---

## 11. Documentos relacionados

- `src/content/nanobook-project/layout/comunicacion-estado-componentes.md` — explica en detalle cómo se comunican los estados entre TOC, Sidebar y ContentWidthToggle.
- `src/content/nanobook-project/table-of-contents/dataset-estado-toc.md` — profundiza en el cambio de `classList` a `data-is-active` en el TOC.

---

## 12. Commits relacionados

- `d75bdb0` — `feat: enforce single expanded panel in constrained mode, allow both in full mode`
- `b505506` — `feat: expand both sidebar and TOC when switching to full mode`
