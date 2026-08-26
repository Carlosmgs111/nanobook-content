---
title: "Módulo de rendering"
description: "Visión general del módulo src/rendering/: conversión de Markdown a HTML en el navegador usando workers y adapters."
date: 2026-08-22
author: "Nanobook Team"
tags: ["editor", "preview", "rendering", "worker", "markdown"]
draft: false
index: true
position: 41
---

> **Estado**: estable. Última revisión tras el fix de persistencia efímera en el flujo edit/preview.

## Propósito

El módulo `src/rendering/` convierte Markdown en HTML dentro del navegador, sin tocar el servidor. Se usa principalmente para el **preview de edición**: el usuario escribe en `/{id}/edit` y previsualiza el resultado en `/{id}/preview` antes de guardar.

El objetivo principal es **no bloquear el hilo principal**: el resaltado de sintaxis con Shiki puede ser costoso, por lo que el trabajo pesado corre en un Web Worker.

## Estructura

```text
src/rendering/
├── core/
│   ├── render-service.ts   # Orquestación entre editor, worker y sessionStorage
│   └── types.ts            # Contratos RenderedDocument y DocumentRenderer
├── workers/
│   ├── index.ts            # MarkdownRenderClient (cliente del worker)
│   └── work.ts             # Worker que instancia el renderer
└── adapters/
    ├── it-mardown.ts       # Renderer basado en markdown-it + Shiki
    ├── astro-markdown.ts   # Renderer basado en @astrojs/markdown-remark
    └── unified-markdown.ts # Renderer basado en unified/remark/rehype
```

## Contrato base

Todo renderer implementa `DocumentRenderer`:

```ts
interface DocumentRenderer {
  render(document: { id: string; content: string }): Promise<RenderedDocument>;
}
```

Y devuelve:

```ts
interface RenderedDocument {
  Content: any;      // HTML renderizado
  headings: Heading[];
}
```

Actualmente `headings` se deja vacío en los adapters; la extracción para el TOC se hace en el consumer con `parseDocument()`.

## Subtemas

- [Servicio de renderizado](./service)
- [Workers](./workers)
- [Adapters](./adapters)

## Flujo de datos resumido

```text
DocumentEditor
     │
     ▼
saveStagedDocument(document) ──► sessionStorage["stagedDocument"]
     │
     ▼
renderStagedDocument(document)
     │
     ▼
MarkdownRenderClient.postMessage({ id, document })
     │
     ▼
Worker → MarkdownItRenderer.render()
     │
     ▼
sessionStorage["renderedStagedDocument"]
     │
     ▼
BroadcastChannel("rendered-document") ► PreviewPage
```

## Decisiones y tradeoffs

1. **Renderizado en cliente**
   - **Pros**: instantáneo, no sobrecarga el servidor, permite previsualizar sin guardar.
   - **Contras**: el preview solo funciona si se pasó primero por el editor en la misma pestaña.

2. **Web Worker**
   - **Pros**: no bloquea el cursor ni el scroll del editor; aísla el pipeline de renderizado.
   - **Contras**: añade complejidad en la comunicación y en el manejo de renders concurrentes.

3. **Claves globales en `sessionStorage`**
   - **Pros**: simple, sin acumular basura por cada documento editado.
   - **Contras**: requiere limpieza explícita al salir del flujo de edición para evitar que un documento anterior contamine uno nuevo.

## Próximos pasos

- Extraer headings del HTML renderizado directamente en el renderer para mantener el TOC en preview.
- Decidir cuál adapter se queda como renderer principal.
- Añadir tests unitarios para `render-service.ts` y los adapters.
