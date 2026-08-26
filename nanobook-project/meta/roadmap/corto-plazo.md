---
title: Corto plazo
description: Mejoras inmediatas para cerrar la fase alpha y estabilizar el día a
  día de Nanobook.
date: 2026-08-21T00:00:00.000Z
author: Nanobook
tags:
  - roadmap
  - corto-plazo
  - nanobook
draft: false
index: false
position: 0
---

# Corto plazo

Horizonte: próximas 2–4 semanas. El objetivo es terminar de pulir la alpha para que Nanobook sea usable de forma cotidiana.

## Navegación y lectura

- [x] **TOC activo con resaltado**: hacer que la tabla de contenidos resalte la sección visible mientras se hace scroll.
- [x] **Sidebar mobile usable**: definir y implementar el comportamiento del sidebar en pantallas pequeñas (drawer, overlay, cierre al hacer click fuera).
- [ ] **Atajos de teclado básicos**: añadir navegación por teclado para abrir/cerrar sidebar (`Cmd/Ctrl + B`) y enfocar la búsqueda (`Cmd/Ctrl + K`).

## Contenido y organización

- [x] **Estructura de `nanobook-project/`**: reorganizar en secciones semánticas (`meta`, `arquitectura`, `interfaz`, `editor`, `flujo-de-trabajo`).
- [x] **Roadmap poblado**: crear fase 0, corto plazo, medio plazo y largo plazo.
- [x] **`position` y descripciones**: revisar y asignar `position` a todos los `index.md` del proyecto.
- [ ] **Guías iniciales**: terminar de pulir `guias/markdown.md` y evaluar si se añade una guía de frontmatter.
- [ ] **Gestion de contenido**:
  - [ ] **Creacion**
  - [x] **Edicion**
  - [ ] **Eliminacion**

## Calidad y build

- [x] **Ordenamiento consistente**: corregir `IndexList.astro` para que respete el campo `position`.
- [ ] **Build limpio**: revisar y eliminar advertencias de build actuales (chunks grandes, `getStaticPaths` en rutas dinámicas, plugins deprecados de markdown).
- [ ] **Accesibilidad básica**: auditar contraste de colores, roles ARIA y orden de foco en los elementos interactivos principales.

## Internacionalización

- [ ] **Base para i18n**: investigar cómo añadir soporte de inglés sin cambiar el español por defecto (rutas, frontmatter, UI).

## Criterios de cierre de corto plazo

- La navegación intra-página funciona y se visualiza correctamente.
- El sidebar es usable en mobile.
- `npm run build` no genera advertencias que distraigan del desarrollo.
- Se tiene una estrategia clara para añadir inglés en el futuro.