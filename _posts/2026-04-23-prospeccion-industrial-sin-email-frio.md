---
layout: post
title: "Prospección B2B industrial sin email frío: cuatro herramientas y una regla"
date: 2026-04-23 14:00:00 +0000
categories: architecture prospeccion
tags: [prospeccion, browseract, sendpilot, firestore, crm, b2b]
excerpt: "El email frío ya no rinde. El uno-a-uno manual tampoco escala. Este es el pipeline que construimos para prospección industrial en Portugal — cuatro piezas, dos decisiones de arquitectura y una regla innegociable."
---

El email frío lleva tiempo sin rendir. Las tasas de respuesta por debajo del 1 % son habituales, los filtros de spam cada vez más duros y los compradores industriales — nuestra audiencia — reconocen el patrón desde la primera línea. La salida fácil es contratar una agencia que lance diez mil emails más. La salida lenta es ir uno a uno, a mano, como en 2008.

Ninguna de las dos nos servía. Levantamos un pipeline intermedio en cuatro fases y lo probamos primero en industria portuguesa. Va el detalle.

![Pipeline en cuatro fases con la regla innegociable arriba: descubrimiento, enriquecimiento, outreach dual y conversión, con las herramientas asignadas a cada paso.](/assets/img/2026-04-23-prospeccion/exhibit-1-pipeline.svg)
*Exhibit 1 — Las cuatro fases y la regla que las gobierna.*

## La regla, antes que la arquitectura

Una regla innegociable antes de elegir herramienta: **nunca contactar a un decisor sin contexto previo**.

Contexto quiere decir dos cosas. Una, que la empresa del decisor ha visto contenido nuestro antes del primer mensaje — un artículo, un post, una mención en prensa sectorial. Dos, que el mensaje habla de algo concreto de su industria, no de "nuestras capacidades". El email genérico al CEO pidiendo una reunión es lo que intentamos no enviar jamás.

Esa regla nos obligó a separar dos flujos que muchas pipelines de prospección funden: el descubrimiento de empresas y la conversación con decisores. Descubrir es rápido y masivo. Conversar es lento y selectivo. Mezclar los tiempos es la raíz de la mayoría de los emails fríos que todos detestamos.

## Fase 1 — Descubrimiento

Fuente principal: Google Maps. No es glamoroso, pero es honesto — cada ficha en Maps corresponde a una empresa que existe, tiene dirección física verificable y declara un sector. Para industria portuguesa eso es suficiente.

El tramo automatizable lo cubre [BrowserAct](https://browseract.com) con dos plantillas encadenadas: una que barre resultados de búsqueda ("metalurgia {ciudad}", "moldes {municipio}", "cerâmica {região}") y otra que entra en cada ficha para sacar web, teléfono y código postal. El output llega a un JSON que procesamos con un script pequeño (`collect-browseract-results.mjs`) que descarta duplicados por dominio.

Sobre esos datos corre un filtro ICP en `enrich-and-qualify.mjs`. Elimina restaurantes, farmacias y servicios de proximidad que se cuelan por el barrido amplio, y asigna un score por tamaño, claridad de industria y presencia web profesional. El criterio de "web profesional" es una heurística tosca pero útil: empresas con sitio propio moderno suelen tener también capacidad para invertir en consultoría.

Orden de magnitud real tras un ciclo completo: **500-700 empresas raw, 300-500 candidatas ICP, 20 targets seleccionados a mano**. El recorte brutal no es casualidad. Preferimos 20 conversaciones posibles que 500 contactos imposibles.

![Embudo descendente de tres pasos: ~600 empresas raw filtradas a ~400 candidatas ICP y reducidas finalmente a 20 targets, con porcentajes de descarte y motivos.](/assets/img/2026-04-23-prospeccion/exhibit-2-funnel.svg)
*Exhibit 2 — El embudo. El −95 % de la última fase es manual y a propósito.*

## Fase 2 — Enriquecimiento

Aquí buscamos los decisores. No el genérico `info@empresa.pt`, que nunca llega a nadie: los nombres, los cargos y los correos personales. Cuatro fuentes en orden de rendimiento decreciente:

1. **Web de la empresa** — página "Equipa" u "Órgãos Sociais". Las grandes la tienen. Las medianas, rara vez.
2. **Hunter.io Domain Search** — devuelve emails con `emailStatus` (valid, accept_all, invalid). Criterio: aceptamos `valid` siempre, `accept_all` con precaución, `invalid` nunca.
3. **Registros públicos portugueses** — Racius devuelve gerentes de sociedades. Información pública, gratuita, fiable.
4. **Social Media Finder en BrowserAct** — para los casos en que la empresa no publica LinkedIn en su web.

La tasa de éxito del Social Media Finder fue del 12 % en nuestra primera ronda. No es espectacular, pero entra a coste casi cero en el flujo. LinkedIn por Playwright lo probamos y lo descartamos: funciona, sí, pero es lento, frágil y los cambios del front de LinkedIn rompen cualquier scraper cada pocas semanas. Para LinkedIn hay herramientas mejores.

## Decisión 1 — Firestore como CRM

Aquí es donde mucha gente elige HubSpot, Pipedrive o Salesforce. Nosotros elegimos Firestore. El argumento:

- **Nuestros datos, nuestra lógica**. Un CRM SaaS te cobra por contacto, te mete en su embudo y convierte tu base comercial en un activo que no controlas. Firestore es almacenamiento ligero — los flujos los modelamos nosotros.
- **Integración nativa con el resto del stack**. Las entidades viven en la misma base que ticketing y soporte, con el mismo `orgId`, las mismas convenciones de `createdAt`, las mismas reglas de auditoría.
- **Coste marginal**. Unas decenas de miles de registros en Firestore cuestan céntimos al mes. Un CRM comercial, varios cientos de euros desde el primer día.

El tradeoff honesto: Firestore no trae interfaz de CRM de caja. Hemos tenido que construir la vista de deals, la cadencia de follow-ups, los dashboards BI. Nos salió a cuenta porque lo necesitábamos también para ticketing — pero no es universalmente buena decisión. Si solo hicieras CRM, el SaaS probablemente gana.

![Tabla comparativa de cinco dimensiones entre Firestore y CRM SaaS: coste a 3 años, time-to-value, integración, propiedad del dato y cuándo gana cada uno.](/assets/img/2026-04-23-prospeccion/exhibit-3-firestore-vs-saas.svg)
*Exhibit 3 — Cuándo gana cada uno. La fila clave es la última.*

## Decisión 2 — Canal dual, no solo email

Aun con dominio verificado y calentado, solo el 30-40 % de los emails llegan a la bandeja de entrada. Eso deja fuera a seis de cada diez destinatarios. Para no perderlos, el canal secundario es LinkedIn.

El orquestador de LinkedIn es [SendPilot](https://sendpilot.com): gestiona conexiones, secuencias de mensajes y *warmup* automáticamente, y nos devuelve estado vía webhook. El integrador local (`sendpilot-routing.mjs`, `sendpilot-status-poll.mjs`, `sendpilot-warmup-queue.mjs`) consume esos eventos, los escribe en la misma entidad Firestore del contacto y deja a Brevo — nuestro canal email — funcionar en paralelo sobre el mismo decisor.

La regla operativa: **primero Brevo, después SendPilot, nunca los dos a la vez**. La cadencia típica es blog post enviado por newsletter → esperar 2-3 días → si no hay apertura, activar secuencia LinkedIn. Nunca empezamos por LinkedIn frío. Nunca mandamos email genérico sin blog detrás. El único mensaje que el decisor recibe de nosotros sin contenido previo es una conexión de LinkedIn, sin pitch.

![Línea temporal de 21 días con cinco hitos: contenido publicado, newsletter, decisión de bifurcación según apertura, follow-up por email o conexión LinkedIn, mensaje LinkedIn final con contenido propio.](/assets/img/2026-04-23-prospeccion/exhibit-4-cadencia.svg)
*Exhibit 4 — La cadencia. La bifurcación ocurre solo si el email se ignora.*

## Lo que no funcionó

Merece la pena anotar los caminos cerrados:

- **Scraping de LinkedIn con Playwright**. Funcionaba el lunes, roto el viernes. Mantener scrapers caseros contra un target que se defiende activamente es un sumidero de horas.
- **Hunter.io Email Verifier como único filtro**. Un `accept_all` parece un email válido, pero el 30-40 % de ellos rebotan. Combinar con otra señal (presencia en LinkedIn, dominio con MX razonable) lo hace fiable.
- **Primer mensaje LinkedIn pidiendo reunión**. Tasa de aceptación de conexión al suelo. Cambiar a "conexión sin pitch + mensaje con contenido dos semanas después" subió la tasa al rango útil.

## Patrón reutilizable

Dos cosas que nos llevamos para replicar en otra región o sector:

1. **Separar descubrimiento de conversación por tiempo y por herramienta**. Un scraper de ICP no es un CRM, y un CRM no es un sistema de outreach. Meterlos en la misma caja es la raíz de la prospección mal hecha.
2. **Dos canales paralelos con una regla de precedencia clara**. Email por delante, LinkedIn detrás, nunca los dos a la vez. Contenido antes que pitch, siempre.

El pipeline completo corre desde el ordenador de una persona en un par de sesiones al mes por región. No hace falta un SDR a tiempo completo. No hay misterio: hay un orden, un filtro duro al principio y una regla innegociable en el último tramo.

---

*Si estás montando algo parecido y atascas en alguna parte concreta, abre una [Discussion](https://github.com/Proportione/proportione-plugins/discussions) o escribe a [contacto@proportione.com](mailto:contacto@proportione.com). Compartimos las decisiones que tomamos, no las cifras que tomaron nuestros clientes.*
