---
title: "Icono dinámico en el toggle del TOC: renderizar ambos y mostrar según data-toc-mode"
description: "Estrategia usada para cambiar el icono del botón de modo del TableOfContents sin manipular el DOM con JavaScript: renderizar ambos SVGs y controlar su visibilidad con data attributes y CSS."
date: 2026-08-04
author: "Nanobook"
tags: ["astro", "toc", "svg", "css", "javascript", "dataset", "iconos"]
draft: false
index: false
---

# Icono dinámico en el toggle del TOC

## 1. Contexto

El componente `TableOfContents` (TOC) de `nanobook` tiene dos modos de visualización:

- **Modo `standard`**: muestra la lista completa de encabezados.
- **Modo `compact`**: muestra un indicador vertical con pequeñas líneas que representan la posición de cada encabezado en el documento.

El usuario cambia de modo mediante un botón de toggle ubicado dentro del propio TOC. Ese botón debe mostrar un icono que indique visualmente a qué modo se va a cambiar al presionarlo.

## 2. El problema

Inicialmente el botón solo mostraba un icono de lista genérico (`list.svg`). Al cambiar a modo compacto, el icono no se actualizaba, por lo que el botón no reflejaba el estado actual del TOC ni el modo al que se cambiaría.

La opción más obvia habría sido usar JavaScript para reemplazar el contenido del botón o alternar clases que muestren un icono u otro. Sin embargo, ese enfoque vuelve a acoplar la lógica de estado con la presentación, justo lo que se había evitado al migrar el estado activo del TOC a `data-is-active`.

## 3. La estrategia: renderizar ambos iconos, mostrar solo uno

En lugar de manipular el DOM desde JavaScript para cambiar el icono, el componente **renderiza ambos SVGs al mismo tiempo** dentro del botón y deja que CSS decida cuál se muestra según el valor de `data-toc-mode` del botón.

### 3.1 Marcado en Astro

```astro
---
import ListSVG from "../icons/list.svg";
import ListCompactSVG from "../icons/list-compact.svg";
---

<button
  id="toc-toggle"
  type="button"
  class="hidden h-9 w-9 items-center justify-center border border-[var(--grid-line-strong)] text-[var(--muted)] hover:border-[var(--accent)] hover:text-[var(--accent)] md:flex"
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

### 3.2 CSS para alternar visibilidad

```css
#toc-toggle {
  position: relative;
}

#toc-toggle[data-toc-mode="standard"] .toc-toggle-icon-standard,
#toc-toggle[data-toc-mode="compact"] .toc-toggle-icon-compact {
  display: none;
}
```

**Nota sobre la semántica**: el icono visible representa el **modo destino**, no el modo actual. Si el TOC está en modo `standard`, el botón muestra el icono compacto para indicar que presionarlo cambiará a modo compacto. Si está en modo `compact`, muestra el icono de lista estándar para indicar que se volverá al modo estándar. Esto sigue la convención común de los botones de toggle.

### 3.3 JavaScript: solo actualiza el estado

```js
function setTocMode(mode) {
  tocRoot.setAttribute("data-toc-mode", mode);
  toggle.setAttribute("data-toc-mode", mode);
  toggle.setAttribute("aria-pressed", mode === "compact");
  if (typeof localStorage !== "undefined") {
    localStorage.setItem("toc-mode", mode);
  }
}

const mode = saved === "compact" ? "compact" : "standard";
setTocMode(mode);

toggle.addEventListener("click", function () {
  const current = tocRoot.getAttribute("data-toc-mode") || "standard";
  const next = current === "standard" ? "compact" : "standard";
  setTocMode(next);
});
```

## 4. ¿Por qué no reemplazar el icono con JavaScript?

| Enfoque | Desventaja |
|---|---|
| `innerHTML` dinámico | Riesgo de XSS si no se sanitiza; destruye referencias y listeners. |
| `classList.toggle` sobre un único icono | Acopla la lógica de estado con los nombres de clase del icono. |
| Cambiar el `src` de una `<img>` | Requiere peticiones adicionales y complica estilos inline. |
| Renderizar ambos + CSS | Mantiene el estado en el DOM, la presentación en CSS y la lógica mínima en JS. |

## 5. Ventajas de la estrategia

1. **Separación de responsabilidades**: JavaScript solo sabe el modo actual; CSS decide qué icono mostrar.
2. **Sin manipulación de DOM innecesaria**: no se crean ni destruyen nodos en cada clic.
3. **Sin riesgo de desincronización**: el icono visible siempre coincide con `data-toc-mode`.
4. **Accesible**: `aria-hidden="true"` en los iconos evita que los lectores de pantalla los anuncien; el botón usa `aria-label` y `aria-pressed` para comunicar el estado.
5. **Predecible**: inspeccionar el botón revela inmediatamente el modo activo.
6. **Extensible**: si se añade un tercer modo, basta agregar otro icono y otro selector CSS.

## 6. Iconos usados

- **`list.svg`**: representa el modo estándar con una lista de tres líneas de distintas longitudes.
- **`list-compact.svg`**: representa el modo compacto con una barra vertical y líneas horizontales cortas, simbolizando la lista comprimida en un indicador estrecho.

## 7. Relación con la estrategia de estado del TOC

Este patrón es coherente con la gestión del estado activo de los enlaces y líneas del TOC mediante `data-is-active`:

- **JavaScript** escribe el estado en atributos de datos (`data-toc-mode`, `data-is-active`).
- **CSS** interpreta esos atributos para aplicar la presentación correcta.
- **El DOM** permanece como la fuente de verdad del estado visible.

## 8. Impacto en el rendimiento

A primera vista, renderizar dos SVGs en lugar de uno podría parecer un desperdicio de recursos, pero en la práctica el impacto es despreciable y, en muchos casos, favorable frente a alternativas que mutan el DOM.

### ¿Por qué no es un problema?

1. **SVGs muy pequeños**: cada icono tiene solo un `viewBox` de 24×24 y un puñado de elementos `<line>`. El número total de nodos añadidos al DOM es de aproximadamente 8–10 nodos por icono.
2. **Sin peticiones de red**: al ser SVGs inline importados por Astro, no se realizan peticiones HTTP adicionales. El navegador ya tiene el markup en el HTML generado estáticamente.
3. **El icono oculto no se pinta**: el SVG con `display: none` (o su contenedor `span`) no participa en el layout ni en el paint. El navegador lo mantiene en memoria, pero no consume ciclos de renderizado.
4. **Sin mutaciones de DOM en el clic**: al no reemplazar `innerHTML` ni crear/destruir nodos, se evitan reflows y repaints costosos cada vez que el usuario cambia de modo.

### Comparación con alternativas

| Enfoque | Costo en clic | Nodos en reposo | Peticiones de red |
|---|---|---|---|
| Renderizar ambos SVGs + CSS | Solo cambia un atributo de datos | ~16–20 nodos extra | Ninguna |
| `innerHTML = svgString` | Reflow + repaint + parseo de SVG | Menos nodos | Ninguna, pero riesgo de XSS |
| Cambiar `src` de `<img>` | Petición de imagen (cacheable) | Menos nodos | Posible petición extra |
| `classList.toggle` sobre icono único | Solo cambia una clase | Menos nodos | Ninguna, pero requiere más CSS complejo |

### ¿Cuándo preocuparse?

Si esta técnica se escalara a cientos de botones con múltiples iconos alternativos, el DOM crecería innecesariamente. Pero para **un único toggle con dos iconos**, la diferencia es imperceptible incluso en dispositivos de gama baja.

### Conclusión sobre rendimiento

Mantener dos SVGs renderizados y alternar su visibilidad con CSS es una estrategia **rendimiento-neutral** para este caso. De hecho, es preferible a mutar el DOM en cada interacción porque el costo de unos pocos nodos extra en el DOM es menor que el costo de destruir y recrear nodos continuamente.

## 9. Conclusión

Para cambiar el icono del toggle del TOC según el modo, la mejor estrategia fue **renderizar ambos iconos y controlar su visibilidad con CSS mediante `data-toc-mode`**. Esto evita manipulaciones de DOM en JavaScript, mantiene la lógica de estado desacoplada de la presentación y permite que el icono refleje fielmente el modo actual del TOC.
