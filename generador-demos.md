---
title: "Generador Automatizado de Demos en Vídeo"
description: "Descripción del post."
date: 2026-07-30
author: "Astro"
tags: []
draft: false
index: false
---

# Generador Automatizado de Demos en Vídeo v1.0

## 1. Visión General

Crear un sistema que automatice la generación de videos de demostración de software cada vez que se publique una nueva release en un repositorio de GitHub.

## 2. Problema del Usuario

Los desarrolladores lanzan nuevas funcionalidades rápidamente gracias a la IA, pero documentarlas y crear vídeos de marketing consume demasiado tiempo y ralentiza el go-to-market.

## 3. Funcionalidades del MVP

Integración con GitHub: Mediante una GitHub App, el sistema detecta lanzamientos de versiones (releases) y lee las notas de cambios automatizados (4:53-5:20).
Generación de Guiones con IA: Uso de modelos de lenguaje para traducir los cambios técnicos en un guion estructurado para la demo (6:40-7:04).
Grabación Automatizada: Ejecución de Playwright en navegadores Chromium para realizar las acciones grabadas en la aplicación del usuario (2:46-2:56, 6:05-6:20).
Almacenamiento: Entrega de videos finalizados en un panel de usuario para su visualización o descarga (7:16-7:27).

## 4. Arquitectura Técnica (Serverless en AWS)

Frontend: React alojado en S3.
Autenticación: AWS Cognito (3:44-3:49).
Backend: Funciones Lambda (gestión de lógica) detrás de API Gateway (4:20-4:49).
Procesamiento de Colas: SQS para gestionar tareas pesadas y evitar bloqueos (5:38-6:05).
Workers de Grabación: AWS Fargate para instanciar contenedores bajo demanda (6:05-6:30).
Base de Datos: DynamoDB para gestionar estados y registros (7:10-7:16).

## 5. Estrategia de Escalabilidad

El sistema es totalmente serverless, pagando únicamente por los segundos de cómputo utilizados durante la grabación y escalando en paralelo según la demanda de los usuarios (4:06-4:16, 6:22-6:35).

## Idea Original

Idea original de el [Rincón Del Dev YT](https://www.youtube.com/@elrincondeldev)

[![](https://i.ytimg.com/vi/EgwY46HidiA/hqdefault.jpg?sqp=-oaymwErCPYBEIoBSFryq4qpAx0IARUAAIhCGAHYAQHiAQoIGBACGAY4AUABuALzGA==&rs=AOn4CLCwIUCyXvrZ8kVqtQL-y7qiIJws7Q)](https://youtu.be/EgwY46HidiA?si=dtiJcmK8Fkvb37h8)
