---
id: "6122d841-b746-4b8e-af5b-9798453f9618"
title: Tipos de buses de eventos
description: Un bus de eventos no es una sola categoría. Diferencias entre buses intra-proceso, inter-proceso y distribuidos, y cuál encaja en cada escenario del proyecto.
date: 2026-09-14T00:00:00.000Z
author: Nanobook Team
tags:
  - arquitectura
  - event-bus
  - desacoplamiento
  - decision
draft: false
index: false
position: 0
---

## Contexto

El patrón **bus de eventos** se suele asociar a sistemas distribuidos con brokers como RabbitMQ, Kafka o SNS. Sin embargo, es también una herramienta válida para desacoplar piezas dentro de un mismo proceso. Tratar todos los buses como si fueran la misma cosa lleva a sobrediseñar soluciones locales o, al revés, a subestimar las necesidades de un sistema distribuido.

## Tipos de buses de eventos

| Tipo | Alcance | Ejemplo típico | Caso de uso | Compromiso |
|---|---|---|---|---|
| **Intra-proceso (in-memory)** | Dentro de un mismo proceso/runtime | Instancia en memoria, registry de handlers | Conectar módulos de la misma aplicación sin imports cruzados | Bajo coste y latencia, pero los eventos se pierden si el proceso cae |
| **Inter-proceso local** | Entre procesos en la misma máquina | Colas del SO, Redis pub/sub, Unix sockets | Servicios separados en el mismo host que deben comunicarse sin acoplamiento directo | Requiere infraestructura local; mayor durabilidad que in-memory |
| **Distribuido** | Entre servicios en red | RabbitMQ, Kafka, NATS, SNS/SQS, Azure Event Grid | Sistemas desacoplados que escalan de forma independiente | Alta disponibilidad y durabilidad, pero más latencia, complejidad operativa y coste |

Lo que tienen en común los tres tipos es el **mismo principio**: el emisor no conoce a los receptores, y los receptores no conocen la implementación interna del emisor. Lo que cambia son las **garantías de entrega, durabilidad y latencia**.

## Aplicación a Nanobook

El proyecto usa actualmente un bus de eventos **intra-proceso** (`InMemoryEventBus` en `src/shared/infraestructure/InMemoryEventBus.ts`). Esto es correcto porque:

- `document` y `publishing` viven en el mismo runtime SSR.
- No se necesita durabilidad de eventos entre reinicios del servidor.
- La latencia debe ser mínima porque la notificación forma parte de la respuesta de una operación síncrona (crear o actualizar un documento).
- Evita añadir infraestructura externa para un desacoplamiento que cabe perfectamente en memoria.

```text
  UpdateDocument
        │
        ▼
  eventBus.publish(DocumentUpdated)
        │
        ▼
  OnDocumentUpdatedHandler (en publishing)
        │
        ▼
  pagePublisher.invalidate(...)
```

## Cuándo cambiar de tipo

### De intra-proceso a inter-proceso local

Cuando se separen `document` y `publishing` en servicios distintos dentro del mismo host, o cuando se quiera que los eventos sobrevivan a un reinicio del servidor de aplicaciones. Redis pub/sub es una opción habitual.

### De inter-proceso local a distribuido

Cuando los servicios vivan en hosts distintos, escalen de forma independiente o necesiten garantías de entrega, ordenamiento o replay de eventos. Kafka o RabbitMQ encajan aquí.

### Regla práctica

> Usa el bus más simple que satisfaga las garantías reales que necesitas.

No añadas un broker distribuido solo porque el patrón se llame "bus de eventos". Un bus in-memory bien diseñado desacopla igual que uno distribuido; solo difiere en durabilidad y alcance.

## Relación con la notificación document/publishing

La decisión de quién declara la necesidad de notificación (ver [Cardinalidad de la notificación document/publishing](./notificacion-cambio-document)) es independiente del tipo de bus. Da igual si `document` declara un puerto o `publishing` se suscribe a eventos: el bus puede ser in-memory para un caso local o distribuido para un caso escalado.

La elección del tipo de bus depende de:

1. ¿Viven emisor y receptor en el mismo proceso?
2. ¿Pueden perderse eventos si el proceso se reinicia?
3. ¿Necesitan ordenarse, persistirse o re-procesarse?
4. ¿Cuál es el coste operativo aceptable?

## Conclusión

Un bus de eventos es un patrón de desacoplamiento, no una tecnología única. En Nanobook, el bus in-memory es la herramienta adecuada para conectar módulos del mismo proceso. Si la arquitectura evoluciona hacia servicios separados, el patrón se mantiene; solo cambia la implementación subyacente.

> El desacoplamiento conceptual no exige infraestructura distribuida.
