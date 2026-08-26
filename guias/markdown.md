---
title: "Guía de Markdown"
description: "Referencia completa de Markdown para escribir contenido en Nanobook, incluyendo sintaxis básica, frontmatter, enlaces internos y Markmap."
date: 2026-08-21
author: "Nanobook"
tags: ["markdown", "guia", "referencia", "markmap"]
draft: false
index: false
---

v2.0

# Guía de Markdown

Nanobook usa Markdown como formato principal para el contenido. Esta guía recoge la sintaxis que puedes utilizar y las convenciones específicas del proyecto.

---

## 1. Estructura de un documento

Todo archivo de contenido en Nanobook comienza con un bloque de **frontmatter**:

```markdown
---
title: "Título del documento"
description: "Breve descripción del contenido"
date: 2026-08-21
author: "Autor"
tags: ["tag1", "tag2"]
draft: false
index: false
position: 0
---
```

### Campos del frontmatter

| Campo | Obligatorio | Descripción |
|---|---|---|
| `title` | Sí | Título del documento. |
| `description` | Sí | Descripción corta que aparece en índices y SEO. |
| `date` | Sí | Fecha de publicación (`YYYY-MM-DD` o ISO completa). |
| `author` | Sí | Autor del contenido. |
| `tags` | Sí | Lista de etiquetas. Usa `[]` si no hay etiquetas. |
| `draft` | No | Si es `true`, el documento no se genera. |
| `index` | Solo para `index.md` | Marca el archivo como representación de una carpeta. |
| `position` | No | Orden del documento dentro de su índice. |
| `cover` | No | URL de imagen de portada para índices visuales. |

---

## 2. Sintaxis básica de Markdown

### Encabezados

```markdown
# Encabezado 1
## Encabezado 2
### Encabezado 3
#### Encabezado 4
```

Los encabezados definen la estructura del documento y son los que usa la tabla de contenidos (TOC) para la navegación intra-página.

### Párrafos y saltos de línea

```markdown
Este es un párrafo.

Este es otro párrafo separado por una línea en blanco.
```

Para forzar un salto de línea dentro del mismo párrafo, deja dos espacios al final de la línea.

### Énfasis

```markdown
**Texto en negrita**
*Texto en cursiva*
***Texto en negrita y cursiva***
~~Texto tachado~~
```

### Citas

```markdown
> Esta es una cita.
> Puede ocupar varias líneas.
```

### Listas

**Desordenadas:**

```markdown
- Elemento 1
- Elemento 2
  - Sub-elemento 2.1
  - Sub-elemento 2.2
- Elemento 3
```

**Ordenadas:**

```markdown
1. Primer paso
2. Segundo paso
   1. Sub-paso 2.1
   2. Sub-paso 2.2
3. Tercer paso
```

### Código

**En línea:**

```markdown
Usa `npm run build` para generar el sitio.
```

**Bloques de código:**

````markdown
```typescript
function saludar(nombre: string): string {
  return `Hola, ${nombre}`;
}
```
````

Nanobook resalta la sintaxis automáticamente cuando indicas el lenguaje después de las tres comillas invertidas.

### Enlaces

**Enlaces externos:**

```markdown
[Astro](https://astro.build)
```

**Enlaces internos entre documentos:**

```markdown
[Volver al inicio](../)
[README del proyecto](../../nanobook-project/meta/readme)
[Roadmap](../../nanobook-project/meta/roadmap)
```

Las rutas son relativas al archivo actual. No incluyas la extensión `.md` ni la barra final.

### Imágenes

```markdown
![Texto alternativo](https://ejemplo.com/imagen.jpg)
```

También puedes usar imágenes locales dentro de `public/`:

```markdown
![Diagrama](/diagrama-arquitectura.png)
```

### Tablas

```markdown
| Campo | Tipo | Requerido |
|---|---|---|
| title | string | Sí |
| draft | boolean | No |
| tags | array | Sí |
```

### Líneas horizontales

```markdown
---
```

### Listas de tareas

```markdown
- [x] Tarea completada
- [ ] Tarea pendiente
- [ ] Otra tarea pendiente
```

Las listas de tareas se renderizan como checkboxes interactivos en visualización, pero no son editables por el lector.

---

## 3. Consejos para escribir en Nanobook

### Usa encabezados descriptivos

La tabla de contenidos y el breadcrumb dependen de una buena jerarquía de encabezados. Evita saltar niveles (`#` seguido de `###` sin `##`).

### Mantiene el frontmatter al día

El `description` se muestra en los índices de navegación. Una buena descripción ayuda a decidir si entrar o no en un documento.

### Organiza con listas de tareas

Las listas de tareas son útiles para roadmaps, checklists de implementación y seguimiento de estados.

### Prefiere enlaces relativos

Para enlazar documentos dentro del mismo sitio, usa rutas relativas (`./`, `../`). Así los enlaces siguen funcionando si cambian las URLs base.

---

## 4. Markmap para mapas mentales

**Markmap** es una extensión de Markdown que permite visualizar una estructura jerárquica como un mapa mental interactivo. Usa la misma sintaxis de Markdown para definir nodos y relaciones.

### Sintaxis de Markmap

Markmap combina encabezados y listas para construir el árbol:

```markdown
# Proyecto de Software

## Fase de Planificación

- Definición de Requisitos
  - Requisitos Funcionales
  - Requisitos No Funcionales
- Análisis de Riesgos

## Fase de Desarrollo

### Diseño

- Arquitectura del Sistema
- Diseño de Base de Datos

### Implementación

- Desarrollo Frontend
  - Uso de React
  - Estilos con CSS
- Desarrollo Backend
  - API con Node.js
  - Base de Datos con MongoDB
```

### Elementos soportados

| Elemento | Sintaxis | Uso |
|---|---|---|
| Nodos | `#`, `##`, `###` | Definen niveles jerárquicos. |
| Subnodos | Listas `-` o `*` | Añaden hijos a un nodo. |
| Texto enriquecido | `**negrita**`, `*cursiva*` | Resalta contenido. |
| Código en línea | `` `código` `` | Muestra fragmentos cortos. |
| Bloques de código | ` ``` ` | Muestra fragmentos multilínea. |
| Tareas | `- [ ]` / `- [x]` | Representan estados. |
| Citas | `>` | Incluyen bloques citados. |
| Tablas | `\|` | Organizan datos. |

### Ejemplo completo

```markdown
# Lanzamiento de Producto

## Preparación

- [x] Definir audiencia objetivo
- [ ] Preparar materiales de marketing
- [ ] Configurar landing page

## Desarrollo

### Frontend

- Diseño responsive
- Integración con API

### Backend

- Endpoints de autenticación
- Base de datos de usuarios

## Lanzamiento

> "Un buen lanzamiento es el resultado de una preparación constante."

- Publicación en redes sociales
- Envío de newsletter
- Recopilación de feedback
```

### Cuándo usar Markmap

Markmap es útil cuando necesitas:

- Explorar ideas de forma no lineal.
- Presentar la estructura de un proyecto o libro.
- Resumir un documento extenso en un mapa mental.
- Facilitar la navegación visual de conceptos relacionados.

---

## 5. Referencias

- [Markdown Guide](https://www.markdownguide.org/)
- [CommonMark Spec](https://commonmark.org/)
- [Markmap](https://markmap.js.org/)
- [Frontmatter en Astro](https://docs.astro.build/es/guides/content-collections/)
