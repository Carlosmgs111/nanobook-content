---
title: "Overview"
description: "Overview of the adaptable architecture template."
date: 2026-07-30
author: "Astro"
tags: []
draft: false
---

# Plantilla para arquitectura adaptable

## Indice

- [Resumen](#resumen)
- [Aspectos Claves](#aspectos-claves)
- [Terminos Claves](#terminos-claves)
- [Escenarios](#escenarios)
  - [Escenario 1: "¡Hola Mundo!"](#escenario-1-hola-mundo)
  - [Escenario 2: Consultar una base de datos](#escenario-2-consultar-una-base-de-datos)
  - [Escenario 3: Consultar con lógica de filtrado](#escenario-3-consultar-con-lógica-de-filtrado)
  - [Escenario 4: Consultar con lógica de negocio](#escenario-4-consultar-con-lógica-de-negocio)
  - [Escenario 5: CRUD completo](#escenario-5-crud-completo-crear-leer-actualizar-eliminar)
  - [Escenario 6: Validación evolutiva](#escenario-6-validación-evolutiva)
    - [Fase 6a: Validación pragmática (controlador)](#fase-6a-validación-pragmática-controlador)
    - [Fase 6b: Validación estructurada (capa de aplicación)](#fase-6b-validación-estructurada-capa-de-aplicación)
  - [Escenario 7: Comunicación entre módulos](#escenario-7-comunicación-entre-módulos)
    - [Fase 7a: Adaptador delegador (monolito)](#fase-7a-adaptador-delegador-monolito)
    - [Fase 7b: Adaptador remoto (servicio separado)](#fase-7b-adaptador-remoto-servicio-separado)
  - [Escenario 8: Eventos y tareas asíncronas](#escenario-8-eventos-y-tareas-asíncronas)
    - [Fase 8a: Bus de eventos en memoria](#fase-8a-bus-de-eventos-en-memoria)
    - [Fase 8b: Broker externo](#fase-8b-broker-externo-servicios-distribuidos)
  - [Escenario 9: Manejo de errores con patrón Result](#escenario-9-manejo-de-errores-con-patrón-result)
    - [Clasificación de errores](#clasificación-de-errores)
    - [Flujo de errores](#flujo-de-errores)
  - [Escenario 10: Interceptores de aplicación](#escenario-10-interceptores-de-aplicación)
  - [Escenario 11: Caching](#escenario-11-caching)
  - [Escenario 12: Monolito modular completo](#escenario-12-monolito-modular-completo-escenario-límite)

## Resumen

Este documento formaliza una estrategia de desarrollo adaptable basada en la evolución progresiva de la arquitectura de un proyecto de software. El propósito es generar una serie de estándares y pautas que permitan construir piezas de software de cualquier escala: desde proyectos pequeños que requieren una estructura mínima, pasando por proyectos medianos con mayor complejidad, hasta sistemas grandes que demanden una arquitectura sofisticada o incluso distribuida.

La filosofía de este método se basa en el **monolito modular**, buscando un equilibrio entre simplicidad y velocidad de desarrollo por un lado, y alta cohesión y bajo acoplamiento por otro. Los módulos son autosuficientes y autocontenidos; si la complejidad del proyecto lo exige, un módulo puede migrar fuera de la base de código principal hacia su propio servicio con un cambio mínimo: basta mover su carpeta y reemplazar su adaptador de comunicación.

---

## Aspectos Claves

La metodología prioriza un enfoque pragmático. Se deliberadamente se deja de lado el rigor académico o las implementaciones mecánicas que, en etapas tempranas, entorpecen la entrega de valor tangible. Esto no significa sacrificar calidad: la arquitectura está diseñada para permitir la incorporación de capas de complejidad sin necesidad de reescribir, reorganizar o mover código existente.

La metodología contempla la existencia de **3 capas**, presentes en toda arquitectura limpia:

- **Capa de Infraestructura** — detalles técnicos (frameworks, bases de datos, APIs externas)
- **Capa de Aplicación** — casos de uso que orquestan el flujo de datos
- **Capa de Dominio** — entidades y reglas de negocio puras

Solo la capa de **Infraestructura** está presente desde el inicio. Esta capa alberga el punto de entrada de la aplicación (servidor, CLI, etc.) y, conforme el proyecto crece y se añaden capas internas, se convierte en la puerta de comunicación con el exterior. Esta progresión se fundamenta en el **Principio de Inversión de Dependencia**: las capas internas no conocen a las externas, pero las externas dependen de abstracciones definidas internamente. 

--- 

## Terminos Claves

- Monolito Modular
- Modulo
- Bajo acoplamiento
- Distribuido
- Autonomo
- Adaptativo
- DDD
- Arquitectura Hexagonal
- SOLID
- YAGNI
- KISS

---

## Escenarios

Los siguientes son escenarios que muestran las etapas de complejización de la arquitectura de un proyecto. Cada escenario incluye un **trigger** (qué necesidad obliga a añadir complejidad), las **capas activas**, el **flujo** de ejecución y el **trade-off** (qué se gana vs qué se añade).



### Escenario 1: "¡Hola Mundo!"

**Trigger:** Solo necesitas responder a una petición sin lógica ni datos externos.

| Infraestructura (inbound)     | Aplicación | Dominio | Infraestructura (outbound) |
| :---------------------------- | :--------- | :------ | :------------------------- |
| controlador (Astro: APIRoute) | N/A        | N/A     | N/A                        |

**Flujo:**

1. Request llega al controlador
2. Controlador responde directamente

**Trade-off:**

- Se gana: máxima simplicidad, cero overhead
- Se pierde: nada, es el punto de partida

---

### Escenario 2: Consultar una base de datos

**Trigger:** Necesitas persistir o recuperar datos de un sistema externo.

| Infraestructura (inbound)     | Aplicación | Dominio | Infraestructura (outbound) |
| :---------------------------- | :--------- | :------ | :------------------------- |
| controlador (Astro: APIRoute) | N/A        | N/A     | sqlite (SELECT)            |

**Flujo:**

1. Request llega al controlador
2. Controlador ejecuta la query directamente
3. Controlador retorna la respuesta

**Trade-off:**

- Se gana: acceso a datos reales
- Se añade: el controlador conoce detalles de la base de datos (query, schema)

---

### Escenario 3: Consultar con lógica de filtrado

**Trigger:** La consulta requiere lógica que no es trivial (filtros dinámicos, paginación, ordenamiento) y empieza a no pertenecer al controlador.

| Infraestructura (inbound)     | Aplicación                          | Dominio | Infraestructura (outbound) |
| :---------------------------- | :---------------------------------- | :------ | :------------------------- |
| controlador (Astro: APIRoute) | caso de uso (consultar con filtros) | N/A     | sqlite (SELECT)            |

**Flujo:**

1. Request llega al controlador
2. Controlador delega al caso de uso con los parámetros
3. Caso de uso construye la consulta y ejecuta
4. Caso de uso retorna resultado al controlador
5. Controlador responde

**Trade-off:**

- Se gana: el controlador se mantiene limpio, la lógica de consulta vive en un lugar con responsabilidad única
- Se añade: una capa intermedia y la coordinación entre controlador y caso de uso

---

### Escenario 4: Consultar con lógica de negocio

**Trigger:** La respuesta depende de reglas del dominio que no son solo filtrado (cálculos, validaciones de negocio, estados).

| Infraestructura (inbound)     | Aplicación                          | Dominio                             | Infraestructura (outbound) |
| :---------------------------- | :---------------------------------- | :---------------------------------- | :------------------------- |
| controlador (Astro: APIRoute) | caso de uso (consultar con filtros) | entidad (aplica reglas del negocio) | sqlite (SELECT)            |

**Flujo:**

1. Request llega al controlador
2. Controlador delega al caso de uso
3. Caso de uso obtiene datos de infraestructura
4. Caso de uso pasa datos a la entidad para aplicar reglas
5. Entidad retorna resultado transformado/validado
6. Caso de uso retorna al controlador
7. Controlador responde

**Trade-off:**

- Se gana: las reglas de negocio viven en el dominio, aisladas de detalles técnicos
- Se añade: la entidad como responsable de las reglas, coordinación de 4 capas

---

### Escenario 5: CRUD completo (Crear, Leer, Actualizar, Eliminar)

**Trigger:** Necesitas operaciones de escritura con validación y transaccionalidad.

| Infraestructura (inbound)               | Aplicación                              | Dominio                           | Infraestructura (outbound)                  |
| :-------------------------------------- | :-------------------------------------- | :-------------------------------- | :------------------------------------------ |
| controlador (POST/PUT/DELETE: APIRoute) | caso de uso (crear/actualizar/eliminar) | entidad (valida, calcula, decide) | sqlite (INSERT/UPDATE/DELETE + transacción) |

**Flujo (Crear como ejemplo):**

1. Request con datos llega al controlador
2. Controlador extrae datos y delega al caso de uso
3. Caso de uso crea la entidad con los datos
4. Entidad valida sus propias reglas (estado inicial válido)
5. Caso de uso inicia transacción
6. Caso de uso persiste via infraestructura
7. Caso de uso confirma transacción
8. Caso de uso retorna resultado al controlador
9. Controlador responde con status apropiado

**Trade-off:**

- Se gana: operaciones de escritura seguras con validación en el dominio y atomicidad
- Se añade: manejo de transacciones, validación de entrada, código de error/éxito

---

### Escenario 6: Validación evolutiva

**Trigger:** Las validaciones se repiten entre casos de uso, dependen del contexto, o las reglas de entrada se mezclan con reglas de negocio.

#### Fase 6a: Validación pragmática (controlador)

En etapas tempranas, validar directamente en el controlador con retornos anticipados es aceptable:

| Infraestructura (inbound)                     | Aplicación  | Dominio | Infraestructura (outbound) |
| :-------------------------------------------- | :---------- | :------ | :------------------------- |
| controlador (valida con `if` / retorna `400`) | caso de uso | entidad | sqlite                     |

**Flujo:**

1. Request llega al controlador
2. Controlador valida campos requeridos, formatos, rangos (`if (!name) return 400`)
3. Si pasa, delega al caso de uso
4. Caso de uso ejecuta y retorna

**Cuándo mantener esta fase:** Las validaciones son simples, no se repiten, y no dependen de contexto externo.

#### Fase 6b: Validación estructurada (capa de aplicación)

Cuando las validaciones crecen, se repiten, o necesitan acceso a datos externos (ej: "¿ya existe este email?"), la validación se mueve a la capa de aplicación:

| Infraestructura (inbound)       | Aplicación              | Dominio                     | Infraestructura (outbound) |
| :------------------------------ | :---------------------- | :-------------------------- | :------------------------- |
| controlador (solo extrae datos) | validador + caso de uso | entidad (reglas de negocio) | sqlite                     |

**Flujo:**

1. Request llega al controlador
2. Controlador extrae datos y delega al validador
3. Validador verifica reglas de entrada (formato, requeridos, unicidad via consulta)
4. Si falla, retorna error al controlador
5. Si pasa, delega al caso de uso
6. Caso de uso crea entidad, entidad aplica reglas de negocio internas
7. Caso de uso persiste y retorna resultado

**Trade-off:**

- Se gana: validaciones reutilizables, testables independientemente, controlador limpio
- Se añade: un componente validador, coordinación adicional

**Criterio de transición (6a → 6b):**

- La misma validación se repite en 2+ casos de uso
- La validación necesita consultar datos externos (ej: verificar unicidad en BD)
- Las reglas de validación dependen del contexto o rol del usuario

---

### Escenario 7: Comunicación entre módulos

**Trigger:** Un módulo necesita datos o acciones de otro módulo sin acoplarse directamente a su implementación.

#### Fase 7a: Adaptador delegador (monolito)

Cada módulo expone puertos en su capa de aplicación. Otro módulo implementa un adaptador en su infraestructura que delega al módulo externo:

| Infraestructura (inbound) | Aplicación  | Dominio | Infraestructura (outbound)                  |
| :------------------------ | :---------- | :------ | :------------------------------------------ |
| controlador               | caso de uso | entidad | adaptador delegador → puerto de otro módulo |

**Flujo (ejemplo: módulo Ventas necesita reservar stock en módulo Inventario):**

1. Request llega al controlador de Ventas
2. Caso de uso de Ventas llama a su puerto `HandleStockPort`
3. El adaptador `HandleStock` (infraestructura de Ventas) recibe la llamada
4. El adaptador delega al puerto `HandleStockForSalePort` del módulo Inventario
5. Inventario ejecuta la lógica y retorna resultado
6. El adaptador retorna el resultado al caso de uso de Ventas
7. Flujo continúa normalmente

**Estructura del adaptador delegador:**

```ts
// Módulo Ventas: infrastructure/HandleStock.ts
// Implementa un puerto local, delega a un puerto de otro módulo
export class HandleStock implements HandleStockPort {
  constructor(private handleStockForSale: HandleStockForSalePort) {}

  async reserveStock(itemId: string, quantity: number): Promise<Result<Error, void>> {
    return await this.handleStockForSale.reserveStock(itemId, quantity);
  }
}
```

**Ventaja clave:** Si el módulo Inventario se extrae a su propio servicio, solo hay que reemplazar el adaptador delegador por uno que use `fetch`/HTTP. El caso de uso y el puerto local no cambian.

#### Fase 7b: Adaptador remoto (servicio separado)

Cuando un módulo migra a servicio independiente, el adaptador cambia su implementación pero no su interfaz:

| Infraestructura (inbound) | Aplicación  | Dominio | Infraestructura (outbound)                       |
| :------------------------ | :---------- | :------ | :----------------------------------------------- |
| controlador               | caso de uso | entidad | adaptador HTTP (fetch) → API de servicio externo |

**Flujo:**

1. Mismo flujo interno hasta el adaptador
2. Adaptador serializa la petición y hace `fetch` al servicio externo
3. Deserializa la respuesta y retorna como `Result`
4. El resto del módulo no nota la diferencia

**Trade-off:**

- Se gana: módulos desacoplados, migración a servicio sin tocar la lógica del módulo
- Se añade: un adaptador por cada puerto inter-módulo, serialización/deserialización

**Criterio de transición (7a → 7b):**

- El módulo crece tanto que necesita su propio despliegue, equipo o base de datos
- Necesitas escalar un módulo independientemente del resto
- Diferentes módulos tienen diferentes requisitos de disponibilidad o rendimiento

---

### Escenario 8: Eventos y tareas asíncronas

**Trigger:** Una acción necesita notificar o disparar efectos secundarios en múltiples módulos sin acoplarse a ellos (enviar email, actualizar logs, notificar a otros módulos).

#### Fase 8a: Bus de eventos en memoria

Se usa un `EventEmitter` nativo como bus de eventos en memoria. Los casos de uso publican eventos; los suscriptores se registran en la composición raíz del proyecto:

| Infraestructura (inbound) | Aplicación                   | Dominio | Infraestructura (outbound)             |
| :------------------------ | :--------------------------- | :------ | :------------------------------------- |
| controlador               | caso de uso (publica evento) | entidad | suscriptores (email, log, otro módulo) |

**Flujo:**

1. Request llega al controlador
2. Caso de uso ejecuta la lógica principal
3. Caso de uso publica un evento en el bus (`eventBus.emit("sale.completed", { saleId })`)
4. Caso de uso retorna respuesta al controlador (no espera a los suscriptores)
5. Los suscriptores registrados reaccionan al evento de forma asíncrona:
   - Envían email de confirmación
   - Actualizan métricas
   - Notifican a otros módulos

**Composición raíz (ejemplo):**

```ts
// app.ts — se ejecuta al iniciar la aplicación
eventBus.on("sale.completed", async ({ saleId }) => {
  await emailService.sendConfirmation(saleId);
});

eventBus.on("sale.completed", async ({ saleId }) => {
  await metricsService.recordSale(saleId);
});
```

**Regla clave:** La publicación ocurre en la capa de aplicación (caso de uso), nunca en el dominio. El dominio no sabe que existe un bus de eventos. Los suscriptores se registran fuera de los módulos, en la composición raíz.

#### Fase 8b: Broker externo (servicios distribuidos)

Cuando los módulos se separan en servicios, el bus en memoria se reemplaza por un broker externo (Redis, RabbitMQ, etc.):

| Infraestructura (inbound) | Aplicación                   | Dominio | Infraestructura (outbound)                    |
| :------------------------ | :--------------------------- | :------ | :-------------------------------------------- |
| controlador               | caso de uso (publica evento) | entidad | adaptador de broker (Redis pub/sub, RabbitMQ) |

**Flujo:**

1. Mismo flujo hasta la publicación del evento
2. El adaptador de infraestructura serializa y envía al broker
3. Cada servicio tiene sus propios suscriptores conectados al broker
4. La respuesta al cliente no espera a los suscriptores

**Trade-off:**

- Se gana: desacoplamiento total entre módulos, escalabilidad independiente, tolerancia a fallos
- Se añade: infraestructura de broker, manejo de reintentos, idempotencia en suscriptores

**Criterio de transición (8a → 8b):**

- Los módulos se despliegan como servicios separados
- Necesitas que los eventos sobrevivan reinicios del servidor
- Múltiples servicios necesitan reaccionar al mismo evento

---

### Escenario 9: Manejo de errores con patrón Result

**Trigger:** Decisión fija de la metodología. El patrón Result se aplica desde el inicio como mecanismo estandarizado de manejo de errores, por su versatilidad y efectividad.

#### Clasificación de errores

| Tipo de error                | Origen                                         | Dónde se crea                | Cómo fluye                                                     |
| :--------------------------- | :--------------------------------------------- | :--------------------------- | :------------------------------------------------------------- |
| **Error de infraestructura** | BD, red, filesystem, API externa               | Adaptador de infraestructura | Se captura y se convierte en `Result.fail()`                   |
| **Error de dominio**         | Reglas de negocio violadas, estado inválido    | Entidad del dominio          | Se retorna como `Result.fail()` con error del dominio          |
| **Error de aplicación**      | Caso de uso no puede completar la orquestación | Caso de uso                  | Retorna `Result.fail()` propagando errores de capas inferiores |

#### Flujo de errores

| Infraestructura (inbound)                  | Aplicación                   | Dominio                  | Infraestructura (outbound)                    |
| :----------------------------------------- | :--------------------------- | :----------------------- | :-------------------------------------------- |
| controlador (traduce Result → HTTP status) | caso de uso (propaga Result) | entidad (retorna Result) | adaptador (captura excepciones → Result.fail) |

**Flujo de éxito:**

1. Request llega al controlador
2. Controlador delega al caso de uso
3. Caso de uso interactúa con el dominio
4. Dominio retorna `Result.ok(valor)`
5. Caso de uso retorna `Result.ok(valor)` al controlador
6. Controlador traduce a respuesta HTTP (200, 201, etc.)

**Flujo de error de infraestructura:**

1. Request llega al controlador
2. Caso de usa llama a infraestructura outbound
3. Infraestructura lanza excepción (BD caída, timeout, etc.)
4. Adaptador captura la excepción y retorna `Result.fail(new InfrastructureError("DB unavailable"))`
5. Caso de uso propaga el `Result.fail()` hacia arriba
6. Controlador recibe `Result.fail()`, traduce a HTTP 500

**Flujo de error de dominio:**

1. Request llega al controlador
2. Caso de uso crea entidad con datos del request
3. Entidad detecta violación de regla de negocio y retorna `Result.fail(new DomainError("Insufficient stock"))`
4. Caso de uso propaga el `Result.fail()`
5. Controlador recibe `Result.fail()`, traduce a HTTP 400 o 409 según el tipo de error

**Estructura del patrón Result:**

```ts
type Result<E, T> =
  | { ok: true; value: T }
  | { ok: false; error: E }
```

**Reglas:**

- Los adaptadores de infraestructura **nunca lanzan excepciones** hacia capas superiores. Capturan y convierten a `Result.fail()`
- Las entidades **nunca lanzan excepciones**. Retornan `Result` con errores del dominio
- Los casos de uso **propagan** los errores de las capas inferiores, no los ocultan
- El controlador es el **único responsable** de traducir `Result` a respuestas HTTP

**Trade-off:**

- Se gana: flujo de errores explícito, sin try/catch anidados, errores tipados y predecibles
- Se añade: boilerplate de crear tipos de error, envolver cada operación en Result

---

### Escenario 10: Interceptores de aplicación

**Trigger:** Necesitas aplicar la misma verificación o comportamiento transversal (auth, autorización, logging, auditoría) a múltiples casos de uso sin depender del middleware del framework.

#### Propuesta: Pipeline de interceptores a nivel de aplicación

En lugar de usar middlewares del framework (Express, Astro, etc.), se define un pipeline de interceptores que envuelven los casos de uso. Cada interceptor implementa una interfaz genérica y se compone en la raíz de la aplicación:

| Infraestructura (inbound) | Interceptors                   | Aplicación  | Dominio | Infraestructura (outbound) |
| :------------------------ | :----------------------------- | :---------- | :------ | :------------------------- |
| controlador               | auth → authorization → logging | caso de uso | entidad | adaptador                  |

**Interfaz del interceptor:**

```ts
interface Interceptor {
  execute(context: Context, next: () => Promise<Result<Error, any>>): Promise<Result<Error, any>>;
}
```

**Flujo:**

1. Request llega al controlador
2. Controlador crea un `Context` con los datos del request
3. Controlador pasa el contexto al pipeline de interceptores
4. `AuthInterceptor` verifica autenticación; si falla, retorna `Result.fail()` y corta el flujo
5. `AuthorizationInterceptor` verifica permisos; si falla, retorna `Result.fail()` y corta el flujo
6. `LoggingInterceptor` registra inicio, ejecuta `next()`, registra fin
7. Todos los interceptores pasan → se ejecuta el caso de uso
8. Caso de uso retorna `Result`
9. `Result` viaja de vuelta por la cadena de interceptores
10. Controlador traduce `Result` a respuesta HTTP

**Composición en la raíz:**

```ts
// app.ts — composición raíz
const createSalePipeline = createPipeline([
  new AuthInterceptor(authService),
  new AuthorizationInterceptor(["admin", "cashier"]),
  new LoggingInterceptor(logger),
]);

// El controlador usa el pipeline, no el caso de uso directamente
const result = await createSalePipeline.execute({ userId, body: request.body });
```

**Ventaja sobre middlewares de framework:**

- Independiente de la tecnología (funciona con Astro, Express, Fastify, CLI, etc.)
- Los interceptores pueden inyectar datos en el `Context` que el caso de uso consume
- Se testean como unidades aisladas
- El orden de ejecución es explícito y controlado por la composición

**Trade-off:**

- Se gana: interceptores framework-agnósticos, composables, testables independientemente
- Se añade: una capa de abstracción sobre el caso de uso, necesidad de definir `Context`

**Criterio de transición:**

- Cuando 2+ casos de uso necesitan la misma verificación (ej: autenticación)
- Cuando necesitas interceptores que inyecten datos al contexto del caso de uso
- Cuando cambias de framework y no quieres reescribir middlewares

---

### Escenario 11: Caching

**Trigger:** La misma consulta se repite frecuentemente y el costo de recomputar o consultar la base de datos es alto.

#### Propuesta: Decorator en infraestructura outbound

El caching vive como un **decorator** en la capa de infraestructura outbound. La capa de aplicación define un puerto (ej: `ProductRepository`), y el caching es un adapter que implementa ese mismo puerto pero internamente delega a otro adapter (el de base de datos). La aplicación no sabe que existe caché.

| Infraestructura (inbound) | Aplicación  | Dominio | Infraestructura (outbound)           |
| :------------------------ | :---------- | :------ | :----------------------------------- |
| controlador               | caso de uso | entidad | `CachedRepository` → `SqlRepository` |

**Flujo (consulta):**

1. Request llega al controlador
2. Caso de uso llama al puerto `ProductRepository.getById(id)`
3. `CachedRepository` recibe la llamada
4. `CachedRepository` busca en caché:
   - **Hit:** retorna el valor cacheado directamente
   - **Miss:** delega a `SqlRepository.getById(id)`, guarda el resultado en caché, retorna el valor
5. Caso de uso recibe los datos y continúa normalmente

**Flujo (escritura):**

1. Caso de uso llama a `ProductRepository.save(product)`
2. `CachedRepository` delega a `SqlRepository.save(product)`
3. `CachedRepository` invalida o actualiza la entrada en caché
4. Retorna resultado

**Estructura:**

```ts
// infrastructure/CachedProductRepository.ts
export class CachedProductRepository implements ProductRepositoryPort {
  constructor(
    private cache: Cache,
    private repository: ProductRepositoryPort // SqlProductRepository
  ) {}

  async getById(id: string): Promise<Result<Error, Product>> {
    const cached = this.cache.get<Product>(id);
    if (cached) return Result.ok(cached);

    const result = await this.repository.getById(id);
    if (result.ok) {
      this.cache.set(id, result.value, { ttl: 300 });
    }
    return result;
  }

  async save(product: Product): Promise<Result<Error, void>> {
    const result = await this.repository.save(product);
    if (result.ok) {
      this.cache.delete(product.id); // invalidación
    }
    return result;
  }
}
```

**Composición en la raíz:**

```ts
// app.ts — composición raíz
const sqlRepo = new SqlProductRepository(db);
const cachedRepo = new CachedProductRepository(redisCache, sqlRepo);

// El caso de uso recibe el puerto, no sabe si está cacheado o no
const useCase = new GetProductById(cachedRepo);
```

**Regla clave:** La capa de aplicación **nunca** decide si usar caché. Eso se resuelve en la composición raíz intercambiando adapters. Si necesitas desactivar el caché, solo cambias `cachedRepo` por `sqlRepo` en la composición.

**Trade-off:**

- Se gana: caché transparente para la aplicación, fácil de activar/desactivar, testable
- Se añade: lógica de invalidación, posible inconsistencia temporal entre caché y BD

**Criterio de transición:**

- La misma query se ejecuta frecuentemente con los mismos parámetros
- El costo de consultar la BD es significativamente mayor que el beneficio de la frescura
- Los datos no cambian frecuentemente o la inconsistencia temporal es aceptable

---

### Escenario 12: Monolito modular completo (escenario límite)

**Trigger:** Este es el "techo" de la primera versión de la metodología. Representa un módulo con todas las capas y mecanismos activos, funcionando como un todo cohesivo dentro de un monolito modular.

| Infraestructura (inbound) | Interceptors                   | Aplicación              | Dominio                 | Infraestructura (outbound)         |
| :------------------------ | :----------------------------- | :---------------------- | :---------------------- | :--------------------------------- |
| controlador (APIRoute)    | auth → authorization → logging | validador → caso de uso | entidad + value objects | cached repo → sql repo + event bus |

**Flujo completo (ejemplo: completar una venta):**

1. **Request** llega al controlador (`POST /api/sales`)
2. **Controlador** extrae datos del body y crea un `Context`
3. **AuthInterceptor** verifica que el usuario esté autenticado → si falla, `Result.fail(401)`
4. **AuthorizationInterceptor** verifica que el usuario tenga rol `cashier` o `admin` → si falla, `Result.fail(403)`
5. **LoggingInterceptor** registra inicio de operación
6. **Validador** verifica datos de entrada (items válidos, cantidades positivas, cliente existe) → si falla, `Result.fail(400)`
7. **Caso de uso** `CompleteSale` recibe datos validados
8. **Entidad** `Sale` se crea y aplica reglas de negocio (calcula totales, verifica descuentos, valida estado)
9. **Caso de uso** llama a `HandleStockPort.reserveStock()` → adaptador delega al módulo Inventario
10. **Módulo Inventario** reserva stock y retorna `Result`
11. **Caso de uso** inicia transacción y persiste via `CachedSaleRepository`
12. **CachedSaleRepository** delega a `SqlSaleRepository` (write-through, no cachea escrituras)
13. **Caso de uso** confirma transacción
14. **Caso de uso** publica evento `sale.completed` en el `EventBus`
15. **Caso de uso** retorna `Result.ok(sale)` al pipeline
16. **LoggingInterceptor** registra fin de operación con duración
17. **Controlador** traduce `Result.ok` a HTTP 201 con body de la venta

**Suscriptores del evento `sale.completed` (registrados en la composición raíz):**

- Envía email de confirmación al cliente
- Actualiza métricas de ventas
- Notifica al módulo de Reportes
- Invalida caché de productos afectados

**Composición raíz:**

```ts
// app.ts
const db = new SQLiteDatabase("./pos.db");
const cache = new InMemoryCache();
const eventBus = new EventEmitter();
const logger = new ConsoleLogger();
const authService = new JwtAuthService();

// Repositorios
const sqlSaleRepo = new SqlSaleRepository(db);
const cachedSaleRepo = new CachedSaleRepository(cache, sqlSaleRepo);

// Interceptores
const salePipeline = createPipeline([
  new AuthInterceptor(authService),
  new AuthorizationInterceptor(["admin", "cashier"]),
  new LoggingInterceptor(logger),
]);

// Caso de uso
const completeSale = new CompleteSale(
  cachedSaleRepo,
  new HandleStock(inventoryModule.handleStockForSale),
  eventBus
);

// Controlador
router.post("/api/sales", async (req) => {
  const context = { userId: req.user.id, body: req.body };
  const result = await salePipeline.execute(context, () =>
    completeSale.execute(context.body)
  );
  return result.ok
    ? { status: 201, body: result.value }
    : { status: mapErrorToStatus(result.error), body: result.error };
});

// Suscriptores de eventos
eventBus.on("sale.completed", async ({ saleId }) => {
  await emailService.sendConfirmation(saleId);
});

eventBus.on("sale.completed", async ({ saleId }) => {
  await metricsService.recordSale(saleId);
});
```

**Lo que este escenario demuestra:**

- Todas las capas coexisten sin conflicto
- Cada capa tiene una responsabilidad clara y acotada
- Los mecanismos (cache, eventos, interceptores) son transparentes para la lógica de negocio
- La composición raíz es el único lugar donde se "cablean" las dependencias
- Si un mecanismo se vuelve innecesario, se retira de la composición sin tocar el caso de uso
- Si un módulo migra a servicio, solo se reemplazan los adapters delegadores por adapters HTTP

**Este es el límite de complejidad para la primera versión de la metodología.** Cualquier necesidad adicional (broker externo, service mesh, etc.) corresponde a una evolución futura cuando el monolito modular se distribuya.
