---
title: "Informe de factibilidad - Recompilado incremental"
description: "Análisis de qué tan viable es implementar recompilado incremental en Nanobook en función de la estructura del árbol de documentos y el estado actual del sistema de build."
date: 2026-08-23
author: "Nanobook Team"
tags:
  ["arquitectura","build","incremental","rendimiento","decision"]
draft: false
index: false
---

> **Nota histórica**: este informe fue redactado cuando Nanobook aún mantenía modos `OUTPUT_MODE=static` y `OUTPUT_MODE=dynamic`. El modo estático fue eliminado en favor de SSR puro; ver [plan de transición a SSR puro](../plan-transicion-ssr-puro). La mayoría del análisis de dependencias sigue siendo válido.

## Resumen ejecutivo

Este informe evalúa la factibilidad de dotar a Nanobook de **recompilado incremental** en producción: la capacidad de regenerar solo las partes del sitio afectadas ante la adición, edición, renombramiento o eliminación de documentos y directorios, sin ejecutar un build completo.

**Conclusión principal**: la factibilidad es **media-baja en el corto plazo** si se mantiene el modelo estático puro, y **media-alta** si se evoluciona hacia un modelo híbrido o dinámico con cacheo. El principal obstáculo no es el tamaño del proyecto (71 documentos, ~10,5 s de build), sino el **acoplamiento global de cada página al árbol completo de documentos**.

| Operación                   | Complejidad incremental | Páginas típicamente afectadas                                                 |
| --------------------------- | ----------------------- | ----------------------------------------------------------------------------- |
| Adición de documento        | Media                   | Documento nuevo + índice padre + sidebar de hermanos                          |
| Edición de contenido        | Baja                    | Solo el documento editado                                                     |
| Edición de metadatos        | Alta                    | Documento + padre + hermanos + descendientes + proxies                        |
| Renombramiento de documento | Muy alta                | Documento + padre + hijos + todos los documentos con links internos + proxies |
| Eliminación de documento    | Muy alta                | Equivalente a renombramiento, más gestión de 404                              |

## Contexto y arquitectura actual

### Stack y modo de build

- Astro 7 con `output: "server"` y adapter `@astrojs/node` en modo `standalone`.
- La fuente de contenido se elige con `CONTENT_SOURCE` (`filesystem` o `github`).
- Las rutas de contenido viven en `src/pages/[...slug]/`:
  - `index.astro`: vista de lectura.
  - `edit.astro`: editor integrado.
  - `preview.astro`: preview del borrador.
- Todas estas rutas declaran `export const prerender = false` y se renderizan bajo demanda.

### Métricas actuales del build

Medición local representativa (`npm run build` sin caché frío):

- **Documentos**: 71 (excluyendo borradores).
- **Páginas estáticas generadas**: ~190 archivos HTML.
  - ~71 vistas de lectura.
  - ~56 vistas de edición (solo documentos no índice).
  - ~63 vistas de preview.
- **Tiempo total de build**: ~10,5 s.
- **Tiempo de prerenderizado**: ~6,2 s.
- **Tamaño de `dist/`**: ~20 MB.

El cuello de botella no es el bundle de JavaScript ni el CSS, sino la generación de HTML: cada página reconstruye el árbol de navegación y renderiza Markdown.

## Modelo de dependencias entre documentos

Para poder regenerar selectivamente es necesario conocer qué páginas dependen de qué documentos. En Nanobook existen varios tipos de dependencias:

### 1. Dependencia del árbol de navegación global

En `src/pages/[...slug]/index.astro`, `edit.astro` y parcialmente `preview.astro`, cada página ejecuta:

```ts
const allDocuments = await repository.list();
const { nodeMap } = buildNavigationTree(allDocuments);
```

Esto significa que, en la implementación actual, **cada página tiene en su ámbito el árbol completo**. Técnicamente, un cambio en cualquier documento podría alterar el orden, la visibilidad o la estructura del árbol, por lo que una estrategia conservadora marcaría **todas las páginas como inválidas** ante cualquier modificación.

En la práctica, los cambios locales solo afectan:

- El propio documento.
- Sus hermanos (orden por `position`).
- Su padre (índice que lista hijos).
- Sus descendientes (si cambia `index`, `draft` o la ruta).
- Documentos proxy que lo referencien.

Pero la arquitectura actual no expone un grafo de dependencias; expone un `Document[]` plano.

### 2. Dependencias de índice

Un documento con `index: true` representa una carpeta y lista a sus hijos mediante `getImmediateChildren()`. Por tanto:

- Crear/renombrar/eliminar un hijo invalida el índice del padre.
- Cambiar `position` o `draft` de un hijo invalida el índice del padre.

### 3. Dependencias de sidebar y breadcrumbs

El sidebar contextual de cada página muestra los hermanos bajo el mismo padre. Los breadcrumbs recorren los ancestros. Por tanto:

- Un cambio en un documento invalida el sidebar de todos sus hermanos.
- Un cambio en un ancestro invalida los breadcrumbs de todos sus descendientes.

### 4. Dependencias de proxy

El sistema de `ref` permite que un documento sea un alias de otro (interno, archivo local o GitHub). Actualmente hay al menos dos documentos proxy reales:

- `proyectos/generador-demos-alias.md` → `proyectos/generador-demos.md`
- `nanobook-project/meta/readme.md` → `/README.md`

Un cambio en el documento destino invalida el documento proxy. Si el destino se renombra o elimina, el proxy queda roto.

### 5. Dependencias de links internos

El contenido Markdown contiene links relativos entre documentos, por ejemplo:

```md
[Arquitectura del modelo de contenido](./content-model-architecture)
```

Astro no valida ni transforma estos links. Un renombramiento rompe los links sin que el build falle. Para mantenerlos coherentes, un sistema incremental debería:

- Detectar links en cada documento.
- Invalidar documentos que apuntan a una ruta modificada.
- Opcionalmente, actualizar los links automáticamente.

## Análisis por operación

### Adición de un documento

**Escenarios**:

- **Nuevo documento en carpeta existente**:
  - Generar HTML para: vista, edición y preview.
  - Regenerar el índice del padre (ahora tiene un hijo más).
  - Regenerar el sidebar de todos los hermanos (nuevo hermano).
  - Regenerar breadcrumbs solo si el nuevo documento es un índice que afecta la jerarquía.
- **Nueva carpeta con `index.md`**:
  - Además del impacto anterior, el índice del abuelo debe regenerarse.
  - Todos los documentos bajo la nueva carpeta deben generarse por primera vez.

**Factibilidad**: **media**. Es posible si se mantiene un registro de qué documentos existen y se comparan con el estado anterior. El reto principal es identificar correctamente el padre y los hermanos.

### Edición de un documento

**Escenarios**:

- **Cambio solo de contenido Markdown** (sin tocar frontmatter):
  - Solo se invalida el documento editado.
  - Nota: si el contenido cambia headings o links, el TOC y los links se actualizan en la misma página; no hay impacto externo.
- **Cambio de metadatos** (`title`, `description`, `position`, `draft`, `index`, `ref`):
  - `title`/`description`: afecta índice del padre, sidebar de hermanos, breadcrumbs de descendientes.
  - `position`: afecta índice del padre y sidebar de hermanos.
  - `draft: true` → `false`: añade el documento al árbol; afecta padre, hermanos, y requiere generar sus páginas.
  - `draft: false` → `true`: remueve el documento; requiere regenerar padre/hermanos y eliminar sus HTML.
  - `index` cambiado: reestructura la jerarquía; impacto masivo.
  - `ref` cambiado: invalida el proxy y requiere resolver el nuevo target.

**Factibilidad**: **baja-media para metadatos**, **alta para contenido puro**. Un sistema incremental robusto debe diferenciar qué campos cambiaron.

### Renombramiento de un documento

Un renombramiento implica cambiar la ruta física del archivo Markdown, lo que cambia su `id`, `slug` y URL. Es la operación más costosa porque:

1. Cambia la URL propia: requiere generar nuevo HTML y eliminar/antiguo o redirigir.
2. Cambia el `parentId` de todos sus descendientes: deben regenerarse.
3. El padre anterior debe actualizar su lista de hijos.
4. El nuevo padre (si aplica) debe actualizar su lista de hijos.
5. Todos los documentos con links internos al documento renombrado deben actualizarse o quedarán rotos.
6. Los proxies que apuntan al documento deben actualizar su referencia.

**Factibilidad**: **baja** en un modelo estático puro. Requiere un motor de renombramiento transaccional que actualice referencias y genere redirecciones. Es más viable delegar esto a un CMS o a una capa de base de datos.

### Eliminación de un documento

Equivalente al renombramiento pero sin destino:

1. Eliminar los HTML del documento (vista, edit, preview).
2. Regenerar índice del padre y sidebar de hermanos.
3. Si tenía hijos, estos quedan huérfanos o se mueven bajo el padre anterior; requiere política definida.
4. Gestionar 404 para URLs antiguas.
5. Actualizar links internos que apuntaban al documento.

**Factibilidad**: **baja** para una solución automática. Requiere políticas de orfandad y redirecciones.

## Opciones de implementación

### Opción A: Build incremental manual sobre `dist/`

Implementar un script o servicio que, ante un evento de cambio:

1. Compare el filesystem actual con un snapshot o manifest del build anterior.
2. Calcule el conjunto mínimo de páginas a regenerar usando el grafo de dependencias.
3. Renderice solo esas páginas con una instancia ligera del renderizador (reutilizando `UnifiedMarkdownRenderer` y `buildNavigationTree`).
4. Escriba/elimine los HTML correspondientes en `dist/`.

**Pros**:

- Compatible con el despliegue actual.
- No requiere cambiar Astro.

**Contras**:

- Hay que reimplementar gran parte de lo que Astro hace en `getStaticPaths`.
- Riesgo alto de inconsistencias si no se rastrean bien las dependencias.
- Eliminación/renombramiento requiere lógica adicional de redirecciones.
- No aprovecha el cacheo de Astro ni de Vite.

**Veredicto**: viable solo como prueba de concepto; no recomendable como solución de producción.

### Opción B: ISR (Incremental Static Regeneration) con Astro + Node

Convertir las páginas dinámicas a `prerender = false` bajo `OUTPUT_MODE=dynamic` y servirlas con cacheo:

- En la primera petición se renderiza bajo demanda y se cachea.
- Un hook externo invalida el cache cuando cambia el contenido.
- Se puede combinar con `Cache-Control` y almacenamiento en disco o Redis.

**Pros**:

- Es el patrón estándar en frameworks modernos.
- Elimina la necesidad de "recompilar" archivos HTML; se invalida el cache.
- Escalable y bien soportado por `@astrojs/node`.

**Contras**:

- Requiere refactorizar las rutas para no depender de `getStaticPaths` en producción.
- Las URLs deben resolverse en runtime, no en build time.
- Necesita una capa de invalidación de cache.

**Veredicto**: **la opción más realista** para producción, pero requiere completar el modo dinámico.

### Opción C: Incremental build con adaptador de terceros

Algunas plataformas (Vercel, Netlify) ofrecen ISR nativo para Astro. Sin embargo, el proyecto actualmente usa `@astrojs/node` en modo `standalone`, por lo que estas soluciones no aplican directamente.

**Veredicto**: no aplicable sin cambiar la infraestructura de despliegue.

### Opción D: Pre-render selectivo con grafo de dependencias propio

Mantener el build estático pero introducir una fase previa que:

1. Parsee todos los documentos y construya un grafo dirigido de dependencias.
2. Detecte cambios mediante hashes de contenido y frontmatter.
3. Propague la invalidación por el grafo.
4. Ejecute `astro build` con un filtro (Astro no soporta esto nativamente, por lo que se haría invocando componentes directamente).

**Pros**:

- Mantiene el sitio 100 % estático.
- Permite optimizaciones a medida.

**Contras**:

- Complejidad alta.
- Astro no está diseñado para builds parciales; habría que evadir su pipeline.
- El mantenimiento es costoso.

**Veredicto**: demasiado complejo para el tamaño actual del proyecto.

## Evaluación de factibilidad

### Factibilidad técnica

| Aspecto                  | Evaluación   | Justificación                                                                                                  |
| ------------------------ | ------------ | -------------------------------------------------------------------------------------------------------------- |
| Tamaño del árbol         | Favorable    | 71 documentos, profundidad máxima 3. El grafo es manejable.                                                    |
| Acoplamiento global      | Desfavorable | Cada página carga `allDocuments` y `buildNavigationTree`.                                                      |
| Identificadores estables | Favorable    | Los IDs se derivan de rutas de archivo; son predecibles.                                                       |
| Referencias explícitas   | Mixta        | Existen proxies (`ref`) y links internos, pero no hay tracking de dependencias.                                |
| Infraestructura actual   | Desfavorable | `output: server` + `prerender = true` es una combinación híbrida que no aprovecha ni SSG puro ni SSR cacheado. |
| Build time actual        | Favorable    | ~10,5 s es aceptable para builds completos frecuentes.                                                         |

### Factibilidad operativa

Dado el tamaño actual del proyecto, un build completo es barato. La complejidad de implementar y mantener un recompilado incremental supera el beneficio de ahorrar ~6-10 segundos en la mayoría de los escenarios.

El recompilado incremental solo se justifica cuando:

- El número de documentos supere varios cientos.
- El tiempo de build completo supere el umbral de tolerancia del flujo de trabajo (por ejemplo, > 60 s).
- Se requiera actualizar contenido en producción sin redeploy.

## Recomendaciones

### A corto plazo (manteniendo SSG)

1. **No implementar recompilado incremental todavía**. El costo-beneficio no es favorable.
2. **Reducir el acoplamiento global**:
   - Refactorizar `src/pages/[...slug]/index.astro` para que no cargue `allDocuments` completo si no es necesario.
   - Introducir helpers como `getSiblings()`, `getParent()`, `getChildren()` que no requieran el árbol global.
3. **Documentar las dependencias**:
   - Añadir a `ContentRepository` métodos de consulta por relaciones (`getAncestors`, `getSiblings`, `getDescendants`).
   - Esto prepara el terreno para un grafo de dependencias futuro.

### A medio plazo (hacia ISR)

1. **Completar el modo dinámico**:
   - Hacer que las rutas de contenido respeten `OUTPUT_MODE` para `prerender`.
   - Implementar un `ContentRepository` que funcione en runtime sin depender de `getCollection` en build time.
2. **Añadir capa de cacheo**:
   - Cachear renders de página con clave por slug + hash del contenido.
   - Invalidar cache mediante webhooks de GitHub o mediante la API de edición.
3. **Mantener modo estático como fallback**:
   - Para desarrollo local y despliegues en entornos sin servidor (static hosting).

### A largo plazo (build incremental real)

1. **Construir un grafo de dependencias**:
   - Nodos: documentos.
   - Aristas: padre-hijo, hermano-hermano (orden), proxy-target, link interno.
2. **Invalidación por hash**:
   - Calcular hash de contenido y frontmatter por documento.
   - Propagar invalidación por aristas.
3. **Renderizador incremental independiente**:
   - Separar la lógica de renderizado de Astro para poder regenerar HTML sin build completo.
   - Esto podría reutilizar `UnifiedMarkdownRenderer` y los componentes de layout.

## Riesgos y mitigaciones

| Riesgo                                           | Impacto | Mitigación                                                                          |
| ------------------------------------------------ | ------- | ----------------------------------------------------------------------------------- |
| Inconsistencia de navegación tras cambio parcial | Alto    | Antes de cualquier solución incremental, desacoplar la navegación del árbol global. |
| Links rotos tras renombramiento                  | Medio   | Introducir validación de links en build time o migrar a IDs estables.               |
| Proxies huérfanos                                | Medio   | Mantener registro de referencias `ref` y revalidar proxies cuando cambien targets.  |
| Complejidad operativa del cacheo                 | Medio   | Empezar con TTL simple y evolucionar a invalidación explícita.                      |
| Astro no soporta builds parciales                | Alto    | No luchar contra el framework; usar SSR cacheado o aceptar build completo.          |

## Conclusión

Implementar un **recompilado incremental puro en SSG** es técnicamente posible pero **no recomendable en la etapa actual** del proyecto. La arquitectura actual carga el árbol completo de documentos en cada página, lo que convierte cualquier cambio de metadatos o estructura en una operación de impacto potencialmente global.

La ruta más pragmática es:

1. **Ahora**: mejorar el desacoplamiento del árbol de navegación y aceptar builds completos.
2. **Después**: completar el modo dinámico con SSR y cacheo (ISR).
3. **Más adelante**, si el volumen lo justifica: construir un motor de dependencias y renderizado incremental propio.

El recompilado incremental debe entenderse no como un objetivo inmediato, sino como una **capacidad que se habilita gradualmente** reduciendo primero el acoplamiento global del árbol de documentos.
