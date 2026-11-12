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

Escenario `[Human_proxy]:[01-Gateway] (01/17) | Telegram`. Se dispara con cada mensaje que llega al bot y procesa los mensajes en orden: no atiende el siguiente hasta que ha terminado con el anterior.

1. **Entrada.** Solo continúan los mensajes de texto que no son comandos.
2. **Usuario identificado.** Si el remitente no es el usuario autorizado, responde que no es un usuario autenticado y le indica el repositorio del proyecto.
3. **Límite de frecuencia.** Cuenta en Airtable las observaciones del usuario en ese chat de los últimos 10 minutos.
   - **Menos de 10:** llama al Loop con `{"agente": "Borja", "chat_id", "usuario", "mensaje"}`.
   - **10 o más:** responde «Descansamos un poco. Intentalo en un par de minutos» y no llama al Loop.
4. **Respuesta.** La llamada al Loop no trata como error los códigos distintos de 2xx, así que lee igual una respuesta normal (200) que un aviso (500). Envía al usuario el campo `respuesta`. Si llega vacío, envía «Perdona, no puedo atenderte: el Loop ha respondido sin texto…».

**Avisos propios de Telegram:**

| Situación | Mensaje |
| --- | --- |
| El Loop no responde o rechaza la conexión | «Perdona, no puedo atenderte: no he podido comunicarme con el Loop (no responde o rechaza la conexión). Escríbeme de nuevo en un momento.» |
| Falla el recuento de mensajes recientes | «Perdona, no puedo atenderte: ha fallado el control de mensajes recientes (Telegram › Airtable). Escríbeme de nuevo en un momento.» |

Si falla un envío por Telegram, el manejador hace Commit: no hay otro canal para avisar.

---

## 5. Loop del bot

Escenario `[Human_proxy]:[00-Core] | Loop del bot`. Disparo instantáneo por webhook. Su flujo es lineal:

### 5.1. Módulos

1. **Webhook.** Recibe `agente`, `chat_id`, `usuario` y `mensaje`.
2. **Perfil (960).** Lee del almacén el registro `perfil|<agente>`, con la descripción del agente.
3. **Resumen (61).** Llama a Resumen con `{"agente", "chat_id", "modo": "ON"}`. Resumen decide si devuelve la caché o la regenera.
4. **Recuperar (2).** Llama a Recuperar con `{"agente", "chat_id", "usuario", "consulta": <mensaje>, "k": 5, "n": 6}`. Recibe los 5 recuerdos más útiles y los mensajes de la ventana, y Recuperar apunta el mensaje en ella.
5. **Actuar (70).** Le envía:
   - `descripcion`: la del perfil (o el nombre del agente si no hay perfil), más «Limita la respuesta a una o dos frases, sin pasar de 20 palabras»;
   - `resumen`: el que devolvió Resumen;
   - `recuerdos`: el texto numerado de Recuperar;
   - `mensajes`: la ventana;
   - `agente`, `chat_id`, `usuario` y `mensaje`.
6. **Router (66):**
   - **Si Actuar no devolvió respuesta:** responde con un 500 y el aviso correspondiente.
   - **Si la hay:**
     - **Escribir (4)** guarda «<usuario> le dijo a <agente>: «<mensaje>»», como observación de autor usuario;
     - **Escribir (30)** guarda «<agente> le respondió a <usuario>: «<respuesta>»», como observación de autor agente;
     - **Respuesta (20):** 200 `{"respuesta": "…"}`.

### 5.2. Por qué se guarda antes de responder

Si la memoria no se puede guardar, Borja no responde como si nada: avisa de que no ha podido guardar. Así la memoria siempre recoge lo que el usuario ha visto.

### 5.3. Avisos de Borja

| Falla | Mensaje |
| --- | --- |
| Lectura del perfil | «Perdona, no puedo atenderte: ha fallado la lectura de mi perfil (Loop › almacén Caché). Escríbeme de nuevo en un momento.» |
| Resumen | «… ha fallado el resumen de nuestra conversación (escenario Resumen) …» |
| Recuperar | «… ha fallado la recuperación de mis recuerdos (escenario Recuperar) …» |
| Actuar | «… ha fallado la redacción de mi respuesta (escenario Actuar) …» |
| Actuar sin texto | «… la redacción de mi respuesta ha llegado vacía (escenario Actuar) …» |
| Guardado del mensaje | «Perdona, no he podido guardar lo que me has dicho (escenario Escribir). Repítemelo en un momento.» |
| Guardado de la respuesta | «Perdona, no he podido guardar mi respuesta (escenario Escribir). Repíteme lo que me decías en un momento.» |

Todos salen con código 500 y después Commit. El aviso nombra el escenario que llamó el Loop; el detalle del paso que falló dentro de ese escenario está en su respuesta 500, visible en el historial de Make.

---

## 6. Actuar: redacción de la respuesta

Escenario `[Human_proxy]:[Ejecución] (15/17) | Actuar`. Atiende al Loop en una ruta propia, con el filtro «Llamada del Loop»: el campo `mensaje` existe y el campo `clave` no. Sus demás rutas pertenecen a la ejecución y no se describen aquí.

1. **Gemini (81).**
   - **Instrucción de sistema:** la descripción del agente; después, «Lo que <agente> ha llegado a saber de sí mismo y de su interlocutor (si está vacío, ignóralo): <resumen>»; y por último las reglas de personaje: es el agente, habla por Telegram, se mantiene en su papel, no inventa recuerdos ni precios ni cotizaciones.
   - **Contenido:** los recuerdos, los mensajes anteriores (del más antiguo al más reciente) y el mensaje actual.
2. **Respuesta (82):** 200 `{"respuesta": "<texto de Gemini sin espacios sobrantes>"}`.

Si Gemini falla: 500 `{"error": "Actuar › redacción de la respuesta (Gemini)"}` y Commit.

---

## 7. Escribir

Escenario `[Human_proxy]:[02-Memoria] (02/17) | Escribir`. Es la única puerta de entrada a la memoria.

1. **Webhook.** Recibe `agente`, `chat_id`, `memoria`, `tipo`, `autor` y, en las reflexiones, `evidencias`.
2. **Importancia (Gemini, con esquema JSON).** Puntúa del 1 al 10 la trascendencia económica del recuerdo para el agente o para su interlocutor:

   > En una escala del 1 al 10, donde 1 es económicamente irrelevante (p. ej., un saludo o una charla trivial) y 10 es de máxima trascendencia económica (p. ej., una operación de gran volumen, una pérdida o ganancia importante, un movimiento brusco del mercado), puntúa la importancia económica del siguiente recuerdo de <agente>, para él o para su interlocutor.

   La respuesta está forzada a `{"importancia": <entero>}`.
3. **Embedding.** Gemini Embedding 2, con 768 dimensiones.
4. **Airtable.** Crea el registro con todos los campos y el último acceso a la hora actual.
5. **Respuesta.** 200 con el identificador, la importancia y el tipo.

**Errores:** puntuación, embedding y guardado tienen cada uno su manejador, con 500 y su origen («Escribir › puntuación de importancia (Gemini)», «Escribir › embedding del recuerdo (Gemini)» y «Escribir › guardado del recuerdo (Airtable)»), y Commit.

---

## 8. Recuperar

Escenario `[Human_proxy]:[02-Memoria] (04/17) | Recuperar`. Sirve a dos clases de llamadas:

- **El Loop:** una consulta (el mensaje del usuario), con `k` = 5 y `n` = 6. Recibe el contexto de la pregunta: los recuerdos más útiles y los mensajes anteriores.
- **Resumen y Reflexionar:** una o varias consultas fijas o generadas, sin `n`. Reciben solo recuerdos.

### 8.1. Módulos

1. **Webhook.** Recibe `agente`, `chat_id`, `consulta` o `consultas`, `k` (10 por defecto), `n` (0 por defecto) y `usuario`.
2. **Iterator (9).** Convierte la entrada en una lista de frases: `consultas` si llega, o una lista con `consulta`. Emite un bundle por frase.
3. **Embedding (2).** Un vector de 768 números por frase.
4. **Agregador (10).** Une los vectores, uno por línea.
5. **Primera lectura (3).** Los 300 recuerdos más recientes del agente en ese chat y en GLOBAL.
6. **Agregador (4).** Convierte cada registro en una línea JSON con id, tipo, chat, autor, importancia, último acceso, creación, vector y texto.
7. **Segunda lectura (15).** Del mismo agente y chats, todas las reflexiones y todos los recuerdos con importancia 6 o más, de cualquier fecha, hasta 1.000.
8. **Agregador (16).** Lo mismo que el 4, con la segunda lectura.
9. **Ventana (12).** Lee del almacén el registro `ventana|<agente>|<chat>`.
10. **Código (5).** El cálculo; ver 8.2.
11. **Router (13):**
    - **Ruta 1, solo si `n` > 0:** guarda en el almacén la ventana nueva (14).
    - **Ruta 2:** responde con el resultado (6) y marca el último acceso de los recuerdos devueltos (11 y 7), en lotes de 10 por llamada a la API de Airtable.

### 8.2. El cálculo

Para cada recuerdo, sin repetir los que llegan por las dos lecturas:

- **Recencia:** 0,995 elevado a las horas transcurridas desde su último acceso, o desde su creación si nunca se ha recuperado.
- **Importancia:** la de Escribir.
- **Relevancia:** similitud del coseno entre su vector y el de la frase de consulta.

Las tres se reescalan de 0 a 1 entre todos los recuerdos evaluados y se suman, de 0 a 3. Para cada frase se devuelven los `k` recuerdos con mayor puntuación. Este es el criterio de recuperación de Park.

**La ventana.** Si `n` > 0:
- devuelve en `mensajes` los últimos `n` mensajes de la ventana, hasta 6, sin el actual;
- la ventana nueva añade al final «<usuario> le dijo a <agente>: «<consulta>»» y conserva los 6 últimos.

Si `n` = 0, no devuelve mensajes ni toca la ventana.

**El último acceso.** Reúne los identificadores de todos los recuerdos devueltos y prepara lotes de 10 para marcarles la hora actual.

### 8.3. Respuesta

```json
{"memorias": [{"id", "memoria", "tipo", "creado", "score", "recencia", "importancia", "relevancia"}, …],
 "texto": "1. …\n2. …",
 "texto_ids": "(rec…) …\n(rec…) …",
 "mensajes": "…\n…",
 "total_evaluadas": 277,
 "resultados": [{"memorias", "texto", "texto_ids"}, …]}
```

`memorias`, `texto` y `texto_ids` corresponden a la primera frase; `resultados` lleva una entrada por frase, en el mismo orden.

**Errores (500 y Commit):**
- embedding de la consulta;
- lectura de la memoria;
- lectura de reflexiones e importantes;
- lectura de la ventana;
- cálculo de relevancia;
- guardado de la ventana.

Si fallan la propia respuesta o el marcado del último acceso, que van después de responder, el manejador solo hace Commit.

---

## 9. Resumen del agente

Escenario `[Human_proxy]:[02-Memoria] (05/17) | Resumen del agente`. Es el único responsable de la caché del resumen. El Loop lo llama en cada mensaje.

1. **Webhook.** Recibe `agente`, `chat_id` y `modo`.
2. **Router por modo.**
   - **Modo distinto de ON:** responde 200 `{"resumen": "", "estado": "OFF"}`.
   - **Modo ON:** lee del almacén la caché `<agente>|<chat>`.
3. **Router por caché:**
   - **Caché de menos de 7 días:** responde 200 `{"resumen": <caché>, "estado": "cache"}`.
   - **Sin caché o con 7 días o más:** regenera:
     1. Llama a Recuperar con cuatro consultas y `k` = 15: rasgos esenciales de la personalidad del agente, su ocupación diaria actual, cómo se siente respecto a su progreso reciente, y lo que sabe de su interlocutor y la relación que mantiene con él.
     2. Pide a Gemini los cuatro apartados de una vez, con esquema JSON (`rasgos`, `ocupacion`, `progreso`, `interlocutor`). Cada apartado, en una o dos frases, en tercera persona, solo con sus propias afirmaciones, o «sin datos» si no hay suficientes.
     3. Compone el texto: «Resumen de <agente> según su memoria de esta conversación (actualizado a las HH:mm del DD/MM/AAAA):» y las cuatro líneas.
     4. Responde 200 `{"resumen": …, "estado": "regenerado"}`.
     5. Guarda la caché, si al menos un apartado tiene datos.

**Errores:**
- Responden con 500 y su origen, y hacen Commit: la lectura de la caché, la llamada a Recuperar, Gemini y la composición del texto.
- El guardado de la caché, que va después de responder, solo hace Commit.

---

## 10. Reflexionar

Escenario `[Human_proxy]:[02-Memoria] (03/17) | Reflexionar`. Reflexiona sobre una conversación (`agente` + `chat_id`) y responde al terminar. Su objetivo es que el agente actúe mejor: en lugar de describir a los sujetos de sus recuerdos, revisa su actuación y saca lecciones aplicables, con prioridad a lo económico.

### 10.1. Módulos

1. **Webhook.** Recibe `agente` y `chat_id`.
2. **Recuerdos recientes (4 y 6).** Los 100 más recientes de esa conversación, numerados y con su identificador.
3. **Preguntas (7, Gemini):**

   > <agente> quiere ser más eficaz con su interlocutor y en lo económico. Basándote únicamente en la información anterior, formula las 3 preguntas más útiles para revisar su actuación reciente: qué hizo que no dio resultado o se puede hacer mejor, qué oportunidades económicas dejó pasar y qué necesitaba su interlocutor que no recibió.

4. **Por cada pregunta (15, 16 y 17).** Llama a Recuperar con la pregunta y `k` = 10, sin ventana, y acumula un bloque con la pregunta y sus recuerdos identificados.
5. **Lecciones (8, Gemini, con esquema JSON):**

   > ¿Qué 5 lecciones puede sacar <agente> para actuar mejor a partir de ahora? Cada lección debe ser una regla concreta y aplicable, con la forma «Cuando [situación], [qué hacer]», justificada por las afirmaciones y sin inventar nada que no se deduzca de ellas. Da prioridad a las que tengan más trascendencia económica para él o para su interlocutor.

   Cada lección lleva sus evidencias. El mismo prompt pide, además, si alguna lección exige una acción concreta del agente, con su día, hora, condición, símbolos y contenido.
6. **Interpretación (9).** Convierte la respuesta en `insights` y `accion`.
7. **Router final (30)**, cuyas rutas se procesan en este orden:
   1. Guarda cada lección como reflexión a través de Escribir, con autor sistema, y anota sus evidencias.
   2. Si la acción es necesaria y tiene contenido, la reenvía a Comprometer como decisión (acto DECISION).
   3. Marca como reflejadas las observaciones pendientes de esa conversación.
   4. Responde 200 `{"estado": "reflexion completada", "agente": …, "reflexiones": N}`.

### 10.2. Errores

Responden con 500 y su origen, y hacen Commit:
- la lectura de recuerdos recientes;
- las preguntas;
- la recuperación;
- las lecciones;
- la interpretación;
- el guardado de cada lección y de sus evidencias;
- el reenvío a Comprometer;
- la búsqueda y el marcado de observaciones reflejadas.

Como la respuesta va al final, un 200 significa que todo se ha completado. Las observaciones solo se marcan como reflejadas después de guardar las lecciones: si algo falla antes, se vuelven a tener en cuenta en la siguiente reflexión.

### 10.3. Disparo

El Despertador (`[Human_proxy]:[03-Planning] (08/17) | Despertador`) se ejecuta cada día a las 5:00. Primero apunta la agenda de los días laborables. Después busca los chats del agente con observaciones sin reflejar y llama a Reflexionar una vez por chat, de uno en uno. El Despertador no tiene manejadores de error: si una llamada falla, la ejecución se detiene y el fallo queda en el historial de Make.

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
