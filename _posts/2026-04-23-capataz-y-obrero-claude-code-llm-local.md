---
layout: post
title: "Capataz y Obrero: por qué Claude Code delega a un LLM local en lugar de hacerlo él"
date: 2026-04-23 13:00:00 +0000
categories: architecture ia-local
tags: [claude-code, mcp, ollama, qwen, llm-local, agente]
excerpt: "Claude Code para todo es caro y lento. Un modelo local para todo es limitado. La combinación correcta no es elegir uno: es dividir el trabajo. Va el patrón que vamos a montar sobre el nodo Aveiro."
image: /assets/img/heroes/post-4-capataz-obrero.svg
image_alt: "Capataz y Obrero: Claude Code delega a un LLM local vía MCP."
---

Quien lleva un mes usando Claude Code en serio nota dos cosas. La primera, que es asombrosamente útil para tareas complejas — refactor, diseño, depuración fina. La segunda, que **no toda tarea merece un Opus**. Pedirle al modelo más capaz del mercado que renombre 40 variables o que clasifique 100 logs por categoría es como contratar a un cirujano para que cambie una bombilla. Funciona. Cuesta una barbaridad.

La trampa simétrica es montarse un modelo local creyendo que sustituye al frontier. No sustituye. Para razonamiento profundo, contexto largo, código nuevo desde cero, **el frontier sigue ganando por mucho**. Quien pretende reemplazarlo entero se encontrará rehaciendo a mano lo que el frontier hubiera resuelto en una iteración.

Entre los dos extremos hay un patrón que sí funciona: que el modelo frontier dirija y un modelo local ejecute la rutina. Lo llamamos **Capataz y Obrero**, y va a ser la primera carga real del servidor Aveiro [del que hablamos en el post anterior](/architecture/infraestructura/2026/04/23/servidor-aveiro-cuatro-opciones-dos-trampas.html).

## El patrón

La división es asimétrica por diseño.

![Diagrama del patrón Capataz y Obrero: Claude Code en el portátil decide qué hacer, descompone, revisa y cierra; un modelo LLM local ejecuta tareas rutinarias delegadas; ambos se comunican vía MCP sobre la red privada Tailscale.](/assets/img/2026-04-23-capataz-obrero/exhibit-1-patron.svg)
*Exhibit 1 — Capataz piensa, Obrero ejecuta. La línea entre ambos es MCP.*

El **Capataz** (Claude Code en el portátil) hace lo que requiere razonamiento amplio: leer el repo, entender qué se pide, descomponer en pasos concretos, decidir cuáles delega y cuáles ejecuta él mismo, revisar el resultado y cerrar el bucle. Es caro por minuto, pero su tiempo se compensa con creces en las decisiones que toma.

El **Obrero** (un LLM local — en nuestro caso, Qwen 2.5 Coder 14B cuantizado a 4 bits, corriendo en Ollama sobre el servidor de Aveiro) recibe instrucciones acotadas y devuelve resultados acotados. No decide qué hacer; hace lo que se le pide. Cuesta cero por petición, está siempre disponible, y su latencia es predecible — del orden de un par de segundos por respuesta corta.

El conector es **MCP** (Model Context Protocol). MCP estandariza cómo un cliente como Claude Code llama a herramientas externas. Un servidor MCP que envuelve Ollama aparece ante Claude Code como un conjunto de funciones invocables: clasifica esto, resume aquello, busca el patrón en este texto. Claude decide cuándo llamarlas, espera el resultado y sigue su trabajo.

## Qué tarea va a quién

La regla de reparto no es "tareas fáciles al obrero". Es más fina: **cualquier tarea con criterio de éxito objetivo y formato de salida estable es candidata a ir al obrero**. Cualquier tarea que requiere juicio sobre el qué, no sobre el cómo, se queda con el capataz.

![Matriz de tipos de tarea con responsable correcto: clasificación con taxonomía cerrada, extracción estructurada de texto y resúmenes cortos van al obrero; arquitectura, depuración, diseño desde cero y revisión final se quedan con el capataz; tareas mixtas se ejecutan en pipeline.](/assets/img/2026-04-23-capataz-obrero/exhibit-2-matriz.svg)
*Exhibit 2 — La regla de reparto. La columna del medio es donde está el dinero.*

Algunos ejemplos concretos del flujo real de Proportione:

- **Clasificar tickets entrantes** por área y departamento. Taxonomía cerrada, salida estructurada, repetitivo. Al obrero.
- **Extraer entidades** (nombre, empresa, asunto) de un correo libre. Formato estable, criterio objetivo. Al obrero.
- **Decidir si un PR introduce regresión arquitectónica.** Requiere entender el repo, el contexto del cambio, la intención del autor. Al capataz.
- **Generar 30 variantes de copy para un A/B test** con tono y restricciones dadas. Al obrero, con plantilla del capataz.
- **Diseñar el módulo nuevo de SLA**. Al capataz, sin discusión.

El patrón gana cuando una sesión real combina las dos cosas. El capataz lee la conversación, identifica que hay 50 tickets que clasificar, los manda en lote al obrero, recibe el resultado y sigue con el problema arquitectónico que le ocupaba. La clasificación cuesta cero, ocurre en segundo plano, y libera al capataz para lo que solo él puede hacer.

## Stack técnico

El montaje físico es deliberadamente simple, en parte para que se pueda replicar en otra organización sin equipo de plataforma.

![Diagrama del stack: portátil con Claude Code y servidor MCP local, conexión por red privada Tailscale al servidor Aveiro, donde Ollama sirve el modelo Qwen 2.5 Coder 14B sobre la GPU; opcionalmente apps móviles Enchanted o Reins consumen el mismo endpoint.](/assets/img/2026-04-23-capataz-obrero/exhibit-3-stack.svg)
*Exhibit 3 — Cuatro componentes, cero dependencias externas tras la instalación.*

Las decisiones que conviene explicar:

**Por qué Ollama y no llama.cpp directo.** Ollama envuelve a llama.cpp con una API estable y un sistema de modelos versionados. Para una organización que va a cambiar de modelo cada pocos meses (porque la frontera abierta se mueve rápido), el coste de mantener invocaciones a llama.cpp directas no compensa.

**Por qué Qwen 2.5 Coder 14B Q4 y no algo más grande.** Es la combinación que cabe holgada en 16 GB de VRAM dejando margen para contexto largo (32 K tokens) sin descargar pesos a CPU. La cuantización a 4 bits degrada la calidad por debajo del umbral perceptible para tareas rutinarias. Cuando entre la GPU de Fase 2, sustituiremos por Qwen 32B o Llama 70B sin tocar el resto del stack.

**Por qué Tailscale y no abrir el puerto 11434.** Tailscale crea una red privada entre los equipos sin exponer servicios a internet. El servidor Aveiro no tiene IP pública, no aparece en escáneres, y solo los dispositivos del *tailnet* corporativo pueden hablarle. Reduce la superficie de ataque a cero sin la fricción habitual de una VPN tradicional.

**Por qué un servidor MCP propio en lugar de uno genérico.** Los servidores MCP genéricos para Ollama existen pero no aplican prompts específicos de Proportione (taxonomías de tickets, formato de extracción, restricciones de copy). Mantener un servidor MCP local de 200 líneas que sí los aplica es trivial y permite versionar los prompts junto al resto del código.

## Tradeoffs honestos

Tres cosas que no funcionan tan bien como una presentación dejaría creer:

**La calidad del obrero degrada con prompts mal diseñados.** Un modelo de 14B cuantizado tolera muchísimo peor un prompt ambiguo que Claude. Si la tarea no se puede explicar en cinco líneas con ejemplos, probablemente no es candidata a delegación. Lo aprendimos con un primer intento de delegar generación de mensajes de commit: el resultado era inconsistente porque el prompt era impreciso. Reescribirlo más estricto resolvió el problema, pero exige tiempo del capataz que hay que contar como inversión.

**Latencia de red al delegar mata el ahorro en lotes pequeños.** Para una sola petición, llamar al servidor MCP, ir hasta Aveiro por Tailscale y volver añade unos 200 ms de overhead. Si la tarea individual cuesta 800 ms al obrero, hablamos del 25 % de la ejecución total. Para una sola tarea no compensa. Para un lote de 50 sí: el overhead se diluye.

**Cuando el obrero falla, el capataz no se entera bien.** Un modelo local que devuelve basura formateada como JSON válido pasa los validadores básicos. Necesitamos validadores semánticos sencillos — ¿el área devuelta está en la taxonomía? ¿el resumen es más corto que el original? — y un fallback explícito al capataz cuando saltan. Todavía afinando estos validadores.

## Cuándo este patrón gana

No siempre compensa. Tres condiciones que conviene verificar antes de montarlo:

1. **Volumen suficiente para amortizar el setup.** Si la previsión es delegar menos de unos cientos de tareas al mes, el ahorro económico es marginal frente al coste de mantener el stack. Mejor pagar al frontier y olvidarse.
2. **Tareas con formato cerrado.** Si el formato de salida varía mucho entre llamadas, el modelo local sufre y la consistencia cae. El patrón requiere disciplina en la definición de las herramientas MCP.
3. **Capacidad de mantener un nodo de cómputo propio.** Si el servidor exige más atención operativa de la que la organización puede sostener, lo barato sale caro. El patrón presupone que la decisión de tener nodo propio ya está tomada y validada por otros motivos — en nuestro caso, los del [post anterior](/architecture/infraestructura/2026/04/23/servidor-aveiro-cuatro-opciones-dos-trampas.html).

Cuando las tres se cumplen, el patrón es genuinamente diferencial. No por el ahorro económico — que existe pero rara vez es el motor — sino por algo más importante: **libera la atención del capataz para lo que solo el capataz puede hacer**. Y esa atención, en un equipo pequeño, es el recurso más escaso.

---

*Próximo post de la serie: cómo el patrón cambia el ticketing y la gestión de SLA cuando uno de los pasos es "esperar respuesta del cliente" y otro es "una persona del equipo está revisando". Para hablar de este, abre una [Discussion](https://github.com/Proportione/proportione-plugins/discussions) o escribe a [contacto@proportione.com](mailto:contacto@proportione.com).*
