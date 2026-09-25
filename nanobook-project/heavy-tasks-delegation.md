---
id: "fa77aa55-f06e-473d-9d67-d80c0cd2e904"
title: "Delegación de Tareas Pesadas"
description: "Delegación de Tareas Pesadas"
date: 2026-09-22T21:15:00.496Z
author: "Nanobook"
tags: []
draft: false
index: false
position: 0
proxyTargetId: null
---

# Uso de Workers en Nanobook: evolución arquitectónica y conclusiones

## 1. Contexto inicial

En Nanobook surgió la necesidad de renderizar Markdown de forma interactiva mientras el usuario edita un documento.

El flujo inicial era aproximadamente:

```text
DocumentEditor
      │
      ▼
Markdown Renderer
      │
      ▼
HTML
```

El problema era que el renderizado podía convertirse en una operación relativamente costosa:

- parsear Markdown;
- procesar GFM;
- generar el árbol HTML;
- generar slugs;
- aplicar syntax highlighting con Shiki;
- extraer información derivada del documento.

Ejecutar todo esto en el hilo principal podía afectar la capacidad de respuesta del editor.

Por esta razón se decidió mover el renderizado a un **Web Worker**.

---

# 2. Primera implementación: el Worker como detalle del `DocumentEditor`

Inicialmente el Worker estaba fuertemente ligado al componente de edición.

Conceptualmente:

```text
DocumentEditor
      │
      │ crea / controla
      ▼
MarkdownRenderClient
      │
      │ postMessage
      ▼
Web Worker
      │
      ▼
Markdown Renderer
```

El `DocumentEditor` terminaba teniendo conocimiento de demasiadas cosas:

- creación del Worker;
- ciclo de vida del Worker;
- comunicación mediante `postMessage`;
- correlación de requests/responses;
- almacenamiento del resultado;
- control de revisiones;
- sincronización con Preview.

El Worker utilizaba un renderer Markdown. Inicialmente se experimentó con un pipeline basado en Unified:

```text
unified
  → remark-parse
  → remark-gfm
  → remark-rehype
  → rehype-slug
  → rehype-shiki
  → rehype-stringify
```

Esto produjo además problemas específicos del runtime Worker, como:

```text
document is not defined
```

Algunas dependencias o configuraciones asumían indirectamente la existencia de un entorno con DOM.

Esto llevó a evaluar una implementación basada en:

```text
markdown-it + shiki
```

más apropiada para ejecutarse en un Worker.

---

# 3. Primera separación importante: UI ≠ renderizado

El primer refactor importante consistió en reconocer que `DocumentEditor` no debería ser responsable de ejecutar directamente toda esta infraestructura.

El editor simplemente necesita una capacidad:

```ts
interface DocumentRenderer {
  render(document: Document): Promise<RenderedDocument>;
}
```

Desde la perspectiva del editor debería ser irrelevante si esa capacidad se ejecuta:

- en el main thread;
- en un Worker;
- en un servidor;
- mediante otro mecanismo.

Por tanto empezamos a separar:

```text
DocumentEditor
      │
      ▼
DocumentRenderer
```

de:

```text
Worker
postMessage
MessageEvent
MarkdownIt
Shiki
```

La primera parte representa **qué necesita el sistema**.

La segunda representa **cómo y dónde se ejecuta**.

---

# 4. El Worker como otro entorno de ejecución

Posteriormente apareció una observación importante:

> Un Worker no es simplemente una función asíncrona. Es otro entorno de ejecución.

Un Web Worker posee:

- su propio global scope;
- su propio event loop;
- memoria aislada respecto al main thread;
- un entrypoint propio;
- un ciclo de vida independiente;
- sus propias dependencias cargadas;
- comunicación explícita mediante mensajes.

Por tanto conceptualmente podemos verlo como:

```text
┌──────────────────────────────┐
│ Main Thread Runtime          │
│                              │
│ DocumentEditor               │
│ Application                  │
│ Adapters                     │
└──────────────┬───────────────┘
               │ IPC
               │ postMessage
               ▼
┌──────────────────────────────┐
│ Worker Runtime               │
│                              │
│ Entry Point                  │
│ Composition                  │
│ Markdown Renderer            │
└──────────────────────────────┘
```

Esto permitió corregir otra deficiencia de la implementación inicial.

No debería existir conceptualmente:

```text
DocumentEditor → worker.ts
```

como si el Worker fuera una dependencia directa de la UI.

El Worker tiene su propio entrypoint y su propia composición.

---

# 5. La confusión: "si es otro runtime, entonces es otra aplicación"

A partir de esta observación llevamos inicialmente el razonamiento demasiado lejos.

Si el Worker tiene:

- lifecycle propio;
- memoria propia;
- entrypoint propio;
- composición propia;
- dependencias propias;

parecía razonable concluir:

> El Worker debería tratarse como una aplicación completamente independiente.

Y de ahí surgía naturalmente la posibilidad de trasladar al Worker no solamente la implementación técnica costosa, sino también:

- casos de uso;
- lógica de aplicación;
- dominio;
- puertos;
- adaptadores.

Por ejemplo:

```text
Main Thread
    │
    │ ejecutar RenderPreview
    ▼
Worker
    │
    ├── RenderPreview
    ├── Domain
    ├── DocumentRenderer
    └── MarkdownItRenderer
```

Esto **es técnicamente válido**.

Un Worker puede perfectamente ser el host de una aplicación completa.

Pero que pueda hacerlo no significa que deba hacerlo.

---

# 6. Entorno de ejecución no implica frontera de negocio

Aquí apareció la distinción fundamental.

Habíamos mezclado dos conceptos diferentes:

```text
frontera de ejecución
```

y:

```text
frontera arquitectónica / de negocio
```

Un Worker introduce necesariamente una frontera de ejecución.

Pero **no introduce automáticamente una nueva frontera de dominio**.

Es decir:

```text
Worker ≠ módulo de negocio
Worker ≠ bounded context
Worker ≠ caso de uso
Worker ≠ dominio
```

El Worker solamente determina **dónde se ejecuta determinado código**.

Por tanto:

> Que una pieza de código se ejecute en un runtime aislado no obliga a trasladar allí la lógica de negocio que utiliza esa pieza.

---

# 7. Separar tres preguntas diferentes

La discusión terminó siendo mucho más clara cuando separamos tres dimensiones.

## Qué se necesita

Por ejemplo:

```text
DocumentRenderer
```

Es una capacidad.

---

## Cómo se implementa

Por ejemplo:

```text
MarkdownItRenderer
```

Es una implementación concreta de esa capacidad técnica.

---

## Dónde se ejecuta

Por ejemplo:

```text
Main Thread
Worker
Server
```

Es una decisión de ejecución/deployment.

Estas tres dimensiones no deberían confundirse.

```text
QUÉ
DocumentRenderer

        │ implementado mediante

CÓMO
MarkdownItRenderer

        │ ejecutado en

DÓNDE
Web Worker
```

El Worker pertenece principalmente a la tercera dimensión.

---

# 8. El Worker como mecanismo de delegación

Esto llevó a una interpretación más precisa.

En Nanobook no necesitamos que el Worker sea dueño de toda la capacidad de aplicación.

Necesitamos que permita **delegar una operación técnica costosa a otro runtime**.

Por tanto, conceptualmente podemos tener:

```text
Caso de uso / capacidad
        │
        ▼
DocumentRenderer
        │
        ▼
WorkerDocumentRenderer
        │
        │ IPC
        ▼
Web Worker
        │
        ▼
MarkdownItRenderer
```

Desde el punto de vista del consumidor:

```ts
await renderer.render(document);
```

Eso es todo.

No conoce:

```ts
new Worker(...)
worker.postMessage(...)
worker.addEventListener(...)
```

ni sabe que el renderizado ocurre fuera del main thread.

---

# 9. `WorkerDocumentRenderer` como adaptador/proxy

La pieza importante pasa a ser algo conceptualmente similar a:

```text
WorkerDocumentRenderer
```

Su responsabilidad no es renderizar Markdown.

Su responsabilidad es **hacer que la capacidad `DocumentRenderer` pueda satisfacerse delegando su ejecución a un Worker**.

Conceptualmente:

```ts
class WorkerDocumentRenderer implements DocumentRenderer {
  async render(document: Document): Promise<RenderedDocument> {
    // serializar request
    // enviarlo al Worker
    // correlacionar response
    // devolver resultado
  }
}
```

Mientras que dentro del Worker existe la implementación real:

```ts
const renderer = new MarkdownItRenderer();
```

y el entrypoint traduce IPC hacia llamadas normales:

```text
message
   │
   ▼
deserialize
   │
   ▼
renderer.render(...)
   │
   ▼
serialize
   │
   ▼
postMessage
```

Así, `postMessage` es solamente el protocolo entre runtimes.

---

# 10. Composición independiente no significa dominio independiente

El Worker sí tiene composición propia.

Por ejemplo:

```text
worker.ts
   │
   ├── crea MarkdownItRenderer
   ├── registra message handler
   └── conecta IPC con renderer
```

Esto sigue siendo correcto.

El Worker es un runtime independiente y por eso necesita saber qué implementaciones existen dentro de él.

Pero esto no significa que necesitemos replicar dentro:

```text
domain/
application/
use-cases/
infra/
```

si únicamente estamos delegando una operación técnica.

La composición independiente deriva de la **frontera de runtime**, no necesariamente de una **frontera de negocio**.

---

# 11. No mover el dominio innecesariamente

La alternativa más pesada habría sido:

```text
Main Thread
     │
     ▼
Worker
     │
     ▼
Use Case
     │
     ▼
Domain
     │
     ▼
DocumentRenderer
     │
     ▼
MarkdownItRenderer
```

Esto introduce varias consecuencias.

Ahora la comunicación entre main thread y Worker deja de representar una simple operación técnica y pasa a representar comandos de aplicación.

También habría que decidir:

- qué parte del dominio vive en cada runtime;
- cómo serializar objetos de dominio;
- cómo manejar estado;
- qué repositorios necesita el Worker;
- cómo manejar errores de aplicación;
- cómo compartir contratos;
- cómo mantener coherencia entre runtimes.

Todo esto sería justificable si realmente necesitáramos ejecutar una aplicación completa dentro del Worker.

Pero Nanobook actualmente no tiene esa necesidad.

---

# 12. La solución adecuada para Nanobook

Para el caso actual, el Worker existe principalmente porque cierta operación técnica puede ser costosa para el main thread.

Por tanto, la arquitectura más simple es:

```text
┌─────────────────────────────────────┐
│ Main Thread                         │
│                                     │
│ Application / Domain                │
│          │                          │
│          ▼                          │
│    DocumentRenderer                 │
│          ▲                          │
│          │ implements               │
│ WorkerDocumentRenderer              │
└──────────┬──────────────────────────┘
           │
           │ IPC / postMessage
           ▼
┌─────────────────────────────────────┐
│ Web Worker                          │
│                                     │
│ Worker Entry Point                  │
│          │                          │
│          ▼                          │
│ MarkdownItRenderer                  │
│          │                          │
│          ▼                          │
│ markdown-it + Shiki                 │
└─────────────────────────────────────┘
```

La lógica de negocio continúa donde estaba.

Lo único que cruza la frontera es la operación que queremos ejecutar fuera del main thread.

---

# 13. Envolver la capacidad técnica, no necesariamente la capacidad completa

Esta fue probablemente la conclusión más importante de toda la discusión.

Inicialmente estábamos pensando en:

```text
Worker
└── capacidad completa
    ├── application
    ├── domain
    └── infraestructura
```

Pero para Nanobook resulta suficiente:

```text
capacidad de aplicación
        │
        ▼
capacidad técnica
        │
        ▼
delegación al Worker
```

O expresado de otra forma:

```text
Use Case
   │
   │ necesita
   ▼
DocumentRenderer
   │
   │ implementación delegada
   ▼
WorkerDocumentRenderer
   │
   ▼
Worker
   │
   ▼
MarkdownItRenderer
```

El Worker **envuelve/delega la capacidad técnica que necesita la capacidad superior**, no toda la capacidad superior junto con su dominio.

Esto mantiene la frontera arquitectónica mucho más limpia.

---

# 14. El Worker sigue pudiendo convertirse en una aplicación completa

Nada de esto significa que un Worker no pueda contener dominio o casos de uso.

Puede hacerlo perfectamente.

Por ejemplo, si en el futuro existiera una capacidad compleja de procesamiento:

```text
Worker
├── Application
├── Domain
├── Ports
├── Adapters
└── Composition Root
```

podría tener sentido tratar ese Worker como una aplicación autónoma.

La regla importante es:

> La frontera del runtime no debe determinar artificialmente la frontera del dominio.

Primero debe existir una razón arquitectónica para mover una capacidad completa.

Después puede elegirse el Worker como runtime para ejecutarla.

No al contrario.

---

# 15. Consecuencia conceptual: Worker como infraestructura de ejecución

Para Nanobook resulta más útil pensar el Worker como una **infraestructura de ejecución/delegación**.

Algo parecido conceptualmente a:

```text
capacidad
    │
    ▼
ejecución delegada
    │
    ├── Local
    └── Worker
```

Aunque discutimos la posibilidad de generalizar esto inmediatamente mediante algo como:

```ts
TaskExecutor
```

o:

```ts
Executor
```

la conclusión fue que no conviene introducir prematuramente una abstracción completamente genérica.

Mientras solamente tengamos un caso concreto, una abstracción semántica como:

```text
DocumentRenderer
        ▲
        │
WorkerDocumentRenderer
```

es más explícita y mantiene mejor el lenguaje arquitectónico del sistema.

Si posteriormente aparecen múltiples capacidades con exactamente la misma necesidad de delegación, entonces sí podría emerger una abstracción transversal de ejecución.

---

# 16. Relación con Ports & Adapters

La solución encaja naturalmente con Ports & Adapters.

La aplicación expresa:

```text
"Necesito poder renderizar un documento"
```

mediante:

```ts
interface DocumentRenderer {
  render(...): Promise<...>;
}
```

La infraestructura puede satisfacerlo mediante diferentes adaptadores:

```text
DocumentRenderer
       ▲
       │
 ┌─────┴─────────────────┐
 │                       │
MarkdownItRenderer   WorkerDocumentRenderer
                         │
                         ▼
                       Worker
                         │
                         ▼
                  MarkdownItRenderer
```

La diferencia es interesante.

`MarkdownItRenderer` implementa directamente la capacidad.

`WorkerDocumentRenderer` actúa como un **proxy/adaptador de ejecución**, haciendo que esa misma capacidad se ejecute en otro runtime.

---

# 17. Qué pertenece a cada runtime

Una separación razonable queda así.

## Main thread

Responsable de:

```text
UI
DocumentEditor
estado del editor
casos de uso
coordinación
sessionStorage
sincronización con Preview
WorkerDocumentRenderer
```

---

## Worker

Responsable de:

```text
entrypoint
recepción de mensajes
correlación request/response
ejecución del renderer
MarkdownItRenderer
markdown-it
Shiki
respuesta al main thread
```

El Worker no necesita conocer:

```text
DocumentEditor
UI
sessionStorage
Preview
Astro components
```

y tampoco necesita contener dominio adicional simplemente por ser un runtime aislado.

---

# 18. Sincronización con Preview

Otra parte relacionada de la implementación fue la sincronización entre:

```text
Editor
```

y:

```text
Preview
```

El Worker no debía asumir esa responsabilidad.

El flujo terminó conceptualmente separado:

```text
DocumentEditor
      │
      ▼
renderPreview()
      │
      ▼
WorkerDocumentRenderer
      │
      ▼
Worker
      │
      ▼
RenderedDocument
      │
      ▼
Main Thread
      │
      ├── sessionStorage
      │
      └── BroadcastChannel
                │
                ▼
             Preview
```

`sessionStorage` conserva el último estado disponible y `BroadcastChannel` funciona como mecanismo de notificación entre las páginas.

Esto reforzó la separación:

> El Worker procesa. El main thread coordina el estado y la interacción entre interfaces.

El Worker no necesita convertirse en propietario del flujo completo solamente porque realiza el trabajo pesado.

---

# 19. Modelo mental final

El modelo mental inicial era aproximadamente:

```text
Worker = otra aplicación
       = otra arquitectura
       = dominio propio
       = casos de uso propios
```

El modelo final es más preciso:

```text
Worker
  =
entorno de ejecución aislado
```

y a partir de ahí:

```text
entorno de ejecución aislado
        │
        ├── puede ejecutar una función
        ├── puede ejecutar una capacidad técnica
        ├── puede ejecutar un caso de uso
        ├── puede alojar un módulo
        └── puede alojar una aplicación completa
```

La elección depende de lo que realmente necesitemos aislar.

En Nanobook actualmente necesitamos principalmente:

```text
aislar trabajo técnico costoso
```

no:

```text
aislar una aplicación o dominio completo
```

Por eso la arquitectura adecuada es:

```text
Domain / Application
        │
        ▼
Technical Capability
        │
        ▼
Delegating Adapter
        │
        ▼
Worker Runtime
        │
        ▼
Concrete Technical Implementation
```

o, aplicado al caso concreto:

```text
Nanobook
   │
   ▼
capacidad que necesita renderizar
   │
   ▼
DocumentRenderer
   │
   ▼
WorkerDocumentRenderer
   │
   │ IPC
   ▼
Web Worker
   │
   ▼
MarkdownItRenderer
   │
   ▼
markdown-it + Shiki
```

---

# Conclusión

La evolución de la arquitectura consistió principalmente en dejar de identificar **aislamiento de ejecución** con **aislamiento arquitectónico**.

Un Worker sí es un entorno de ejecución independiente y, por ello, tiene:

- entrypoint;
- lifecycle;
- memoria;
- composición;
- protocolo de comunicación;

propios.

Pero esto no obliga a convertirlo en una aplicación de negocio independiente.

Para Nanobook, el Worker puede mantenerse como un mecanismo de infraestructura encargado de **delegar la ejecución de una capacidad técnica costosa**.

La lógica de dominio y aplicación permanece en su módulo correspondiente, mientras el adaptador basado en Worker hace transparente el lugar donde se ejecuta el trabajo.

La idea puede resumirse así:

> **No mover al Worker todo lo que usa una operación costosa. Mover únicamente la operación que necesita ejecutarse fuera del runtime principal.**

Y, sobre todo:

> **La frontera de ejecución y la frontera de negocio son dimensiones independientes. Un Worker introduce necesariamente la primera, pero solamente debe introducir la segunda cuando el diseño del sistema realmente lo requiera.**