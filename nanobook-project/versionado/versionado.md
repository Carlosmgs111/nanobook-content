---
title: "Versionado de nanobook"
description: "Estrategia de versionado y flujo de trabajo para mantener el proyecto de forma consistente usando Semantic Versioning, Conventional Commits y un CHANGELOG manual."
date: 2026-08-09
author: "Nanobook"
tags: ["git", "versioning", "semver", "conventional-commits", "workflow"]
draft: false
index: false
---

v1.0

# Versionado de nanobook

## 1. Objetivo

Definir una forma clara y ligera de versionar `nanobook` a medida que pasa de proyecto experimental a producto de consumo real. La estrategia busca equilibrio entre disciplina y poca fricción operativa.

## 2. Versionado semántico (SemVer)

El proyecto usa [Semantic Versioning](https://semver.org/lang/es/) con el formato:

```text
MAJOR.MINOR.PATCH
```

Dado que aún estamos en la fase `0.x.x`:

- `0.0.x` → correcciones menores o parches.
- `0.x.0` → nuevas funcionalidades, mejoras significativas de UX o refactorizaciones visibles.
- `1.0.0` → primera versión considerada estable para uso público. Se alcanzará cuando la estructura de contenido, URLs y componentes públicos sean estables.

## 3. Commits convencionales

Los mensajes de commit siguen [Conventional Commits](https://www.conventionalcommits.org/):

```text
tipo(alcance): descripción breve

[opcional: cuerpo explicativo]

[opcional: referencias]
```

### Tipos principales

| Tipo | Uso |
|---|---|
| `feat` | Nueva funcionalidad. Ejemplo: `feat(toc): añade blur al modo compacto`. |
| `fix` | Corrección de bug. Ejemplo: `fix(sidebar): evita corte de texto en mobile`. |
| `refactor` | Cambio de estructura sin alterar comportamiento. Ejemplo: `refactor(toc): atomiza componentes`. |
| `docs` | Cambios en documentación. Ejemplo: `docs: actualiza arquitectura de estado`. |
| `style` | Cambios puramente visuales (CSS, colores, espaciado). |
| `chore` | Tareas de mantenimiento (dependencias, build, configuración). |

### Alcances comunes

- `theme`
- `layout`
- `sidebar`
- `toc`
- `content-width`
- `docs`

## 4. Flujo de trabajo

Usamos un modelo **trunk-based** simplificado:

```text
main
  │
  ├── feat/nombre-feature
  ├── fix/nombre-bug
  └── docs/nombre-documento
```

### Reglas

1. `main` siempre debe poder build-arse (`npm run build` sin errores).
2. Cada cambio nace en una rama corta y se fusiona a `main` con merge commit o rebase.
3. El mensaje de merge/commit debe seguir Conventional Commits.
4. Antes de fusionar se verifica:
   - `npm run build` pasa.
   - No hay archivos sin intención en el stage.
5. Cuando se acumulan cambios significativos, se actualiza `CHANGELOG.md` y se crea un tag de versión.

## 5. Changelog

`CHANGELOG.md` en la raíz sigue el formato [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/):

- Sección `[Unreleased]` para cambios que aún no tienen versión.
- Al releasear, se mueven los items de `[Unreleased]` a una nueva versión numerada.

Ejemplo de ciclo:

1. Se desarrollan varios cambios en `main`.
2. Se actualiza `[Unreleased]` con los cambios.
3. Se decide releasear `v0.2.0`.
4. Se mueven los items a `## [0.2.0] - YYYY-MM-DD`.
5. Se crea el tag anotado:

```bash
git tag -a v0.2.0 -m "Release v0.2.0: descripción corta"
git push origin v0.2.0
```

6. Opcionalmente se crea un Release en GitHub copiando la sección del CHANGELOG.

## 6. Tags y releases

Los tags son anotados y se crean sobre `main`:

```bash
git tag -a v0.1.0 -m "Release v0.1.0: layout base, sidebar y TOC atómicos"
```

Para listar versiones:

```bash
git tag -l
```

Para ver el changelog de una versión:

```bash
git show v0.1.0
```

## 7. Roadmap tentativo

| Versión | Enfoque |
|---|---|
| `v0.1.0` | Layout funcional, sidebar/TOC atómicos, tema, docs. Estado actual. |
| `v0.2.0` | Estabilizar arquitectura de estado (store central o patrón comando/notificación). |
| `v0.3.0` | Pulido de UX/UI: animaciones, accesibilidad, mobile. |
| `v0.4.0` | Contenido real: migrar o escribir libros y artículos. |
| `v1.0.0` | Release público: estructura de URLs estable, docs completas, responsive sólido. |

## 8. Automatización futura

Cuando el proyecto crezca, se pueden agregar herramientas como:

- `commitlint` + `husky` para validar mensajes de commit.
- `release-it` o `standard-version` para generar CHANGELOG y tags automáticamente.
- `changesets` si el proyecto evoluciona a monorepo.

Por ahora, el proceso se mantiene manual para evitar fricción innecesaria.

## 9. Referencias

- `CHANGELOG.md` — registro de cambios del proyecto.
- `AGENTS.md` — convenciones para agentes de código.
- [Semantic Versioning](https://semver.org/lang/es/)
- [Conventional Commits](https://www.conventionalcommits.org/)
- [Keep a Changelog](https://keepachangelog.com/es-ES/1.0.0/)
