---
id: "3d6ae747-fb1e-410b-8bdb-db8f91e04339"
title: "Renderizado local confirmado en edition"
description: "Cómo la página pública muestra el HTML que edition confirmó tras guardar."
date: 2026-09-24
author: "Nanobook Team"
tags:
  - arquitectura
  - edition
  - renderizado
  - cache
draft: false
index: false
---

# Renderizado local confirmado en edition

## Decisión

La página pública puede reemplazar el cuerpo recibido del servidor con el HTML
que `edition` guardó en `sessionStorage`, siempre que corresponda exactamente a
una versión confirmada por una escritura exitosa.

## Flujo

1. `SaveDocument` renderiza el documento staged y espera ese resultado.
2. Tras el `PATCH` exitoso, guarda el `SerializedEntry` como documento
   confirmado.
3. La página pública lee el preview cacheado mediante `GetRenderedDocument` y
   reemplaza `#document-body` solo cuando su fuente coincide con el documento
   confirmado.
4. Un cambio posterior en el editor elimina la confirmación.

## Consecuencias

La navegación a la página pública sigue obteniendo el documento inicial desde
el servidor, pero no realiza una consulta adicional para resolver la versión
recién guardada. El cuerpo mostrado se toma de la versión renderizada por
`edition`, por lo que no queda sujeto a una caché de servidor todavía antigua.
