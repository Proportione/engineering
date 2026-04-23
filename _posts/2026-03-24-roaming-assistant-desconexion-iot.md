---
layout: post
title: "Cuando el router inteligente desconecta tus IoT: el caso del Roaming Assistant"
date: 2026-03-24 23:00:00 +0000
categories: post-mortem redes
tags: [iot, wifi, asus, router, alexa, post-mortem]
excerpt: "Seis Alexas dejaron de responder a las 22:00 y no volvieron solas. La causa no estaba en las Alexas ni en Amazon — estaba en un parámetro del router que tiene sentido en una oficina y ninguno en casa."
image: /assets/img/heroes/post-1-roaming.svg
image_alt: "Roaming Assistant: post-mortem de un router que echa los dispositivos IoT por la noche."
---

Seis Alexas dejaron de responder a las 22:00 del 23 de marzo. Ninguna reconectó por sí sola. No había corte de luz, no había actualización pendiente, no había cambios en la red durante el día.

La causa no estaba en las Alexas ni en Amazon. Estaba en un parámetro del router que tiene sentido en una oficina densa y casi ninguno en una casa.

## Síntoma

- Seis dispositivos Alexa (Echo Dot mezclados con Show 5) sin conectividad simultáneamente.
- Ninguno aparecía en la tabla de clientes DHCP del router.
- Reiniciar una Alexa la devolvía a la lista, pero solo durante unos minutos.
- Otros dispositivos 2.4 GHz (termostato, sensores Zigbee vía hub) funcionaban con normalidad.

## Sospechas descartadas

Una por una:

- **Canal 2.4 GHz saturado** — ya estaba fijado en canal 1 con 20 MHz desde el fix de marzo (un episodio anterior de ACSD cambiando canal por la noche). No era eso.
- **DHCP agotado** — la tabla tenía espacio.
- **Interferencia bluetooth o microondas** — las Alexas son robustas a esto; además no explicaba las seis a la vez.
- **Firmware del router obsoleto** — ya era la última (3.0.0.4.388_24305 en el ASUS TUF-AX5400).

## Causa raíz

En el `syslog` del router aparecían decenas de eventos en `eth5` (la interfaz 2.4 GHz):

```
Disassociated due to inactivity
```

Ese mensaje no significa "el dispositivo dejó de responder". Significa "yo, el router, decidí expulsarlo". Y lo decidió por una razón concreta: el **Roaming Assistant** con umbral de `-70 dBm`.

## Qué hace el Roaming Assistant

El Roaming Assistant está pensado para entornos con varios puntos de acceso. Su trabajo es empujar a un cliente a reasociarse con otro AP cuando su señal cae por debajo de un umbral. Para conseguirlo, lo *desauthentica* activamente, confiando en que el cliente buscará un AP mejor a continuación.

La lógica tiene dos problemas en una casa:

1. **No hay otro AP** al que migrar. El cliente queda desconectado y punto.
2. **Los IoT baratos** no implementan bien el reconnect tras una desasociación forzada. Algunos dejan de intentarlo hasta un ciclo de corriente.

Con la señal de las Alexas fluctuando alrededor de `-70 dBm` (el salón no estaba justo al lado del router), el Roaming Assistant las iba expulsando una tras otra. Tras varios intentos fallidos de reasociarse, se rendían.

## Fix aplicado

Cinco cambios en la configuración 2.4 GHz (NVRAM persistente):

| Parámetro | Antes | Después | Motivo |
|-----------|-------|---------|--------|
| Roaming Assistant (`wl0_user_rssi`) | `-70 dBm` | `0` (OFF) | Causa directa. No hay AP alternativo. |
| TX Beamforming (`wl0_txbf`) | ON | OFF | Inestabilidad observada con IoT baratos. |
| Implicit Beamforming (`wl0_itxbf`) | ON | OFF | Ídem. |
| WPS (`wps_enable`) | ON | OFF | Vector de ataque + reconexiones espontáneas. |
| Turbo QAM 256 (`wl0_turbo_qam`) | ON | OFF | Modulación no soportada por algunos IoT. |

Configuración final 2.4 GHz verificada: canal 1 fijo, 20 MHz, WiFi 6 desactivado, WPA2-Personal/AES, MU-MIMO y OFDMA desactivados, Smart Connect desactivado. Un router "tonto" en la banda 2.4, que es exactamente lo que los IoT necesitan.

## Lecciones

1. **El router "inteligente" no es siempre tu amigo.** Las optimizaciones pensadas para entornos empresariales densos pueden romper un setup doméstico con IoT heterogéneo.

2. **El `syslog` del router es infravalorado.** El mensaje `Disassociated due to inactivity` era la pista completa. Sin acceso al log, habríamos seguido culpando a las Alexas.

3. **2.4 GHz merece configuración separada de 5 GHz.** Los IoT viven en 2.4 GHz y no necesitan nada de lo que hace atractivo a 5 GHz. Mezclarlos con "Smart Connect" empeora las dos bandas.

4. **Documentar el fix en el punto donde alguien lo leerá.** En nuestro caso, [`router_credentials.md`](https://github.com/Proportione) con el "Known issue" anotado. El próximo episodio — y lo habrá — empezará por ahí.

## Acción manual restante

Para cerrar el incidente hubo que desenchufar y reenchufar las seis Alexas (una por una, 10-15 segundos entre cada una). El ciclo de corriente es la única forma fiable de que un Echo que lleva rato "rendido" vuelva a intentar asociarse.

---

*Si ves algo que está mal o tienes un caso parecido, abre un [Issue](https://github.com/Proportione/proportione-plugins/issues) o comparte en [LinkedIn](https://www.linkedin.com/company/proportione).*
