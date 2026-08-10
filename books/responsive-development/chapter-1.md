---
title: "Guía completa de elementos Markdown"
description: "Capítulo de referencia que incluye todos los elementos renderizables de Markdown para experimentar con el diseño."
date: 2026-07-30
author: "Astro"
tags: ["markdown", "referencia", "diseño"]
draft: false
---

# Guía completa de elementos Markdown

Este capítulo contiene **todos los elementos** que pueden marcarse con Markdown. Sirve como referencia visual para iterar sobre el diseño del libro.

## Encabezados

# Encabezado de nivel 1

## Encabezado de nivel 2

### Encabezado de nivel 3

#### Encabezado de nivel 4

##### Encabezado de nivel 5

###### Encabezado de nivel 6

## Formato de texto

Este es un párrafo normal. Dentro de un párrafo puedes usar **negrita**, _cursiva_, o **_negrita y cursiva combinadas_**.

También existe el texto ~~tachado~~, el texto `monoespaciado en línea`, y el [enlace a un documento](/).

Puedes escapar caracteres especiales: \*asteriscos\*, \`backticks\`, \[corchetes\].

## Enlaces

- [Enlace interno](/)
- [Enlace externo](https://astro.build)
- [Enlace con título](https://docs.astro.build "Documentación de Astro")

---

## Listas

### Lista desordenada

- Elemento uno
- Elemento dos
  - Sub-elemento A
  - Sub-elemento B
    - Sub-sub-elemento
- Elemento tres

### Lista ordenada

1. Primer paso
2. Segundo paso
   1. Sub-paso A
   2. Sub-paso B
3. Tercer paso

### Lista de tareas

- [x] Tarea completada
  - [x] Subtarea A
  - [x] Subtarea B
- [ ] Tarea pendiente
- [ ] Otra tarea pendiente

## Citas

> Esta es una cita en bloque. Puedes incluir párrafos dentro de una cita.
>
> > También puedes anidar citas dentro de otras citas.

---

## Código

Código en línea: `const result = await useCase.execute(input)`.

Bloque de código sin especificar lenguaje:

```
function greet(name) {
  return `Hola, ${name}`;
}
```

Bloque de código con resaltado de sintaxis:

```ts
interface Repository<T> {
  findById(id: string): Promise<Result<Error, T>>;
  save(entity: T): Promise<Result<Error, void>>;
}

class UserRepository implements Repository<User> {
  async findById(id: string) {
    return Result.ok(new User(id));
  }
}
```

## Tablas

| Capa            | Responsabilidad   | Ejemplo                     |
| :-------------- | :---------------- | :-------------------------- |
| Dominio         | Reglas de negocio | Entidades, value objects    |
| Aplicación      | Orquestación      | Casos de uso                |
| Infraestructura | Detalles técnicos | Repositorios, controladores |

Tabla con alineación:

| Izquierda | Centro | Derecha |
| :-------- | :----: | ------: |
| A         |   B    |       C |
| D         |   E    |       F |

---

## Línea horizontal

---

## Imágenes

![Imagen de ejemplo con texto alternativo](https://placehold.co/600x200/333/fff?text=Imagen+de+ejemplo)

Imagen con enlace:

[![Imagen dentro de enlace](https://placehold.co/300x100/444/fff?text=Click+me)](https://astro.build)

---

## Bloques de contenido

Algunos renderizadores permiten HTML inline:

**Haz clic para expandir**

Este contenido está oculto hasta que se expande.

| Indice 1 | Indice 2 |
| -------- | -------- |
| Contenido 1 | Contenido 2 |

**Nota importante**

Este es un callout de tipo nota. Se puede usar para resaltar información relevante.

**Atención**

Este es un callout de advertencia. Úsalo para llamar la atención sobre posibles problemas.

**Consejo**

Este es un consejo práctico para mejorar el flujo de trabajo.

**Sobre esto**

Información adicional o contextual para complementar la lectura.

### Sección con relleno diagonal

Este bloque se envolvía en un `hatch-section`. El relleno de líneas diagonales aparece a la izquierda, marcando visualmente una sección técnica del contenido.

---

## Notas al pie

Aquí hay una referencia a una nota al pie[^1]. Y aquí hay otra[^2].

[^1]: Esta es la primera nota al pie.
[^2]: Esta es la segunda nota al pie con más contexto.

## Subíndice y superíndice

Algunos motores soportan H~2~O como subíndice y x^2^ como superíndice.

## Escape de caracteres especiales

\# No es un encabezado \* No es una lista
\> No es una cita

## Párrafos con saltos de línea

Este párrafo tiene un salto de línea  
forzado usando dos espacios al final de la línea.

## Énfasis combinado

Este texto combina **negrita con _cursiva interna_** y viceversa: \*cursiva con **negrita interna\***.

## Lista de definiciones (Markdown extendido)

Término 1
: Definición del término 1.

Término 2
: Definición del término 2.

## Bloque de código con sangría

    function oldStyle() {
      console.log("Bloque de código indentado");
    }

## Referencias

> "La simplicidad es la máxima sofisticación."
> — Leonardo da Vinci

## Fin del capítulo

Este capítulo cubre la mayoría de elementos Markdown disponibles. Úsalo para probar tipografía, espaciado, colores y otros detalles visuales del diseño.
