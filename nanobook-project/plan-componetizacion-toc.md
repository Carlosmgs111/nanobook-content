---
title: "Plan de componetización del TableOfContents"
description: "Propuesta para dividir el componente TableOfContents de nanobook en subcomponentes más pequeños, manteniendo la lógica de estado declarativa y los scripts inline de Astro."
date: 2026-08-04
author: "Nanobook"
tags: ["astro", "toc", "refactor", "componentes", "arquitectura"]
draft: false
index: false
---

# Plan de componetización del `TableOfContents`

## 1. Diagnóstico actual

El componente `src/components/TableOfContents.astro` actualmente acumula varias responsabilidades:

1. **Renderizado del contenedor y layout**: `toc-root`, altura, sticky, etc.
2. **Indicador compacto**: líneas que representan la posición de los headings en el documento.
3. **Navegación estándar**: lista de enlaces a los headings.
4. **Botón de modo**: toggle para cambiar entre `standard` y `compact` con iconos dinámicos.
5. **Lógica de scroll spy**: detectar qué heading está activo según el scroll.
6. **Posicionamiento de líneas**: traducir la posición de los headings al alto del contenedor.
7. **Persistencia de modo**: leer/escribir `localStorage`.
8. **Estilos CSS**: todo el CSS del TOC en un único bloque `<style>`.

Esto lo convierte en un componente largo y difícil de testear, reutilizar o modificar sin afectar otras partes.

## 2. Objetivo de la componetización

Separar el TOC en piezas más pequeñas y especializadas que:

- Tengan una única responsabilidad clara.
- Sean más fáciles de testear y mantener.
- Permitan reutilizar partes (por ejemplo, el indicador compacto en otro contexto).
- Mantengan la estrategia de estado declarativa basada en `data-*` y CSS.
- No rompan el funcionamiento actual ni la experiencia de usuario.

## 3. Propuesta de estructura

### 3.1 Opción recomendada: componentes de presentación + orquestador

La opción más sencilla y compatible con Astro es dividir el componente en **subcomponentes de presentación** y mantener un **orquestador principal** que contenga la lógica inline.

```
src/
  components/
    TableOfContents/
      index.astro          # Orquestador: contenedor, estado y script
      TocIndicator.astro   # Indicador compacto: líneas
      TocNav.astro         # Lista de enlaces a headings
      TocToggle.astro      # Botón con iconos dinámicos
```

#### Ventajas

- Cada subcomponente es fácil de entender y modificar.
- No requiere cambios en la arquitectura de scripts de Astro.
- Se mantiene el `data-*` + CSS como mecanismo de estado.
- Facilita futuras variantes (por ejemplo, un TOC sin modo compacto).

#### Desventajas

- El script inline sigue viviendo en el orquestador y sigue siendo relativamente largo.
- La lógica de scroll spy no está en un archivo separado testeable.

### 3.2 Opción alternativa: extraer lógica a un módulo TypeScript

Si se quiere testear la lógica de scroll spy o reutilizarla en otros componentes, se puede mover a un módulo:

```
src/
  components/
    TableOfContents/
      index.astro
      TocIndicator.astro
      TocNav.astro
      TocToggle.astro
  lib/
    scroll-spy.ts        # Lógica pura y genérica: scroll spy
    toc.ts               # Lógica específica del TOC (opcional)
```

#### Ventajas

- La lógica se puede testear unitariamente.
- Separa completamente presentación de lógica.
- El orquestador Astro queda más limpio.

#### Desventajas

- Astro no empaqueta automáticamente módulos importados dentro de `<script is:inline>`.
- Para usar `import` en el cliente, el script debe ser `type="module"` y Vite/Astro deben resolver la ruta. Esto es posible, pero añade complejidad.
- Si se usa un módulo, puede perderse la ejecución inmediata sin hidratación que ofrece `is:inline`.

#### Compromiso recomendado (enfoque híbrido)

Extraer **solo la lógica pura y genérica** a `src/lib/scroll-spy.ts` (por ejemplo `findActiveIndex`), manteniendo en el orquestador Astro las funciones que interactúan directamente con el DOM y que son específicas del TOC:

- `initTocMode`, `setTocMode` — modo del TOC.
- `collectHeadings` — resolución de links con hash a headings.
- `positionIndicatorLines` — aplicación de estilos al indicador visual.
- `updateActive` — escritura de `data-is-active` en links y líneas.

Esto permite testear y reutilizar la utilidad genérica sin perder la ejecución inmediata del script del TOC.

### 3.3 Opción avanzada: custom element

Convertir todo el TOC en un custom element (`<toc-widget>`) y cada subcomponente en elementos internos.

```
src/
  components/
    toc/
      TocWidget.astro      # Define <toc-widget>
      TocIndicator.astro   # Define <toc-indicator>
      TocNav.astro         # Define <toc-nav>
      TocToggle.astro      # Define <toc-toggle>
```

#### Ventajas

- Encapsulación completa.
- Reutilizable fuera de Astro.
- Comportamiento autocontenido.

#### Desventajas

- Añade mucho JavaScript al cliente.
- Mayor complejidad.
- Riesgo de conflictos con SSR y hidratación.
- Probablemente excesivo para este caso.

## 4. División propuesta de responsabilidades

### 4.1 `TableOfContents/index.astro` (orquestador)

Responsabilidades:

- Recibir `headings` como prop.
- Renderizar el contenedor `.toc-root`.
- Importar y usar `TocIndicator`, `TocNav` y `TocToggle`.
- Contener el script que inicializa:
  - Modo del TOC (`standard` / `compact`).
  - Scroll spy (importando `findActiveIndex` desde `src/lib/scroll-spy.ts`).
  - Posicionamiento de líneas.
  - Persistencia en `localStorage`.

Ejemplo de estructura:

```astro
---
import TocIndicator from "./TocIndicator.astro";
import TocNav from "./TocNav.astro";
import TocToggle from "./TocToggle.astro";

interface Heading {
  depth: number;
  slug: string;
  text: string;
}

interface Props {
  headings: Heading[];
}

const { headings } = Astro.props;
---

<div class="h-[calc(100vh-10rem)] sticky top-32 flex-shrink-0">
  <div
    class="toc-root md:block"
    data-toc-mode="standard"
    data-is-visible={headings.length > 0}
  >
    <TocIndicator headings={headings} />
    <TocNav headings={headings} />
  </div>
</div>

<script is:inline>
  // Inicialización de modo, scroll spy y posicionamiento
</script>
```

### 4.2 `TocIndicator.astro`

Responsabilidades:

- Recibir `headings`.
- Renderizar el contenedor `.toc-indicator`.
- Generar una línea por cada heading.

```astro
---
interface Heading {
  slug: string;
}

interface Props {
  headings: Heading[];
}

const { headings } = Astro.props;
---

<div class="toc-indicator" aria-hidden="true">
  {headings.map((heading) => (
    <div class="toc-indicator-line" data-slug={heading.slug} />
  ))}
</div>
```

### 4.3 `TocNav.astro`

Responsabilidades:

- Recibir `headings`.
- Renderizar `<nav class="toc-nav">`.
- Renderizar el encabezado con el título y el botón de toggle.
- Generar la lista de enlaces con indentación según `depth`.

```astro
---
import TocToggle from "./TocToggle.astro";

interface Heading {
  depth: number;
  slug: string;
  text: string;
}

interface Props {
  headings: Heading[];
}

const { headings } = Astro.props;
---

<nav class="toc-nav" aria-label="En esta página">
  <div class="flex items-end mb-3">
    <span class="tech-label block w-full">En esta página</span>
    <TocToggle />
  </div>
  <ul class="space-y-1 border-l border-[var(--grid-line-strong)]">
    {headings.map((heading) => (
      <li
        class="toc-item"
        data-depth={heading.depth}
        data-slug={heading.slug}
        style={`padding-left: ${(heading.depth - 2) * 0.75}rem`}
      >
        <a href={`#${heading.slug}`} class="block py-1 pl-3 text-sm">
          {heading.text}
        </a>
      </li>
    ))}
  </ul>
</nav>
```

### 4.4 `TocToggle.astro`

Responsabilidades:

- Renderizar el botón con ambos iconos.
- No contener lógica de estado; solo presentación.

```astro
---
import ListSVG from "../../icons/list.svg";
import ListCompactSVG from "../../icons/list-compact.svg";
---

<button
  id="toc-toggle"
  type="button"
  class="..."
  aria-label="Cambiar modo del índice"
  aria-pressed="false"
  data-toc-mode="standard"
>
  <span class="toc-toggle-icon-standard" aria-hidden="true">
    <ListSVG />
  </span>
  <span class="toc-toggle-icon-compact" aria-hidden="true">
    <ListCompactSVG />
  </span>
</button>
```

## 5. Manejo de estilos

### 5.1 Opción A: estilos en cada subcomponente

Cada subcomponente lleva su propio bloque `<style>`. Esto mantiene los estilos cerca del markup.

#### Ventajas

- Estilos co-localizados.
- Fácil de modificar un componente sin tocar otros.

#### Desventajas

- Puede haber duplicación de variables o reglas comunes.
- Astro puede generar selectores con mayor especificidad si no se tiene cuidado.

### 5.2 Opción B: estilos centralizados en el orquestador

Mantener todos los estilos del TOC en `TableOfContents/index.astro`.

#### Ventajas

- Un solo lugar para ver todos los estilos del TOC.
- Evita duplicación.

#### Desventajas

- El orquestador sigue siendo largo.
- Pierde la ventaja de co-localización.

### 5.3 Opción recomendada

Estilos **co-localizados** en cada subcomponente, excepto reglas compartidas o de estado global que viven en el orquestador:

- `TocIndicator.astro`: estilos de `.toc-indicator` y `.toc-indicator-line`.
- `TocNav.astro`: estilos de `.toc-nav` y `.toc-item`.
- `TocToggle.astro`: estilos de `#toc-toggle` y visibilidad de iconos.
- `TableOfContents/index.astro`: estilos de `.toc-root`, modos (`data-toc-mode`), y visibilidad (`data-is-visible`).

## 6. Manejo del script inline

### 6.1 Estructura sugerida dentro del orquestador

Organizar el IIFE en funciones con una sola responsabilidad, importando desde `src/lib/scroll-spy.ts` la lógica pura y genérica:

```js
import { findActiveIndex } from "../../lib/scroll-spy.ts";

(function () {
  const tocRoot = document.querySelector(".toc-root");
  if (!tocRoot || tocRoot.offsetParent === null) return;

  const toggle = document.getElementById("toc-toggle");

  initTocMode(tocRoot, toggle);
  initScrollSpy(tocRoot);
})();

function initTocMode(root, toggle) { ... }
function initScrollSpy(root) { ... }
function setTocMode(root, toggle, mode) { ... }
function positionIndicatorLines(root) { ... }
function updateActiveState(root) { ... }
```

### 6.2 Posible mejora futura: módulo externo

Si la lógica del TOC crece, se puede mover helpers específicos a `src/lib/toc.ts`, manteniendo la lógica genérica en `src/lib/scroll-spy.ts`.

Para integrarlo con Astro sin perder la ejecución inmediata, se puede usar un script de módulo:

```html
<script type="module">
  import { initToc } from "/src/lib/toc.ts";
  initToc();
</script>
```

Esto requiere verificar que Vite resuelva correctamente la ruta en el cliente. En Astro, una forma más segura es importar el módulo en el frontmatter y usar `client:load` en una island, pero eso cambia el modelo actual.

## 7. Pasos de implementación sugeridos

1. **Crear la carpeta** `src/components/TableOfContents/`.
2. **Mover** `TableOfContents.astro` a `TableOfContents/index.astro`.
3. **Crear** `TocIndicator.astro`, `TocNav.astro` y `TocToggle.astro` con su markup inicial.
4. **Mover** los estilos correspondientes a cada subcomponente.
5. **Actualizar** `TableOfContents/index.astro` para importar y usar los subcomponentes.
6. **Refactorizar** el script en funciones con responsabilidad única.
7. **Actualizar** todas las importaciones que referencian el componente antiguo (actualmente solo `Layout.astro` importa `TableOfContents`).
8. **Extraer** a `src/lib/scroll-spy.ts` la lógica pura y genérica (por ejemplo `findActiveIndex`).
9. **Ejecutar build** y verificar visualmente ambos modos, scroll spy y persistencia.
10. **Opcional**: extraer helpers específicos del TOC a `src/lib/toc.ts` si el script sigue siendo muy largo.

## 8. Qué NO cambiar

Para mantener la estabilidad del componente, se recomienda conservar:

- El uso de `data-toc-mode` y `data-is-active` para el estado.
- La estrategia de iconos renderizados ambos y ocultados por CSS.
- El `localStorage` para persistir el modo.
- El scroll spy con búsqueda binaria (`findActiveIndex`).
- El posicionamiento de líneas con `position: absolute` y `top: X%`.

## 9. Conclusión

La componetización del TOC no requiere una reescritura profunda. La opción más pragmática es dividirlo en **tres subcomponentes de presentación** (`TocIndicator`, `TocNav`, `TocToggle`) y mantener un **orquestador principal** con el script refactorizado. Además, la lógica pura y genérica del scroll spy se extrae a `src/lib/scroll-spy.ts`, lo que permite testearla y reutilizarla en otros componentes. Esto reduce la complejidad del archivo principal, mejora la mantenibilidad y conserva la arquitectura declarativa basada en `data-*` y CSS que ya se estableció.
