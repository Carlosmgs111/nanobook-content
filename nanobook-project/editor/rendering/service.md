---
title: "Servicio de renderizado"
description: "Cómo funciona render-service.ts: orquestación del renderizado Markdown, estado en sessionStorage y coordinación con editor y preview."
date: 2026-08-22
author: "Nanobook Team"
tags: ["editor", "preview", "rendering", "service", "sessionstorage"]
draft: false
index: false
position: 42
---

`src/edition/client/render-service.ts` es el único punto de contacto entre el editor, el preview y el Web Worker. Vive en el hilo principal y gestiona el estado compartido en `sessionStorage`.

## Responsabilidades

- Recibir peticiones de renderizado (`renderStagedDocument(document)`).
- Evitar renders duplicados del mismo documento usando `renderPendingDocument`.
- Reutilizar un render previo si el documento no cambió (`renderedStagedDocument` + `renderedStagedDocumentSource`).
- Verificar que el documento staged sigue siendo el actual antes de escribir un resultado, evitando que renders antiguos sobreescriban renders nuevos.
- Notificar a través de `BroadcastChannel("rendered-document")` cuando hay HTML listo.
- Limpiar `renderedStagedDocument` y `renderedStagedDocumentSource` cuando se sale del flujo de edición.

## API pública

```ts
getStagedDocument(): Document | null
getRenderedDocument(): RenderedDocument | null
getRenderedDocumentSource(): Document | null
clearRenderedDocument(): void
renderStagedDocument(stagedDocument: Document): Promise<RenderedDocument | null>
ensureRendered(): Promise<RenderedDocument | null>
```

> `clearStagedDocument()` vive en `src/edition/client/stage-document.ts` porque es quien controla la clave `stagedDocument`.

## Estado en `sessionStorage`

| Clave | Contenido | Quién lo escribe | Vida |
|---|---|---|---|
| `stagedDocument` | `Document` con el contenido editado | `DocumentEditor` | Solo dentro del flujo edit/preview del mismo documento |
| `renderPendingDocument` | `Document` en renderizado activo | `render-service.ts` | Durante el render |
| `renderedStagedDocumentSource` | `Document` fuente del último HTML | `render-service.ts` | Solo dentro del flujo edit/preview |
| `renderedStagedDocument` | `RenderedDocument` con HTML y headings | `render-service.ts` | Solo dentro del flujo edit/preview |

> **Regla de oro**: estas claves se borran al navegar fuera de `/{id}/edit` o `/{id}/preview`. Si se cierra la pestaña también desaparecen porque `sessionStorage` no sobrevive.

## Flujo interno de `renderStagedDocument`

```text
renderStagedDocument(document)
        │
        ▼
¿Hay un render pendiente del mismo documento?
        │ Sí ──► waitForRender() ──► devuelve HTML
        │ No
        ▼
¿Ya existe HTML renderizado para este documento?
        │ Sí ──► devuelve HTML cacheado
        │ No
        ▼
Marcar renderPendingDocument
        │
        ▼
worker.render(document)
        │
        ▼
¿stagedDocument sigue siendo el mismo?
        │ No ──► descarta resultado y devuelve HTML previo o null
        │ Sí
        ▼
Guardar renderedStagedDocument + Source
        │
        ▼
BroadcastChannel.postMessage("")
        │
        ▼
Devolver HTML
```

La comprobación final es clave: si el usuario sigue escribiendo mientras el worker renderiza una versión anterior, el resultado obsoleto no sobreescribe el estado.

## `ensureRendered`

Es un punto de entrada de conveniencia usado por `PreviewPage`. Si no hay `stagedDocument` devuelve `null`; si lo hay, pide el render.

```ts
export async function ensureRendered(): Promise<RenderedDocument | null> {
  const staged = getStagedDocument();
  if (!staged) return null;
  return renderStagedDocument(staged);
}
```

## Coordinación con editor y preview

### Editor

- Escribe en `stagedDocument` en cada cambio (con debounce).
- Pide el render y espera el HTML.
- En `astro:before-swap`, fuerza un último render para que el preview no se quede vacío si el usuario navega rápido.
- Al salir del flujo de edición, borra todo el estado.

### Preview

- Lee `renderedStagedDocument` y su fuente.
- Si no hay nada o el contenido pertenece a otro documento, pide un render o muestra "No hay contenido para previsualizar".
- Escucha el `BroadcastChannel` para refrescarse cuando el editor termine un render.
- Al salir del flujo de edición, borra todo el estado.
