---
layout: post
title: "Servidor propio para una consultoría AI-first: cuatro opciones, dos trampas y una decisión"
date: 2026-04-23 13:00:00 +0000
categories: architecture infraestructura
tags: [servidor, aveiro, proxmox, zfs, hardware, ia-local]
excerpt: "Decidimos no migrar a un hyperscaler ni encadenar Macs. Va el marco con el que elegimos servidor propio, la BOM real, las dos trampas que casi rompen la decisión y la lección sobre comprar hardware enterprise en marketplaces europeos."
image: /assets/img/heroes/post-3-servidor-aveiro.svg
image_alt: "Servidor propio para una consultoría AI-first: cuatro opciones, dos trampas y una decisión."
---

Una consultoría pequeña con foco en IA tiene tres caminos para su capa de cómputo: alquilarlo todo en la nube, encadenar Macs como puestos de trabajo, o montar un servidor propio. Cada camino esconde un coste distinto. Decidimos el tercero — un servidor x86 en nuestra sede de Aveiro — y este post cuenta cómo se llega ahí sin caer en la tentación del catálogo más caro ni del más barato.

Por el camino acumulamos dos lecciones que nos vinieron por sorpresa: una sobre el desfase entre la placa que eliges en hoja de cálculo y la que de verdad puedes comprar; otra sobre cómo los marketplaces europeos rompen los pedidos enterprise en pedazos.

## Marco de decisión: cuatro opciones reales

Antes de elegir hardware, listamos qué soluciones cumplían los criterios mínimos: cómputo suficiente para inferencia LLM local, almacenamiento ampliable, un canal de réplica al nodo Madrid y vida útil mayor de cinco años. Salieron cuatro candidatos.

![Matriz comparativa de cuatro opciones para la capa de cómputo de una consultoría pequeña: nube hyperscaler, Mac Mini en estantería, NAS Synology con módulo de cálculo y servidor x86 propio, evaluadas en cinco dimensiones.](/assets/img/2026-04-23-aveiro/exhibit-1-opciones.svg)
*Exhibit 1 — Las cuatro opciones que evaluamos. La fila clave es la última.*

La nube hyperscaler perdió por una razón concreta: nuestro perfil de carga es continuo y predecible (inferencia LLM más servicios internos), no a ráfagas. Pagar por hora un cómputo que necesitamos siempre es la peor combinación. Cuando hicimos números a tres años, cualquier instancia con GPU equivalente nos salía entre tres y cinco veces más cara que comprar el hardware una vez.

El Mac Mini quedó fuera por el techo de RAM (24 GB en el modelo base, 64 GB en el M2 Pro) y por la imposibilidad de instalar Proxmox o pasar GPU a un contenedor. Es un equipo excelente como puesto de trabajo. Como nodo de cómputo central, no.

El NAS con módulo de cálculo lo descartamos por un detalle que solo se ve cuando ya tienes uno: el cómputo va al ritmo del NAS, y el NAS se diseña para ser silencioso y ahorrador, no para mover modelos de lenguaje. Mantenemos el NAS Synology de Madrid como nodo de réplica fría y de Time Machine, que es exactamente para lo que es bueno.

Quedó el servidor x86 propio. La decisión no era qué comprar, sino con qué placa, qué procesador y qué GPU.

## La BOM y por qué cada componente

Apuntamos a una arquitectura de cuatro fases. Fase 1 es funcional desde el día uno: hipervisor, almacenamiento, red interna y un modelo LLM local de tamaño medio. Las fases siguientes añaden GPU mayor, más RAM y redundancia.

![Tabla con la lista de componentes de la fase 1 del servidor de Aveiro: placa server-grade con IPMI, CPU EPYC con AVX-512, RAM ECC, almacenamiento mixto NVMe + SATA, GPU 16 GB, fuente Titanium dimensionada para upgrades futuros, caja silenciosa.](/assets/img/2026-04-23-aveiro/exhibit-2-bom.svg)
*Exhibit 2 — Cada componente lleva una razón. La razón vale más que el modelo.*

Tres principios marcaron las elecciones:

1. **Nada se desperdicia al escalar**. La fuente está dimensionada para una GPU de gama alta futura. La caja admite el largo de tarjetas de la próxima generación. La RAM se añade en módulos, no se sustituye. La GPU actual pasa a tarjeta secundaria cuando entre la siguiente.
2. **Server-grade siempre que el sobrecoste sea pequeño**. Placa con IPMI fuera de banda, RAM ECC, fuente Titanium con garantía de diez años, discos enterprise. La diferencia frente a componentes consumer está entre el 10 % y el 30 %, y compra años de tranquilidad.
3. **Comprable desde Aveiro**. Todo el catálogo se podía pedir a Amazon.es, PCComponentes o Worten con envío a Portugal. Sin importaciones internacionales, sin segunda mano, sin "lo trae alguien que viaja".

## Trampa 1: la placa que eliges no es la placa que llegas a comprar

Aquí ocurrió el episodio más educativo del proceso. La hoja de cálculo dijo claramente que la placa correcta era una Supermicro server-grade con IPMI maduro y dos NVMe nativos. La validación externa lo confirmó: una alternativa de ASRock Rack que también encajaba tenía dos bugs documentados en el foro de Proxmox — uno relacionado con passthrough de GPU, otro con la NIC integrada que desaparecía después de ciertos reinicios. Decisión clara: Supermicro.

Cuando fuimos a comprar, la Supermicro elegida no estaba disponible ni en Amazon, ni en PCComponentes, ni en Senetic, ni en los proveedores españoles habituales con envío a Portugal. La estimación más optimista era de seis a ocho semanas. La ASRock con los bugs sí estaba disponible, en stock, con envío inmediato.

![Diagrama de decisión flip-flop en cuatro pasos: especificación inicial elige A, validación externa cambia a B por bugs documentados, compra real vuelve a A por ruptura de stock de B, mitigación de bugs con BIOS y software open-source.](/assets/img/2026-04-23-aveiro/exhibit-3-trampa-stock.svg)
*Exhibit 3 — La realidad pragmática del stock. Lo importante no es no cambiar de opinión: es saber por qué cambias.*

Volvimos a la ASRock con los bugs documentados. Pero no como antes — esta vez con un plan de mitigación específico para cada bug:

- **Bug del passthrough de GPU**: actualización de BIOS a la última versión del fabricante (donde el bug está corregido) y BMC a la versión 6.01 o superior. Verificación durante el montaje, antes de poner el servidor en producción.
- **Bug de la NIC integrada**: tener a mano la utilidad de recuperación del fabricante de la NIC y, como salvaguarda, una NIC PCIe de Intel para sustituirla si el problema persiste.

La lección no es "Supermicro es mejor que ASRock" ni al contrario. Es que **una decisión de hardware sin verificar disponibilidad real es solo una preferencia**. Y que cambiar de opinión está bien siempre que el motivo del cambio se documente en algún sitio donde el yo del futuro pueda releerlo.

## Trampa 2: comprar hardware enterprise en marketplaces europeos

Esta nos sorprendió más. Los componentes enterprise — placa, CPU, fuente, NIC — viven en Amazon.es como vendedores marketplace, no como artículos de Amazon directo. Eso tiene tres consecuencias que conviene saber antes:

**Un pedido se convierte en varios.** Confirmas siete artículos en una transacción y el sistema los fragmenta en cinco o seis subpedidos, uno por vendedor. Cada uno tiene su propia fecha de entrega, su propio código y su propia política de devolución. La gestión postventa se multiplica.

**Anti-fraude se activa con valor alto en poco tiempo.** Confirmar dos pedidos con varios cientos de euros cada uno en menos de diez minutos disparó los algoritmos de Amazon. El pedido más grande quedó cancelado automáticamente, "por seguridad". Resolverlo requirió enviar documentación de empresa por su canal de recurso. Aprendido para la siguiente vez: separar las compras grandes por al menos un día.

**Amazon Locker no funciona con marketplace.** Probablemente el detalle más doloroso. Los puntos de recogida automática que tan bien funcionan para artículos directos están vetados para vendedores externos. Hay que recibir en dirección física durante ventana laboral, lo que en una sede pequeña requiere coordinación o dejar plantón a alguien.

Ninguna de las tres es un drama, pero las tres juntas alargaron el proceso de un día previsto a una semana real. Si vas a montar un servidor enterprise por catálogo, presupuestar tiempo de gestión de pedidos como si fuera una tarea aparte, no como un click final.

## Plan de fases y por qué importa contarlo

El servidor no nace adulto. Lo levantamos por fases para que cada euro tenga una utilidad inmediata, y para que el upgrade futuro a una GPU de gama alta no obligue a tirar nada.

![Línea temporal en cuatro fases: cimientos con hipervisor y modelo de lenguaje medio, IA potente con GPU mayor y más RAM, redundancia con segundo nodo y SAI, expansión de almacenamiento RAIDZ.](/assets/img/2026-04-23-aveiro/exhibit-4-fases.svg)
*Exhibit 4 — Cuatro fases. Cada una entrega valor sin depender de la siguiente.*

La Fase 1 entrega el grueso del valor: hipervisor con contenedores, almacenamiento espejado, réplica al nodo Madrid, modelo LLM local de tamaño medio funcional desde el primer día y *tailnet* con acceso desde portátil y móvil sin abrir puertos. El retorno frente al ahorro de la API externa es directo y medible — del orden de varias decenas de euros al mes desde la primera semana.

Las fases siguientes son optativas. Si Fase 1 cubre la carga real durante seis meses, no hay urgencia. Si el modelo medio se queda corto, la Fase 2 sustituye la GPU por una mayor — y la GPU original pasa a tarjeta secundaria para *embeddings*, voz a texto y generación de imagen, sin tirar nada.

## Patrón reutilizable

Tres cosas nos llevamos para repetir en futuras decisiones de hardware:

1. **Empezar por el marco, no por el componente**. La pregunta correcta no es "¿qué placa?" sino "¿qué carga real, qué horizonte temporal, qué presupuesto operativo a tres años?". El componente cae solo cuando esas tres respuestas están claras.
2. **Validar disponibilidad antes de cerrar la decisión**. La especificación ideal sin stock es un mock-up, no una decisión. Reservar siempre una alternativa con plan B documentado.
3. **Server-grade donde el sobrecoste sea moderado**. ECC, IPMI, fuente Titanium y discos enterprise no son lujo: son tiempo que no gastarás depurando inestabilidades raras dentro de tres años.

El servidor entra en servicio durante mayo. El siguiente post de esta serie contará el patrón "Capataz y Obrero" que monta encima — cómo Claude Code en el portátil delega tareas rutinarias a un modelo LLM local que vive en este hardware.

---

*Si estás evaluando algo parecido y quieres contraste honesto sobre alguna decisión concreta, abre una [Discussion](https://github.com/Proportione/proportione-plugins/discussions) o escribe a [info@proportione.com](mailto:info@proportione.com).*
