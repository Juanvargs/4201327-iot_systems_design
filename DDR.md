# Design & Decision Record (DDR)
**GreenField Technologies | IoT Systems Design**

**Team Members:**
1. Valeria Corredor García
2. Juan Felipe Camacho García
3. Juan Pablo Vargas Córdoba
---

## 1. System Overview

*   **System Type:** [ ] Component (Lab 1-2) | [x] System (Lab 3-6) | [ ] Environment (Lab 7-8)
*   **Description:** En Lab 2, tres placas ESP32-C6 (dos C6 normales y una Super Mini) formaron una red Thread sobre radio 802.15.4. En Lab 3 se usaron dos C6 normales: S ofrece temperatura y válvula mediante CoAP; C es el cliente y leader de Thread.

---

## 2. Lab Log & Stakeholder Summaries

### Lab 1: RF Characterization
*   **To Samuel (Architect):**
*   **To Edwin (Ops):**

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

### Lab 4: Sensors & Control
*   **To Edwin:**

### Lab 5: Border Router
*   **To Daniela:**

### Lab 6: Security
*   **To Edward (Security):**

### Lab 7: Dashboard
*   **To Gustavo (Product):**

### Lab 8: Final Integration
*   **To All:**

---

## 3. Architecture Decision Records (ADRs)
**ADR-001: Channel Selection**
*   **Context:** Se escanearon los canales 802.15.4 del 11 al 26 durante 500 ms por canal. Los canales 24 y 25 registraron el valor más bajo: −104 dBm.
*   **Decision:** Usar el canal 25.
*   **Rationale:** El aire y la interferencia pertenecen al PED. Aunque 24 y 25 empataron en la medición, la guía sitúa el canal 24 en el solapamiento con Wi-Fi 11 y el 25 en un espacio entre canales Wi-Fi. No se midieron los canales de los routers cercanos.
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**Para Felipe:** Usar el canal 25 (−104 dBm en este escaneo). Si aparece pérdida de paquetes, repetir el escaneo y revisar las redes Wi-Fi cercanas

**Comprobación al iniciar Lab 2:** Se repitió `ot scan energy 500` en A antes de fijar el canal del nuevo dataset. Los canales 17, 23, 24 y 25 empataron en −104 dBm; se mantuvo el canal 25 conforme al ADR-001. En A se eligieron 10 dBm de potencia de transmisión para esta prueba, en lugar de los 0 dBm indicados por la guía. Se detectó una lectura de 0 dBm en otra placa y se indicó fijar 10 dBm en B y C, pero no quedaron registradas lecturas de ambas después de cada reinicio.

**ADR-002: Topología de la red del campo**
*   **Context:** El campo necesita que un nodo alcance a otros aunque el enlace directo sea pobre y que la comunicación se recupere en menos de 120 s si cae un router. La guía pide comparar una malla Thread con una estrella de estilo LoRaWAN, cuyos dispositivos finales se comunican por un salto con gateways, sin usar a otro dispositivo final como relé.
*   **Decision:** Usar una malla Thread dentro del SCD para la conectividad entre nodos, con routers alimentados de forma continua y nodos finales dormidos cuando el requisito de batería lo exija. La autonomía de tres meses queda pendiente de validación en Lab 4.
*   **Rationale:** En esta prueba, A llegó a C a través de B con 20/20 respuestas, RTT promedio de 30,350 ms y `TxUnicast: 40` en B; al desconectar B, los tres tiempos estimados de recuperación fueron 8, 10 y 9 s, todos menores que 120 s. La estrella LoRaWAN no proporcionaría ese relé entre dispositivos finales en la topología ensayada. No se midió una red LoRaWAN en este laboratorio, por lo que la comparación con ella es arquitectónica. [Especificación LoRaWAN 1.0.4](https://lora-alliance.org/wp-content/uploads/2021/11/LoRaWAN-Link-Layer-Specification-v1.0.4.pdf).
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**ADR-003: CoAP/UDP y CBOR para telemetría y órdenes en la malla**
*   **Context:** El enlace Thread es restringido y el sistema necesita leer `/env/temp`, recibir cambios y enviar órdenes a `/act/valve` sin mantener una conexión TCP abierta. Los contratos están documentados en §4.
*   **Decision:** Usar CoAP sobre UDP con payload CBOR para GET/Observe de temperatura y PUT/GET de válvula; usar CON para órdenes y heartbeats, y NON para avisos de temperatura por umbral.
*   **Rationale:** En la Parte 6 se midieron 15 B de solicitud y 17 B de respuesta CoAP para GET de temperatura; se calcularon 19 B por notificación Observe y 18/13 B para PUT y respuesta de válvula, respectivamente. El comparador MQTT/JSON de la guía ronda 40 B por lectura sobre una conexión TCP ya establecida: el tamaño de un PUBLISH no incluye mantener el socket, los keepalives ni el broker. Por ello no se eligió MQTT para el lado radio de la malla. La prueba Observe dio 31 avisos en 5 min frente a 300 GET a 1 Hz, pero no demostró ahorro frente a GET cada 60 s; el beneficio depende de la frecuencia real de cambios. Los tamaños calculados no son mediciones de tramas.
*   **Status:** [ ] Proposed | [x] Accepted | [ ] Deprecated

**ADR-004: Período de sondeo de la válvula a batería**
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
| Thread routers, links and MLE | SCD | Communication subsystem of the routed mesh |
| ML-EID reachability for later application services | SCD ↔ ASD | Stable device addressing exposed to application services from Lab 3 onward |
| CoAP `/env/temp` y su contrato CBOR/CDDL | ASD | Servicio de lectura y observación de temperatura, con representación publicada |
| CoAP `/act/valve` y su contrato CBOR/CDDL | ASD | Servicio de control y lectura del estado de la válvula |
| SED y sondeo al router padre | SCD | Comunicación del dispositivo cuando la radio receptora duerme |

### Lab 3 — Contratos de las interfaces de aplicación (ASD)

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

### Lab 2 — Direcciones Thread

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
| Supporting | Time and crypto | Sincronización de tiempo y acelerador criptográfico; aún no usados en las pruebas | Latent | Lab 1 |
| Latent | Other radios | Wi-Fi y BLE no usados por Thread | Latent | Lab 1 |

### Lab 2 — Functional viewpoint: roles Thread

| Rol | Función | Perfil de energía | Presente en esta prueba |
|-----|---------|-------------------|-------------------------|
| Leader | Coordina la red, los ID de router y los datos de red | Receptor encendido de forma continua | Sí: A durante las mediciones iniciales |
| Router | Reenvía tráfico entre nodos | Receptor encendido de forma continua | Sí: B y C durante las mediciones iniciales |
| REED | Nodo final apto para convertirse en router; reserva de enrutamiento | Receptor encendido de forma continua | No se configuró uno |
| End Device (FED) | Hoja que usa un router padre y no reenvía tráfico | Receptor encendido de forma continua | No se configuró uno para la prueba |
| Sleepy End Device (SED) | Hoja que duerme y consulta periódicamente a su padre | Puede mantener el receptor apagado gran parte del tiempo | No se configuró uno en Lab 2; se probó en Lab 3 |

Los routers y el leader mantienen la recepción activa, lo que dificulta cumplir el objetivo de tres meses con dos pilas AA de Lab 1. Para el campo requieren alimentación continua o una solución energética equivalente; los sensores a batería se evaluaron como SED en Lab 3. El consumo eléctrico no se midió aquí ni en Lab 3.

---

## 5. First Principles Reflections

**Lab 1:**
1.
2.

**Lab 2:**
1. **¿Cómo cabe IPv6 en una trama 802.15.4 de 127 bytes?** 6LoWPAN usa IPHC para no transmitir completos los campos predecibles de la cabecera IPv6 de 40 bytes: el prefijo mesh-local compartido se recupera del contexto y otros campos comunes se codifican de forma abreviada. El RLOC de A, `fd63:ae07:8739:36b7:0:ff:fe00:3000`, contiene un IID derivable del identificador corto `0x3000` cuando coincide con la dirección de enlace; en cambio, el IID de su ML-EID, `b456:2f8:216a:a556`, es aleatorio y debe enviarse porque no puede reconstruirse de la cabecera MAC. En el mejor caso la cabecera comprimida puede ocupar 2 bytes, pero ese mínimo no debe asumirse para nuestros pings de varios saltos. Si el datagrama aún no cabe, 6LoWPAN lo fragmenta. [RFC 6282](https://www.rfc-editor.org/rfc/rfc6282.html), [RFC 4944](https://www.rfc-editor.org/rfc/rfc4944.html).
2. **¿Por qué la recuperación tomó segundos y no unos 100 s?** Se observaron 8, 10 y 9 pings perdidos al desconectar B mientras había tráfico. La explicación de la guía y la lectura es que los intentos de transmisión hacia B dejan de recibir ACK MAC; al declararse fallido el enlace, los routers actualizan la ruta mediante MLE y A usa el enlace directo A–C. No se capturaron los ACK ni los anuncios MLE, por lo que este mecanismo es una explicación del protocolo, no una medición directa. Sin tráfico, la detección dependería más del vencimiento de los anuncios del vecino (aproximadamente 100 s en la guía).
3. **¿Por qué A–C debía ser un enlace pobre y no inexistente?** Con B activo, su ruta de dos saltos era preferible; al fallar B, el enlace A–C de LQI 1 seguía disponible y A pudo alcanzar C directamente (`Next Hop 63`). Una lista MAC que negara por completo el enlace A–C habría eliminado ese respaldo y la comunicación no se habría recuperado con solo estas tres placas. En el campo se necesita cobertura solapada y al menos una ruta alternativa; su alcance real con obstáculos y vegetación aún requiere pruebas.

La pregunta 4 de la guía (cambio de RLOC tras volver a asociar C) es opcional y no se realizó.

**Lab 3 — Parte 7: preguntas de primeros principios**

1. **¿Por qué CoAP usa UDP en este laboratorio y qué aportan CON y NON?** El HTTP del Lab 0 usó TCP, que establece una conexión y ofrece entrega fiable de bytes; CoAP envía mensajes pequeños sobre UDP sin establecer esa conexión. Cuando una solicitud requiere confirmación, CON añade ACK y retransmisiones a nivel de mensaje: con S fuera de la red, C terminó en `ResponseTimeout` tras 95 s; al volver S, un PUT CON obtuvo la respuesta `{"v": 0}` y S registró `2.04`. NON no exige ACK ni retransmite a nivel de mensaje: el emisor no puede deducir del silencio si la orden llegó, aunque S sí registró algunos PUT NON en otras pruebas. [RFC 7252](https://www.rfc-editor.org/rfc/rfc7252.html).
2. **¿Cuándo ahorra tráfico Observe frente a `GET` cada 60 s?** El servidor conoce cuándo cambia la temperatura y puede notificar solo cuando cruza el umbral, mientras que un cliente que consulta periódicamente mantiene tráfico aunque no haya cambios. Ese ahorro es condicional: en nuestros 5 min el sensor simulado produjo **31 notificaciones de umbral y 0 heartbeats**; una consulta cada 60 s habría hecho aproximadamente **5 GET**, por lo que **esta prueba no demostró ahorro de mensajes frente a ese intervalo**. Frente al comparador de la tarea 3.2, un GET cada segundo (300 GET), sí hubo muchas menos notificaciones (31/300 = 10,3 %; además hubo una solicitud y respuesta iniciales de Observe). Esto compara tráfico, no corriente ni autonomía medidas. [RFC 7641](https://www.rfc-editor.org/rfc/rfc7641.html).
3. **¿Por qué repetir un PUT sería inocuo y repetir “alternar válvula” sería peligroso?** La pregunta supone dos entregas en la tarea 5.3, pero en nuestro registro de un único comando con S como SED y `pollperiod 5000` apareció **una sola** línea PUT y C recibió `{"v": 1}`. La línea `OPEN, no change` muestra que la válvula ya estaba abierta, **no** prueba una entrega duplicada. Como el primer intervalo de retransmisión CON (2–3 s) puede vencer antes del siguiente sondeo de 5 s, un duplicado es posible; `PUT {"v": 1}` vuelve a dejar la válvula abierta. Una orden “alternar” aplicada dos veces la abriría y luego la cerraría, con un resultado incorrecto para Edwin. La duplicación no se observó en este ensayo. [RFC 7252](https://www.rfc-editor.org/rfc/rfc7252.html).
4. **¿Por qué CBOR y no un `struct` binario propio?** CBOR estandariza tipos y estructura de mapas en bytes, de modo que otro decodificador pudo interpretar nuestro payload de 6 B `a16174f94e03` como `{"t": 24.046875}` sin conocer el código C ni su disposición de memoria. El contrato CDDL describe el significado y los tipos (`env-reading = { t: float16 }` y `valve-state = { v: 0 / 1 }`), lo que facilita documentar cambios de esquema para otros consumidores. Un `struct` propio obligaría a acordar manualmente tamaño, alineación, orden de bytes y evolución de campos. [RFC 8949](https://www.rfc-editor.org/rfc/rfc8949.html), [RFC 8610](https://www.rfc-editor.org/rfc/rfc8610.html).

...

---

## 6. Performance Baselines

| Metric | Target | Measured | Status |
|--------|--------|----------|--------|
| Lab 1: Max Range | > 20m | ___ m | [ ] Pass |
| Lab 2: Healing Time | < 120s | ≈ 8, 10 y 9 s; promedio ≈ 9 s | [x] Pass |
| Lab 3: CoAP Latency | < 200ms | 52,9 ms, GET C → S en un salto (una muestra, tiempo observado desde el PC) | [x] Pass en esta muestra |
| Lab 4: Poll Latency | < 5s | ___ s | [ ] Pass |
| Lab 6: DTLS Time | < 3s | ___ s | [ ] Pass |

### Lab 3 — tiempos de respuesta y sondeo

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

**Tarea 3.2 — Observe, ventana de 5 min:** S registró **31** notificaciones por umbral y **0** heartbeats. Frente a los **300 GET** que enviaría el comparador de la guía a 1 Hz, `31/300 = 10,3 %` de esa cantidad de mensajes de actualización, además del intercambio inicial de registro Observe. Esta comparación no demuestra ahorro frente a un GET cada 60 s, que habría enviado aproximadamente 5 consultas.

**Tarea 5.2 — S como SED.** Se usó `ot ping <S-mleid> 16 10 1.3 64 40` desde C. Cada serie recibió 10/10 respuestas; la latencia de sondeo es una medida de ping a S, no el RTT del GET CoAP. La fracción de radio encendida estima **10 ms por sondeo** divididos entre el período y no mide corriente.

| Período de sondeo | RTT mínimo / promedio / máximo | Pérdida | Fracción RX estimada |
|---|---|---:|---:|
| 1 s | 42 / 483,900 / 932 ms | 0/10 | 1 % |
| 5 s | 197 / 2192,200 / 4191 ms | 0/10 | 0,2 % |
| 15 s | 174 / 7231,800 / 13845 ms | 0/10 | 0,067 % |

**Estimación de energía de la válvula SED (sondeo propuesto de 5 s).** La [hoja de datos ESP32-C6 v1.5 de Espressif, tablas 5-9 a 5-11](https://documentation.espressif.com/esp32-c6_datasheet_en.html) indica **74 mA pico** para RX 802.15.4 (tabla 5-9), **17 mA típicos** en modem-sleep con CPU inactiva a 160 MHz y relojes periféricos apagados (tabla 5-10), y **35 µA típicos** en light-sleep con CPU, radio y periféricos apagados (tabla 5-11). Son condiciones del *chip*, no consumo medido de nuestras placas. La [guía del Lab 1](en/labs/lab1.md) usa 3000 mAh a 3 V para dos AA **en serie**: el voltaje se suma, la capacidad en mAh no.

Con `t_RX = 10 ms` y `T = 5000 ms`, la fracción RX es `d = 10/5000 = 0,002`. En un **escenario ideal de light-sleep** entre sondeos: `I_prom = 74 mA × 0,002 + 0,035 mA × 0,998 = 0,18293 mA`; duración ideal `3000 mAh / 0,18293 mA ≈ 16 399 h ≈ 683 días`. En un **escenario de modem-sleep** a 17 mA: `I_prom = 74 × 0,002 + 17 × 0,998 = 17,114 mA`; duración ideal `3000 / 17,114 ≈ 175 h ≈ 7,3 días`. Para llegar a tres meses (90 días), el presupuesto medio sería como máximo `3000/(90 × 24) ≈ 1,39 mA`.

El firmware actual solo configuró `ot mode -` y `ot pollperiod`; no habilita explícitamente gestión de energía ni se midió que alcance light-sleep. Por ello **683 días no es una autonomía demostrada**, y el escenario de 17 mA tampoco predice con precisión la placa real. Ambos modelos omiten TX, procesamiento, LED, regulador, pérdidas y variación de capacidad de las AA. Se mezclan el pico RX y valores típicos de reposo únicamente para esta estimación solicitada. Antes de aceptar el ADR-004 o prometer tres meses se debe medir la corriente de la placa funcionando como SED.

### Lab 3 — Parte 6: presupuesto de bytes

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

---

## 7. Ethics & Sustainability Checklist

### Lab 3 — seguridad de la válvula ante pérdida de red

**Escenario y decisión:** S recibe `PUT /act/valve` con `{"v": 1}` y confirma OPEN; luego pierde el contacto con la red antes de recibir CLOSE. Si Thread pasa a `detached` mientras la válvula está abierta, S inicia un temporizador **local de 30 s**. Si continúa sin padre al vencer ese plazo, S ordena CLOSED por sí mismo y registra el cierre. Si vuelve a asociarse antes, cancela el temporizador. Es un período de gracia ante cambios breves de padre, tomado expresamente de la [parte B.2 del SOP-08](en/labs/sops/sop08_consolidation.md#b2--the-fail-safe-valve-fail-closed); **no** es un valor que aparezca en la guía del Lab 3 ni se deriva de los 95 s de `ResponseTimeout` de CoAP. Las recuperaciones de 8, 10 y 9 s de Lab 2 quedaron por debajo de ese margen, aunque fueron ensayos de routers y no validan todavía el comportamiento del SED. El plazo se cuenta desde que Thread **detecta** la desconexión, no necesariamente desde la pérdida física de la última trama. Al reiniciar, el estado seguro previsto es CLOSED; ante pérdida de alimentación, esto requiere además una válvula física normalmente cerrada.

**Alcance y verificación:** el firmware actual de Lab 3 **no implementa** ese cierre automático: `valve_open` solo cambia al procesar un PUT. La prueba futura debe usar S como **SED child**, abrir la válvula, dejar a S sin ningún padre Thread y comprobar que se cierre aproximadamente 30 s después de observar `detached`, sin recibir un PUT CLOSE. Con S como router, perder enlaces no implica necesariamente `detached`: podría formar otra partición como leader, por lo que este detector no basta para cubrir ese rol. Esta protección contra pérdida de red tampoco fija la duración máxima de un riego cuando la red sigue conectada. Para ese segundo límite hacen falta el caudal `Q` (L/min), el volumen admisible `V_max` (L) y la duración normal del ciclo; entonces `T_max ≤ V_max/Q` minutos. Ninguno de esos valores consta en Lab 3.

*   [ ] **Lab 1:** Verified interference doesn't disrupt neighbors.
*   [x] **Lab 2:** Se creó un dataset aleatorio propio y se ejecutó `ot thread stop` en las tres placas al terminar.
*   [ ] **Lab 4:** Data collection minimized (Privacy).
*   [ ] **Lab 5:** System works locally without cloud (Sustainability).
*   [ ] **Lab 6:** Encryption enabled (Privacy).
*   [ ] **Lab 8:** End-of-Life plan considered.

---

## 8. Viewpoint Analysis

| Viewpoint | Labs Addressed | Key Concerns Documented |
|-----------|----------------|-------------------------|
| Foundational | Lab 1-2 | Placas ESP32-C6, radio 802.15.4 y red Thread de tres nodos |
| Business |               |                         |
| Usage |                  |                         |
| Functional | Lab 1-2 | PED: propagación e interferencia; SCD: placas, routers y MLE; interfaz SCD–ASD mediante ML-EID para servicios posteriores |
| Trustworthiness |        |                         |
| Construction |           |                         |

---

## 9. Trustworthiness Audit (Lab 6+)

| Characteristic | Addressed? | How | Gaps |
|----------------|------------|-----|------|
| Availability |            |     |      |
| Confidentiality |         |     |      |
| Integrity |               |     |      |
| Reliability |             |     |      |
| Resilience |              |     |      |
| Safety |                  |     |      |
| Compliance |              |     |      |

---

## 10. Construction Viewpoint - IoT System Pattern (Lab 8)

| Pattern Element | Category | Your System |
|-----------------|----------|-------------|
| IoT System |            |             |
| IoT Components |         |             |
| Digital Network |        |             |
| IoT Devices |            |             |
| Primary Capability (observation) | |   |
| Primary Capability (control) | |       |
| Secondary Capability (processing) | |  |
| Secondary Capability (transferring) | | |
| Secondary Capability (storage) | |     |
| Interface (network) |    |             |
| Interface (human UI) |   |             |
| Interface (application) | |            |
| Supplemental (security) | |            |
| Supplemental (orchestration) | |       |
| Supplemental (management) | |          |

### Lab 2 — Parte 3: comprobación de la ruta A → C

Ambas pruebas usaron 20 pings de 64 bytes, enviados cada 0,5 s desde A a la ML-EID de C. Se reiniciaron los contadores MAC de B antes de cada prueba.

| Condición | Pings recibidos | Pérdida | RTT mín./prom./máx. (ms) | `TxUnicast` en B |
|-----------|-----------------|---------|--------------------------|------------------|
| Antes de los filtros MAC | 20/20 | 0 % | 15/19,100/31 | 6 |
| Con A–C fijado en LQI 1 en ambos sentidos | 20/20 | 0 % | 30/30,650/33 | 40 |

Los 40 paquetes unicast transmitidos por B durante la segunda prueba coinciden con el reenvío esperado de 20 solicitudes y 20 respuestas. La comparación con los 6 anteriores verifica que el tráfico A–C pasó por B con los filtros activos. El RTT promedio aumentó 11,55 ms entre ambas pruebas.

En una tabla de A tomada después de reconectar B, el enlace A–B marcó LQ 3/3, C mantuvo LQ 1/1 y la fila de C mostró `Next Hop 51` (B) con `Path Cost 1`. Con los costos de la guía, la ruta directa A–C cuesta 4 y la ruta A–B–C cuesta 1 + 1 = 2. Esta comparación corresponde a esa tabla posterior; las calidades fueron menores en la primera tabla tomada después de aplicar los filtros.

### Lab 2 — Tarea 3.2: latencia por saltos

Las dos mediciones de Lab 2 usaron 20 pings de 64 bytes a intervalos de 0,5 s, una prueba a la vez.

| Ruta | Saltos | RTT mín./prom./máx. (ms) | Pérdida |
|------|--------|--------------------------|---------|
| A → B | 1 | 15/16,300/31 | 0 % (20/20) |
| A → C, vía B | 2 | 30/30,350/31 | 0 % (20/20) |
| Lab 1 A → B | 1 | Pendiente: no consta una medición comparable de 20 pings | Pendiente |

En esta prueba, el segundo salto añadió 14,05 ms al RTT promedio respecto de A → B.

### Lab 2 — Parte 4: recuperación al desconectar B

Cada prueba usa 200 pings de 64 bytes desde A a la ML-EID de C, a un ping por segundo. B se desconecta después de unas diez respuestas correctas. El tiempo de recuperación se estima por los pings perdidos, según el procedimiento de la guía.

| Ensayo | Pings perdidos | Recuperación estimada | Ruta observada después |
|--------|----------------|-----------------------|------------------------|
| 1 | 8 de 200 (4,0 %) | ≈ 8 s | A → C directa: C `Next Hop 63`; B ya no aparece en la tabla tras finalizar los 200 pings |
| 2 | 10 de 200 (5,0 %) | ≈ 10 s | A → C directa: C `Next Hop 63`; B ausente de la tabla tras finalizar los 200 pings |
| 3 | 9 de 200 (4,5 %) | ≈ 9 s | A → C directa: C `Next Hop 63`; B ausente de la tabla tras finalizar los 200 pings |

Los tres ensayos quedaron por debajo del objetivo de 120 s de la guía. Estos tiempos son aproximaciones a partir del número de pings perdidos a un ping por segundo; no se anotaron marcas de tiempo independientes del corte y la reanudación.

**Límites del registro:** No quedaron anotadas lecturas de `ot txpower` de B y C tras cada reconexión, ni una salida explícita de `ot state` de A antes del tercer ensayo. Sí se registró que B y C estaban como routers y que la tabla de A enviaba a C por B antes de ese ensayo. Estas verificaciones faltantes impiden afirmar que las tres pruebas mantuvieron condiciones idénticas de potencia y liderazgo.
