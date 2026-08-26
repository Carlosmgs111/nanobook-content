---
title: "Decisión: cache isomórfico en lugar de ISR nativo de plataforma"
description: "Registro de la decisión de usar un sistema de cache de cuerpos isomórfico en lugar de depender de ISR nativo de Vercel."
date: 2026-08-24
author: "Nanobook Team"
tags:
  - arquitectura
  - decision
  - isr
  - cache
  - incremental
draft: false
index: false
---

# Decisión: cache isomórfico en lugar de ISR nativo de plataforma

## Contexto

Durante la planificación de la Fase 7 del recompilado incremental se evaluaron dos enfoques para servir y regenerar páginas de contenido en producción:

1. Usar ISR nativo de Vercel como mecanismo principal de cache y revalidación.
2. Usar un sistema de cache de cuerpos propio, abstracto, con backends intercambiables, y cabeceras HTTP estándar para cache de páginas completas.

## Opciones evaluadas

### Opción A: ISR nativo de Vercel

Ventajas:
- Configuración mínima con @astrojs/vercel.
- Revalidación automática por TTL.
- Revalidación bajo demanda mediante revalidate().
- CDN global integrada.

Desventajas:
- Vendor lock-in: revalidate() y la configuración isr del adapter solo funcionan en Vercel.
- El cache de cuerpos implementado en fases previas quedaría relegado a un rol secundario y no portable.
- Cambiar a Netlify, Cloudflare o Node requeriría reescribir la lógica de invalidación y cacheo.
- En entornos serverless, FileSystemRenderedPageCache no persiste entre invocaciones.

### Opción B: Cache isomórfico con RenderedPageCache abstracto

Ventajas:
- El mismo código funciona en Vercel, Netlify, Cloudflare, Node o cualquier entorno serverless.
- El cache de cuerpos se convierte en un contrato central con múltiples implementaciones.
- La invalidación es genérica: un endpoint /api/invalidate limpia el cache sin depender de APIs propietarias.
- El cache de página completa se resuelve con cabeceras HTTP estándar como stale-while-revalidate.
- Es más fácil de testear localmente.

Desventajas:
- Requiere definir e implementar la interfaz RenderedPageCache y sus backends.
- No aprovecha optimizaciones específicas de Vercel de forma automática.
- Es necesario configurar el backend de cache adecuado para cada entorno.

## Decisión

Se elige la Opción B: cache isomórfico con RenderedPageCache abstracto.

## Justificación

El objetivo del proyecto es que Nanobook sea portable y no dependa de una plataforma específica de despliegue. Un sistema que funcione solo en Vercel contradice ese principio. Además, el trabajo invertido en fases previas para construir PageRenderer y FileSystemRenderedPageCache debe seguir siendo relevante en producción, no solo en desarrollo local.

La cabecera Cache-Control: stale-while-revalidate es un estándar web soportado por la mayoría de CDNs, lo que permite obtener un comportamiento similar al ISR nativo sin vendor lock-in.

## Consecuencias

- RenderedPageCache se convierte en un contrato central del dominio rendering.
- Se mantienen e implementan múltiples backends: filesystem, memory, redis, kv.
- El endpoint /api/invalidate es genérico y no usa revalidate() de Vercel.
- La elección de adapter de Astro pasa a ser una decisión de despliegue, no de arquitectura.
- Se mantiene la puerta abierta a usar ISR nativo de Vercel más adelante como optimización adicional, pero nunca como requisito.

## Estado

Aceptada. El plan de Fase 7 se actualizó para reflejar este enfoque.
