---
id: "7af13e98-2a12-43d8-815c-f990117dc8a7"
title: "Migración del módulo edition a arquitectura hexagonal con DDD"
description: "Decisiones y estructura de la migración del módulo edition al mismo patrón DDD/hexagonal que document, navigation y publishing."
date: 2026-09-16
author: "Nanobook"
tags: ["arquitectura", "ddd", "hexagonal", "edition"]
draft: false
index: false
position: 0
---

# Migración del módulo edition a arquitectura hexagonal con DDD

## Contexto

El módulo `edition` contenía únicamente código de cliente (`client/`) y componentes Astro (`ui/`). A medida que creció, la lógica de negocio quedó mezclada con detalles de infraestructura del navegador (`sessionStorage`, `fetch`, `BroadcastChannel`, Web Worker), dificultando testearla y reutilizarla.

Los demás módulos del proyecto (`document`, `navigation`, `publishing`) ya siguen una arquitectura hexagonal con DDD:

- `domain/` para modelos, puertos, servicios puros y errores.
- `application/` para casos de uso.
- `infraestructure/` para adaptadores concretos.
- `index.ts` como composition root.

## Decisión

Replicar el mismo patrón en `edition`, extrayendo la lógica pura a `domain/`, los casos de uso a `application/` y los adaptadores del navegador a `infraestructure/`.

## Estructura resultante

```text
src/edition/
  domain/
    model/          # StagedDocument, RenderedPreview
    ports/          # DocumentStorage, PreviewRenderer, PageWarmingService, DocumentContentParser
    services/       # DocumentFlow, RenderConfirmation
    errors.ts       # EditionStorageError, EditionRenderError, InvalidDocumentContentError
  application/
    BuildStagedDocument.ts
    StageDocument.ts
    LoadStagedDocument.ts
    ClearEditionStorage.ts
    RenderPreview.ts
    ConfirmRenderedPreview.ts
    WarmDocumentPage.ts
    MarkDocumentSaved.ts
    InitializeEditor.ts
    GetEditorInitialContent.ts
    SaveDocument.ts
    HandleEditorChange.ts
    PrepareEditorNavigation.ts
    test/factories.ts
  infraestructure/
    storage/SessionStorageDocumentStorage.ts
    renderer/WorkerPreviewRenderer.ts
    warming/FetchPageWarmingService.ts
    parser/YamlDocumentContentParser.ts
  ui/               # componentes Astro que usan EditionModule directamente
  index.ts          # EditionModule (singleton `edition`)
```

## Principios aplicados

- `domain/` no depende de Astro ni de APIs del navegador.
- `application/` orquesta casos de uso a través de los puertos definidos en `domain/`.
- `infraestructure/` contiene las implementaciones con side effects.
- Los componentes Astro importan directamente `edition` desde `src/edition/index.ts`; no hay fachada intermedia.
- Helpers de API de documentos (`createDocument`, `updateDocument`) se ubicaron en `src/document/client/api/` porque pertenecen al dominio de documentos, no a edición.

## Errores

Se introdujeron errores propios del módulo:

- `EditionStorageError`: fallos de lectura/escritura en el almacenamiento de edición.
- `EditionRenderError`: fallos al renderizar el preview.
- `InvalidDocumentContentError`: contenido Markdown o frontmatter inválido.

## Parseo de contenido

El parseo del frontmatter YAML no es un servicio de dominio puro porque depende de la librería `yaml`. Por eso:

- `DocumentContentParser` se define como **puerto** en `src/edition/domain/ports/DocumentContentParser.ts`.
- `YamlDocumentContentParser` es la **implementación concreta** en `src/edition/infraestructure/parser/YamlDocumentContentParser.ts`.
- `BuildStagedDocument` recibe el parser por inyección de dependencias, manteniendo el dominio desacoplado de la librería.

## Casos de uso de flujo

Los casos de uso primitivos (`BuildStagedDocument`, `StageDocument`, `RenderPreview`, etc.) se combinan en casos de uso de alto nivel que modelan los flujos reales del editor. Esto evita que el componente Astro repita la secuencia *build → stage → render* en cada handler y elimina los wrappers locales sobre `Result`.

- `InitializeEditor` — carga el borrador, limpia storage si pertenece a otro documento y stagea el documento base si no hay borrador.
- `GetEditorInitialContent` — decide si usar el borrador en `sessionStorage` o el contenido original.
- `SaveDocument` — construye, persiste en servidor, stagea, marca como guardado, precalienta la página y renderiza el preview.
- `HandleEditorChange` — construye, stagea y renderiza el preview en cada cambio del editor.
- `PrepareEditorNavigation` — guarda el estado actual, renderiza el preview y limpia storage si la navegación sale del flujo de edición.

## Consecuencias

- La lógica de edición ahora es testeable unitariamente sin necesidad de un navegador.
- Los adaptadores de infraestructura pueden reemplazarse o mockearse fácilmente.
- Los componentes Astro delegan la orquestación en casos de uso de application; solo se encargan de conectar eventos del DOM y mostrar feedback visual.
