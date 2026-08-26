---
title: Adapters de rendering
description: "Los tres adapters que implementan DocumentRenderer en
  src/rendering/adapters/: markdown-it, @astrojs/markdown-remark y unified."
date: 2026-08-22T00:00:00.000Z
author: Nanobook Team
tags:
  - editor
  - preview
  - rendering
  - markdown-it
  - astro
  - unified
draft: false
index: false
position: 44
---

Los adapters implementan el contrato `DocumentRenderer`:

```ts
interface DocumentRenderer {
  render(document: { id: string; content: string }): Promise<RenderedDocument>;
}
```

Actualmente existen tres implementaciones. El worker usa `MarkdownItRenderer` como renderer por defecto.

## `MarkdownItRenderer` (`it-mardown.ts`)

Renderer por defecto del worker. Basado en:

- `markdown-it` para el parseo Markdown.
- `@shikijs/markdown-it` para el resaltado de sintaxis.
- `markdown-it-anchor` para generar anclas en los headings.
- Motor de regex de JavaScript de Shiki (`createJavaScriptRegexEngine`) para evitar cargar WASM dentro del worker.

### Configuración

```ts
const processor = new MarkdownIt({
  html: true,
  linkify: true,
  typographer: true,
})
  .use(anchor, { slugify: ... })
  .use(shiki);
```

Soporta temas claro/oscuro:

```ts
{
  light: "github-light",
  dark: "github-dark",
}
```

### Por qué no usa Oniguruma

Inicialmente se intentó usar el motor Oniguruma de Shiki, que depende de un archivo `.wasm`. Dentro de un Web Worker en Vite, la carga dinámica de ese WASM fallaba con:

```bash
TypeError: Failed to fetch dynamically imported module: .../wasm-XXXX.js
```

La solución fue cambiar al **motor de regex de JavaScript** (`createJavaScriptRegexEngine` de `@shikijs/engine-javascript`). No requiere WASM, es más ligero para el worker y, con `forgiving: true`, ignora patrones de grammars que no pueda emular.

## `AstroMarkdownRenderer` (`astro-markdown.ts`)

Alternativa basada en `@astrojs/markdown-remark`. Usa el mismo pipeline de renderizado que Astro en servidor, lo que la hace interesante para evaluar paridad de salida.

```ts
this.processor = await createMarkdownProcessor({
  syntaxHighlight: "shiki",
  shikiConfig: {
    theme: "github-dark",
  },
});
```

No se usa actualmente en el worker, pero se mantiene disponible para comparar resultados con el renderer principal.

## `UnifiedMarkdownRenderer` (`unified-markdown.ts`)

Alternativa basada en el ecosistema `unified`:

```text
remark-parse → remark-gfm → remark-rehype → rehype-slug → rehype-shiki → rehype-stringify
```

Pipeline:

1. `remarkParse`: convierte Markdown a AST de Markdown.
2. `remarkGfm`: añade soporte para GitHub Flavored Markdown.
3. `remarkRehype`: convierte el AST de Markdown a AST de HTML.
4. `rehypeSlug`: añade IDs a los headings.
5. `rehypeShiki`: resalta bloques de código.
6. `rehypeStringify`: convierte el AST a HTML.

Tampoco se usa activamente; se mantiene como experimento.

## Comparativa

| Adapter | Motor | Uso actual | Pros | Contras |
|---|---|---|---|---|
| `MarkdownItRenderer` | `markdown-it` + Shiki JS regex | Worker por defecto | Rápido, ligero, funciona en worker | Menos parecido al renderizado de Astro |
| `AstroMarkdownRenderer` | `@astrojs/markdown-remark` | No activo | Máxima paridad con Astro server | Más pesado, posiblemente problemas en worker |
| `UnifiedMarkdownRenderer` | `unified`/`remark`/`rehype` | No activo | Pipeline modular y estandarizado | Requiere ajuste para igualar salida de Astro |

## Próximos pasos

- Decidir cuál adapter se queda como renderer principal.
- Extraer headings del HTML renderizado dentro del adapter para mantener el TOC en preview.
- Añadir tests de paridad entre adapters.