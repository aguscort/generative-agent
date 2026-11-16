# Human_proxy · Documentación técnica: núcleo y memoria

Human_proxy implementa en Make.com un agente conversacional que habla por Telegram y recuerda, inspirado en *Generative Agents* (Park et al., 2023) para la memoria, la recuperación y la reflexión, y en *Reflexion* (Shinn et al., 2023) para que las reflexiones sirvan para actuar mejor. El agente de ejemplo es Borja, un asesor financiero ficticio.

Este documento cubre dos partes:

- **El núcleo:** la entrada por Telegram, el Loop del bot y la redacción de la respuesta en Actuar.
- **La memoria:** Escribir, Recuperar, Resumen del agente y Reflexionar.

La planificación y la ejecución quedan fuera de este documento.

| Escenario en Make | Parte | Disparo | Función |
| --- | --- | --- | --- |
| `[Human_proxy]:[01-Gateway] (01/17) \| Telegram` | Núcleo | Mensaje en Telegram | Filtra, limita la frecuencia, llama al Loop y envía la respuesta |
| `[Human_proxy]:[00-Core] \| Loop del bot` | Núcleo | Webhook | Reúne la memoria, pide la respuesta, la guarda y la devuelve |
| `[Human_proxy]:[Ejecución] (15/17) \| Actuar` (ruta del Loop) | Núcleo | Webhook | Redacta la respuesta con Gemini |
| `[Human_proxy]:[02-Memoria] (02/17) \| Escribir` | Memoria | Webhook | Puntúa, vectoriza y guarda un recuerdo |
| `[Human_proxy]:[02-Memoria] (04/17) \| Recuperar` | Memoria | Webhook | Devuelve los recuerdos mejor puntuados y la ventana de mensajes |
| `[Human_proxy]:[02-Memoria] (05/17) \| Resumen del agente` | Memoria | Webhook | Devuelve el resumen del agente, con caché de 7 días |
| `[Human_proxy]:[02-Memoria] (03/17) \| Reflexionar` | Memoria | Webhook | Saca lecciones de la actuación reciente y las guarda |

Los modelos son Gemini 3.8 Flash para el lenguaje y Gemini Embedding 2, con 768 dimensiones, para los vectores. Los cálculos de puntuación los hace el módulo Make Code, no el modelo. La memoria vive en Airtable; el perfil, la caché del resumen y la ventana de mensajes, en un almacén de datos de Make.

---

## 1. Recorrido de un mensaje

```
Telegram ──► Loop del bot ──► Resumen del agente
                         ├──► Recuperar
                         ├──► Actuar (redacta)
                         ├──► Escribir (tu mensaje)
                         └──► Escribir (la respuesta)
         ◄── respuesta ──┘
```

1. Telegram recibe tu mensaje, comprueba que eres un usuario identificado y que no has superado el límite de frecuencia, y llama al Loop.
2. El Loop lee el perfil del agente y pide a Resumen el resumen de la conversación. Resumen lo devuelve desde su caché o lo regenera.
3. El Loop pide a Recuperar los 5 recuerdos más útiles para tu mensaje y los 6 mensajes anteriores de la conversación. Recuperar apunta tu mensaje en la ventana de la conversación.
4. Actuar redacta la respuesta con el perfil, el resumen, los recuerdos, los mensajes anteriores y tu mensaje.
5. El Loop guarda en la memoria tu mensaje y la respuesta, a través de Escribir, y devuelve la respuesta a Telegram.
6. Telegram te la envía.

Todo se guarda antes de responder, para que cualquier fallo se pueda comunicar en la misma respuesta.

Reflexionar no forma parte de este recorrido: lo llama el Despertador cada día a las 5:00.

---

## 2. Datos

### 2.1. La memoria: tabla de Airtable

Base `appZSaY9QBp9Kw8AA`, tabla `tblHjYljQ167ySe8M`. Cada registro es un recuerdo.

| Campo | Contenido |
| --- | --- |
| Agente | Nombre del agente, por ejemplo «Borja» |
| Chat_ID | Conversación a la que pertenece, o «GLOBAL» si vale para todas |
| Memoria | El recuerdo en lenguaje natural, por ejemplo «Agus le dijo a Borja: «…»» |
| Tipo | `observacion`, `reflexion` u otros tipos que escriben otros escenarios |
| Autor | `usuario`, `agente` o `sistema` |
| Importancia | Entero del 1 al 10 |
| Embedding | Vector de 768 números, guardado como texto |
| Creado | Fecha de creación |
| Ultimo_acceso | Última vez que Recuperar lo devolvió |
| Evidencias | En las reflexiones, los identificadores de los recuerdos en que se basan |
| Reflejada | Casilla que marca Reflexionar en las observaciones ya tenidas en cuenta |

### 2.2. El almacén de datos Caché

Almacén de Make con la estructura «[Human_proxy] Resumen agente». Guarda tres clases de registros, distinguidos por la clave:

| Clave | Contenido | Quién lo escribe | Quién lo lee |
| --- | --- | --- | --- |
| `perfil\|Borja` | Descripción del agente, en el campo Resumen | Ningún escenario de este documento | Loop, Despertador |
| `Borja\|<chat>` | Resumen del agente en esa conversación y su fecha de actualización | Resumen | Resumen |
| `ventana\|Borja\|<chat>` | Ventana de los 6 últimos mensajes de la conversación, como texto JSON | Recuperar | Recuperar |

Los campos de la estructura son Agente, Chat ID, Resumen, Actualizado, sistemas, token y ventana.

---

## 3. Comunicación entre escenarios

### 3.1. Autenticación

Las llamadas entre escenarios son peticiones HTTP POST a los webhooks de Make, autenticadas con una conexión de clave API de Make, que envía la clave en la cabecera. Las rutas descritas en este documento no filtran por token.

### 3.2. Gestión de errores

Todos los escenarios siguen la misma regla: si algo falla, se informa del origen y la ejecución se detiene.

- **Escenarios llamados (Resumen, Recuperar, Escribir, Actuar y Reflexionar).** Cada módulo que puede fallar tiene un manejador que responde con un código 500 y un cuerpo como `{"error": "Recuperar › lectura de la memoria (Airtable)"}`, con el escenario, el paso y el servicio, y después hace Commit. Commit detiene la ejecución sin marcarla como fallida, de modo que el escenario no se desactiva.
- **Loop.** Cuando una llamada falla, su manejador responde a Telegram con un 500 y un aviso de Borja que dice qué ha fallado (sección 5.3).
- **Telegram.** Envía al usuario el texto que devuelva el Loop, sea respuesta o aviso. Si el Loop no contesta, envía su propio aviso.
- **Fallos sin canal.** Si el fallo ocurre después de haber respondido, o si falla el propio envío por Telegram, el manejador solo hace Commit y el fallo queda en el historial de Make.

Un escenario con disparador instantáneo se desactiva en cuanto tiene un error no gestionado. Por eso no hay ningún módulo de estos escenarios que pueda fallar sin manejador.

### 3.3. Contratos

| Escenario | Recibe | Devuelve |
| --- | --- | --- |
| Loop | `agente`, `chat_id`, `usuario`, `mensaje` | 200 `{"respuesta": "…"}`, o 500 `{"respuesta": "<aviso>"}` |
| Resumen | `agente`, `chat_id`, `modo` | 200 `{"resumen": "…", "estado": "cache" \| "regenerado" \| "OFF"}`, o 500 `{"error": "…"}` |
| Recuperar | `agente`, `chat_id`, `consulta` o `consultas`, `k`, `n`, `usuario` | 200 con `memorias`, `texto`, `texto_ids`, `mensajes`, `total_evaluadas`, `resultados`, o 500 `{"error": "…"}` |
| Actuar (ruta del Loop) | `agente`, `chat_id`, `usuario`, `mensaje`, `descripcion`, `resumen`, `recuerdos`, `mensajes` | 200 `{"respuesta": "…"}`, o 500 `{"error": "…"}` |
| Escribir | `agente`, `chat_id`, `memoria`, `tipo`, `autor`, `evidencias` | 200 con `id`, `importancia` y `tipo` del registro, o 500 `{"error": "…"}` |
| Reflexionar | `agente`, `chat_id` | 200 `{"estado": "reflexion completada", "agente": "…", "reflexiones": N}`, o 500 `{"error": "…"}` |

### 3.4. Tiempos de espera

| Llamada | Espera máxima |
| --- | --- |
| Telegram → Loop | 120 s |
| Loop → Resumen | 180 s |
| Loop → Recuperar | 60 s |
| Loop → Actuar | 120 s |
| Loop → Escribir (cada una) | 120 s |
| Resumen → Recuperar | 120 s |
| Reflexionar → Recuperar | 60 s |
| Reflexionar → Escribir | 120 s |
| Despertador → Reflexionar | 300 s |

---

## 4. Telegram


---

## 5. Loop del bot


---

## 6. Actuar: redacción de la respuesta


---

## 7. Escribir


---

## 8. Recuperar


---

## 9. Resumen del agente

 
---

## 10. Reflexionar


---

## 11. Rendimiento medido

Medida del 11/11/2026, anterior a la segunda lectura de Recuperar y a la ventana en caché: la regeneración del resumen tardó 31,5 s, de los que 7,3 s fueron la llamada a Recuperar y 23,9 s la de Gemini. En los mensajes siguientes, Resumen devolvió la caché. Los tiempos del resto de la cadena con la versión actual están por medir.

---

## 12. Limitaciones y pendientes

- **Ventana de mensajes.** Hoy la ventana solo recibe los mensajes del usuario. Falta que Escribir añada las respuestas del agente cuando guarde una observación de autor agente.
- **Importancias.** El criterio económico se aplica a lo guardado desde el 11/11/2026; lo anterior conserva la puntuación por intensidad emocional. Algún recuerdo antiguo con trascendencia económica puede no entrar en la segunda lectura de Recuperar.
- **Segunda lectura de Recuperar.** Está limitada a 1.000 reflexiones o recuerdos importantes.
- **Reflexionar:**
  - El paso de interpretación (9) espera texto JSON, mientras que Gemini con esquema entrega datos estructurados. Está pendiente comprobar si sobra o si falla.
  - El marcado de reflejadas actualiza los registros de uno en uno, lo que alarga la ejecución con muchas observaciones pendientes.
  - La decisión de actuar pertenece a la planificación y está pendiente de decidir si sigue aquí. Su llamada a Comprometer todavía envía token.
- **Disparo de la reflexión.** Está pendiente de definir si el Despertador llama a Reflexionar por chat o solo con el agente.
- **Autenticación.** No se ha comprobado en la configuración de cada webhook que exija la clave API.
- **Tiempos.** No se ha comprobado el tiempo máximo que Make mantiene abierta la respuesta de un webhook. Las esperas de Telegram y del Despertador deben ajustarse con medidas reales.
- **Pruebas.** Los cambios de la ventana, la segunda lectura de Recuperar, los nuevos prompts de Reflexionar, su gestión de errores y la llamada del Despertador no se han ejecutado todavía.

---

## 13. Referencias

- Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P. y Bernstein, M. S. (2023). *Generative Agents: Interactive Simulacra of Human Behavior*. [arXiv 2304.03442](https://arxiv.org/abs/2304.03442). Memoria, recuperación (recencia, importancia y relevancia) y reflexión (preguntas de alto nivel, recuperación por pregunta y conclusiones con evidencias).
- Shinn, N., Cassano, F., Berman, E., Gopinath, A., Narasimhan, K. y Yao, S. (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning*. [arXiv 2303.11366](https://arxiv.org/abs/2303.11366). Reflexión orientada a corregir la actuación con lecciones aplicables.
