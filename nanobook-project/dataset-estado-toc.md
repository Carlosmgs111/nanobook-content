---
title: "De classList a data attributes: estado declarativo en el TOC"
description: "Por qué el TableOfContents de nanobook dejó de manipular clases CSS desde JavaScript para expresar estados y pasó a usar atributos de datos interpretados por CSS."
date: 2026-08-04
author: "Nanobook"
tags: ["astro", "toc", "css", "javascript", "estado", "dataset"]
draft: false
index: false
---

# De `classList` a `data attributes`: estado declarativo en el TOC

## 1. El problema: estados expresados con clases

Durante el desarrollo del `TableOfContents` (TOC), el estado activo de cada enlace se controlaba directamente desde JavaScript manipulando clases CSS:

```js
const isActive = link.getAttribute("href") === "#" + activeId;

if (isActive) {
  link.classList.remove("border-transparent", "text-[var(--muted)]");
  link.classList.add(
    "border-[var(--accent)]",
    "text-[var(--text)]",
    "bg-white/5"
  );
} else {
  link.classList.add("border-transparent", "text-[var(--muted)]");
  link.classList.remove(
    "border-[var(--accent)]",
    "text-[var(--text)]",
    "bg-white/5"
  );
}
```

Y para el indicador compacto se usaba:

```js
line.classList.toggle("active", i === activeIndex);
```

### 1.1 Por qué esto es frágil

1. **Acoplamiento entre lógica y presentación**: el script debe conocer los nombres exactos de las clases y sus combinaciones. Si se renombra una clase o se cambia una variable, el script deja de aplicar el estilo correcto.
2. **Difícil de escalar**: cada nuevo estado implica más ramas de `classList.add` / `classList.remove` / `classList.toggle`.
3. **Inconsistencia**: los enlaces y las líneas del indicador usaban mecanismos distintos (clases de Tailwind para unos, una clase `.active` para otras).
4. **Flash visual inicial**: al renderizar primero con el estilo "activo" por defecto, todos los enlaces se veían resaltados hasta que el script corrige el estado.
5. **Colisiones con hover**: las clases de Tailwind (`hover:border-[var(--grid-line-strong)]`) podían anular el estado activo al pasar el mouse.

## 2. La solución: prescindir de la administración de clases

El cambio de diseño fue simple en concepto: **el script ya no administra clases CSS para expresar estado**. Solo escribe un atributo de datos (`data-is-active`) y deja que CSS interprete ese estado.

### 2.1 El script solo dice "está activo o no"

```js
const activeId = headings[activeIndex].id;

tocLinks.forEach((link) => {
  link.dataset.isActive = link.getAttribute("href") === "#" + activeId;
});

tocLines.forEach((line, i) => {
  line.dataset.isActive = i === activeIndex;
});
```

Nota clave: el script no sabe qué color, borde o fondo corresponde al estado activo. Solo sabe **cuál** es el elemento activo.

### 2.2 CSS define la apariencia según el estado

```css
/* Enlaces del TOC: estado base inactivo */
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
/* Líneas del indicador compacto */
.toc-indicator-line {
  width: 12px;
  background: var(--muted);
}

.toc-indicator-line[data-is-active="true"] {
  width: 24px;
  background: var(--accent);
}
```

## 3. Comparación directa

| Aspecto | Con `classList` dinámica | Con `data-is-active` |
|---|---|---|
| **Responsabilidad del script** | Conocer y mutar clases CSS | Solo escribir un booleano en el dataset |
| **Mantenimiento del estilo** | Disperso entre JS y CSS | Centralizado en CSS |
| **Flash visual inicial** | Todos los enlaces se veían activos | El estado base es inactivo por defecto |
| **Escalabilidad** | Cada estado nuevo requiere más `classList` | Solo se añade un selector `[data-*]` |
| **Consistencia** | Links y líneas usaban mecanismos distintos | Ambos usan `data-is-active` |
| **Colisiones con hover** | Tailwind `hover:` podía anular el estado activo | CSS controla explícitamente ambos estados |
| **Testabilidad** | Hay que inspeccionar clases aplicadas | Basta leer `dataset.isActive` |
| **Acoplamiento** | Alto: el script depende de nombres de clase | Bajo: el script depende de una semántica de estado |

## 4. Separación de responsabilidades

Este enfoque refuerza una arquitectura clara:

- **JavaScript** responde a eventos y calcula estado: "este enlace corresponde al heading activo".
- **HTML** expresa ese estado: `data-is-active="true"`.
- **CSS** decide cómo se ve: selectores por atributo.

```
Evento de scroll
      |
      v
JS calcula activeIndex
      |
      v
JS escribe data-is-active="true|false"
      |
      v
CSS aplica estilos según [data-is-active]
```

## 5. Ventajas a largo plazo

1. **Rediseño sin tocar JavaScript**: si en el futuro el estado activo debe cambiar de color, tamaño o animación, basta modificar CSS.
2. **Estados más ricos**: si se necesitan estados intermedios (`data-is-active="near"`, `data-is-visited="true"`), el script solo escribe el atributo; CSS se encarga de la presentación.
3. **Menor riesgo de regressions**: al no tocar clases desde JS, se evitan errores como clases que no se remueven correctamente o nombres de clase que cambian.
4. **Mayor claridad en el DOM**: inspeccionar un elemento revela inmediatamente su estado semántico (`data-is-active="true"`), no solo una lista de clases opaca.

## 6. Cuándo aplicar este patrón

Este patrón es especialmente útil cuando:

- Un componente tiene varios estados visuales mutuamente excluyentes.
- El estado se calcula en JS pero la presentación es compleja.
- Se usa Tailwind o un sistema de utilidades donde las clases pueden colisionar.
- Se quiere mantener el script pequeño y libre de detalles de diseño.

## 7. Conclusión

La decisión de **prescindir de la administración de clases CSS desde JavaScript** y expresar estados mediante `data-is-active` fue una mejora arquitectónica clave en el `TableOfContents`. El script se simplificó notablemente, el estilo se centralizó en CSS y el componente se volvió más robusto, escalable y fácil de mantener. El estado pasó a ser una propiedad semántica del DOM, mientras que la presentación sigue siendo responsabilidad exclusiva de las hojas de estilo.
