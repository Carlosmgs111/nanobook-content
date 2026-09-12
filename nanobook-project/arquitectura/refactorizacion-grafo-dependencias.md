---
title: "Refactorización del grafo de dependencias"
description: "Decisión de mover DocumentsGraph a navigation y de calcular invalidatedIds a través del grafo en lugar de invalidar directamente los IDs recibidos."
date: 2026-09-11
author: "Nanobook Team"
tags:
  - arquitectura
  - navigation
  - document-graph
  - invalidacion
  - decision
draft: false
index: false
---

# Refactorización del grafo de dependencias

## Contexto

Nanobook modela el contenido como una jerarquía de documentos con relaciones entre sí. Esas relaciones generan dependencias de renderizado:

- Un padre muestra breadcrumbs, sidebar y navegación que dependen de sus hijos.
- Los hermanos se ordenan entre sí en índices y sidebars.
- Un proxy replica el contenido de otro documento.
- Los enlaces internos pueden quedar rotos si cambia el destino.

Originalmente el grafo que modelaba estas relaciones vivía en `src/document/domain/DocumentsGraph.ts`, mezclado con tipos de navegación como `NavigationNode`, `Crumb` y `ParentEntry`.

Eso generaba confusión: `document` terminaba conteniendo conceptos de navegación, y `navigation` solo tenía servicios que consumían estructuras definidas en otro dominio.

## Decisión

Mover `DocumentsGraph` a `src/navigation/infraestructure/DocumentsGraph.ts` y dejar en `document/domain/types.ts` solo los tipos puros de documento.

El grafo sigue construyéndose a partir de `Document[]`, pero su responsabilidad es modelar la **estructura navegable y las dependencias de invalidación**, no el documento en sí.

## Tipos de aristas

El grafo modela cuatro tipos de dependencias:

| Arista | Significado |
|--------|-------------|
| `parent-child` | Relación jerárquica entre padre e hijo. |
| `sibling-order` | Orden relativo entre documentos hermanos. |
| `proxy-target` | Un documento proxy depende del documento destino. |
| `internal-link` | Un documento enlaza a otro en su cuerpo. |

## Cálculo de invalidatedIds

`DocumentsGraph.computeInvalidatedIds(changes)` recibe un conjunto de `DocumentChange` y devuelve los IDs que deben invalidarse, incluyendo dependientes transitivos.

El algoritmo:

1. Agrega los documentos cambiados directamente.
2. Según el `scope` del cambio (`content`, `metadata`, `all`), decide qué tipos de arista pueden propagar la invalidación.
3. Recorre el grafo por BFS desde los nodos cambiados hacia sus dependientes.
4. Devuelve `invalidatedIds`, `addedIds` y `removedIds`.

### ¿Por qué no invalidar solo los IDs que llegan en el webhook?

El webhook sabe qué archivo cambió, pero no sabe qué otras páginas renderizadas dependen de él. Por ejemplo:

- Si cambia `guia/completa.md`, `guia/resumen.md` (proxy) debe re-renderizarse.
- Si cambia el título de `guia/index.md`, los hijos deben actualizar breadcrumbs y sidebar.
- Si se mueve un hermano, el orden en el sidebar de todos los hermanos cambia.

Invalidar solo el archivo modificado dejaría esas páginas con caché desactualizado.

## Justificación

### 1. El grafo es una proyección, no una entidad

Un documento no cambia porque exista un grafo. El grafo es una vista derivada de los documentos para navegar e invalidar. Eso encaja mejor en `navigation`.

### 2. `document` se mantiene puro

`document` debe preocuparse de qué es un documento, su contenido y sus reglas de negocio. No de cómo se organiza en un sidebar.

### 3. Reutilización

El mismo grafo sirve para:

- Navegación: breadcrumbs, sidebar, hijos, padres.
- Invalidación: calcular qué páginas dependen de un cambio.

## Consecuencias

- `navigation` ahora contiene `DocumentsGraph` y `NavigationService`.
- `document` exporta `Document[]` y tipos puros; `navigation` construye el grafo a partir de ellos.
- `publishing` puede consumir el cálculo de invalidación a través de `NavigationService` o de un puerto futuro.

## Estado

Aceptada. `DocumentsGraph` vive en `src/navigation/infraestructure/DocumentsGraph.ts` y `NavigationService` lo consume para navegación e invalidación.

## Véase también

- [Composición raíz modular](./composicion-raiz-modular)
- [Webhook de GitHub: ubicación y dependencias](./webhook-github-arquitectura)
- [Thundering herd contra la API de GitHub](./thundering-herd-github)
