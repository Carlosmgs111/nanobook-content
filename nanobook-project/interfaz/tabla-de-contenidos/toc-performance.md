---
title: "Evolución del componente TableOfContents: rendimiento y escalabilidad"
description: "Comparativa entre la implementación inicial del scroll spy y la solución actual optimizada, con análisis de rendimiento y posibles mejoras futuras."
date: 2026-08-03
author: "Nanobook"
tags: ["rendimiento", "table-of-contents", "scroll-spy", "astro", "frontend"]
draft: false
index: false
---

# Evolución del componente TableOfContents: rendimiento y escalabilidad

Este documento expone la evolución del componente `TableOfContents` de Nanobook, encargado de la navegación intra-página. Se comparan la solución inicial y la actual, se analiza el impacto en rendimiento y se proponen mejoras futuras para escenarios más exigentes.

## Contexto

El componente `TableOfContents` genera una lista de anclas a partir de los encabezados `h2` y `h3` del contenido de cada documento. Además, resalta dinámicamente el encabezado visible mientras el usuario hace scroll.

## Solución inicial: IntersectionObserver

La primera implementación utilizaba `IntersectionObserver` para detectar qué encabezado estaba visible en el viewport.

```js
const observer = new IntersectionObserver(
  (entries) => {
    entries.forEach((entry) => {
      if (entry.isIntersecting && entry.target.id) {
        setActive(entry.target.id);
      }
    });
  },
  {
    rootMargin: "-10% 0px -70% 0px",
    threshold: 0,
  }
);
```

### Problemas detectados

- **Saltos de encabezados**: con `rootMargin` restrictivo, encabezados pequeños o consecutivos a veces no cruzaban el área de intersección, por lo que el resaltado se saltaba ítems.
- **Poca predecibilidad**: el comportamiento dependía de la altura de cada sección y de la velocidad de scroll.
- **Dificultad para afinar**: ajustar los márgenes para cubrir todos los casos era complejo y propenso a errores.

## Solución actual: offsets cacheados + búsqueda binaria

La implementación actual mide las posiciones de los encabezados una sola vez al cargar la página y, durante el scroll, compara `window.scrollY` contra esas posiciones cacheadas.

### Cambios técnicos

1. **Medición única de posiciones**: la función `measure()` lee `offsetTop` de cada heading al cargar y guarda los valores en un array.

2. **Cálculo en scroll sin lecturas de layout**: durante el scroll solo se lee `window.scrollY`, que no fuerza reflow.

3. **Búsqueda binaria**: el encabezado activo se encuentra con complejidad `O(log n)` en lugar de `O(n)`.

4. **Actualizaciones DOM condicionales**: solo se actualizan las clases si el índice activo cambia.

5. **Early exit en mobile**: si el TOC no es visible (`offsetParent === null`), el script no se ejecuta.

6. **Recálculo ante cambios de layout**: se vuelven a medir los offsets en los eventos `resize` y `orientationchange`.

### Código clave

```js
let offsets = [];

function measure() {
  offsets = headings.map((heading) => heading.element.offsetTop);
  updateActive();
}

function findActiveIndex(scrollY) {
  const target = scrollY + offset;
  let left = 0;
  let right = offsets.length - 1;
  let result = 0;

  while (left <= right) {
    const mid = (left + right) >>> 1;
    if (offsets[mid] <= target) {
      result = mid;
      left = mid + 1;
    } else {
      right = mid - 1;
    }
  }

  return result;
}
```

## Beneficios de rendimiento

| Métrica | Solución anterior | Solución actual |
|---------|-------------------|-----------------|
| Lecturas de layout por frame de scroll | `O(n)` | `0` |
| Complejidad para encontrar el heading activo | `O(n)` lineal | `O(log n)` binaria |
| Reflows forzados durante scroll | Múltiples | Ninguno |
| Actualizaciones DOM por frame | Siempre | Solo si cambia el activo |
| Ejecución en mobile | Siempre | Solo si el TOC es visible |

### Impacto en carga

- Se fuerza **un único reflow** al cargar la página, durante `measure()`.
- El script es inline y pequeño; no bloquea el render inicial.
- La memoria adicional es `O(n)` para el array de offsets, despreciable incluso para documentos grandes.

### Escalabilidad estimada

| Cantidad de headings | Tiempo por frame de scroll | Percepción |
|----------------------|----------------------------|------------|
| < 50 | < 0.01 ms | Imperceptible |
| 100 | ~0.01 ms | Imperceptible |
| 1,000 | ~0.02 ms | Imperceptible |
| 10,000 | ~0.03 ms | Aún fluido |

> Los valores son aproximados y dependen del dispositivo, pero la aritmética sobre un array ordenado es muy barata para la CPU moderna.

## Posibles mejoras futuras

### 1. Recálculo con ResizeObserver

Si en el futuro el sitio incluye contenido dinámico —acordeones, imágenes lazy-load, bloques expandibles— los offsets pueden cambiar después del evento `load`. Una mejora robusta sería observar el contenedor principal con `ResizeObserver` y llamar a `measure()` solo cuando sea necesario.

```js
const main = document.querySelector("main");
if (main && typeof ResizeObserver !== "undefined") {
  const resizeObserver = new ResizeObserver(() => measure());
  resizeObserver.observe(main);
}
```

### 2. Virtualización del TOC para documentos enormes

Si un documento llega a tener miles de headings, renderizar todos los ítems en el DOM del TOC puede volverse costoso. Una solución sería virtualizar la lista, mostrando solo los ítems visibles dentro del panel del TOC.

### 3. Uso de IntersectionObserver híbrido

Para algunos casos, un enfoque híbrido puede ser más eficiente:

- `IntersectionObserver` con `rootMargin` cubriendo toda la ventana para detectar qué headings están visibles.
- Lógica de scroll para refinar cuál es el activo entre los visibles.

Esto delega la detección de visibilidad al navegador, que puede optimizarlo mejor que JavaScript puro.

### 4. Paginación o colapso de niveles

En documentos muy extensos, agrupar headings de nivel 3 bajo su padre de nivel 2 y colapsarlos por defecto mejora tanto la usabilidad como el rendimiento del DOM.

### 5. Web Worker para documentos masivos

Para documentos realmente enormes (miles de headings con cálculos complejos), el análisis de posiciones podría delegarse a un Web Worker. En la práctica actual esto es excesivo, pero es una opción si se llega a ese escenario.

## Conclusión

La solución actual ofrece un balance óptimo entre simplicidad y rendimiento:

- Es predecible y no se salta encabezados.
- No fuerza reflows durante el scroll.
- Escala bien para documentos grandes gracias a la búsqueda binaria.
- Es fácil de mantener y entender.

Para un sitio de documentación típico, esta implementación es más que suficiente. Las mejoras propuestas solo serían necesarias ante requisitos específicos de contenido dinámico o volúmenes de headings muy por encima de lo habitual.
