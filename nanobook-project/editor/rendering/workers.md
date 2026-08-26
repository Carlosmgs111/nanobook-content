---
title: "Workers de renderizado"
description: "Cómo funciona la capa de workers: MarkdownRenderClient, el worker work.ts y la comunicación con el hilo principal."
date: 2026-08-22
author: "Nanobook Team"
tags: ["editor", "preview", "rendering", "worker", "markdown-it"]
draft: false
index: false
position: 43
---

El renderizado Markdown pesado se ejecuta en un Web Worker para mantener fluida la interfaz del editor, especialmente con documentos grandes o con resaltado de sintaxis.

## `MarkdownRenderClient` (`workers/index.ts`)

Es un singleton (`export const worker`) que actúa como cliente del worker desde el hilo principal.

### Responsabilidades

- Crear y mantener la instancia del `Worker` a partir de `work.ts`.
- Generar IDs numéricos crecientes para cada petición.
- Mantener un `Map<id, resolve>` de promesas pendientes.
- Resolver la promesa correcta cuando el worker responde, incluso si las respuestas llegan desordenadas.

### Uso

```ts
import { worker } from "../rendering/client/workers";

const rendered = await worker.render(stagedDocument);
```

### Implementación clave

```ts
export class MarkdownRenderClient {
  private worker: Worker;
  private pending = new Map<number, (document: RenderedDocument) => void>();
  private nextId = 0;

  render(document: Document): Promise<RenderedDocument> {
    const id = ++this.nextId;
    return new Promise((resolve) => {
      this.pending.set(id, resolve);
      this.worker.postMessage({ id, document });
    });
  }
}
```

Cada petición lleva un ID numérico. El worker responde con el mismo ID, y `MarkdownRenderClient` resuelve la promesa correspondiente. Esto permite que múltiples renders se solapen sin perderse respuestas.

## `work.ts`

El worker propiamente dicho. Es un módulo independiente que:

1. Instancia un `MarkdownItRenderer` al iniciarse.
2. Escucha mensajes del hilo principal.
3. Renderiza el documento.
4. Devuelve el resultado con el mismo ID de petición.

```ts
import { MarkdownItRenderer } from "../adapters/it-mardown";

const renderer = new MarkdownItRenderer();

self.onmessage = async function (e: MessageEvent<RenderRequest>) {
  const { id, document } = e.data;
  const renderedDocument = await renderer.render(document);
  self.postMessage({ id, document: renderedDocument });
};
```

## Por qué un worker

- **No bloquear el hilo principal**: el renderizado con Shiki puede ser costoso; en un worker no se congela el cursor ni el scroll del editor.
- **Aislar el pipeline de renderizado**: permite cambiar entre adapters sin tocar el editor.
- **Reutilizar el mismo contrato**: el renderer puede usarse también en servidor si en el futuro se decide renderizar bajo demanda.

## Problemas resueltos

- **Worker perdía respuestas al cancelar renders**: una versión intermedia cancelaba renders terminando el worker y recreándolo, lo que perdía mensajes en tránsito. Se volvió a un modelo de IDs numéricos con un `Map` de promesas pendientes, sin terminar el worker.
