---
title: "Informe de estado: arquitectura basada en IDs/referencias"
description: "Análisis del gap entre el ideal de componentes desacoplados por referencias y la implementación actual de nanobook, con fases de refactorización propuestas."
date: 2026-08-21
author: "Nanobook"
tags: ["arquitectura", "ids", "referencias", "refactorización", "componentes"]
draft: false
index: false
---

# Informe de estado: arquitectura basada en IDs/referencias

## 1. Resumen ejecutivo

El proyecto ha documentado una arquitectura de componentes "basada en IDs" ([`id-based-component-architecture.md`](./id-based-component-architecture)) en la que cada pieza de UI se comunica con otras a través de referencias explícitas al DOM (IDs), atributos `data-*` y eventos personalizados. El objetivo implícito es que los componentes sean autónomos, agnósticos del contexto y reutilizables sin tener que conocer la estructura interna de otros.

La realidad actual es **parcial**: existen buenas prácticas aisladas (TOC, sidebar móvil) pero también una cantidad significativa de **IDs hardcodeadas, selectores globales y responsabilidades mezcladas**. Esto genera acoplamiento accidental, dificulta la reutilización y deja la arquitectura a medio camino entre el ideal de referencias y un modelo clásico de componentes acoplados al DOM.

Este informe describe el estado actual, identifica los puntos de fricción y propone una refactorización por fases que respete la decisión de mantener el paradigma de IDs/referencias, pero aplicándolo de forma real y consistente.

## 2. Estado actual: lo que ya funciona

| Aspecto | Estado | Ejemplos |
|---|---|---|
| **Comunicación por eventos custom** | Aplicado en TOC y coordinación layout | `toc:mode`, `sidebar:mode`, `toc:active`, `toc:headings`, `content:mode` |
| **Estado declarativo con `data-*`** | Consistente en sidebar, TOC y ancho de contenido | `data-sidebar-mode`, `data-toc-mode`, `data-content-mode`, `data-is-active` |
| **IDs parametrizadas en componentes atómicos** | Parcial | `OpenSidebar.astro`, `CloseSidebar.astro`, `SidebarModeToggle.astro`, `TocToggle.astro`, `TocNav.astro`, `TocIndicator.astro` |
| **Separación scroll-spy/helper genérico** | Correcto | `src/shared/utils/scroll-spy.ts` no toca el DOM |
| **Documentación de decisiones** | Buena | Docs en `src/content/nanobook-project/` |

## 3. Hallazgos: dónde la implementación se desvía del ideal

### 3.1 IDs hardcodeadas globales

Un componente que implementa el patrón de referencias debería recibir sus IDs desde fuera. Actualmente varios componentes definen IDs fijas en su propio markup y luego las usan directamente:

| ID fija | Definida en | Usada también en | Problema |
|---|---|---|---|
| `theme-toggle` | `ThemeToggle.astro` | scripts internos, CSS interno | No se puede tener más de un toggle de tema; la lógica vive dentro del botón |
| `content-width-toggle` | `ContentWidthToggle.astro` | scripts internos, `Header.astro` (CSS) | El toggle conoce `#content-wrapper`, `#sidebar`, `#toc-root` |
| `content-wrapper` | `LayoutShell.astro` | `ContentWidthToggle.astro`, `LayoutShell.astro`, `StateRestoreScripts.astro` | Cualquier cambio de ID rompe múltiples archivos |
| `header-inner` | `Header.astro` | `LayoutShell.astro`, `StateRestoreScripts.astro` | El header debería sincronizarse solo a través de datos, no por ID global |
| `sidebar` / `sidebar-backdrop` | `SidebarNav/index.astro` | `SidebarModeToggle.astro`, `SidebarHeader.astro`, `ContentWidthToggle.astro` | El modo del sidebar se hardcodea en varios estilos y scripts |
| `toc-root` / `toc-toggle` | `TableOfContents/index.astro` | `TocNav.astro`, `TocIndicator.astro`, `ContentWidthToggle.astro`, CSS de `TocNav.astro` | El TOC asume una única instancia por página |
| `preview-container` | `preview.astro` | script interno de la misma página | Menos grave, pero sigue siendo un ID global hardcodeado |

### 3.2 Componentes que saben demasiado

#### `ContentWidthToggle.astro`
Es el caso más representativo. Un botón que debería solo alternar un modo (`constrained`/`full`) actualmente:

- Lee y escribe `localStorage` para `content-width-mode`.
- Escucha `sidebar:mode` y `toc:mode`.
- Dispara eventos sobre `#sidebar` y `#toc-root`.
- Aplica la regla de exclusión "un solo panel expandido en modo constrained".
- Conoce la semántica de los modos de sidebar (`expanded`/`collapsed`) y TOC (`standard`/`compact`).

Esto convierte al toggle en un **coordinador de layout camuflado**, no en un botón agnóstico.

#### `ThemeToggle.astro`
Contiene:
- Renderizado del botón.
- Lógica de cambio de tema (`light`/`dark`).
- Lectura/escritura en `localStorage`.
- Restauración antes del primer paint.
- Estilos condicionales por icono.

Si mañana se necesita un segundo toggle de tema (por ejemplo, en un menú móvil o en preferencias), habría que duplicar lógica o extraerla a posteriori.

#### `LayoutShell.astro`
Contiene un script que escucha `content:mode` sobre `#content-wrapper` para actualizar `#header-inner`. Esto es una coordinación que debería hacerse mediante referencias pasadas como props o mediante un controlador de layout, no con IDs globales dentro del layout.

#### `StateRestoreScripts.astro`
Restaura el estado del tema y del ancho de contenido usando IDs globales. Parte de esta lógica está duplicada en los propios componentes (`ThemeToggle`, `ContentWidthToggle`).

### 3.3 Botones desarrollados directamente por los consumidores

No existe un componente `Button` base. Cada botón o enlace de acción se implementa de forma particular:

- `DocumentControls.astro` define inline los botones `Editar` y `Ver`.
- `ThemeToggle.astro` es un botón especializado.
- `ContentWidthToggle.astro` es otro botón especializado.
- `SidebarModeToggle.astro`, `TocToggle.astro`, `OpenSidebar.astro`, `CloseSidebar.astro` son todos botones con lógica propia.

Esto contradice el ejemplo que señala el usuario: si se necesita un botón base cuya lógica varíe según el entorno, hoy no hay dónde inyectar esa variación.

### 3.4 Referencias globales por clase en lugar de por ID

`DocumentEditor.astro` no usa IDs, pero tampoco usa referencias propias: busca `.editor-wrapper`, `[data-editor-container]`, `.status` y `[data-action='save']` dentro del wrapper. Si bien esto funciona porque solo hay un editor por página, no es reutilizable y es propenso a colisiones si la estructura cambia.

### 3.5 Inconsistencia en la restauración de estado

Cada componente con estado persistente implementa su propio script `is:inline` con su propia bandera `window.__nanobook*Restored`:

- `__nanobookThemeRestored` en `ThemeToggle.astro` y `StateRestoreScripts.astro`.
- `__nanobookContentWidthRestored` en `ContentWidthToggle.astro` y `StateRestoreScripts.astro`.
- `__nanobookSidebarRestored` en `SidebarNav/index.astro`.
- `__nanobookTocRestored` en `TableOfContents/index.astro`.

Esto dispersa la responsabilidad de "arranque del estado" y genera duplicación.

## 4. Puntos críticos a mejorar

1. **Contrato de referencias incompleto**: no todos los componentes reciben sus IDs/referencias por props.
2. **Acoplamiento al DOM global**: muchos scripts usan `document.getElementById` con IDs literales.
3. **Mezcla de responsabilidades**: botones que coordinan layout, layouts que escuchan eventos de botones, etc.
4. **Ausencia de abstracciones base**: no hay `Button`, `Toggle`, `StateManager` o `Controller` reutilizables.
5. **Persistencia dispersa**: cada componente habla directamente con `localStorage`.
6. **Coordinación layout ad hoc**: la lógica de exclusión entre sidebar y TOC vive en `ContentWidthToggle`.
7. **Dificultad para múltiples instancias**: IDs fijas impiden reutilizar toggles, sidebars o TOCs en la misma página.

## 5. Fases de refactorización propuestas

La propuesta respeta la decisión de mantener el paradigma de IDs/referencias. Lo que cambia es que las referencias se **inyectan desde fuera** y los componentes se vuelven **verdaderamente agnósticos**.

### Fase 0 — Inventario y contratos (sin cambios de comportamiento)

- Crear un mapa actualizado de todos los IDs/referencias del DOM.
- Documentar qué componente define cada ID y quién la consume.
- Decidir qué IDs son realmente globales (layout) y cuáles deberían ser parametrizables.
- Asegurar que `npm run build` pase antes de empezar.

### Fase 1 — Componente `Button` base agnóstico

Crear un componente base que no conozca su comportamiento:

```astro
---
// src/shared/components/Button.astro
interface Props {
  id?: string;
  type?: "button" | "submit" | "reset";
  variant?: "default" | "icon" | "ghost";
  ariaLabel?: string;
  ariaPressed?: boolean;
  ariaExpanded?: boolean;
  ariaControls?: string;
  data?: Record<string, string>;
  class?: string;
}
const { id, type = "button", variant = "default", ariaLabel, ariaPressed, ariaExpanded, ariaControls, data = {}, class: className = "" } = Astro.props;
---
<button
  id={id}
  type={type}
  class={`nb-button nb-button--${variant} ${className}`}
  aria-label={ariaLabel}
  aria-pressed={ariaPressed}
  aria-expanded={ariaExpanded}
  aria-controls={ariaControls}
  {...Object.fromEntries(Object.entries(data).map(([k, v]) => [`data-${k}`, v]))}
>
  <slot />
</button>
```

Migrar primero los casos más simples (`OpenSidebar`, `CloseSidebar`, `Editar`/`Ver`).

### Fase 2 — Extraer controladores de estado

Separar la UI del comportamiento. Crear módulos puros de TypeScript (sin JSX/Astro) que contengan la lógica de negocio:

| Controlador | Responsabilidad |
|---|---|
| `theme/controller.ts` | Leer/escribir preferencia de tema, aplicar clase en root, notificar cambios |
| `contentWidth/controller.ts` | Alternar modo, persistir, emitir `content:mode` |
| `sidebar/controller.ts` | Cambiar modo, persistir, emitir `sidebar:mode` |
| `toc/controller.ts` | Cambiar modo, persistir, emitir `toc:mode` |
| `layout/controller.ts` | Escuchar `content:mode`, `sidebar:mode`, `toc:mode`; aplicar regla de exclusión |

Los componentes Astro solo:
- Renderizan markup.
- Reciben referencias (`id`, `targetId`, `controller`).
- Delegan clicks al controlador.

### Fase 3 — Referencias inyectadas, no hardcodeadas

Refactorizar los componentes para que reciban todas sus referencias por props:

```astro
<!-- Antes -->
<ThemeToggle />

<!-- Después -->
<ThemeToggle id="theme-toggle" rootId="html" controller={themeController} />
```

```astro
<!-- Antes -->
<ContentWidthToggle />

<!-- Después -->
<ContentWidthToggle
  id="content-width-toggle"
  wrapperId="content-wrapper"
  controller={contentWidthController}
/>
```

El controlador de layout (no el toggle) será quien escuche `content:mode` y coordine sidebar/TOC mediante sus propias referencias.

### Fase 4 — Consolidar restauración de estado

Reemplazar los scripts `is:inline` dispersos por un único `StateBootstrap` o `StateRestoreScripts` que:

- Se ejecute una sola vez por carga de página.
- Lea las preferencias de `localStorage`.
- Aplique atributos iniciales a los elementos referenciados.
- No duplique lógica con los controladores.

Eliminar las banderas `window.__nanobook*Restored` y reemplazarlas por un mecanismo centralizado de inicialización.

### Fase 5 — Migrar el editor a referencias explícitas

`DocumentEditor.astro` debería recibir un `id` o una referencia, y todos sus selectores internos deberían partir de ese root:

```astro
<DocumentEditor id="main-editor" document={entry} previewHref={previewHref} />
```

```js
const wrapper = document.getElementById("main-editor");
const container = wrapper.querySelector("[data-editor-container]");
```

Esto elimina la dependencia de clases globales como `.editor-wrapper`.

### Fase 6 — Revisar CSS global por ID

Reemplazar selectores como `:global(#toc-root[data-toc-mode="compact"])` o `:global(#sidebar[data-sidebar-mode="collapsed"])` por clases/contextos parametrizables:

- Usar clases semánticas (`.toc-root--compact`, `.sidebar--collapsed`) o
- Aplicar los estilos dentro del componente que posee el contexto, recibiendo el modo como prop/estado.

Esto reduce el acoplamiento entre hojas de estilo de diferentes componentes.

### Fase 7 — Validación y documentación

- Ejecutar `npm run build` y corregir regresiones.
- Actualizar `id-based-component-architecture.md` y los documentos específicos de sidebar/TOC/layout.
- Añadir ejemplos de uso con referencias inyectadas.
- Considerar tests estáticos (lint) que detecten IDs hardcodeadas en scripts.

## 6. Recomendaciones inmediatas (próximos pasos concretos)

1. **Definir la interfaz `Button`** y migrar `OpenSidebar`, `CloseSidebar`, `Editar` y `Ver`.
2. **Crear un `ThemeController`** y hacer que `ThemeToggle` solo renderice y delegue.
3. **Parametrizar `ThemeToggle` y `ContentWidthToggle`** para que acepten `id` y `targetId` por props.
4. **Mover la coordinación sidebar/TOC desde `ContentWidthToggle` a un `LayoutController`** independiente.
5. **Consolidar la restauración de estado** en un único script de bootstrap.

## 7. Conclusión

La arquitectura basada en IDs/referencias es viable y ya está parcialmente implementada, pero la capa de UI todavía arrastra acoplamiento global. La refactorización no implica abandonar el enfoque: consiste en aplicarlo de verdad, haciendo que cada componente reciba sus referencias desde fuera, delegue la lógica en controladores y no asuma IDs globales.

El ejemplo del botón es el mejor punto de partida: crear un `Button` agnóstico e inyectarle comportamiento mediante referencias y controladores demuestra el valor del enfoque y sirve de patrón para el resto de componentes.

## 8. Referencias

- [`id-based-component-architecture.md`](./id-based-component-architecture)
- [`../interfaz/sidebar/sidebar-architecture.md`](../interfaz/sidebar/sidebar-architecture)
- [`../interfaz/tabla-de-contenidos/toc-architecture.md`](../interfaz/tabla-de-contenidos/toc-architecture)
- [`../interfaz/layout/coordinacion-eventos-layout.md`](../interfaz/layout/coordinacion-eventos-layout)
- [`../interfaz/layout/comunicacion-estado-componentes.md`](../interfaz/layout/comunicacion-estado-componentes)
