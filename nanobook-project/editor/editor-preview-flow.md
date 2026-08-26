---
title: "Flujo de edición y preview"
description: "Cómo funciona el ciclo editor → preview en Nanobook, incluyendo el uso de workers para renderizar Markdown sin bloquear la UI."
date: 2026-08-19
author: "Nanobook Team"
tags: ["editor", "preview", "client-router", "workers", "codemirror"]
draft: false
index: false
---

## Contexto

Nanobook incluye una vista de edición accesible desde `/{slug}/edit`. En ella se puede modificar el Markdown de un documento y, sin guardar en disco, previsualizar el resultado en `/{slug}/preview`. Este flujo es completamente cliente: el contenido editado viaja por `sessionStorage` y se renderiza en un worker para no bloquear el editor.

## Componentes principales

### `src/edition/client/render-service.ts`

Servicio compartido entre editor y preview. Es el único punto de contacto con el worker y con el `BroadcastChannel` de notificaciones.

Responsabilidades:

- Recibir peticiones de renderizado (`renderStagedDocument`).
- Gestionar un flag `renderPendingDocument` en `sessionStorage` para evitar renders duplicados del mismo documento.
- Escribir `renderedStagedDocument` solo cuando el documento que se renderizó sigue siendo el actual.
- Notificar a través de `BroadcastChannel("rendered-document")` cuando el HTML está listo.
- Limpiar `renderedStagedDocument` y `renderedStagedDocumentSource` cuando el usuario sale del flujo de edición.

### `DocumentEditor.astro`

Renderiza el editor con CodeMirror 6.

Responsabilidades:

- Montar CodeMirror en el contenedor del editor.
- Cargar el contenido inicial del documento (del prop de Astro o de `sessionStorage` si existe un borrador).
- Escuchar cambios, actualizar el `Document` en memoria y guardar el borrador en `sessionStorage`.
- Pedir al servicio de renderizado que convierta el borrador en HTML.
- En `astro:before-swap`, capturar el contenido actual del editor y forzar un render, de modo que el preview pueda mostrarse aunque el usuario navegue antes de que termine el debounce.
- Guardar cambios en disco mediante `PATCH /api/{id}`.
- Limpiar `stagedDocument` y `renderedStagedDocument` cuando el usuario sale del flujo de edición (cualquier navegación que no sea `/{id}/edit` o `/{id}/preview`).

Puntos clave para `ClientRouter`:

- El script es un módulo que persiste entre transiciones, por lo que no puede cachear referencias al DOM.
- Se consulta el wrapper del editor fresco en cada `astro:page-load`.
- La instancia de CodeMirror se destruye en `astro:before-swap` y se vuelve a crear en `astro:page-load`.
- El botón de guardar se clona antes de asignar el listener para evitar listeners duplicados.

### Vista `preview.astro`

Página dinámica (`prerender = false`) que muestra el HTML previamente renderizado.

Responsabilidades:

- Leer `renderedStagedDocument` de `sessionStorage`.
- Si no hay contenido renderizado, o si el contenido renderizado pertenece a otro documento, pedir al servicio de renderizado que lo genere y mostrar un estado de carga mientras tanto.
- Inyectar el HTML renderizado en `#preview-container`.
- Escuchar el `BroadcastChannel("rendered-document")` para refrescarse cuando otro componente (el editor o el propio servicio) termine un render.
- Limpiar `stagedDocument` y `renderedStagedDocument` cuando el usuario sale del flujo de edición.

Puntos clave para `ClientRouter`:

- Al igual que el editor, el script de preview escucha `astro:page-load` para renderizar en cada transición.
- Si `sessionStorage` está vacío, no falla: inicia el render desde el preview.

## Flujo de datos

```text
Usuario escribe en CodeMirror
        │
        ▼
updateListener de CodeMirror
        │
        ▼
debouncedOnChange (1 s)
        │
        ▼
updateDocument(editor.getContent())
        │
        ▼
sessionStorage.setItem("stagedDocument", ...)
        │
        ▼
renderService.renderStagedDocument(stagedDocument)
        │
        ▼
worker.render(stagedDocument)
        │
        ▼
sessionStorage.setItem("renderedStagedDocument", ...)
        │
        ▼
BroadcastChannel("rendered-document") notifica a preview
        │
        ▼
preview.astro lee renderedStagedDocument
        │
        ▼
#preview-container.innerHTML = renderedContent
```

Si el usuario navega a preview antes de que termine el debounce, `DocumentEditor` captura el contenido en `astro:before-swap` y también pide un render. Si el usuario navega antes de que el worker termine, `preview.astro` muestra "Generando preview..." y espera a que el servicio termine (ya sea porque el editor lo inició o porque el preview mismo lo inicia como respaldo).

`BroadcastChannel` permite que la vista de preview se refresque automáticamente cuando el worker termina de renderizar, sin necesidad de que el usuario vuelva a cargar la página.

## Workers de renderizado

El renderizado del borrador se ejecuta en un Web Worker para mantener fluida la interfaz del editor, especialmente con documentos grandes o con resaltado de sintaxis.

### Arquitectura

- `src/services/render.ts` — Servicio compartido que coordina el renderizado entre editor y preview. Mantiene el estado en `sessionStorage`, evita duplicados y notifica por `BroadcastChannel`.
- `src/workers/index.ts` — `MarkdownRenderClient`, una pequeña clase que envía peticiones numeradas al worker y devuelve una promesa por cada petición. Mantiene un `Map<id, resolve>` para poder resolver cada respuesta independientemente, incluso si llegan desordenadas.
- `src/workers/work.ts` — El worker propiamente dicho. Recibe un `RenderRequest`, llama al renderer y responde con el `RenderedDocument` incluyendo el mismo `id`.
- `src/rendering/adapters/markdown/markdown-it.ts` — Implementación basada en `markdown-it` + `@shikijs/markdown-it` + `markdown-it-anchor`. Usa el motor de regex de JavaScript de Shiki para evitar cargar WASM dentro del worker.
- `src/rendering/adapters/markdown/astro-markdown.ts` — Implementación alternativa basada en `@astrojs/markdown-remark`.

### Contrato

```ts
interface DocumentRenderer {
  render(document: { id: string; content: string }): Promise<RenderedDocument>;
}

interface RenderedDocument {
  Content: string;
  headings: Heading[];
}
```

El worker serializa el `Document` completo, pero el renderer solo necesita `id` y `content`.

### ¿Por qué un worker?

- **No bloquear el hilo principal**: el renderizado con Shiki puede ser costoso; en un worker no se congela el cursor ni el scroll del editor.
- **Aislar el pipeline de renderizado**: permite cambiar entre `markdown-it` y `@astrojs/markdown-remark` sin tocar el editor.
- **Reutilizar el mismo contrato**: el renderer puede usarse también en servidor si en el futuro se decide renderizar bajo demanda.

### Motor de Shiki

Inicialmente el renderer usaba el motor Oniguruma de Shiki, que depende de un archivo `.wasm`. Dentro de un Web Worker en Vite, la carga dinámica de ese WASM fallaba con:

```
TypeError: Failed to fetch dynamically imported module: .../wasm-XXXX.js
```

La solución fue cambiar al **motor de regex de JavaScript** (`createJavaScriptRegexEngine` de `@shikijs/engine-javascript`). Este motor no requiere WASM, es más ligero para el worker y, con `forgiving: true`, ignora patrones de grammars que no pueda emular.

## Estado en `sessionStorage`

| Clave | Contenido | Quién lo escribe |
|---|---|---|
| `stagedDocument` | `Document` con el contenido editado | `DocumentEditor` (`debouncedOnChange`, `astro:before-swap`) |
| `renderPendingDocument` | `Document` que se está renderizando en este momento | `src/services/render.ts` |
| `renderedStagedDocumentSource` | `Document` fuente del último HTML renderizado | `src/services/render.ts` |
| `renderedStagedDocument` | `RenderedDocument` con el HTML y headings | `src/services/render.ts` |

Ambos valores se limpian al cerrar la pestaña. Además, se borran activamente cuando el usuario navega fuera del flujo de edición de un documento (cualquier URL que no sea `/{id}/edit` o `/{id}/preview`). Esto garantiza que abrir el editor de otro documento nunca cargue un borrador ajeno.

No son persistentes entre sesiones porque la edición es un borrador temporal; el guardado definitivo ocurre con el botón **Guardar**.

## Decisiones y tradeoffs

1. **Renderizado en cliente, no en servidor**
   - El preview no hace una petición al servidor; todo ocurre en el navegador.
   - Ventaja: instantáneo y sin sobrecargar el servidor.
   - Costo: el preview solo funciona si el usuario pasó primero por el editor en la misma pestaña.

2. **`sessionStorage` como transporte**
   - Simple y suficiente para el flujo editor → preview.
   - No requiere un backend de borradores.
   - Limitación: no se comparte entre pestañas ni dispositivos.

3. **Worker `markdown-it` como renderer por defecto**
   - Da control total sobre plugins (Shiki, anclas, GFM).
   - Se mantiene `astro-markdown.ts` como alternativa para evaluar paridad de salida.

## Problemas resueltos recientemente

- **Editor vacío al volver de preview**: el script del editor capturaba el wrapper en el scope del módulo; tras una transición de `ClientRouter` apuntaba a un nodo desconectado. Se movió la consulta del DOM a `astro:page-load` y se añadió destrucción en `astro:before-swap`.
- **Preview vacío en la segunda visita**: el script de preview era un IIFE que solo corría una vez. Se convirtió a listener de `astro:page-load`.
- **Doble inicialización del editor**: se eliminó la llamada inmediata a `initEditor()`; `astro:page-load` ya dispara en la carga inicial.
- **Worker perdía respuestas al cancelar renders**: una versión intermedia cancelaba renders terminando el worker y recreándolo, lo que perdía mensajes en tránsito. Se volvió a un modelo de IDs numéricos con un `Map` de promesas pendientes, sin terminar el worker.
- **Renders obsoletos sobreescribiendo `sessionStorage`**: si un render anterior terminaba después de uno más reciente, podía dejar el preview desactualizado. El servicio de renderizado verifica que el `stagedDocument` de `sessionStorage` siga siendo el mismo que se renderizó antes de escribir el resultado.
- **Preview vacío si se navegaba antes de que terminara el debounce**: si el usuario hacía click en preview dentro del segundo de debounce, el render nunca se iniciaba. Ahora `DocumentEditor` captura el contenido actual en `astro:before-swap` y pide un render, y `preview.astro` también puede iniciar el render si es necesario.
- **Editor mostraba siempre el mismo documento staged**: al usar una única clave global `stagedDocument`, editar cualquier documento después de haber editado otro cargaba el contenido del documento anterior. Se resolvió validando el `id` del documento staged al cargar el editor y borrando las claves de `sessionStorage` al salir del flujo de edición.

## Próximos pasos

- Decidir cuál renderer se queda como principal (`markdown-it` vs `@astrojs/markdown-remark`).
- Extraer headings del HTML renderizado para mantener el TOC en preview.
- Añadir indicador de "guardando..." y manejo de errores de red en el botón de guardar.
- Evaluar reducir el debounce de 1 s o renderizar incrementalmente mientras se escribe.
