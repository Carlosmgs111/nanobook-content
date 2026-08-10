---
title: "Sobrepensar"
description: "La necesidad de encontrar soluciones a problemas que no existen."
date: 2026-07-30
author: "Astro"
tags: []
draft: false
---

# 🤯 Sobrepensar

## La necesidad de encontrar soluciones a problemas que no existen

Estaba programando tranquilamente mi proyecto experimental: un sistema POS. 💻

Una de las características que quería conseguir era una interfaz fluida y con una respuesta inmediata a las acciones del usuario. Para ello decidí implementar **actualizaciones optimistas** (_optimistic updates_).

La idea es sencilla: asumir que todo va a salir bien. El usuario recibe un feedback inmediato mientras el cambio se sincroniza con el backend en segundo plano. Si algo falla, el sistema intenta recuperarse automáticamente sin que el usuario lo note y, solo si después de varios intentos sigue sin conseguirlo, revierte el estado anterior y muestra el error correspondiente.

Hasta ahí, todo parecía bastante simple. 😌

El estado que necesitaba controlar era, literalmente, un número: la cantidad de unidades de un producto en el carrito. 🛒

El motivo para utilizar actualizaciones optimistas surgía de un escenario muy común: un usuario presiona el botón de incrementar o disminuir la cantidad varias veces en pocos milisegundos.

Si cada clic dispara una petición al servidor, el resultado es algo como esto:

```text
n clics
    ↓
n fetches
    ↓
n llamadas al backend
    ↓
n respuestas
    ↓
n actualizaciones del frontend
    ↓
n renderizados
```

No hace falta ser un genio para darse cuenta de que eso es insostenible y ofrece una experiencia de usuario bastante pobre. 📉

La solución parecía obvia: **debounce**. ⏳

En lugar de enviar una petición por cada clic, la interfaz actualiza el estado local inmediatamente y retrasa la sincronización con el backend hasta que la ráfaga de clics termina. Al final solo se envía el estado definitivo.

                ┌─────────────────────────┐
                │ Usuario pulsa (+/-)     │
                └─────────────┬───────────┘
                              │
                              ▼
                 Actualizar Store (Optimista)
                              │
                              ▼
               Actualizar UI inmediatamente
                              │
                              ▼
                  Reiniciar temporizador
                     (debounce 300 ms)
                              │
                     ¿Más clics antes de
                      terminar el tiempo?
                    ┌─────────┴─────────┐
                    │                   │
                  Sí│                   │No
                    ▼                   ▼
           Reiniciar debounce      Enviar fetch
                                        │
                                        ▼
                              ¿La petición fue exitosa?
                               ┌────────┴────────┐
                               │                 │
                             Sí│                 │No
                               ▼                 ▼
               Copiar estado actual        Reintentar N veces
              del Store → sessionStorage         │
                                                 ▼
                                   ¿Todos los intentos fallaron?
                                        ┌────────┴────────┐
                                        │                 │
                                      No│                 │Sí
                                        ▼                 ▼
                                  Confirmar cambio   Recuperar snapshot
                                                     desde sessionStorage
                                                             │
                                                             ▼
                                                Actualizar Store (rollback)
                                                             │
                                                             ▼
                                                  La UI se actualiza sola
                                                             │
                                                             ▼
                                                 Mostrar mensaje de error

Problema resuelto.

O eso creía... 😅

Entonces apareció esa pequeña voz en mi cabeza:

> 🤔 "¿Y si el backend falla?"
>
> 🤔 "¿Cómo recupero el estado anterior?"
>
> 🤔 "¿Y si el usuario sigue interactuando mientras tanto?"
>
> 🤔 "¿Qué pasa si fallan varios intentos consecutivos?"
>
> 🤔 "¿Necesito un mecanismo de reconciliación?"
>
> 🤔 "¿Versionado?"
>
> 🤔 "¿Snapshots?"
>
> 🤔 "¿Una cola de operaciones?"

Y ahí empezó el desastre. 🌪️

Lo que era un contador terminó convirtiéndose, en mi cabeza, en un sistema distribuido con mecanismos de recuperación dignos de una plataforma financiera. 🏦

Probé varias ideas. Algunas eran interesantes. Otras incluso podrían tener sentido en escenarios mucho más complejos.

Pero no en este.

Estaba intentando resolver problemas que todavía no existían. 😵‍💫

Así que hice algo que debí hacer desde el principio: dejar de pensar durante un momento, volver al problema original y recorrer el flujo paso a paso. 🚶‍♂️

¿Qué necesitaba realmente?

Muy poco.

Cada vez que el estado cambiaba correctamente, bastaba con persistirlo en `sessionStorage`, donde ya tenía una copia del estado que manejaba el _store_. 💾

Si la sincronización con el backend fallaba después de varios intentos, simplemente recuperaba ese estado desde `sessionStorage` y lo aplicaba nuevamente al _store_.

Y ya. 🙃

No hacían falta _snapshots_ sofisticados, ni mecanismos de reconciliación, ni arquitecturas complejas.

Solo necesitaba guardar un número y restaurarlo cuando fuera necesario.

A veces programar no consiste en encontrar la solución más ingeniosa.

Consiste en dejar de buscar soluciones para problemas que todavía no existen. 🚀