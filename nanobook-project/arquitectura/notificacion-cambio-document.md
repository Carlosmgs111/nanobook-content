---
id: "e23e0e3b-3da1-4e8b-87a9-3a58278f41eb"
title: Cardinalidad de la necesidad de notificación entre document y publishing
description: Por qué document debe seguir siendo la fuente del evento de cambio incluso cuando publishing declara la necesidad de ser notificado, y alternativas para mantener el desacoplamiento.
date: 2026-09-14T00:00:00.000Z
author: Nanobook Team
tags:
  - arquitectura
  - document
  - publishing
  - notificaciones
  - decision
draft: false
index: false
position: 0
---

## Contexto

El dominio `document` experimenta cambios de estado: se crea, edita, guarda y sus metadatos o contenido mutan. El dominio `publishing` necesita reaccionar ante esos cambios para invalidar cachés, regenerar salidas o actualizar vistas publicadas.

La pregunta arquitectónica es: ¿quién debe declarar la necesidad de notificación? Hay dos cardinalidades posibles:

1. **`document` declara la capacidad de notificar** → `publishing` se suscribe.
2. **`publishing` declara la necesidad de ser notificado** → `document` debe proveer un mecanismo que la satisfaga.

## Decisión vigente

La cardinalidad actual —`document` expone una necesidad o capacidad de notificación y `publishing` actúa como consumidor— es correcta porque:

- La fuente del cambio es la única que puede activar un flujo de notificación de forma fiable.
- `document` no conoce a `publishing`; solo expone un contrato de eventos genérico.
- `publishing` elige suscribirse, manteniendo la dirección de la dependencia hacia adentro.

> Cambiar quién declara la necesidad no elimina el requisito de que `document` sea reactivo al cambio de estado.

## Cardinalidad inversa: publishing declara la necesidad

Si `publishing` fuese quien dice "necesito mantenerme informado de cambios en documentos", `document` igual debería disponer de un mecanismo para activar un flujo de notificación. Lo que cambia es el **titular del contrato**, no la **existencia del mecanismo emisor**.

El riesgo de esta inversión es que `document` termine conociendo a `publishing` si no se introduce un intermediario. A continuación se listan alternativas para evitar ese acoplamiento.

## Alternativas para satisfacer la necesidad sin acoplamiento directo

### 1. Event bus / message broker centralizado

`document` publica eventos genéricos del tipo `DocumentChanged` en un bus. `publishing` se suscribe a esos eventos. Ambos dominios solo conocen el contrato del evento, no se conocen entre sí.

```text
  document ──► event bus ◄── publishing
```

- **Ventaja**: desacoplamiento total, fácil añadir nuevos consumidores.
- **Desventaja**: requiere infraestructura adicional y puede dificultar la trazabilidad local.

### 2. Ports and adapters con observable genérico

`publishing` define un puerto `DocumentChangeSubscriber`. `document` expone un `DocumentChangeNotifier` genérico. La composición se realiza en la raíz de la aplicación (`Application.ts` o un composition root), nunca dentro de los dominios.

```text
  document ──DocumentChanged event──► Application
                                       │
                                       ▼
                                  publishing
```

- **Ventaja**: inversión de dependencias clásica, testeable sin mocks de todo el sistema.
- **Desventaja**: requiere un punto de ensamblaje explícito.

### 3. Log de cambios / change stream

`document` escribe cada cambio en un stream o log inmutable. `publishing` lee del stream y reacciona. El desacoplamiento es temporal: `publishing` puede reconectarse y procesar eventos históricos.

```text
  document ──► change log ◄── publishing
```

- **Ventaja**: auditabilidad y tolerancia a caídas del consumidor.
- **Desventaja**: mayor complejidad operativa.

### 4. Puntos de extensión (hooks) en document

`document` expone un registro de hooks genéricos: `onDocumentChanged(hook)`. `publishing` registra su hook desde fuera, típicamente durante la inicialización de la aplicación.

```typescript
// En document
const hooks: DocumentChangeHook[] = [];
export function onDocumentChanged(hook: DocumentChangeHook) {
  hooks.push(hook);
}

async function afterSave(document: Document) {
  for (const hook of hooks) await hook(document);
}

// En Application.ts (o equivalente)
onDocumentChanged(publishing.handleDocumentChanged);
```

- **Ventaja**: simple de implementar, no requiere bus de eventos.
- **Desventaja**: puede volverse opaco si crecen demasiados hooks.

### 5. Observer pattern clásico

`document` mantiene una lista de observadores que implementan `DocumentObserver`. `publishing` implementa esa interfaz. La suscripción se inyecta desde el composition root.

```typescript
interface DocumentObserver {
  onChanged(document: Document): Promise<void> | void;
}

class DocumentNotifier {
  private observers: DocumentObserver[] = [];
  addObserver(observer: DocumentObserver) { this.observers.push(observer); }
  notifyChanged(document: Document) {
    for (const observer of this.observers) observer.onChanged(document);
  }
}
```

- **Ventaja**: patrón familiar, contrato explícito.
- **Desventaja**: `document` debe gestionar el registro de observadores.

## Ejemplo concreto sobre el codebase actual

El proyecto ya dispone de dos mecanismos que permiten invertir la cardinalidad sin crear un acoplamiento directo: el puerto `DocumentChangeNotifier` y el `EventBus` compartido.

### Cardinalidad actual (document declara el puerto)

El contrato vive en `src/document/application/ports/DocumentChangeNotifier.ts`:

```typescript
export interface DocumentChangeNotifier {
  onDocumentCreated(documentId: string): Promise<Result<DocumentNotificationError, void>>;
  onDocumentUpdated(documentId: string): Promise<Result<DocumentNotificationError, void>>;
}
```

`UpdateDocument` y `CreateDocument` lo reciben como dependencia y lo invocan tras persistir. `publishing` implementa el adaptador `PublishingDocumentChangeNotifier`, que invalida la caché de páginas renderizadas. La composición ocurre en `Application.ts`:

```typescript
const documentChangeNotifier = new PublishingDocumentChangeNotifier(
  publishingModule.pagePublisher
);
const documentModule = await DocumentModule.create(eventBus, documentChangeNotifier);
```

Aquí `publishing` depende de un contrato definido en `document`, lo cual preserva la estabilidad del dominio base.

### Cardinalidad inversa (publishing declara la necesidad) con EventBus

En este codebase, la forma más limpia de invertir la declaración sin acoplar es usar el bus de eventos ya existente en `src/shared/domain/bus/EventBus.ts`.

`publishing` declararía su necesidad como un conjunto de handlers:

```typescript
// src/publishing/application/event-handlers/OnDocumentUpdatedHandler.ts
import type { EventHandler } from "../../../shared/domain/bus/EventBus";
import type { DocumentUpdated } from "../../../document/domain/events/DocumentUpdated";
import type { PagePublisher } from "../PagePublisher";

export class OnDocumentUpdatedHandler implements EventHandler<DocumentUpdated> {
  constructor(private pagePublisher: PagePublisher) {}

  async handle(event: DocumentUpdated): Promise<Result<EventBusError, void>> {
    return this.pagePublisher.invalidate([event.payload.id]);
  }
}
```

`document` seguiría publicando eventos de dominio genéricos tras cada cambio, sin saber quién los consume:

```typescript
// En UpdateDocument.ts (sin cambios esenciales)
await this.eventBus.publish(DocumentUpdated.create({ id: updatedDocumentId }));
```

La composición se movería a `Application.ts`:

```typescript
const publishingModule = await PublishingModule.create(eventBus);

// publishing declara qué eventos necesita escuchar
publishingModule.registerEventHandlers();

const documentModule = await DocumentModule.create(eventBus);
```

Con este enfoque:

- `document` no conoce a `publishing`.
- `publishing` no conoce la implementación interna de `document`, solo los eventos de dominio públicos.
- El contrato de suscripción lo define `publishing` (a través de `registerEventHandlers`), pero la activación sigue siendo responsabilidad de `document` al publicar eventos.

### Comparación directa

| Aspecto | Cardinalidad actual | Cardinalidad inversa con EventBus |
|---|---|---|
| Titular del contrato | `DocumentChangeNotifier` en `document` | Handlers en `publishing` + eventos en `shared` |
| Quién activa el flujo | `document` llama al notifier | `document` publica en el bus |
| Quién consume | Adaptador en `publishing` | Handlers en `publishing` |
| Acoplamiento | `publishing` depende de `document` | Ambos dependen de `shared` |
| Ventaja | Directo, sincrónico, fácil de trazar | Desacoplado, múltiples consumidores, asíncrono |
| Desventaja | Acopla la entrega al caso de uso | Requiere bus y manejo de fallos asíncronos |

## Conclusión

Da igual quién declare la necesidad: `document` siempre termina siendo responsable de activar el flujo de notificación al cambiar de estado. La cardinalidad actual es preferible porque el dominio origen del evento declara su propia capacidad de notificación, dejando a `publishing` como consumidor opcional.

Si en el futuro se invierte la cardinalidad, se debe adoptar una de las alternativas anteriores para evitar que `document` dependa de `publishing`. La regla de oro es:

> El dominio que cambia no debe conocer a los dominios que reaccionan.
