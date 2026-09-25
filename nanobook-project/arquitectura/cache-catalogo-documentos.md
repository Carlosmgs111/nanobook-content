---
id: "0ad0bcf1-8e2a-4d7a-b7cb-1a87e71e3a0e"
title: "Cache del catálogo de documentos"
description: "Diseño para mover la cache de Document[] desde GitHubRepository al módulo document."
date: 2026-09-24
author: "Nanobook Team"
tags:
  - arquitectura
  - cache
  - documentos
  - github
draft: false
index: false
---

# Cache del catálogo de documentos

## Objetivo

El módulo `document` debe controlar la política de cache del catálogo de
documentos sin depender del adaptador que persiste el contenido. Los adaptadores
de filesystem, GitHub o memoria deben concentrarse en leer y escribir su fuente.

El cambio busca conservar la mejora de latencia obtenida al actualizar una sola
entrada cacheada y eliminar de `GitHubRepository` la responsabilidad de manejar
el TTL, la deduplicación de cargas de `Document[]` y la consistencia del catálogo.

## Alcance

- Cache en memoria del catálogo `Document[]` dentro de cada instancia de
  `DocumentModule`.
- TTL configurable, inicialmente habilitado para la fuente GitHub con el valor
  actual de 300.000 ms.
- Una sola carga del repositorio fuente cuando varias lecturas concurrentes
  encuentran la cache vacía.
- Actualización puntual de una entrada después de un `update` confirmado.
- Inserción puntual después de un `create` confirmado cuando ya existe un
  catálogo cacheado.
- Invalidación segura si el documento persistido no puede reconstruirse con el
  parser del módulo.
- Preservación del comportamiento de filesystem y memoria sin cache de catálogo
  en la primera integración.

Quedan fuera de este cambio la cache HTTP/CDN, `RenderedPageCache`, la
reconstrucción dinámica del grafo de navegación y una cache distribuida del
catálogo entre procesos.

## Responsabilidades

| Componente | Responsabilidad |
|---|---|
| `DocumentModule` | Compone repositorio fuente, parser, cache y casos de uso. Decide si la cache está habilitada. |
| `CachedContentRepository` | Aplica cache-aside a lecturas, deduplica cargas concurrentes y sincroniza la cache después de escrituras confirmadas. |
| `DocumentCatalogCache` | Define las operaciones sobre un catálogo ya construido. No conoce GitHub ni filesystem. |
| `InMemoryDocumentCatalogCache` | Mantiene el snapshot y su expiración dentro del proceso. |
| `GitHubRepository` | Lee y escribe archivos en GitHub. Conserva solamente la cache del árbol remoto y la deduplicación de esa petición. |
| `PublishingModule` | Mantiene la cache de HTML renderizado y responde a eventos de documento. |

## Diseño

```text
CreateDocument / UpdateDocument / GetDocument / GetAllDocuments
                            │
                            ▼
                 CachedContentRepository
                    │               │
                    │               └── DocumentCatalogCache
                    │                    (memoria + TTL)
                    ▼
                 ContentRepository
                    │
        ┌───────────┼────────────┐
        ▼           ▼            ▼
    GitHub       Filesystem    InMemory
```

`CachedContentRepository` implementa el mismo puerto `ContentRepository`, por
lo que los casos de uso y los resolutores internos no necesitan conocer la
cache.

### Contrato de cache

```typescript
export interface DocumentCatalogCache {
  get(): Document[] | null;
  set(documents: Document[]): void;
  insert(document: Document): boolean;
  replace(document: Document): boolean;
  invalidate(): void;
}
```

`insert` y `replace` devuelven `false` cuando no existe un snapshot vigente o
cuando la operación no puede aplicarse de forma consistente. El decorador
invalida el snapshot ante una reconstrucción inválida o un conflicto.

La implementación devuelve copias del array para impedir que un consumidor
modifique accidentalmente el catálogo almacenado. El TTL pertenece a la
implementación, no al puerto.

### Lecturas

1. `list()` consulta `DocumentCatalogCache.get()`.
2. Si existe un snapshot vigente, lo devuelve.
3. Si no existe, comparte una única promesa de carga entre llamadas concurrentes.
4. Solo guarda el resultado cuando el repositorio fuente devuelve `Result.ok`.
5. Cada escritura confirmada incrementa una generación interna. Una carga que
   empezó antes de esa escritura puede completar la petición que ya estaba en
   curso, pero no puede poblar la cache con el snapshot anterior.
6. Un fallo limpia la promesa en vuelo para permitir un reintento posterior.
7. `getById()` y `listChildren()` derivan su resultado de `list()` para que todas
   las lecturas observen el mismo snapshot.

### Actualización

1. El decorador llama a `source.update(document)`.
2. Si la escritura falla, devuelve el error y conserva el snapshot anterior.
3. Si GitHub confirma el `PUT`, reconstruye el documento mediante
   `createDocumentFromRaw` y el parser configurado por `DocumentModule`.
4. Si hay snapshot vigente, reemplaza únicamente el documento con el mismo ID.
5. Si no hay snapshot, no inicia una carga remota adicional.
6. Si la reconstrucción o el reemplazo falla, invalida el catálogo completo; la
   siguiente lectura vuelve a obtener una representación autoritativa de la
   fuente.

Esta sincronización ocurre antes de devolver el resultado del repositorio. No
depende de `DocumentUpdated`, porque el evento también atiende efectos de otros
módulos y no debe dejar una ventana donde la petición siguiente vea contenido
antiguo.

### Creación

1. El decorador llama a `source.create(document)`.
2. Si falla, la cache permanece intacta.
3. Si se confirma y existe snapshot, reconstruye e inserta el documento.
4. Si el snapshot no existe, no lo crea parcialmente.
5. Un conflicto o reconstrucción inválida invalida el catálogo.

Crear un archivo cambia el árbol de GitHub. Por ello `GitHubRepository.create()`
debe invalidar su cache del árbol remoto después de confirmar el `PUT`. Esa
cache sigue siendo responsabilidad del adaptador porque almacena una respuesta
propia de la API de GitHub, no objetos del dominio.

## Composición

`DocumentModule.create()` construye un único `UnifiedDocumentParser` y lo
comparte con el repositorio fuente y el decorador. La cache de catálogo se
habilita inicialmente para `CONTENT_SOURCE=github`; filesystem conserva lecturas
directas para que cambios externos durante desarrollo sean visibles de inmediato.

```typescript
const parser = new UnifiedDocumentParser();
const sourceRepository = await createContentRepository({ source, parser });
const contentRepository = source === "github"
  ? new CachedContentRepository(
      sourceRepository,
      new InMemoryDocumentCatalogCache(300_000),
      parser
    )
  : sourceRepository;
```

No se usa un `Map` global. `getApp()` ya conserva una instancia compartida de
`Application` por proceso y, con ella, una instancia de `DocumentModule`.

## Errores y consistencia

- Los errores del repositorio fuente se propagan sin convertirse en aciertos de
  cache.
- Un fallo de lectura no se cachea.
- Una escritura fallida no modifica la cache.
- Una escritura confirmada y una reconstrucción fallida invalidan la cache.
- Una cache vencida se trata como ausente.
- La cache no corrige documentos inválidos en la fuente; la siguiente lectura
  devuelve el error de parseo correspondiente.

## Observabilidad

La primera versión no agrega una dependencia de métricas. El diseño deja un
punto único (`CachedContentRepository`) donde posteriormente se pueden registrar
hits, misses, cargas compartidas, invalidaciones y duración de la carga remota.

## Evolución

Si el catálogo debe compartirse entre procesos, no se deben serializar objetos
`Document` directamente porque contienen el parser en memoria. Una cache
distribuida deberá almacenar una representación serializable y reconstruir los
documentos con el parser del módulo al leerla.

La invalidación causada por webhooks externos y la reconstrucción incremental
del grafo de navegación requieren un diseño coordinado con `NavigationModule` y
se abordarán como un cambio separado.
