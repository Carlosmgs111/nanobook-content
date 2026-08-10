---
title: "Coordinación de layout mediante eventos: estado actual y posibles mejoras"
description: "Documentación del enfoque actual para sincronizar Sidebar, TOC y modo de ancho de contenido mediante eventos personalizados, junto con sus ventajas, desventajas y alternativas de mejora."
date: 2026-08-08
author: "Nanobook"
tags: ["astro", "javascript", "state", "events", "sidebar", "toc", "layout", "content-width"]
draft: false
index: false
---

# Coordinación de layout mediante eventos: estado actual y posibles mejoras

## 1. Resumen del enfoque actual

El layout de `nanobook` coordina tres componentes principales:

- `SidebarNav` — navegación contextual del directorio actual.
- `TableOfContents` — mapa de encabezados de la página actual.
- `ContentWidthToggle` — alternador entre modo de contenido `constrained` y `full`.

Cada componente mantiene su propio estado en atributos `data-*` y `localStorage`. La comunicación entre ellos se realiza mediante eventos personalizados disparados sobre elementos DOM estables:

- `sidebar:mode` se dispara sobre `#sidebar`.
- `toc:mode` se dispara sobre `#toc-root`.

`ContentWidthToggle` actúa como coordinador: escucha ambos eventos y, cuando el modo de contenido es `constrained`, fuerza el estado opuesto del otro panel para mantener la regla de exclusión.

---

## 2. Archivos involucrados

| Archivo | Responsabilidad |
|---|---|
| `src/components/ContentWidthToggle.astro` | Alterna `content-width-mode`; escucha `sidebar:mode` y `toc:mode`; sincroniza el panel opuesto en modo `constrained`. |
| `src/components/SidebarNav/SidebarModeToggle.astro` | Alterna `sidebar-mode` al hacer click; emite `sidebar:mode`; actualiza su UI al escuchar `sidebar:mode`. |
| `src/components/TableOfContents/TocToggle.astro` | Alterna `toc-mode` al hacer click; emite `toc:mode`; actualiza su UI al escuchar `toc:mode`. |
| `src/layouts/Layout.astro` | Contenedor que aloja `SidebarNav`, `TableOfContents` y `ContentWidthToggle`. Ya no contiene lógica de coordinación. |

---

## 3. Flujo de eventos

### 3.1 El usuario expande el sidebar

```text
SidebarModeToggle (click)
  └─► set sidebar-mode = expanded
  └─► dispatch sidebar:mode { mode: "expanded" }
        └─► ContentWidthToggle escucha
              └─► wrapper está en constrained
              └─► set toc-mode = compact
              └─► dispatch toc:mode { mode: "compact", source: "content-width" }
                    └─► TocToggle escucha → actualiza su botón
                    └─► ContentWidthToggle ignora (source propio)
```

### 3.2 El usuario expande el TOC

```text
TocToggle (click)
  └─► set toc-mode = standard
  └─► dispatch toc:mode { mode: "standard" }
        └─► ContentWidthToggle escucha
              └─► wrapper está en constrained
              └─► set sidebar-mode = collapsed
              └─► dispatch sidebar:mode { mode: "collapsed", source: "content-width" }
                    └─► SidebarModeToggle escucha → actualiza su botón
                    └─► ContentWidthToggle ignora (source propio)
```

### 3.3 El usuario cambia a modo full

```text
ContentWidthToggle (click)
  └─► set content-mode = full
  └─► dispatch sidebar:mode { mode: "expanded", source: "content-width" }
  └─► dispatch toc:mode { mode: "standard", source: "content-width" }
        └─► SidebarModeToggle y TocToggle actualizan sus botones
        └─► ContentWidthToggle ignora ambos (source propio)
```

### 3.4 Prevención del ciclo infinito

`ContentWidthToggle` marca los eventos que emite como reacción con `source: "content-width"`. Su propio listener ignora cualquier evento que contenga ese `source`, evitando que un evento genere el opuesto en bucle.

```js
function isUserEvent(event) {
  return !event.detail || event.detail.source !== "content-width";
}
```

---

## 4. Ventajas del enfoque actual

1. **Componentes autónomos.** Cada toggle maneja su propio botón, iconos y persistencia. No dependen de clases CSS internas de otros componentes.
2. **Estado visible en el DOM.** `data-sidebar-mode`, `data-toc-mode` y `data-content-mode` permiten depurar inspeccionando el HTML.
3. **CSS declarativo.** Los estilos condicionales se aplican directamente sobre los atributos de datos, sin necesidad de manipular clases desde JavaScript.
4. **Comunicación desacoplada.** Los toggles no conocen la existencia de `ContentWidthToggle`; solo emiten eventos sobre sus respectivos roots.
5. **No hay lógica de layout en `Layout.astro`.** La coordinación vive en el componente que tiene sentido semántico: el que controla el ancho de contenido.

---

## 5. Desventajas y problemas actuales

1. **Coordinador con demasiados conocimientos.** `ContentWidthToggle` conoce los IDs `#sidebar` y `#toc-root`, los modos permitidos de cada uno y la regla de exclusión. Esto lo aleja del ideal de componente atómico.
2. **Persistencia duplicada.** `ContentWidthToggle` escribe `sidebar-mode` y `toc-mode` en `localStorage`, aunque `SidebarModeToggle` y `TocToggle` ya lo hacen al escuchar los eventos. Esto es necesario para evitar condiciones de carrera en la carga, pero genera redundancia.
3. **Condición de carrera en la inicialización.** `SidebarModeToggle` emite `sidebar:mode` al montarse. Si `TocToggle` aún no está listo, el evento se pierde. La mitigación actual es escribir `localStorage` desde `ContentWidthToggle`, lo que no garantiza que el DOM quede sincronizado si un componente se monta tarde.
4. **Doble mutación del DOM.** `ContentWidthToggle` actualiza `dataset.tocMode` o `dataset.sidebarMode` y luego emite el evento para que el otro toggle actualice su botón. Esto implica que el coordinador conoce implícitamente que el otro componente escuchará el evento.
5. **Estados inválidos posibles.** Nada impide que, en modo `constrained`, otro script o un error de sincronización deje `sidebar=expanded` y `toc=standard` simultáneamente.
6. **El flag `source` es un parche.** Soluciona el bucle, pero introduce una convención frágil. Si aparece otro coordinador, habrá que agregar más fuentes y más condiciones.

---

## 6. Posibles mejoras

### Opción A — Store central ligero

Crear un módulo pequeño, por ejemplo `src/scripts/layoutStore.ts`, que sea la única fuente de verdad para `contentMode`, `sidebarMode` y `tocMode`. Los componentes leen y escriben del store; el store aplica la lógica de exclusión, persiste en `localStorage` y emite eventos de notificación.

- **Pros:** lógica centralizada, fácil de testear, sin ciclos, sin condiciones de carrera.
- **Contras:** rompe parcialmente el enfoque ID-based/atómico; requiere compartir módulo entre scripts de Astro.

### Opción B — Separar comando de notificación

Los toggles emiten eventos de **petición** (`layout:request-sidebar-mode`, `layout:request-toc-mode`, `layout:request-content-mode`). Un coordinador escucha las peticiones, aplica las reglas, actualiza `data-*` + `localStorage` y emite eventos de **cambio** (`sidebar:mode-changed`, `toc:mode-changed`). Los toggles solo escuchan los eventos de cambio.

- **Pros:** imposible generar bucles por diseño; los toggles no pueden causar cascadas; la lógica de negocio queda aislada.
- **Contras:** más verboso; sigue necesitando un coordinador central.

### Opción C — Source of truth derivado

Guardar solo dos valores:

- `content-width-mode`: `full` | `constrained`.
- `active-panel`: `none` | `sidebar` | `toc`.

En modo `constrained`, el estado de sidebar y TOC se deriva de `active-panel`:

- `active-panel=sidebar` → `sidebar=expanded`, `toc=compact`.
- `active-panel=toc` → `sidebar=collapsed`, `toc=standard`.
- `active-panel=none` → ambos colapsados.

- **Pros:** imposible tener estados inválidos; un solo estado determina todo; predecible.
- **Contras:** refactor más profundo; los toggles dejan de ser autónomos y actualizan `active-panel`.

### Opción D — Eliminar la exclusión automática

Dejar que sidebar y TOC se activen independientemente. Si no caben, el usuario cambia a modo `full`.

- **Pros:** desaparece casi toda la complejidad; cero riesgo de bucles.
- **Contras:** puede no cumplir con el diseño visual deseado; en pantallas medianas ambos paneles expandidos podrían no caber.

### Opción E — Coordinación por MutationObserver

`ContentWidthToggle` observa cambios en `data-sidebar-mode` y `data-toc-mode` en lugar de escuchar eventos. Cuando detecta un cambio, aplica la regla de exclusión mutando el otro atributo.

- **Pros:** los toggles no necesitan emitir eventos para la coordinación; menos acoplamiento por convenciones de eventos.
- **Contras:** puede generar ciclos si no se detectan cambios propios; más difícil de seguir que los eventos explícitos.

---

## 7. Recomendación provisional

La **Opción B** (separar comando de notificación) es la que mejor conserva la arquitectura de eventos actual mientras elimina el `source`, aisla la lógica de exclusión y evita bucles por diseño.

Si la prioridad es eliminar la complejidad por completo, la **Opción D** (eliminar exclusión automática) es la más simple y robusta, asumiendo que el diseño visual lo permita.

---

## 8. Commits relacionados

- `c398c85` — `refactor(sidebar): declara estilos condicionales en hijos`.
- `b46ca98` — `fix(content-width): rompe ciclo infinito entre sidebar:mode y toc:mode`.

---

## 9. Referencias

- `src/components/ContentWidthToggle.astro`
- `src/components/SidebarNav/SidebarModeToggle.astro`
- `src/components/TableOfContents/TocToggle.astro`
- `src/content/nanobook-project/comunicacion-estado-componentes.md` — evolución anterior basada en `MutationObserver`.
- `src/content/nanobook-project/id-based-component-architecture.md` — principios de la arquitectura basada en IDs.
