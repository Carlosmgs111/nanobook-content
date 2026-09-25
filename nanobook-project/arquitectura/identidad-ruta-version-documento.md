---
id: "8d1d0ceb-2515-4942-a6f3-e653ac79fdb1"
title: "Identidad, ruta y versión de los documentos"
description: "Separación entre identidad estable, ubicación mutable y versión de estado."
date: 2026-09-25
author: "Nanobook"
tags: ["document", "identity", "path", "version", "cache"]
draft: false
index: false
---

# Identidad, ruta y versión de los documentos

Un documento tiene tres propiedades distintas:

- `documentId`: identidad estable de la entidad.
- `path`: ubicación mutable dentro del árbol y URL pública.
- `version`: identidad del estado actual del contenido y sus metadatos.

El path no debe usarse como clave primaria de una futura base de datos. Mover o renombrar un documento debe actualizar su path, no eliminar y recrear la entidad.

La versión se calcula a partir de los hashes del contenido y los metadatos, pero no incluye el path. Por tanto, mover un documento no invalida su versión de contenido, aunque sí puede requerir recalcular navegación y rutas.

Los documentos nuevos persisten su `documentId` en el frontmatter. Los documentos heredados pueden usar temporalmente una identidad `legacy:` derivada del path hasta completar la migración de identidades.
