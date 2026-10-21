# Design & Decision Record (DDR)

**GreenField Technologies | IoT Systems Design**

**Integrantes:** Valeria Corredor García; Juan Felipe Camacho García; Juan Pablo Vargas Córdoba.

**Alcance del registro:** laboratorios 1–4. Los apartados de los laboratorios posteriores se conservan según la plantilla y se indican como pendientes. Las cifras de ensayos se distinguen de las estimaciones y de la fila simulada del DHT22.

---

## 1. System Overview

*   **System Type:** [ ] Component (Lab 1-2) | [x] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:** En Lab 2, tres placas ESP32-C6 (dos C6 normales y una Super Mini) formaron una red Thread sobre radio 802.15.4. En Lab 3 se usaron dos C6 normales: S ofreció temperatura simulada y control de válvula mediante CoAP; C actuó como cliente. En Lab 4, S incorporó un módulo OKY3068 (DHT11) real y C consultó su temperatura y observó los cambios. Los roles Thread de las placas variaron durante las pruebas.

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization

*   **To Samuel (Architect):** El escaneo de energía de 500 ms por canal encontró −104 dBm en los canales 24 y 25; se eligió el canal 25 según ADR-001. Un segundo escaneo al iniciar Lab 2 encontró el mismo mínimo en 17, 23, 24 y 25, por lo que se mantuvo el 25.
*   **To Edwin (Ops):** Si aumenta la pérdida de paquetes, repetir el escaneo y comprobar los canales Wi-Fi cercanos; no se midieron los canales de los routers del entorno ni consta una distancia máxima de alcance.

### Lab 2: 6LoWPAN

*   **To Edwin (Ops):** Recuperación estimada tras desconectar B: 8, 10 y 9 s (promedio ≈ 9 s).
*   **To Edwin (Ops):** Los tres ensayos cumplieron el objetivo de menos de 120 s.
*   **To Edwin (Ops):** Ante una placa sin comunicación, comprobar vecinos en alcance y la ruta con `ot router table`.
*   **To Samuel (Architect):** El RTT promedio medido fue 16,300 ms en A → B (un salto) y 30,350 ms en A → C vía B (dos saltos): diferencia de 14,05 ms.
*   **To Samuel (Architect):** El firmware de aplicación debe dirigir tráfico a la ML-EID de cada nodo.
*   **To Samuel (Architect):** La ML-EID identifica al nodo; el RLOC codifica su ubicación y puede cambiar al volver a unirse.

### Lab 3: Thread & CoAP

*   **To Daniela (Customer):** El GET de temperatura llevó 15 B de solicitud CoAP y 17 B de respuesta, con un payload CBOR de 6 B. En cinco minutos, Observe produjo 31 notificaciones frente a 300 consultas hipotéticas a 1 Hz; no se midió corriente ni se demostró ahorro frente a consultas cada 60 s.
*   **To Edwin (Ops):** Un PUT CON se confirma con la respuesta CoAP y el estado de válvula `{"v": 0|1}`; S registró `2.04 Changed` al ejecutar la orden.
*   **To Edwin (Ops):** Con S fuera de la red, C reintentó CON y declaró `ResponseTimeout` tras 95 s; el silencio de NON no permite saber si llegó.
*   **To Edwin (Ops):** Como SED con sondeo cada 5 s, S respondió a 10/10 pings; RTT promedio 2,192 s y máximo 4,191 s en esta prueba, por debajo del objetivo provisional de 5 s.

### Lab 4: Real Sensors

*   **To Edwin:** Si el sensor deja de entregar lecturas pero S sigue en la red, C recibió `no fresh reading` 51 s después de retirar DATA en la prueba cronometrada; S responde 5.03 cuando ya no hay una lectura válida de menos de 60 s.
*   **To Edwin:** Si S sale de Thread, C no recibe respuesta: el GET CON terminó en `ResponseTimeout` a los 90 s en nuestra prueba. El silencio de Observe por sí solo no identifica la causa.
*   **To Edwin:** Al reconectar DATA, S registró `Sensor back` en unos 2 s y declaró el GET disponible con 2.05 otros 516 ms después. Tras 5.03 o RESET, C debe registrar Observe otra vez; después del RESET hubo que iniciar Thread manualmente en S.

### Lab 5: Border Router

*   **To Daniela:** Pendiente de Lab 5.

### Lab 6: Security

*   **To Edward (Security):** Pendiente de Lab 6.

### Lab 7: Dashboard

*   **To Gustavo (Product):** Pendiente de Lab 7.

### Lab 8: Final Integration

*   **To All:** Pendiente de Lab 8.

---

## 3. Architecture Decision Records (ADRs)

### ADR-001: Selección del canal 802.15.4 (Lab 1)

*   **Context:** Se escanearon los canales 802.15.4 del 11 al 26 durante 500 ms por canal. Los canales 24 y 25 registraron el valor más bajo: −104 dBm.
*   **Decision:** Usar el canal 25.
*   **Rationale:** El aire y la interferencia pertenecen al PED. Aunque 24 y 25 empataron en la medición, la guía sitúa el canal 24 en el solapamiento con Wi-Fi 11 y el 25 en un espacio entre canales Wi-Fi. No se midieron los canales de los routers cercanos.
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**Comprobación al iniciar Lab 2:** Se repitió `ot scan energy 500` en A antes de fijar el canal del nuevo dataset. Los canales 17, 23, 24 y 25 empataron en −104 dBm; se mantuvo el canal 25 conforme al ADR-001. En A se eligieron 10 dBm de potencia de transmisión para esta prueba, en lugar de los 0 dBm indicados por la guía. Se detectó una lectura de 0 dBm en otra placa y se indicó fijar 10 dBm en B y C, pero no quedaron registradas lecturas de ambas después de cada reinicio.

### ADR-002: Topología de la red del campo (Lab 2)

*   **Context:** El campo necesita que un nodo alcance a otros aunque el enlace directo sea pobre y que la comunicación se recupere en menos de 120 s si cae un router. La guía pide comparar una malla Thread con una estrella de estilo LoRaWAN, cuyos dispositivos finales se comunican por un salto con gateways, sin usar a otro dispositivo final como relé.
*   **Decision:** Usar una malla Thread dentro del SCD para la conectividad entre nodos, con routers alimentados de forma continua y nodos finales dormidos cuando el requisito de batería lo exija. La autonomía de tres meses sigue sin validarse: en Lab 4 no se midió corriente de la placa.
*   **Rationale:** En esta prueba, A llegó a C a través de B con 20/20 respuestas, RTT promedio de 30,350 ms y `TxUnicast: 40` en B; al desconectar B, los tres tiempos estimados de recuperación fueron 8, 10 y 9 s, todos menores que 120 s. La estrella LoRaWAN no proporcionaría ese relé entre dispositivos finales en la topología ensayada. No se midió una red LoRaWAN en este laboratorio, por lo que la comparación con ella es arquitectónica. [Especificación LoRaWAN 1.0.4](https://lora-alliance.org/wp-content/uploads/2021/11/LoRaWAN-Link-Layer-Specification-v1.0.4.pdf).
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

### ADR-003: CoAP/UDP y CBOR para telemetría y órdenes (Lab 3)

*   **Context:** El enlace Thread es restringido y el sistema necesita leer `/env/temp`, recibir cambios y enviar órdenes a `/act/valve` sin mantener una conexión TCP abierta. Los contratos están documentados en §4.
*   **Decision:** Usar CoAP sobre UDP con payload CBOR para GET/Observe de temperatura y PUT/GET de válvula; usar CON para órdenes y heartbeats, y NON para avisos de temperatura por umbral.
*   **Rationale:** En la Parte 6 se midieron 15 B de solicitud y 17 B de respuesta CoAP para GET de temperatura; se calcularon 19 B por notificación Observe y 18/13 B para PUT y respuesta de válvula, respectivamente. El comparador MQTT/JSON de la guía ronda 40 B por lectura sobre una conexión TCP ya establecida: el tamaño de un PUBLISH no incluye mantener el socket, los keepalives ni el broker. Por ello no se eligió MQTT para el lado radio de la malla. La prueba Observe dio 31 avisos en 5 min frente a 300 GET a 1 Hz, pero no demostró ahorro frente a GET cada 60 s; el beneficio depende de la frecuencia real de cambios. Los tamaños calculados no son mediciones de tramas.
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

### ADR-004: Período de sondeo de la válvula a batería (Lab 3)

*   **Context:** Una válvula SED apaga la recepción entre consultas a su padre. Edwin reportó una orden que tardó 14 min; el DDR fija provisionalmente un objetivo de latencia de sondeo menor a 5 s. Se probaron períodos de 1, 5 y 15 s en la tarea 5.2.
*   **Decision:** Proponer un sondeo cada 5 s para el siguiente prototipo de válvula, sujeto a validar corriente real y el plazo aceptable con Operaciones.
*   **Rationale:** Con 5 s se obtuvieron 10/10 pings, RTT promedio 2,192 s y máximo 4,191 s; con 15 s el máximo fue 13,845 s, y con 1 s fue 0,932 s pero aumenta la frecuencia de sondeo. En el modelo de energía de §6, 5 s da 17,114 mA y ~7,3 días si el reposo consume 17 mA; el firmware no ha demostrado light sleep, así que el objetivo de tres meses con 2× AA no está validado. El período se elige por el límite provisional de latencia, no por preferencia subjetiva.
*   **Status:** [x] Proposed | [ ] Accepted | [ ] Deprecated

---

## 4. ISO/IEC 30141 Mapping

### Domain Mapping

| Component | ISO Domain | Justification |
|-----------|------------|---------------|
| ESP32-C6 SoC | SCD | Sensing/controlling device (§6.4–6.5) |
| 802.15.4 radio + antenna | SCD | Communication subsystem |
| Air (RF medium) | PED | Physical entity — EM propagation |
| Temperatura del aire junto al DHT11 | PED | Magnitud física observada en Lab 4; la referencia colocada al lado marcó 24,1 °C cuando el DHT11 marcó 23,0 °C |
| Módulo OKY3068 (DHT11) | SCD | Transductor que mide temperatura y humedad; S lo consulta aproximadamente cada 2 s |
| Lectura y comprobaciones de plausibilidad en S | SCD | Adquisición y rechazo de valores fuera de −40..85 °C o de saltos mayores de 5 °C respecto de una lectura reciente |
| Thread routers, links and MLE | SCD | Communication subsystem of the routed mesh |
| ML-EID reachability for later application services | SCD ↔ ASD | Stable device addressing exposed to application services from Lab 3 onward |
| CoAP `/env/temp` y su contrato CBOR/CDDL | ASD | Servicio de lectura y observación de temperatura; en Lab 4 declara precisión DHT11 ±2 °C, frescura mediante Max-Age y respuesta 5.03 si no hay lectura reciente |
| CoAP `/act/valve` y su contrato CBOR/CDDL | ASD | Servicio de control y lectura del estado de la válvula |
| SED y sondeo al router padre | SCD | Comunicación del dispositivo cuando la radio receptora duerme |

### Component Capabilities

| Capability Category | Subcategory | Component/Feature | Active/Latent | Lab Introduced |
|---------------------|-------------|-------------------|---------------|----------------|
| Transducer | Actuation | RGB LED: muestra el rol Thread | Active | Lab 2 |
| Data | Radio | 802.15.4 TX/RX y RSSI usados | Active | Lab 1 |
| Data | Storage | NVS guarda el dataset | Active | Lab 1 |
| Data | Routing | Thread, MLE, 6LoWPAN e IPv6 en uso | Active | Lab 2 |
| Interface | Network | Red 802.15.4 en uso | Active | Lab 1 |
| Interface | Console | CLI `ot` y monitor serie en uso | Active | Lab 1 |
| Interface | MAC | Filtro MAC habilitado y usado para fijar A–C en LQI 1 durante la Parte 3 | Active | Lab 2 |
| Interface | Application | CoAP `/env/temp` y `/act/valve` sobre UDP/5683, con contratos CBOR/CDDL | Active | Lab 3 |
| Transducer | Sensing | DHT11 OKY3068: temperatura y humedad reales | Active | Lab 4 |
| Data | Validation | Rechazo por rango y salto de temperatura; límite de frescura de 60 s | Active | Lab 4 |
| Supporting | Time and crypto | Sincronización de tiempo y acelerador criptográfico; aún no usados en las pruebas | Latent | Lab 1 |
| Latent | Other radios | Wi-Fi y BLE no usados por Thread | Latent | Lab 1 |

### Lab 2: direcciones Thread

| Placa | Dirección | Tipo | Alcance y uso |
|-------|-----------|------|---------------|
| A | `fd63:ae07:8739:36b7:0:ff:fe00:fc00` | ALOC | Toda la malla; identifica el rol de leader, no una placa específica |
| A | `fd63:ae07:8739:36b7:0:ff:fe00:3000` | RLOC | Toda la malla; localizador para el enrutamiento |
| A | `fd63:ae07:8739:36b7:b456:2f8:216a:a556` | ML-EID | Toda la malla; identidad estable para las aplicaciones |
| A | `fe80:0:0:0:e8b5:95fb:b264:be7` | Link-local | Un salto; MLE entre vecinos |
| B | `fd63:ae07:8739:36b7:0:ff:fe00:cc00` | RLOC | Toda la malla; localizador para el enrutamiento |
| B | `fd63:ae07:8739:36b7:d6a7:9217:f5b:f529` | ML-EID | Toda la malla; identidad estable para las aplicaciones |
| B | `fe80:0:0:0:a0c0:582c:dd91:3c14` | Link-local | Un salto; MLE entre vecinos |
| C | `fd63:ae07:8739:36b7:0:ff:fe00:0` | RLOC | Toda la malla; localizador para el enrutamiento |
| C | `fd63:ae07:8739:36b7:9a76:9516:57d5:2ca7` | ML-EID | Toda la malla; identidad estable para las aplicaciones |
| C | `fe80:0:0:0:b478:3:8156:9e90` | Link-local | Un salto; MLE entre vecinos |

**Derivación link-local (tarea 2.2):** El IID de `fe80::` se obtiene de la MAC extendida invirtiendo el bit 1 (`0x02`) del primer byte. En A, `ea` → `e8` (`eab595fbb2640be7`); en B, `a2` → `a0` (`a2c0582cdd913c14`); en C, `b6` → `b4` (`b678000381569e90`). Los demás bytes coinciden.

**Derivación del ID de router (tarea 2.2):** Los seis bits superiores de RLOC16 dan el ID de la tabla de routers: A `0x3000 >> 10 = 12`, B `0xcc00 >> 10 = 51` y C `0x0000 >> 10 = 0`. Estos valores coinciden con `ot router table` en A.

**Multicast (tarea 2.3):** `ot ipmaddr` mostró en A, B y C los cuatro grupos solicitados por la guía:

| Grupo | Alcance y destinatarios |
|-------|-------------------------|
| `ff02::1` | Todos los nodos del enlace |
| `ff03::1` | Todos los nodos de la malla |
| `ff02::2` | Routers del enlace |
| `ff03::2` | Routers de la malla |

### Lab 2: roles Thread (vista funcional)

| Rol | Función | Perfil de energía | Presente en esta prueba |
|-----|---------|-------------------|-------------------------|
| Leader | Coordina la red, los ID de router y los datos de red | Receptor encendido de forma continua | Sí: A durante las mediciones iniciales |
| Router | Reenvía tráfico entre nodos | Receptor encendido de forma continua | Sí: B y C durante las mediciones iniciales |
| REED | Nodo final apto para convertirse en router; reserva de enrutamiento | Receptor encendido de forma continua | No se configuró uno |
| End Device (FED) | Hoja que usa un router padre y no reenvía tráfico | Receptor encendido de forma continua | No se configuró uno para la prueba |
| Sleepy End Device (SED) | Hoja que duerme y consulta periódicamente a su padre | Puede mantener el receptor apagado gran parte del tiempo | No se configuró uno en Lab 2; se probó en Lab 3 |

Los routers y el leader mantienen la recepción activa, lo que dificulta cumplir el objetivo de tres meses con dos pilas AA de Lab 1. Para el campo requieren alimentación continua o una solución energética equivalente; los sensores a batería se evaluaron como SED en Lab 3. El consumo eléctrico no se midió aquí ni en Lab 3.

### Lab 3: contratos de las interfaces de aplicación (ASD)

En Lab 3, S fue el servidor CoAP en UDP/5683 y C fue el cliente. Se usó la ML-EID de S `fd78:5b46:d53e:718c:b456:2f8:216a:a556` en las pruebas; esta dirección corresponde al dataset de Lab 3 y no sustituye las direcciones históricas de Lab 2.

| Recurso | Métodos | Solicitud | Respuesta y formato | Política / códigos |
|---|---|---|---|---|
| `/env/temp` | GET; GET + Observe | Sin payload | Mapa CBOR `{"t": float16}` en °C, siempre 6 B; Content-Format 60 (`application/cbor`) | `2.05 Content`, Max-Age 60 s. Notificación NON si cambia >0,5 °C desde el último aviso; heartbeat CON tras 45 s sin aviso. |
| `/act/valve` | PUT (fijar estado); GET (leer) | En PUT: CBOR `{"v": 0|1}` con Content-Format 60, o texto `0`/`1` sin Content-Format como envía la CLI | Mapa CBOR `{"v": 0|1}`; Content-Format 60 | `2.04 Changed` para PUT, `2.05 Content` para GET, `4.00 Bad Request` para otra carga. PUT es idempotente: repetir el mismo estado no lo invierte. |

Bytes de ejemplo del contrato: `A1 61 74 F9 4E 20` representa `{"t": 24.5}`; `A1 61 76 00` y `A1 61 76 01` representan válvula cerrada y abierta. Nuestra lectura `A1 61 74 F9 4E 03` se decodificó como `{"t": 24.046875}` y S registró `2.05`.

```cddl
env-reading = { t: float16 }        ; °C
valve-state = { v: 0 / 1 }          ; 0 = closed, 1 = open
```

### Lab 4: contrato de `/env/temp` con sensor real (ASD)

S usa el módulo OKY3068 (DHT11). El recurso conserva GET, GET + Observe, el payload CBOR `{"t": float16}` en °C y Content-Format 60 de Lab 3. La tabla describe el contrato del firmware; las respuestas ante fallas se comprobaron en la Parte 4.

| Promesa | Contrato |
|---|---|
| Precisión declarada | ±2 °C para el DHT11, según la tabla de la guía de Lab 4; no equivale a la precisión comprobada de esta unidad. |
| Frescura | Max-Age indica los segundos restantes de una ventana de 60 s desde la última lectura aceptada. |
| Plausibilidad | Se rechazan temperaturas menores de −40 °C o mayores o iguales a 85 °C, y cambios mayores de 5 °C respecto de una lectura previa reciente (menos de 10 s en el firmware; normalmente se lee cada 2 s). Una lectura rechazada no reemplaza la última lectura buena. |
| Sin lectura fresca | Si no hay una lectura aceptada de menos de 60 s, GET responde `5.03 Service Unavailable`, Max-Age 10 y el texto `no fresh reading`. |
| Observe | Mientras hay lecturas frescas, mantiene la política de Lab 3: aviso NON por cambio mayor de 0,5 °C y heartbeat CON tras 45 s. Cuando dejan de estar frescas, envía una notificación 5.03 CON y termina la observación. |

```cddl
env-reading = { t: float16 }        ; °C, precisión según sensor, fresco durante Max-Age s
```

---

## 5. First Principles Reflections

**Lab 1:** No constan respuestas a preguntas de primeros principios de ese laboratorio en el registro disponible.

### Lab 2: red Thread y 6LoWPAN

1. **¿Cómo cabe IPv6 en una trama 802.15.4 de 127 bytes?** 6LoWPAN usa IPHC para no transmitir completos los campos predecibles de la cabecera IPv6 de 40 bytes: el prefijo mesh-local compartido se recupera del contexto y otros campos comunes se codifican de forma abreviada. El RLOC de A, `fd63:ae07:8739:36b7:0:ff:fe00:3000`, contiene un IID derivable del identificador corto `0x3000` cuando coincide con la dirección de enlace; en cambio, el IID de su ML-EID, `b456:2f8:216a:a556`, es aleatorio y debe enviarse porque no puede reconstruirse de la cabecera MAC. En el mejor caso la cabecera comprimida puede ocupar 2 bytes, pero ese mínimo no debe asumirse para nuestros pings de varios saltos. Si el datagrama aún no cabe, 6LoWPAN lo fragmenta. [RFC 6282](https://www.rfc-editor.org/rfc/rfc6282.html), [RFC 4944](https://www.rfc-editor.org/rfc/rfc4944.html).
2. **¿Por qué la recuperación tomó segundos y no unos 100 s?** Se observaron 8, 10 y 9 pings perdidos al desconectar B mientras había tráfico. La explicación de la guía y la lectura es que los intentos de transmisión hacia B dejan de recibir ACK MAC; al declararse fallido el enlace, los routers actualizan la ruta mediante MLE y A usa el enlace directo A–C. No se capturaron los ACK ni los anuncios MLE, por lo que este mecanismo es una explicación del protocolo, no una medición directa. Sin tráfico, la detección dependería más del vencimiento de los anuncios del vecino (aproximadamente 100 s en la guía).
3. **¿Por qué A–C debía ser un enlace pobre y no inexistente?** Con B activo, su ruta de dos saltos era preferible; al fallar B, el enlace A–C de LQI 1 seguía disponible y A pudo alcanzar C directamente (`Next Hop 63`). Una lista MAC que negara por completo el enlace A–C habría eliminado ese respaldo y la comunicación no se habría recuperado con solo estas tres placas. En el campo se necesita cobertura solapada y al menos una ruta alternativa; su alcance real con obstáculos y vegetación aún requiere pruebas.

La pregunta 4 de la guía (cambio de RLOC tras volver a asociar C) es opcional y no se realizó.

### Lab 3: preguntas de la Parte 7

1. **¿Por qué CoAP usa UDP en este laboratorio y qué aportan CON y NON?** El HTTP del Lab 0 usó TCP, que establece una conexión y ofrece entrega fiable de bytes; CoAP envía mensajes pequeños sobre UDP sin establecer esa conexión. Cuando una solicitud requiere confirmación, CON añade ACK y retransmisiones a nivel de mensaje: con S fuera de la red, C terminó en `ResponseTimeout` tras 95 s; al volver S, un PUT CON obtuvo la respuesta `{"v": 0}` y S registró `2.04`. NON no exige ACK ni retransmite a nivel de mensaje: el emisor no puede deducir del silencio si la orden llegó, aunque S sí registró algunos PUT NON en otras pruebas. [RFC 7252](https://www.rfc-editor.org/rfc/rfc7252.html).
2. **¿Cuándo ahorra tráfico Observe frente a `GET` cada 60 s?** El servidor conoce cuándo cambia la temperatura y puede notificar solo cuando cruza el umbral, mientras que un cliente que consulta periódicamente mantiene tráfico aunque no haya cambios. Ese ahorro es condicional: en nuestros 5 min el sensor simulado produjo **31 notificaciones de umbral y 0 heartbeats**; una consulta cada 60 s habría hecho aproximadamente **5 GET**, por lo que **esta prueba no demostró ahorro de mensajes frente a ese intervalo**. Frente al comparador de la tarea 3.2, un GET cada segundo (300 GET), sí hubo muchas menos notificaciones (31/300 = 10,3 %; además hubo una solicitud y respuesta iniciales de Observe). Esto compara tráfico, no corriente ni autonomía medidas. [RFC 7641](https://www.rfc-editor.org/rfc/rfc7641.html).
3. **¿Por qué repetir un PUT sería inocuo y repetir “alternar válvula” sería peligroso?** La pregunta supone dos entregas en la tarea 5.3, pero en nuestro registro de un único comando con S como SED y `pollperiod 5000` apareció **una sola** línea PUT y C recibió `{"v": 1}`. La línea `OPEN, no change` muestra que la válvula ya estaba abierta, **no** prueba una entrega duplicada. Como el primer intervalo de retransmisión CON (2–3 s) puede vencer antes del siguiente sondeo de 5 s, un duplicado es posible; `PUT {"v": 1}` vuelve a dejar la válvula abierta. Una orden “alternar” aplicada dos veces la abriría y luego la cerraría, con un resultado incorrecto para Edwin. La duplicación no se observó en este ensayo. [RFC 7252](https://www.rfc-editor.org/rfc/rfc7252.html).
4. **¿Por qué CBOR y no un `struct` binario propio?** CBOR estandariza tipos y estructura de mapas en bytes, de modo que otro decodificador pudo interpretar nuestro payload de 6 B `a16174f94e03` como `{"t": 24.046875}` sin conocer el código C ni su disposición de memoria. El contrato CDDL describe el significado y los tipos (`env-reading = { t: float16 }` y `valve-state = { v: 0 / 1 }`), lo que facilita documentar cambios de esquema para otros consumidores. Un `struct` propio obligaría a acordar manualmente tamaño, alineación, orden de bytes y evolución de campos. [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html), [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html).

### Lab 4: preguntas de la Parte 7

1. **¿Por qué `/env/temp` responde 5.03 en vez de servir el último valor?** Max-Age promete cuánto tiempo más puede considerarse fresca una lectura: el firmware parte de 60 s desde la última lectura aceptada y reduce ese plazo conforme envejece. Con DATA desconectado, S todavía sirvió 23,00 °C con edades de 15, 30 y 45 s y Max-Age de 45, 30 y 15 s. Al agotarse la ventana, registró 5.03 y C recibió `no fresh reading`. Presentar esos 23,00 °C como una medición actual después de perder el sensor podría llevar a Daniela a decidir un riego con información vencida. El 5.03 distingue la falta de dato fresco de una temperatura estable medida; su Max-Age de 10 s indica cuándo volver a intentar, no que el valor antiguo siga siendo válido. [Guía de Lab 4](en/labs/lab4.md#part-4--availability-can-the-client-get-the-data).
2. **¿Cómo distingue C un sensor muerto de un nodo inalcanzable?** Si falla el DHT11 pero S sigue en Thread, el nodo responde al GET: después de 60 s sin lectura válida devuelve **5.03** y el texto `no fresh reading`; S también termina Observe. Si S deja la red, no puede emitir esa respuesta. En la prueba con `ot thread stop`, un GET CON de C terminó en `ResponseTimeout` tras **90 s**, sin respuesta del nodo ni nuevas notificaciones Observe. Por tanto, **5.03 requiere una respuesta recibida de S** y apunta a falta de dato fresco; el silencio o un timeout indica que C no consiguió respuesta, pero por sí solo no identifica la causa física ni prueba que el sensor haya fallado.
3. **¿Por qué la plausibilidad no detecta una lectura incorrecta pero creíble?** El control local solo conoce límites y cambios respecto de la última lectura: rechazó **90,00 °C** por rango y **31,50 °C** por un salto de **+8,5 °C**, pero aceptó la inyección deliberada de **25,50 °C** desde una lectura cercana a 23 °C y hasta la notificó a C. Una segunda medición independiente junto al sensor o una lectura vecina contextualizada permitiría detectar discrepancias, aunque tampoco demostraría por sí sola cuál instrumento es correcto. Una firma o autenticación, prevista para Lab 6, podría verificar quién envió el dato; **no convierte en correcta una medición errónea del sensor**. La comparación posterior con un termómetro junto al DHT11 dio 23,0 frente a 24,1 °C (offset −1,1 °C), pero fue otra medición puntual y no valida la inyección anterior.
4. **¿Por qué puede fallar una lectura cuando la radio está ocupada?** El DHT11 codifica bits en la duración de pulsos y su controlador mide esos tiempos con la CPU; una interrupción de radio/OpenThread durante la lectura de aproximadamente **22 ms** puede alterar la medida o la comprobación y producir un fallo. En nuestra serie sin pings falló **1/69 (1,4 %)** y en la serie con pings **12/86 (14,0 %)**; C informó **3,5 %** de pérdida de ping en la segunda serie. La asociación es compatible con interferencia temporal entre lectura y actividad de radio, pero los contadores de S cubrieron **172 s**, más que los 120 s nominales de pings, y no se instrumentó cada interrupción: no se demostró la causa de cada fallo. Un sensor en un bus gestionado por hardware, como el BMP280 por I²C, no exige que la CPU mida cada pulso del dato; por eso es menos sensible a esta forma concreta de interrupción. **No se midió un BMP280 en esta pareja**, así que esa última comparación es de mecanismo, no una tasa experimental. [SOP-04](en/labs/sops/sop04_sensor_integration.md#b-the-dht-and-the-radio).

---

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20 m | No consta una distancia máxima medida | [ ] Sin verificar |
| Lab 2: Healing Time | < 120s | ≈ 8, 10 y 9 s; promedio ≈ 9 s | [x] Pass |
| Lab 3: CoAP Latency | < 200ms | 52,9 ms, GET C → S en un salto (una muestra, tiempo observado desde el PC) | [x] Pass en esta muestra |
| Lab 3: Poll Latency (5 s period) | < 5s | 4,191 s máximo (10/10 pings; una serie) | [x] Pass en esta serie |
| Lab 4: Time to see a lost sensor | ≤ 60s | 51 s desde retirar DATA hasta que C mostró `no fresh reading`, según el cronómetro del equipo; S registró el paso a 5.03 al agotarse la frescura de 60 s desde la última lectura válida. | [x] Pass en la prueba cronometrada |
| Lab 4: Read failures, radio busy | < 5% | 12/86 lecturas fallidas (14,0 %) en la repetición con pings; contador de S: 172 s | [ ] No cumple |
| Lab 6: DTLS Time | < 3 s | Pendiente de Lab 6 | [ ] Sin verificar |

### Lab 2: rendimiento y recuperación de la malla

#### Parte 3: comprobación de la ruta A → C

Ambas pruebas usaron 20 pings de 64 bytes, enviados cada 0,5 s desde A a la ML-EID de C. Se reiniciaron los contadores MAC de B antes de cada prueba.

| Condición | Pings recibidos | Pérdida | RTT mín./prom./máx. (ms) | `TxUnicast` en B |
|-----------|-----------------|---------|--------------------------|------------------|
| Antes de los filtros MAC | 20/20 | 0 % | 15/19,100/31 | 6 |
| Con A–C fijado en LQI 1 en ambos sentidos | 20/20 | 0 % | 30/30,650/33 | 40 |

Los 40 paquetes unicast transmitidos por B durante la segunda prueba coinciden con el reenvío esperado de 20 solicitudes y 20 respuestas. La comparación con los 6 anteriores verifica que el tráfico A–C pasó por B con los filtros activos. El RTT promedio aumentó 11,55 ms entre ambas pruebas.

En una tabla de A tomada después de reconectar B, el enlace A–B marcó LQ 3/3, C mantuvo LQ 1/1 y la fila de C mostró `Next Hop 51` (B) con `Path Cost 1`. Con los costos de la guía, la ruta directa A–C cuesta 4 y la ruta A–B–C cuesta 1 + 1 = 2. Esta comparación corresponde a esa tabla posterior; las calidades fueron menores en la primera tabla tomada después de aplicar los filtros.

#### Tarea 3.2: latencia por saltos

Las dos mediciones de Lab 2 usaron 20 pings de 64 bytes a intervalos de 0,5 s, una prueba a la vez.

| Ruta | Saltos | RTT mín./prom./máx. (ms) | Pérdida |
|------|--------|--------------------------|---------|
| A → B | 1 | 15/16,300/31 | 0 % (20/20) |
| A → C, vía B | 2 | 30/30,350/31 | 0 % (20/20) |
| Lab 1 A → B | 1 | Pendiente: no consta una medición comparable de 20 pings | Pendiente |

En esta prueba, el segundo salto añadió 14,05 ms al RTT promedio respecto de A → B.

#### Parte 4: recuperación al desconectar B

Cada prueba usa 200 pings de 64 bytes desde A a la ML-EID de C, a un ping por segundo. B se desconecta después de unas diez respuestas correctas. El tiempo de recuperación se estima por los pings perdidos, según el procedimiento de la guía.

| Ensayo | Pings perdidos | Recuperación estimada | Ruta observada después |
|--------|----------------|-----------------------|------------------------|
| 1 | 8 de 200 (4,0 %) | ≈ 8 s | A → C directa: C `Next Hop 63`; B ya no aparece en la tabla tras finalizar los 200 pings |
| 2 | 10 de 200 (5,0 %) | ≈ 10 s | A → C directa: C `Next Hop 63`; B ausente de la tabla tras finalizar los 200 pings |
| 3 | 9 de 200 (4,5 %) | ≈ 9 s | A → C directa: C `Next Hop 63`; B ausente de la tabla tras finalizar los 200 pings |

Los tres ensayos quedaron por debajo del objetivo de 120 s de la guía. Estos tiempos son aproximaciones a partir del número de pings perdidos a un ping por segundo; no se anotaron marcas de tiempo independientes del corte y la reanudación.

**Límites del registro:** No quedaron anotadas lecturas de `ot txpower` de B y C tras cada reconexión, ni una salida explícita de `ot state` de A antes del tercer ensayo. Sí se registró que B y C estaban como routers y que la tabla de A enviaba a C por B antes de ese ensayo. Estas verificaciones faltantes impiden afirmar que las tres pruebas mantuvieron condiciones idénticas de potencia y liderazgo.

### Lab 3: tiempos de respuesta, sondeo y presupuesto de bytes

#### Tiempos de respuesta CoAP

| Prueba | Resultado observado | Alcance de la medición |
|---|---|---|
| GET `/env/temp`, ida y vuelta en un salto | **52,9 ms** | En C se envió el GET por el puerto serie y se cronometró hasta recibir `coap response ... a16174f94def`. C era leader y S router; la tabla de vecinos de C mostraba a S (`eab595fbb2640be7`, RLOC16 `0x3000`) como vecino directo. Es una sola muestra de tiempo observado desde el PC, que incluye puerto serie y CLI; no mide exclusivamente el tránsito de radio. |
| PUT CON con S inalcanzable | **95 s** hasta `ResponseTimeout` (`error 28`) | Tiempo observado por C en la tarea 4.2; depende de los reintentos y la configuración de esta ejecución. |
| PUT NON con S inalcanzable | Sin confirmación; C seguía sin respuesta a los 2 min | El silencio no permite afirmar entrega ni pérdida. Los PUT NON registrados después por S no demuestran recepción durante el corte. |

**Predicción teórica del timeout CON (calculada con la guía después del ensayo):** sea `t` la primera espera aleatoria, entre 2 y 3 s. Cada espera siguiente se duplica y el cliente permite cuatro retransmisiones:

| Etapa | Espera de esa etapa | Tiempo acumulado |
|---|---:|---:|
| Primera retransmisión | `t = 2–3 s` | 2–3 s |
| Segunda retransmisión | `2t = 4–6 s` | 6–9 s |
| Tercera retransmisión | `4t = 8–12 s` | 14–21 s |
| Cuarta retransmisión | `8t = 16–24 s` | 30–45 s |
| Abandono tras la última espera | `16t = 32–48 s` | **`31t = 62–93 s`** |

La [lectura del Lab 3](en/labs/lectures/lab3_lecture.md) llama a ese intervalo `MAX_TRANSMIT_WAIT`. El ensayo registró **95 s**, unos 2 s por encima del extremo calculado. La medición desde la consola incluye tiempo de aplicación y cronómetro; no se capturaron las retransmisiones para atribuir con certeza la diferencia. **Esta predicción se reconstruyó después, no se anotó antes del ensayo**, como solicitaba la secuencia de la guía.

#### Tarea 3.2: Observe durante cinco minutos

S registró **31** notificaciones por umbral y **0** heartbeats. Frente a los **300 GET** que enviaría el comparador de la guía a 1 Hz, `31/300 = 10,3 %` de esa cantidad de mensajes de actualización, además del intercambio inicial de registro Observe. Esta comparación no demuestra ahorro frente a un GET cada 60 s, que habría enviado aproximadamente 5 consultas.

#### Tarea 5.2: S como SED y estimación de energía

Se usó `ot ping <S-mleid> 16 10 1.3 64 40` desde C. Cada serie recibió 10/10 respuestas; la latencia de sondeo es una medida de ping a S, no el RTT del GET CoAP. La fracción de radio encendida estima **10 ms por sondeo** divididos entre el período y no mide corriente.

| Período de sondeo | RTT mínimo / promedio / máximo | Pérdida | Fracción RX estimada |
|---|---|---:|---:|
| 1 s | 42 / 483,900 / 932 ms | 0/10 | 1 % |
| 5 s | 197 / 2192,200 / 4191 ms | 0/10 | 0,2 % |
| 15 s | 174 / 7231,800 / 13845 ms | 0/10 | 0,067 % |

**Estimación de energía de la válvula SED (sondeo propuesto de 5 s).** La [hoja de datos ESP32-C6 v1.5 de Espressif, tablas 5-9 a 5-11](https://documentation.espressif.com/esp32-c6_datasheet_en.html) indica **74 mA pico** para RX 802.15.4 (tabla 5-9), **17 mA típicos** en modem-sleep con CPU inactiva a 160 MHz y relojes periféricos apagados (tabla 5-10), y **35 µA típicos** en light-sleep con CPU, radio y periféricos apagados (tabla 5-11). Son condiciones del *chip*, no consumo medido de nuestras placas. La [guía del Lab 1](en/labs/lab1.md) usa 3000 mAh a 3 V para dos AA **en serie**: el voltaje se suma, la capacidad en mAh no.

Con `t_RX = 10 ms` y `T = 5000 ms`, la fracción RX es `d = 10/5000 = 0,002`. En un **escenario ideal de light-sleep** entre sondeos: `I_prom = 74 mA × 0,002 + 0,035 mA × 0,998 = 0,18293 mA`; duración ideal `3000 mAh / 0,18293 mA ≈ 16 399 h ≈ 683 días`. En un **escenario de modem-sleep** a 17 mA: `I_prom = 74 × 0,002 + 17 × 0,998 = 17,114 mA`; duración ideal `3000 / 17,114 ≈ 175 h ≈ 7,3 días`. Para llegar a tres meses (90 días), el presupuesto medio sería como máximo `3000/(90 × 24) ≈ 1,39 mA`.

El firmware actual solo configuró `ot mode -` y `ot pollperiod`; no habilita explícitamente gestión de energía ni se midió que alcance light-sleep. Por ello **683 días no es una autonomía demostrada**, y el escenario de 17 mA tampoco predice con precisión la placa real. Ambos modelos omiten TX, procesamiento, LED, regulador, pérdidas y variación de capacidad de las AA. Se mezclan el pico RX y valores típicos de reposo únicamente para esta estimación solicitada. Antes de aceptar el ADR-004 o prometer tres meses se debe medir la corriente de la placa funcionando como SED.

#### Parte 6: presupuesto de bytes

La guía compara tamaños de **mensajes CoAP**, antes de sumar UDP e IPv6. La consola de S midió 15 B para la solicitud `GET /env/temp` y 17 B para su respuesta (`CBOR a16174f94e03`, 6 B). El firmware no imprime el tamaño de las notificaciones Observe ni de los mensajes de válvula; esas celdas se calculan a partir del formato CoAP y del código, no son capturas de paquetes.

| Intercambio | Solicitud | Respuesta | Paquetes | Procedencia y nota |
|---|---:|---:|---:|---|
| HTTP/JSON lectura (Lab 0) | ~150 B + TCP | ~120 B + TCP | 7+ | Estimación de la guía: conexión, GET, respuesta y cierre; no medida aquí. |
| MQTT/JSON lectura (Lab 0) | — | ~40 B | 1 | Estimación de la guía para una conexión TCP ya abierta; excluye conexión y keepalives. |
| CoAP GET `/env/temp` | **15 B** | **17 B** | 2 | Ambos tamaños registrados por S en la tarea 3.1. |
| CoAP Observe, notificación | — | **19 B** | 1 | Cálculo para token de 2 B y secuencia Observe de 1 B; no registrado como tamaño por S. |
| CoAP PUT `/act/valve` (CON) | **18 B** | **13 B** | 2 | Cálculo para el texto `0` o `1`, sin Content-Format en la solicitud, y respuesta CBOR de 4 B; no registrado como tamaño por S. |

**Cuentas CoAP:** GET solicitado = 4 B de cabecera + 2 B de token + 4 B para `env` + 5 B para `temp` = **15 B**. GET respondido = 4 + 2 + 2 de Content-Format + 2 de Max-Age + 1 marcador de payload + 6 de CBOR = **17 B**. La notificación añade una opción Observe de 2 B: **19 B** con secuencia de un byte; puede crecer si crece la secuencia. PUT solicitado = 4 + 2 + 4 para `act` + 6 para `valve` + 1 marcador + 1 byte de texto = **18 B**. PUT respondido = 4 + 2 + 2 de Content-Format + 1 marcador + 4 de CBOR = **13 B**. El token de 2 B también consta en la [lectura del Lab 3](en/labs/lectures/lab3_lecture.md).

**Encaje en 802.15.4:** agregar 8 B de UDP a cada mensaje da, respectivamente, 23/25 B para GET, 27 B para Observe y 26/21 B para PUT, más IPv6 comprimido mediante 6LoWPAN. Incluso tomando como cota conservadora la cabecera IPv6 sin comprimir de 40 B, el mensaje mayor suma 19 + 8 + 40 = **67 B**, por debajo del presupuesto aproximado de 80–100 B para datos en una trama de 127 B descrito en la [lectura del Lab 2](en/labs/lectures/lab2_lecture.md). Por tamaño, estos mensajes caben en una trama; no se capturaron tramas de radio para medir su longitud efectiva ni confirmar directamente la ausencia de fragmentación.

### Lab 4: sensor real, disponibilidad y resiliencia

#### Tarea 3.1: resolución y variación en reposo

Con el OKY3068 (DHT11) quieto y `soilsense log on`, S registró 30 lecturas entre `00:26:01.620` y `00:26:59.620`, separadas aproximadamente 2 s. La temperatura fue 25,00 °C y luego 24,00 °C: el menor cambio **no nulo observado** entre lecturas fue 1 °C y la amplitud de la serie (máximo − mínimo) fue 1 °C. La humedad descendió de 60,0 a 58,0 %RH, en pasos observados de 1 punto porcentual. La amplitud de temperatura es la variación observada en este minuto; con una sola serie que cambia de nivel no se puede atribuirla por completo al ruido intrínseco del sensor. La exactitud frente a una referencia se evaluó en la tarea 3.2.

#### Tarea 3.2: comparación puntual con referencia

S registró DHT11 a **23,00 °C** (`last good`, 1 s de antigüedad) y el termómetro de referencia colocado junto al DHT11 marcó **24,1 °C en ese momento**, según confirmó el equipo. La diferencia calculada como `T_DHT11 − T_referencia` es **−1,1 °C**; su magnitud, **1,1 °C**, está dentro de los **±2 °C** que la guía indica para el DHT11. El mismo `soilsense stats` registró 320 lecturas en 639 s, 2 fallidas (0,6 %), 0 rechazadas y tiempo de lectura mínimo/promedio/máximo de 21 754/21 939/22 290 µs. Esta es una sola comparación; no valida la precisión del sensor en todo su rango. La referencia de 18 °C consultada antes en una página web no se usó como medición local.

#### Tarea 3.3: respuesta al aliento

**Primera serie.** Antes del cambio, S registró 25,00 °C y 53–54 %RH; la primera lectura de 26,00 °C fue a `00:40:39.620` y volvió a 25,00 °C a `00:41:13.620`. El aumento de temperatura observado fue **1 °C**, con dos notificaciones NON por umbral registradas en S (`00:40:40.036` y `00:41:14.071`). C mostró tres respuestas Observe con payloads CBOR correspondientes a **25 → 26 → 25 °C**; la primera no se atribuye a un nuevo cambio por umbral en esta ventana. La humedad alcanzó 90 %RH a `00:40:41.620` y descendió hasta 56 %RH en el fragmento compartido. No se anotó el instante de inicio del aliento y el DHT11 solo mostró pasos enteros de 1 °C: no apareció una lectura intermedia de 25,5 °C, así que el tiempo hasta la mitad del aumento **no se puede medir con esta serie**.

**Segunda serie.** El registro comenzó con 24,00 °C y 59 %RH. La temperatura alcanzó 28,00 °C a `00:47:29.619`, un aumento de **4 °C**; el nivel de mitad de subida, 26,00 °C, apareció por primera vez a `00:47:21.620`. Se registraron cuatro notificaciones NON por umbral durante la subida (25, 26, 27 y 28 °C) y una más al bajar a 27 °C a `00:47:37.620`; no se dispone de la salida correspondiente de C para esta serie. La persona marcó `00:47:19.620` como inicio del aliento, lo que daría 2 s hasta la lectura de 26 °C **respecto de esa marca**. Sin embargo, la humedad ya había subido de 59 a 66 %RH a `00:47:17.619` y era 86 %RH en la lectura marcada; por eso la exposición comenzó antes de la marca y el tiempo real hasta mitad de subida sigue sin determinarse. La hora de fin del aliento tampoco se anotó. La última temperatura de ese fragmento fue 27 °C; el registro siguiente permite confirmar que la temperatura volvió a 24 °C, sin precisar cuándo.

**Tercera serie, con marca de inicio.** A `00:49:53.619` S ya registraba 24,00 °C, de modo que la serie anterior volvió al valor inicial en algún momento del intervalo sin log; su hora exacta de recuperación no quedó medida. En esta serie se marcó `00:50:03.619` como inicio del cambio, con 24,00 °C y 67 %RH. La temperatura alcanzó **29,00 °C** a `00:50:19.620`, una subida de **5 °C**. El punto medio es 26,5 °C: la lectura fue 26 °C a `00:50:11.620` (8 s tras la marca) y la primera lectura igual o superior a 26,5 °C fue 27 °C a `00:50:13.620` (**10 s tras la marca**, resolución temporal de muestreo de 2 s). Volvió a 24 °C a `00:51:31.620`, 72 s después del máximo y 88 s después de la marca. S registró cinco notificaciones NON durante la subida (25–29 °C), cinco durante la bajada (28–24 °C) y un heartbeat CON a `00:50:04.621`. La humedad alcanzó 95 %RH y todavía era 92 %RH al final del log: estaba cerca del límite superior indicado para el DHT11, aun cuando la temperatura ya había vuelto a 24 °C. No se aportó la hora de fin del aliento; la recepción en C se documenta a continuación.

**Recepción en C para la nueva serie:** se recibieron respuestas Observe con temperaturas decodificadas de **25 → 26 → 27 → 28 → 29 → 28 → 27 → 26 → 25 → 24 → 23 → 23 °C** (`OBS=26` a `OBS=38`, sin una línea `OBS=35` en la salida compartida). Las diez primeras lecturas contienen la subida y vuelta a 24 °C que S registró; las dos lecturas de 23 °C son posteriores al fragmento de log de S. La ausencia de una línea `OBS=35` en esta copia no basta para concluir pérdida de paquetes ni para atribuirla a una causa. **No se comparó la curva con otro tipo de sensor porque el equipo no dispone de una pareja con otro sensor para esta prueba**, según informó el usuario; esa comparación solicitada por la guía queda sin datos.

#### Tarea 3.4: inyección y plausibilidad

A `00:55:40.802`, `soilsense inject 90` fue rechazado por estar fuera de −40..85 °C; el heartbeat posterior siguió anunciando 23,00 °C. A `00:56:45.349`, `soilsense inject 31.5` fue rechazado por un salto de +8,5 °C en una muestra. A `00:57:11.065`, `soilsense inject 25.5` pasó los controles: S notificó 25,50 °C (NON, cambio de 2,50 °C) y un segundo después notificó de nuevo 23,00 °C al leer el DHT11. Los controles locales de rango y salto no pueden demostrar que 25,5 °C fuera la temperatura real: era un valor plausible. Una referencia independiente o un segundo sensor permitirían contrastarlo; autenticar el origen en Lab 6 ayudaría frente a falsificación de mensajes, pero no corregiría por sí sola un sensor defectuoso.

#### Tarea 4.1: sensor desconectado

**Registro de S.** Se registraron lecturas fallidas a `01:00:41.616` y `01:00:43.616`, seguidas de `Sensor lost: 3 failed reads in a row` a `01:00:45.616`. S siguió respondiendo GET con el último valor bueno, 23,00 °C: a `01:00:55.449` tenía 15 s de edad y Max-Age 45; a `01:01:10.533`, 30 s y Max-Age 30; a `01:01:25.558`, 45 s y Max-Age 15. A `01:01:40.343`, S registró que ya no había lectura menor de 60 s, que GET pasaba a 5.03 y que terminaba una observación. Los GET de `01:01:40.725` y `01:01:55.687` quedaron registrados como 5.03 con reintento en 10 s. No se anotó la hora absoluta de la desconexión física ni se cronometró hasta la primera indicación en C.

**Registro de C.** C recibió primero respuestas GET con CBOR `a16174f94dc0` (23,00 °C) y después respuestas con payload hexadecimal `6e6f2066726573682072656164696e67`, que decodifica a `no fresh reading`. El shell de C muestra el payload pero no el código; S registró que esos GET recibieron 5.03. Algunas líneas de C quedaron intercaladas con el prompt y los comandos, por lo que no se atribuye con certeza una línea truncada a una respuesta específica de GET o Observe. El fin de una observación consta en S. Los **1 min 15 s** comunicados por el equipo fueron la duración durante la que enviaron GET desde C a S, no el tiempo desde desconectar DATA hasta el primer `no fresh reading`. Ese último intervalo no se cronometró en esta primera serie; en la repetición de la tarea 5.2 el equipo reportó 51 s hasta `no fresh reading`, resultado consignado en la tabla de §6.

#### Tarea 4.2: reconexión del sensor y pérdida del nodo

**Reconexión de DATA.** S registró `Sensor back after 24 failed reads` a `01:13:05.619`. El equipo midió **aproximadamente 2 s desde volver a conectar DATA hasta `Sensor back`**. Las 24 lecturas fallidas se acumularon durante la desconexión; el intervalo de 44 s entre el último `Sensor lost` aportado (`01:12:21.616`) y `Sensor back` no es el tiempo de recuperación desde la reconexión. C volvió a ejecutar `ot coap observe` y recibió `OBS=51` con CBOR `a16174f94e00` (**24 °C**). Esa respuesta con Observe y lectura confirma que `/env/temp` volvió a servir un valor fresco; conforme al contrato del firmware corresponde a 2.05. No se aportó la línea de S con su código ni se midió el tiempo entre reconexión y esa respuesta.

**S fuera de Thread.** Después de `ot thread stop` en S, C ejecutó `ot coap get <S-mleid> env/temp con`. La consola mostró `Done` y luego `coap receive response error 28: ResponseTimeout`; el cronómetro del equipo marcó **1 min 30 s (90 s)** desde enviar el GET hasta ese error. A diferencia del sensor desconectado con S activo, aquí no se obtuvo una respuesta 5.03 del nodo. Durante esos 90 s, C no imprimió ninguna línea Observe; este silencio no identifica por sí solo la causa, mientras que el GET CON permitió detectar que el nodo no respondía. Tras `ot thread start`, S volvió a la red con rol `child`.

#### Tarea 5.1: fallos de lectura con la radio quieta y ocupada

**Referencia sin pings.** S pasó posteriormente a `router`. Después de `soilsense reset` y **137 s** sin la carga de pings de la segunda fase, `soilsense stats` registró DHT11 a 24,00 °C y 62,0 %RH, **69 lecturas, 1 fallida (1,4 %) y 0 rechazadas**. El tiempo de lectura fue mínimo 21 874 µs, promedio 21 986 µs y máximo 22 110 µs. Esta serie es la referencia de la comparación con la radio ocupada; aún no demuestra cómo cambia la tasa bajo carga.

**Primera serie con pings.** C completó `ot ping <S-mleid> 64 1200 0.1`: **1200 enviados, 282 recibidos, 76,5 % de pérdida**, RTT mínimo/promedio/máximo de **15/19,691/104 ms** entre las respuestas recibidas. En S, el contador reiniciado marcó primero **49 lecturas en 98 s, 3 fallidas (6,1 %) y 0 rechazadas**; posteriormente marcó **133 lecturas en 265 s, 4 fallidas (3,0 %) y 0 rechazadas**, con tiempo de lectura mínimo/promedio/máximo de **21 874/22 001/23 145 µs**. El contador final cubre 265 s, mientras que los 1200 pings a intervalos de 0,1 s representan aproximadamente 120 s: **3,0 % es la tasa de todo el intervalo, no una tasa aislada durante la carga de radio**. Tampoco se midió si la pérdida de pings y los fallos de lectura tuvieron la misma causa; por ello se hizo una segunda serie con los contadores de S reiniciados justo antes.

**Segunda serie con pings y contadores reiniciados.** Tras `soilsense reset` en S, C completó `ot ping <S-mleid> 64 1200 0.1`: **1200 enviados, 1157 recibidos, 3,5 % de pérdida según la consola**, RTT mínimo/promedio/máximo de **15/21,385/257 ms** entre respuestas recibidas. Al finalizar C, `soilsense stats` en S mostró **86 lecturas en 172 s, 12 fallidas (14,0 %) y 0 rechazadas**, con tiempo de lectura mínimo/promedio/máximo de **19 141/22 106/23 233 µs**; la última lectura válida era 24,00 °C y 61,0 %RH. Frente a la referencia sin pings (1/69, 1,4 %), el porcentaje observado en este intervalo aumentó **12,6 puntos porcentuales** y superó el objetivo < 5 %. El contador de S cubrió 172 s, por lo que incluye tiempo adicional al nominal de 120 s de pings; no se aisló un porcentaje exclusivo de los paquetes en vuelo ni se demostró la causa de cada fallo. La cifra 3,5 % de la consola se conserva literalmente; 43 respuestas ausentes sobre 1200 envíos equivalen a 3,58 % si se calcula la fracción exacta. **No hubo una pareja con otro sensor disponible para contrastar estas tasas**; se conserva solo la comparación dentro de nuestra pareja.

#### Tarea 5.2: recuperación tras reconectar DATA

El equipo corrigió el tiempo de detección en C a **51 s desde retirar DATA hasta recibir el payload `6e6f2066726573682072656164696e67` (`no fresh reading`)**; la cifra anterior de 2 s para ese evento fue un error. Es una medición de cronómetro comunicada por el equipo, sin marca horaria de C en la salida conservada. En el registro de **S**, el heartbeat de `01:47:36.104` todavía anunció **24,00 °C con una lectura de 46 s de antigüedad**. A `01:47:50.120`, **14,016 s después**, S registró `GET now answers 5.03; 3 observation(s) ended`, coherente con la ventana de frescura de 60 s desde la última lectura válida. Esos tiempos y los **51 s desde la desconexión física** tienen puntos de partida distintos y no se deben restar entre sí. Tras reconectar DATA, el equipo midió **aproximadamente 2 s** hasta `Sensor back after 37 failed reads` a `01:48:05.620`; habían pasado **15,500 s** desde el 5.03, intervalo que incluye espera antes de reconectar y por tanto no mide recuperación. A `01:48:06.136`, **516 ms después de `Sensor back`**, S registró `Fresh readings again: GET answers 2.05; observers must register again`: el servidor volvió a estar listo para responder 2.05 aproximadamente **2,5 s desde reconectar DATA**, combinando el cronómetro del equipo con las marcas de S. Un GET posterior, a `01:51:43.142`, **217,006 s después de la disponibilidad**, confirmó **2.05, 24,00 °C, lectura de 1 s, Max-Age 59, CBOR `a16174f94e00`**; se solicitó tarde y no mide la latencia de recuperación del cliente. Las tres observaciones se terminaron al entrar en 5.03 y, según el firmware y el aviso de S, **no se reanudan solas: C debe registrarlas de nuevo**. No se infiere que fueran tres clientes distintos.

#### Tarea 5.3: recuperación tras RESET

Antes del reinicio, C tenía un Observe activo sobre S y recibió `OBS=2` con CBOR `a16174f94dc0`. Tras pulsar RESET, S mostró `ot state` = `disabled`: **Thread no arrancó automáticamente**. El sensor sí volvió a producir lecturas: `soilsense stats` mostró DHT11 a **23,00 °C**, lectura de 1 s, **14 lecturas en 27 s, 0 fallidas y 0 rechazadas**; son contadores nuevos tras el arranque, no los acumulados antes del reinicio. S recuperó la asociación solo después de que el equipo ejecutó `ot ifconfig up` y `ot thread start`: registró `detached` a `00:02:00.875`, `child` a `00:02:15.689` (unos 15 s después) y `router` a `00:02:53.642`; `ot state` confirmó `router`. C no recibió nuevas líneas `OBS=` durante el reinicio ni después de la reconexión de S. La observación previa no se recuperó sola; C tuvo que registrarla otra vez. El primer `ot coap observe` posterior al reinicio devolvió `Done` pero no mostró respuesta en C. Tras un segundo intento, C recibió `coap response from fdde:ad00:beef:0:65dc:e974:8408:adc9 OBS=2 with payload: a16174f94d80` (**22,00 °C**), lo que confirma la nueva suscripción. El equipo indicó que había un multímetro interfiriendo durante el primer intento; no se midió su efecto ni se establece como causa demostrada de la falta de respuesta. El reingreso sin importar manualmente otro dataset es compatible con que la configuración de red persistió, pero el servicio Thread necesitó inicio manual. A `00:03:07.980` S registró un fallo de lectura DHT11 `-5` posterior al arranque, sin evidencia aquí de una nueva pérdida permanente del sensor.

#### Parte 6: tabla de comparación (medición y simulación)

| Sensor | Resolución | Variación en 1 min | Offset frente a referencia | Aliento: subida / mitad de subida | Notificaciones en tarea 3.3 | Tiempo de lectura | Fallos reposo / radio ocupada | ¿Se recupera solo? |
|---|---|---|---|---|---|---|---|---|
| OKY3068 (DHT11) | Paso no nulo observado de 1 °C | Amplitud observada de 1 °C; no aislada como ruido intrínseco | −1,1 °C: S 23,0 °C y referencia 24,1 °C, juntos y al mismo tiempo | +5 °C / ≈10 s desde la marca de inicio, con muestreo cada 2 s | 10 avisos NON por umbral (5 subida, 5 bajada) y 1 heartbeat CON en la serie marcada | ≈22 ms promedio | 1/69 (1,4 %) / 12/86 (14,0 %); intervalo ocupado de S: 172 s | El sensor volvió a leerse ≈2 s tras reconectar DATA; el servicio quedó listo para 2.05 ≈0,5 s después. Observe requirió nuevo registro; tras RESET hubo que iniciar Thread manualmente. |
| DHT22 **(simulación)** | 0,1 °C | 0,2 °C | +0,3 °C | +3,0 °C / 6 s | 6 | 22 ms | 0/60 / 2/60 | Sí, tras reconectar DATA |

La fila del DHT11 contiene las mediciones de nuestra pareja. **Todos los números de la fila DHT22 son datos simulados** para ilustrar cómo se compararía otro sensor; no proceden de una placa ni de otra pareja. En esta tabla ilustrativa, la fila simulada tendría menor variación y menos fallos bajo carga que el DHT11 medido, pero esa diferencia no es un resultado experimental. **Esta tabla local es la versión final de la Parte 6 para el equipo con los datos disponibles**, según lo acordado; la guía menciona una tabla compartida de la clase, cuya ubicación y demás filas no se facilitaron. El DDR cita la tabla local completa en esta sección sin atribuir la simulación a otros estudiantes.

---

## 7. Ethics & Sustainability Checklist

*   [ ] **Lab 1:** No se verificó que la interferencia no afectara a redes vecinas.
*   [x] **Lab 2:** Se creó un dataset aleatorio propio y se ejecutó `ot thread stop` en las tres placas al terminar.
*   [x] **Lab 4:** Se definió un valor por defecto seguro para dato ausente en el diseño siguiente; todavía no se implementó ni se ensayó el cierre automático de la válvula.
*   [ ] **Lab 4:** No se documentó minimización de datos recolectados (privacidad).
*   [ ] **Lab 5:** Pendiente comprobar funcionamiento local sin nube (sostenibilidad).
*   [ ] **Lab 6:** Pendiente comprobar cifrado (privacidad).
*   [ ] **Lab 8:** Pendiente documentar el plan de fin de vida.

### Lab 3 — seguridad de la válvula ante pérdida de red

**Escenario y decisión:** S recibe `PUT /act/valve` con `{"v": 1}` y confirma OPEN; luego pierde el contacto con la red antes de recibir CLOSE. Si Thread pasa a `detached` mientras la válvula está abierta, S inicia un temporizador **local de 30 s**. Si continúa sin padre al vencer ese plazo, S ordena CLOSED por sí mismo y registra el cierre. Si vuelve a asociarse antes, cancela el temporizador. Es un período de gracia ante cambios breves de padre, tomado expresamente de la [parte B.2 del SOP-08](en/labs/sops/sop08_consolidation.md#b2--the-fail-safe-valve-fail-closed); **no** es un valor que aparezca en la guía del Lab 3 ni se deriva de los 95 s de `ResponseTimeout` de CoAP. Las recuperaciones de 8, 10 y 9 s de Lab 2 quedaron por debajo de ese margen, aunque fueron ensayos de routers y no validan todavía el comportamiento del SED. El plazo se cuenta desde que Thread **detecta** la desconexión, no necesariamente desde la pérdida física de la última trama. Al reiniciar, el estado seguro previsto es CLOSED; ante pérdida de alimentación, esto requiere además una válvula física normalmente cerrada.

**Alcance y verificación:** el firmware actual de Lab 3 **no implementa** ese cierre automático: `valve_open` solo cambia al procesar un PUT. La prueba futura debe usar S como **SED child**, abrir la válvula, dejar a S sin ningún padre Thread y comprobar que se cierre aproximadamente 30 s después de observar `detached`, sin recibir un PUT CLOSE. Con S como router, perder enlaces no implica necesariamente `detached`: podría formar otra partición como leader, por lo que este detector no basta para cubrir ese rol. Esta protección contra pérdida de red tampoco fija la duración máxima de un riego cuando la red sigue conectada. Para ese segundo límite hacen falta el caudal `Q` (L/min), el volumen admisible `V_max` (L) y la duración normal del ciclo; entonces `T_max ≤ V_max/Q` minutos. Ninguno de esos valores consta en Lab 3.

### Lab 4 — decisión de seguridad ante temperatura ausente

**Valor por defecto propuesto:** no iniciar un riego nuevo y solicitar CLOSED para una válvula que esté abierta cuando no haya temperatura vigente. Si `/env/temp` responde **5.03**, aplicar la decisión **inmediatamente**: S ya agotó los 60 s desde su última lectura válida y no es correcto usarla. Si C no obtiene respuesta, deja de usar la lectura almacenada **al vencer su Max-Age** (como máximo 60 s desde que se tomó la lectura), no espera a que una temperatura antigua parezca actual. En la prueba de nodo inalcanzable, el GET CON declaró `ResponseTimeout` tras **90 s**; en ese momento C clasifica la comunicación como fallida y avisa a Edwin, pero no debe esperar esos 90 s para invalidar un dato cuyo Max-Age ya venció.

**Límite físico:** si S es inalcanzable, un PUT CLOSE desde C no garantiza cerrar una válvula que ya está abierta. Para que la decisión sea segura en campo, S necesita ejecutar localmente el cierre cuando pierda datos frescos y un límite independiente de duración del riego. El cierre local a los **30 s desde `detached`** descrito para Lab 3 sigue siendo solo una propuesta y no cubre el caso en que S forma otra partición como leader. Faltan caudal y volumen máximo para fijar un tiempo máximo de riego por ciclo. **El firmware de Lab 4 no implementa ni valida este comportamiento de cierre**; se consigna como diseño y brecha de seguridad, no como resultado experimental.

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Lab 1-2 | Placas ESP32-C6, radio 802.15.4 y red Thread de tres nodos |
| Business | Pendiente | No documentado todavía en este DDR |
| Usage | Lab 3-4 | C consulta y observa la temperatura de S; Edwin distingue falta de dato fresco (`5.03`) de falta de respuesta (`ResponseTimeout`). Véase §2. |
| Functional | Lab 1-4 | PED: propagación e interferencia y temperatura del aire; SCD: placas, routers, DHT11 y controles de plausibilidad; ASD: servicios CoAP y contratos de §4. |
| Trustworthiness | Lab 4 | Integridad: offset −1,1 °C y filtros de plausibilidad con un valor falso aceptado; disponibilidad: 5.03 frente a timeout; fiabilidad: 1,4 % de fallos sin pings frente a 14,0 % en la serie con pings; resiliencia: sensor recuperado en ≈2 s y servicio listo 516 ms después, con Thread y Observe restaurados manualmente tras RESET. Véanse §5–§7 y la tabla completa de la Parte 6 en §6. |
| Construction | Pendiente de Lab 8 | El patrón de construcción de §10 aún no se ha documentado |

---

## 9. Trustworthiness Audit (Lab 4+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability | Parcial | Con DATA desconectado, S redujo Max-Age 45 → 30 → 15 s y pasó a 5.03 al agotar 60 s desde la última lectura válida; C recibió `no fresh reading` 51 s después de retirar DATA, según el cronómetro del equipo. Con S fuera de Thread, C obtuvo `ResponseTimeout` tras 90 s. | La medición de 51 s no tiene marca horaria de C conservada; Observe terminó y requirió un nuevo registro tras recuperar el sensor. |
| Confidentiality | → Lab 6 | Lab 4 no ensayó protección de confidencialidad. | Cifrado y ataques quedan para Lab 6. |
| Integrity | Parcial | DHT11 23,0 °C frente a referencia colocada al lado de 24,1 °C; se rechazaron 90 °C y un salto de +8,5 °C, pero pasó una inyección falsa de 25,5 °C. | Una comparación puntual no valida todo el rango; faltan referencia continua y protección frente a alteración maliciosa → Lab 6. |
| Reliability | Parcial; objetivo incumplido | 1/69 lecturas fallidas (1,4 %) sin pings frente a 12/86 (14,0 %) en la serie con pings; el objetivo de radio ocupada era <5 %. | El contador ocupado abarcó 172 s; no se aisló causalmente cada fallo ni se comparó con otro sensor medido. La fila DHT22 de la tabla §6 es simulada. |
| Resilience | Parcial | Al reconectar DATA, `Sensor back` apareció en ≈2 s y S declaró 2.05 otros 516 ms después. Tras RESET volvió a leer, pero Thread arrancó `disabled`; C tuvo que registrar Observe de nuevo. | Inicio de Thread y restauración de Observe requirieron acciones manuales; no se probó recuperación autónoma de extremo a extremo. |
| Safety | Diseño, sin prueba | Valor por defecto de §7: no iniciar riego sin temperatura fresca; ante 5.03 cerrar de inmediato, y ante silencio invalidar el dato al vencer Max-Age. | La orden CLOSE puede no llegar si S está fuera de la red; cierre local y límite de duración de riego aún no implementados. |
| Compliance | Pendiente | No se evaluó cumplimiento normativo en Lab 4. | Requiere requisitos y evidencias de cumplimiento específicos. |

---

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

Se conserva la tabla de la plantilla para Lab 8. Sus celdas permanecen sin completar porque el patrón de construcción final aún no se ha evaluado.

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System | Pendiente | Pendiente de Lab 8 |
| IoT Components | Pendiente | Pendiente de Lab 8 |
| Digital Network | Pendiente | Pendiente de Lab 8 |
| IoT Devices | Pendiente | Pendiente de Lab 8 |
| Primary Capability (observation) | Pendiente | Pendiente de Lab 8 |
| Primary Capability (control) | Pendiente | Pendiente de Lab 8 |
| Secondary Capability (processing) | Pendiente | Pendiente de Lab 8 |
| Secondary Capability (transferring) | Pendiente | Pendiente de Lab 8 |
| Secondary Capability (storage) | Pendiente | Pendiente de Lab 8 |
| Interface (network) | Pendiente | Pendiente de Lab 8 |
| Interface (human UI) | Pendiente | Pendiente de Lab 8 |
| Interface (application) | Pendiente | Pendiente de Lab 8 |
| Supplemental (security) | Pendiente | Pendiente de Lab 8 |
| Supplemental (orchestration) | Pendiente | Pendiente de Lab 8 |
| Supplemental (management) | Pendiente | Pendiente de Lab 8 |
