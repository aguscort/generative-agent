# Human_proxy · Documentación técnica

Human_proxy implementa en Make.com la arquitectura de *Generative Agents: Interactive Simulacra of Human Behavior* (Park et al., 2023, [arXiv 2304.03442](https://arxiv.org/abs/2304.03442)) para agentes que conversan por Telegram. La iniciativa de los agentes sigue la programación orientada a agentes de Shoham (*Agent-oriented programming*, Artificial Intelligence 60, 1993).

| Bloque | Escenarios | Estado |
| --- | --- | --- |
| Memoria y diálogo (Park) | Gateway, Escribir, Reflexionar, Recuperar, Responder, Resumen del agente | Activos |
| Compromisos (Shoham) | Comprometer, Cumplir, Despertador. Percibir es un paso de Responder | Activos |
| Asesor financiero | Estudiar, Interpelar, Noticias, Cotizar. La cartera y la calculadora están en Responder | Activos |
| Ejecución por pasos | Actuar, Observar, Refinar | Creados y desactivados ([sección 18](#18-ejecución-en-desarrollo-15-actuar-16-observar-y-17-refinar)) |
| Obsoletos | Planificar y la versión anterior de Percibir | Desactivados, con el prefijo «[OBSOLETO]» |

## 1. Visión general

El paper describe un agente con capacidades encadenadas sobre un único registro de memoria, el *memory stream*. Percibe y almacena cada experiencia como una observación en lenguaje natural, con su marca temporal y una puntuación de importancia. Recupera los recuerdos pertinentes combinando lo reciente, lo importante y lo relevante. Reflexiona periódicamente, sintetizando observaciones en conclusiones que vuelven al mismo registro. Y planifica su día, reaccionando a lo que percibe.

Human_proxy conserva la memoria, la recuperación y la reflexión del paper, pero no su planificación. La iniciativa sale del modelo de Shoham: el estado mental del agente son sus creencias, sus capacidades y sus compromisos, y una decisión es un compromiso consigo mismo.

Un compromiso nace de tres sitios:

1. **Lo que pide el usuario.** Antes de contestar, Responder clasifica el mensaje como petición (REQUEST), liberación (UNREQUEST) o ninguna de las dos. Comprometer aplica las reglas y decide.
2. **Las reflexiones del agente.** Si una conclusión exige actuar, Reflexionar envía a Comprometer una decisión (DECISION).
3. **Reglas fijas.** El Despertador crea cada laborable la orden de hacer la agenda, y la agenda crea una orden de estudiar por evento.

Cada compromiso queda en el almacén Compromisos con su momento y su estado. Cumplir lo revisa cada media hora y ejecuta los que han vencido. Nada se borra: un compromiso pasa a cumplido, fallido, liberado o caducado (sección 17).

El repositorio incluye un agente de ejemplo, Borja: un asesor financiero ficticio de una firma de hedge funds del Madrid de los 90. Su personalidad está en un único sitio, su descripción, guardada en un almacén de datos. Lo propio de su oficio está en la acción «agendar», en Noticias, en Cotizar y en las rutas de cartera, cotizaciones y cálculo de Responder. Un agente de otro ámbito solo necesita su descripción: no tendrá agenda financiera, pero conserva la memoria, la reflexión y los compromisos.

| Escenario | Disparo | Función |
| --- | --- | --- |
| (1) Gateway Telegram | Mensaje en Telegram | Aplica el límite de frecuencia, pide la respuesta, la envía, la memoriza y lanza Estudiar |
| (2) Memoria: Escribir | Webhook | Puntúa, vectoriza y guarda un recuerdo; dispara la reflexión si toca |
| (3) Memoria: Reflexionar | Webhook | Genera preguntas focales, recupera evidencias y guarda 5 reflexiones; si una exige actuar, la envía a Comprometer |
| (4) Memoria: Recuperar | Webhook | Devuelve los k recuerdos con mejor puntuación para una consulta |
| (6) Responder | Webhook | Percibe el mensaje, pide a Comprometer la decisión si hay una petición, y genera la respuesta; registra la cartera, consulta cotizaciones y calcula cuando hace falta |
| (7) Resumen del agente | Webhook | Resume quién es el agente según su memoria, con caché de 7 días |
| (8) Estudiar | Webhook | Detecta preguntas profesionales, investiga en Google y guarda conocimiento |
| (9) Interpelar | Webhook | Escribe al usuario por iniciativa propia cuando le falta un dato |
| (10) Despertador | Diario, 5:00 | Los laborables deja la orden «agendar» para las 8:00 |
| (11) Cumplir | Cada 30 minutos, laborables de 8:00 a 20:30 | Ejecuta los compromisos vencidos y marca su resultado |
| (12) Noticias | Laborables, 8:00 y 16:00 | Guarda los titulares que afectan a la cartera de cada usuario |
| (13) Cotizar | Webhook | Devuelve precio, cierre anterior, moneda y hora de hasta 5 valores |
| (14) Comprometer | Webhook | Aplica las reglas de compromiso y responde con la decisión |
| (15) Actuar | Webhook · desactivado | *En desarrollo.* Da un paso de un compromiso con una herramienta |
| (16) Observar | Webhook · desactivado | *En desarrollo.* Comprueba el paso y decide si el compromiso se ha cumplido |
| (17) Refinar | Webhook · desactivado | *En desarrollo.* Corrige, aplaza o da por fallido un compromiso que no sale |

Los números son los de esta documentación. En Make los escenarios llevan otra numeración y una etiqueta de grupo:

| Documentación | Nombre en Make |
| --- | --- |
| (1) Gateway | [Human_proxy]:[Gateway] (01/13) \| Telegram |
| (2) Escribir | [Human_proxy]:[Memoria] (02/13) \| Escribir |
| (3) Reflexionar | [Human_proxy]:[Memoria] (03/13) \| Reflexionar |
| (4) Recuperar | [Human_proxy]:[Memoria] (04/13) \| Recuperar |
| (14) Comprometer | [Human_proxy]:[Planning] (05/13) \| Comprometer |
| (6) Responder | [Human_proxy]:[Skills] (06/13) \| Responder |
| (11) Cumplir | [Human_proxy]:[Planning] (07/13) \| Cumplir: ejecución de compromisos |
| (8) Estudiar | [Human_proxy]:[Skills] (08/13) \| Estudiar |
| (9) Interpelar | [Human_proxy]:[Skills] (09/13) \| Interpelar |
| (7) Resumen del agente | [Human_proxy]:[Self] (10/13) \| Resumen del agente |
| (10) Despertador | Despertador (11/13) |
| (12) Noticias | [Human_proxy]:[Skills] (12/13) \| Noticias |
| (13) Cotizar | [Human_proxy]:[Skills] (13/13) \| Cotizar |
| (15) Actuar | [Human_proxy]:[Ejecución] (14/16) \| Actuar |
| (16) Observar | [Human_proxy]:[Ejecución] (15/16) \| Observar |
| (17) Refinar | [Human_proxy]:[Ejecución] (16/16) \| Refinar |

Los modelos son Gemini 3.8 Flash para el lenguaje natural y Gemini Embedding 2, con 768 dimensiones, para los vectores. Las cuentas las ejecuta el módulo Make Code, no el modelo. La memoria vive en Airtable; la cartera, las cachés y los compromisos, en almacenes de datos de Make; las cotizaciones salen de Yahoo Finance.

## 2. Arquitectura

Cada escenario es una función: un webhook de entrada, un cuerpo JSON con sus parámetros y una respuesta, o un «Accepted» inmediato si trabaja en segundo plano. Solo el Gateway recibe de Telegram. Solo Escribir crea recuerdos y solo Recuperar hace la búsqueda puntuada.

Todas las llamadas entre escenarios llevan un campo `token` en el cuerpo. Cada escenario con webhook lo comprueba en un filtro, «Token correcto», justo después del webhook, y no hace nada si no coincide (sección 22).

Un mensaje recorre el sistema así:

1. El Gateway lee el perfil del agente, comprueba el límite de frecuencia y llama a Responder.
2. Responder pide el resumen del agente, lee lo que sabe del interlocutor, sus compromisos de hoy, los recuerdos relevantes, el historial y la cartera.
3. Responder percibe el mensaje. Si es una petición o una liberación, llama a Comprometer, que responde con la decisión.
4. Si el mensaje comunica posiciones, las registra; si pregunta por precios, llama a Cotizar; si pide una cuenta, la calcula.
5. Responder genera la respuesta conociendo la decisión: Borja no promete nada que no esté comprometido.
6. El Gateway envía la respuesta, guarda el mensaje y la respuesta como dos observaciones y pasa el mensaje a Estudiar.
7. Cada escritura suma la importancia de las observaciones no reflejadas y, por encima de 150, lanza Reflexionar.

```mermaid
flowchart LR
    TG([Telegram · mensaje]) --> GW["(1) Gateway"]
    GW --> RS["(6) Responder<br/>percibe y responde"]
    RS --> CMP["(14) Comprometer<br/>reglas"]
    RS --> COT["(13) Cotizar"]
    RS --> RES["(7) Resumen"]
    RS --> REC["(4) Recuperar"]
    GW --> ESC["(2) Escribir"]
    GW --> EST["(8) Estudiar"]
    ESC --> REF["(3) Reflexionar"]
    REF --> REC
    REF -- reflexiones --> ESC
    REF -- decisiones --> CMP
    DSP["(10) Despertador<br/>5:00"] -. agendar .-> CS
    CMP -.-> CS[("Compromisos")]
    CS -.-> CUM["(11) Cumplir<br/>cada 30 min"]
    CUM --> COT
    CUM --> EST
    CUM --> INT["(9) Interpelar"]
    CUM --> ESC
    CUM --> TGO([Telegram · mensaje propio])
    EST --> INT
    EST --> REC
    EST --> ESC
    INT --> ESC
    INT --> TGO
    RES --> REC
    NOT["(12) Noticias"] --> ESC
    COT -.-> YF([Yahoo Finance])
    ESC -.-> AT[(Airtable<br/>Memory Stream)]
    REC -.-> AT
```

Las flechas continuas son llamadas a webhook; las discontinuas, accesos a datos o a fuentes externas. Compromisos lo escriben Comprometer, el Despertador y Cumplir, y lo leen Comprometer, Cumplir y Responder.

Sin que nadie escriba, el Despertador deja a las 5:00 la orden de agendar, Cumplir revisa los compromisos cada media hora y Noticias lee la prensa dos veces al día.

| Escenario | Respuesta al llamante | Procesamiento |
| --- | --- | --- |
| (1) Gateway | No aplica | Secuencial, para respetar el orden de los mensajes |
| (2) Escribir | Síncrona, en cuanto el registro existe | Paralelo |
| (3) Reflexionar | 202 inmediato | Paralelo |
| (4) Recuperar | Síncrona; actualiza accesos después | Paralelo |
| (6) Responder | Síncrona | Paralelo |
| (7) Resumen del agente | Síncrona; regenera después | Paralelo |
| (8) Estudiar | «Accepted» inmediato | Paralelo |
| (9) Interpelar | «Accepted» inmediato | Paralelo |
| (10) Despertador | No aplica | Paralelo |
| (11) Cumplir | No aplica | Paralelo |
| (12) Noticias | No aplica | Paralelo |
| (13) Cotizar | Síncrona | Paralelo |
| (14) Comprometer | Síncrona: la decisión en texto | Paralelo |
| (15) Actuar · (16) Observar · (17) Refinar | «Accepted» inmediato | Paralelo |

Si el filtro del token bloquea una llamada, Make responde igualmente 200 «Accepted». Quien llama no distingue ese caso de uno correcto.

Make no admite el módulo de respuesta de webhook en un escenario secuencial y desactiva un escenario secuencial en cuanto una ejecución falla. Por eso los escenarios que no deben pararse llevan un manejador de error Ignore en sus módulos externos.

### Ajustes de cada escenario

| Escenario | Programación | Procesamiento | Manejador Ignore |
| --- | --- | --- | --- |
| (1) Gateway | Inmediata | Secuencial | En los 11 módulos externos |
| (2) Escribir | Inmediata | Paralelo | No |
| (3) Reflexionar | Inmediata | Paralelo | En la llamada a Comprometer |
| (4) Recuperar | Inmediata | Paralelo | No |
| (6) Responder | Inmediata | Paralelo | En las rutas de percepción, cartera, cotizaciones y cálculo |
| (7) Resumen del agente | Inmediata | Paralelo | No |
| (8) Estudiar | Inmediata | Paralelo | En los 5 módulos externos |
| (9) Interpelar | Inmediata | Paralelo | En los 5 módulos externos |
| (10) Despertador | Diaria, 5:00 (Europe/Madrid) | Paralelo | No |
| (11) Cumplir | Cada 30 minutos, laborables de 8:00 a 20:30 (Europe/Madrid) | Paralelo | En todos sus módulos externos y de escritura |
| (12) Noticias | Laborables, 8:00 y 16:00 (Europe/Madrid) | Paralelo | En los 6 módulos externos |
| (13) Cotizar | Inmediata | Paralelo | En la consulta a Yahoo Finance |
| (14) Comprometer | Inmediata | Paralelo | No |
| (15) Actuar · (16) Observar · (17) Refinar | Inmediata, desactivados | Paralelo | En todos sus módulos externos y de escritura |

Todos usan el máximo de 3 errores seguidos de Make antes de desactivarse y confirman cada operación al terminarla (auto-commit).

Cada escenario lleva notas de Make en sus webhooks y en sus llamadas, con el origen y el destino:

| Webhook de destino | Lo llaman (escenario y módulo) |
| --- | --- |
| (2) Escribir | Gateway 4 y 5, Reflexionar 11, Estudiar 6, Interpelar 6, Cumplir 10, 45, 55, 60 y 64, Noticias 11 |
| (3) Reflexionar | Escribir 8 |
| (4) Recuperar | Reflexionar 16, Responder 2 y 10, Resumen 10, 12 y 14, Estudiar 4 |
| (6) Responder | Gateway 2 |
| (7) Resumen del agente | Responder 9 |
| (8) Estudiar | Gateway 15, Cumplir 62 |
| (9) Interpelar | Estudiar 7, Cumplir 11 y 61 |
| (13) Cotizar | Responder 23, Cumplir 50 y 63 |
| (14) Comprometer | Responder 51, Reflexionar 31 |
| (15) Actuar | Observar 12, Refinar 9 |
| (16) Observar | Actuar 34 |
| (17) Refinar | Observar 6, 13 y 16 |

Responder 51 y Reflexionar 31 no llevan el token escrito: reenvían el que recibió su propio webhook, `{{1.token}}`. Actuar, Observar y Refinar hacen lo mismo en todas sus llamadas.

> [!NOTE]
> Actuar llama además a Cotizar, Recuperar, Interpelar y Escribir, y Observar a Escribir. No figuran en la tabla porque los tres están desactivados.

## 3. Modelo de datos

### La tabla Memory Stream

Toda la memoria vive en una sola tabla de Airtable, igual que en el paper, donde observaciones, reflexiones y planes son objetos del mismo registro y se recuperan con el mismo mecanismo. Cada fila es un recuerdo y el campo Tipo distingue su naturaleza.

| Campo | Tipo Airtable | Quién lo escribe | Significado |
| --- | --- | --- | --- |
| Memoria | Texto largo (campo primario) | Escribir | Descripción en lenguaje natural del recuerdo |
| Tipo | Selección única: observacion, reflexion, conocimiento; plan en registros antiguos | Escribir | Naturaleza del recuerdo |
| Agente | Texto de una línea | Escribir | Agente propietario del recuerdo |
| Chat_ID | Texto de una línea | Escribir | Chat de Telegram; segmenta la memoria por conversación. Lo que no es de una conversación, como la agenda del día, usa `GLOBAL` |
| Autor | Selección única: usuario, agente, sistema | Escribir | Quién originó el contenido |
| Importancia | Número | Escribir | Intensidad emocional de 1 a 10 asignada por el LLM |
| Creado | Fecha de creación (automático) | Airtable | Marca temporal de creación |
| Ultimo_acceso | Fecha con hora | Escribir y Recuperar | Última vez que el recuerdo se creó o se recuperó; base de la recencia |
| Embedding | Texto largo | Escribir | Vector de 768 dimensiones serializado como `[n1, n2, …]` |
| Evidencias | Vínculo a registros de la misma tabla | Reflexionar | Recuerdos que justifican una reflexión |
| Citada_por | Vínculo recíproco de Evidencias | Airtable | Reflexiones que citan este recuerdo |
| Reflejada | Casilla | Reflexionar | La observación ya se ha contado para una reflexión |
| Plan_ubicacion | Texto de una línea | Solo registros antiguos de Planificar | Lugar del bloque de plan |
| Plan_inicio | Fecha con hora | Solo registros antiguos de Planificar | Inicio del bloque |
| Plan_duracion_min | Número | Solo registros antiguos de Planificar | Duración en minutos |

Tres decisiones conviene tenerlas presentes. El paper guarda para cada objeto la descripción, la marca de creación y la de último acceso; aquí son Memoria, Creado y Ultimo_acceso, y el resto son añadidos para operar con varios agentes y conversaciones. El embedding se guarda como texto porque Airtable no tiene un tipo vectorial; ocupa unos 8–10 KB por fila y es la razón de que la recuperación tenga un techo de 300 registros. Evidencias es un vínculo real entre filas, así que el árbol de reflexiones del paper se puede navegar desde Airtable siguiendo Citada_por.

Reflejada no está en el paper. Allí la reflexión se dispara cuando «la suma de importancia de los eventos más recientes» supera un umbral; sin un estado explícito, cada observación nueva volvería a sumar las anteriores y el umbral se superaría en todos los mensajes a partir del primero que lo cruza. Reflejada funciona como un contador que se pone a cero.

Escribir crea los registros con la conversión de tipos de Airtable (typecast) activada, para que acepte valores nuevos en Tipo e intente convertir a número una importancia que llegue como texto.

Algunos textos siguen una convención porque otros escenarios buscan por ellos. Las observaciones que nacen de Interpelar empiezan por «{agente} le planteó una duda a». La agenda que guarda Cumplir empieza por «Agenda financiera de {agente} para hoy» y lo que escribe a un usuario por iniciativa propia, por «{agente} le escribió por iniciativa propia». Las lecturas de prensa de Noticias empiezan por «{agente} leyó en prensa».

Los registros de tipo `plan` que dejó Planificar, y los de tipo `evento` de versiones anteriores, siguen en la tabla, pero ningún escenario los busca. Los compromisos no viven aquí sino en el almacén Compromisos: son el trabajo del agente, no su memoria. Lo que el agente hace o aprende al cumplirlos sí entra en la memoria, a través de Escribir.

### Almacenes de datos de Make

| Almacén | Clave | Contenido |
| --- | --- | --- |
| Caché (campos agente, chat_id, resumen, actualizado) | `perfil\|<agente>` | Descripción del personaje, en el campo resumen, con chat_id `GLOBAL` |
| | `<agente>\|<chat_id>` | Resumen del agente en ese chat (escenario 7) |
| | `rel\|<agente>\|<chat_id>` | Lo que el agente sabe de su interlocutor (escenario 6) |
| Cartera (campos agente, chat_id, cartera, posiciones, actualizado) | `<agente>\|<chat_id>` | En `cartera`, un resumen en texto de las posiciones, los intereses y el horizonte del usuario; en `posiciones`, la lista estructurada en JSON. La escribe Responder y la leen Responder, Cumplir y Noticias |
| Compromisos (campos agente, comprometido_con, destinatario, accion, momento, condicion, simbolos, contenido, linea, origen, intentos, nivel, creado, estado, fin) | La genera Make; `agendar\|<agente>\|<fecha>` para la orden del Despertador | Un compromiso por registro, con su momento y su estado (sección 17) |

El perfil es la única fuente de la personalidad. Lo leen el Gateway, el Despertador y Cumplir, y el Gateway lo pasa a Responder y Estudiar. El límite de longitud de las respuestas no está en el perfil, porque solo aplica a Telegram: el Gateway añade «Limita la respuesta a una o dos frases, sin pasar de 20 palabras» únicamente en la llamada a Responder.

El campo `posiciones` es un texto con una lista JSON, un objeto por valor:

```json
[{"simbolo": "ITX.MC", "nombre": "Inditex", "cantidad": 50, "precio_medio": 45, "moneda": "EUR"}]
```

El símbolo es el de Yahoo Finance, para que la calculadora pueda cruzarlo con las cotizaciones. Una cantidad o un precio medio de 0 significan que el usuario no los dio.

El módulo Data store · Get a record exige `returnWrapped: false` en el blueprint aunque la interfaz no lo muestre; sin él falla con `BundleValidationError`.

## 4. (1) Gateway Telegram

El Gateway es el único escenario que conoce Telegram; los demás son funciones independientes del canal. Responde primero y memoriza después, para que el usuario reciba la respuesta cuanto antes y para que el mensaje actual no compita consigo mismo en la recuperación.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Telegram Bot · Watch Updates | Recibe los mensajes del bot |
| filtro | Mensaje de texto (no comandos) | Solo pasa si `message.text` existe y no empieza por «/» |
| 6 | Airtable · Search records | Mensajes del usuario en este chat en los últimos 10 minutos, hasta 10 |
| 7 | Tools · Numeric aggregator | Los cuenta con `if(6.id; 1; 0)`, para que una búsqueda vacía cuente 0 |
| 8 | Router | Ruta 1 si hay menos de 10 mensajes; ruta 2 si hay 10 o más |
| R2 · 10 | Telegram Bot · Send a Text Message | Responde «Descansamos un poco. Inténtalo en un par de minutos» y no sigue |
| R1 · 960 | Data store · Get a record | Lee `perfil\|<agente>` |
| R1 · 2 | HTTP · POST a Responder | agente, `modo` (ON/OFF del resumen), chat_id, mensaje, usuario y descripción |
| R1 · 957 | Slack · Create a message | Registro opcional de cada intercambio en un canal |
| R1 · 3 | Telegram Bot · Send a Text Message | Envía la respuesta al chat |
| R1 · 4 | HTTP · POST a Escribir | Guarda «{nombre} le dijo a {agente}: «{mensaje}»», autor usuario |
| R1 · 5 | HTTP · POST a Escribir | Guarda «{agente} le respondió a {nombre}: «{respuesta}»», autor agente |
| R1 · 15 | HTTP · POST a Estudiar | Pasa el mensaje para que decida si investigar; no espera |

Todas las llamadas HTTP llevan el campo `token` al principio del cuerpo.

Todos los módulos que pueden fallar llevan un manejador Ignore: si uno falla, esa ejecución termina y el Gateway sigue activo para el siguiente mensaje. Sin ellos, Make desactivaría el escenario al primer error porque es secuencial.

El límite de frecuencia protege el consumo frente a ráfagas. Cuenta solo los mensajes del usuario ya guardados; como el mensaje en curso se guarda después de responder, no se cuenta a sí mismo. La condición de fecha, `IS_AFTER({Creado}, DATEADD(NOW(), -10, 'minutes'))`, compara instantes absolutos y no depende de la zona horaria.

La búsqueda de Airtable en Make devuelve un paquete vacío (`__IMTLENGTH__` = 0) cuando no encuentra nada. Por eso los contadores suman `if(id; 1; 0)` en lugar de 1.

Las observaciones se redactan en tercera persona y con el nombre del interlocutor porque el paper guarda las percepciones como frases narrativas («Isabella Rodriguez is setting out the pastries»). Así el texto que luego lee el LLM es autoexplicativo.

El procesamiento secuencial hace que tres mensajes seguidos se ejecuten uno detrás de otro. Sin él, el segundo podría recuperar memoria antes de que se guardara el primero. El coste es la latencia: cada mensaje espera al anterior, unos 10–20 segundos.

Todos los textos que entran en un cuerpo JSON pasan por `escapeJSON()`, para que unas comillas o un salto de línea no rompan la llamada. Telegram solo admite un webhook por bot: si el bot ya lo usaba otro escenario, este deja de recibir al crear el del Gateway.

Para tener varios agentes basta con duplicar el Gateway y cambiar el nombre del agente, la clave del perfil y la conexión de Telegram: la memoria ya está separada por el campo Agente.

## 5. (2) Memoria: Escribir

Escribir es la única puerta de entrada a la memoria: toda observación, reflexión o pieza de conocimiento pasa por aquí y queda puntuada y vectorizada con el mismo criterio. Responde en cuanto el registro existe y después comprueba si hay que reflexionar.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Gemini · Generate a response | Filtro «Token correcto»; puntúa la importancia del recuerdo de 1 a 10 |
| 3 | Gemini · Generate embeddings | Vector de la memoria, 768 dimensiones |
| 4 | Airtable · Create a record | Crea la fila con typecast, Importancia, Ultimo_acceso = ahora y Embedding |
| 5 | Webhooks · Webhook response | 200 con `{"id", "importancia", "tipo"}` |
| 9 | Router | Una ruta: la de la reflexión |
| 6 | Airtable · Search records | Observaciones del mismo agente y chat sin reflejar (hasta 500) |
| 7 | Tools · Numeric aggregator | Suma su importancia |
| 8 | HTTP · POST a Reflexionar | Solo si la suma supera 150 y el recuerdo nuevo es una observación |

Hasta noviembre de 2026, Escribir también detectaba promesas en cada recuerdo y las enviaba a Comprometer. Esa regla desapareció: veía solo la frase del agente, sin el mensaje que la motivó, y reaccionaba también a recuerdos del sistema. Ahora los compromisos se deciden donde está el contexto: en Responder y en Reflexionar.

### Importancia

El paper pide al LLM que puntúe la intensidad del recuerdo con una escala anclada en dos ejemplos extremos. El prompt es su traducción:

```text
En una escala del 1 al 10, donde 1 es puramente mundano (p. ej., cepillarse los dientes, hacer la cama) y 10 es extremadamente conmovedor (p. ej., una ruptura, ser admitido en la universidad), puntúa la probable intensidad emocional del siguiente recuerdo de {{1.agente}}.
Responde únicamente con un número entero del 1 al 10.
Recuerdo (autor: {{1.autor}}): {{1.memoria}}
Puntuación:
```

El módulo 4 guarda como importancia la primera línea de la respuesta, `trim(first(split(trim(2.result); newline)))`. La importancia se calcula una vez y no cambia: por eso un acontecimiento importante sigue apareciendo semanas después aunque su recencia haya caído.

### Embedding

El vector se guarda como texto entre corchetes. Su dimensión debe coincidir con la de la consulta en Recuperar (768 en ambos). Si se cambia el modelo o la dimensión en uno de los dos, hay que cambiarla en el otro y recalcular los embeddings existentes: la similitud entre vectores de distinta longitud se trata como 0.

### Disparo de la reflexión

El filtro del módulo 8 exige dos condiciones: suma mayor que 150 y tipo observacion. La segunda evita un bucle, porque las reflexiones también se guardan llamando a Escribir. Con mensajes de chat corrientes, que el modelo suele puntuar entre 2 y 4, el umbral se alcanza tras unas 40–60 observaciones, es decir, 20–30 intercambios.

### Entrada y salida

Entrada: `token`, `memoria` (obligatorio), `tipo`, `agente`, `chat_id`, `autor` y, opcionales, `evidencias` (IDs de registro) y `nivel`, que se acepta pero ya no se usa. Los campos `plan_ubicacion`, `plan_inicio` y `plan_duracion_min` se siguen aceptando, aunque ya nadie los envía. Salida: `{"id": "rec…", "importancia": "4", "tipo": "observacion"}`.

## 6. (3) Memoria: Reflexionar

Reflexionar convierte una acumulación de observaciones en cinco conclusiones de nivel superior, cada una vinculada a los recuerdos que la justifican, con los tres pasos del paper: preguntas focales, recuperación de evidencias e inferencia de insights. Lo llama Escribir al cruzar el umbral; también se puede llamar a mano con `{"token", "agente", "chat_id"}`. Además, se pregunta si alguna conclusión exige actuar y, si es así, se compromete consigo mismo.

### Respuesta inmediata y router

El módulo 2 lleva el filtro «Token correcto» y responde 202 nada más recibir la llamada, porque la reflexión tarda alrededor de un minuto y Escribir no debe esperarla. Un router reparte después la ejecución en dos rutas que Make recorre en orden.

La ruta 1 (módulos 12 y 13) marca como reflejadas las observaciones pendientes. Va primero a propósito: si durante la reflexión entran mensajes nuevos, Escribir no volverá a encontrar las mismas observaciones por encima de 150 y no lanzará una reflexión duplicada.

### Ruta 2: la reflexión

| # | Módulo | Qué hace |
| --- | --- | --- |
| 4 | Airtable · Search records | Los 100 registros más recientes del agente y chat, de todos los tipos |
| 6 | Tools · Text aggregator | Una línea por registro: `n. (recID) texto` |
| 7 | Gemini | Genera las 3 preguntas de alto nivel, una por línea |
| 15 | Iterator | Recorre las preguntas |
| 16 | HTTP · POST a Recuperar | Cada pregunta como consulta, k = 10 |
| 17 | Tools · Text aggregator | Reúne por pregunta las memorias recuperadas con sus IDs |
| 8 | Gemini (salida JSON con esquema) | Infiere 5 insights, cada uno con su lista de evidencias |
| 9 | JSON · Parse JSON | Convierte la respuesta |
| 30 | Router | Dos rutas: las conclusiones y la acción |
| R1 · 10 | Iterator | Recorre los insights |
| R1 · 11 | HTTP · POST a Escribir | Guarda el insight como reflexion, autor sistema |
| R1 · 14 | Airtable · Update a record | Rellena Evidencias con los IDs citados |
| R2 · 31 | HTTP · POST a Comprometer | Solo si la reflexión dice que hace falta actuar: envía la acción como DECISION |

El prompt de preguntas focales traduce el del paper:

```text
Recuerdos recientes de {{1.agente}}:
{{6.text}}

Basándote únicamente en la información anterior, ¿cuáles son las 3 preguntas de alto nivel más relevantes que podemos responder sobre los sujetos de estas afirmaciones?
Responde solo con las 3 preguntas, una por línea, sin numeración.
```

Y el de insights, ampliado para obtener las evidencias en un formato que Make pueda procesar:

```text
Para cada pregunta de alto nivel, estas son las afirmaciones relevantes que recuerda {{1.agente}} (cada una con su identificador entre paréntesis):
{{17.text}}

¿Qué 5 conclusiones de alto nivel puedes inferir de las afirmaciones anteriores? (formato de ejemplo: conclusión (por 1, 5, 3))
Devuelve exclusivamente un JSON con esta forma: {"insights": [{"insight": "texto de la conclusión", "evidencias": ["recXXXXXXXXXXXXXX"]}]}. En "evidencias" pon los identificadores (los que van entre paréntesis, empiezan por rec) de las afirmaciones que justifican cada conclusión, copiados literalmente.
```

Al final del prompt de insights se añade la conciencia de acción:

```text
Después, piensa como {{1.agente}}: ¿alguna de estas conclusiones le hace ver que tiene que hacer algo concreto que sabe hacer? Sus capacidades son informar (escribir por iniciativa propia a su interlocutor), preguntar (hacerle una pregunta), investigar (buscar información sobre un tema), cotizar (consultar precios de valores para sí mismo) y estudiar (analizar un evento o un dato económico cuando se publique). Si alguna lo exige, describe en "accion" la más necesaria, solo una y de un solo paso: necesaria true; accion; dia (AAAA-MM-DD) y hora (HH:mm) en hora de Madrid, vacíos si es para ahora; condicion, la condición de la que depende o vacía; simbolos, símbolos de Yahoo Finance separados por comas o vacío; y contenido, qué hará, en una frase con los datos concretos. Si ninguna conclusión exige actuar, pon necesaria false y deja vacíos los demás campos; lo normal es que no haga falta actuar. Ahora es {{formatDate(now; "YYYY-MM-DD [a las] HH:mm"; "Europe/Madrid")}}, hora de Madrid.
```

El módulo 8 fija `responseMimeType: application/json` y un `responseSchema` con dos claves obligatorias: `insights` y `accion`, un objeto con `necesaria`, `accion`, `dia`, `hora`, `condicion`, `simbolos` y `contenido`. Gemini no puede devolver prosa alrededor del JSON.

El módulo 31 pasa si `necesaria` es true y hay acción y contenido. Envía a Comprometer `acto: DECISION`, el chat de la reflexión, `origen: reflexion` y `nivel: 1`, con el token recibido. Comprometer la valida con las mismas reglas que una petición: si es informar o preguntar en un chat `GLOBAL`, por ejemplo, la rechaza por falta de destinatario. Una reflexión produce como mucho una acción.

La recuperación de evidencias reutiliza Recuperar para usar exactamente el mismo mecanismo que en la conversación, como hace el paper. De paso, los recuerdos usados como evidencia actualizan su último acceso. Como la búsqueda incluye reflexiones anteriores, una reflexión nueva puede citar a otra antigua: es el árbol de reflexiones del paper.

El insight se guarda primero llamando a Escribir, para que tenga importancia y embedding como cualquier recuerdo, y después el módulo 14 añade los vínculos. Así se evita reserializar el array de evidencias dentro de otro cuerpo JSON.

## 7. (4) Memoria: Recuperar

Recuperar recibe una consulta en lenguaje natural y devuelve los k recuerdos del agente y chat con mayor puntuación combinada de recencia, importancia y relevancia: la función de recuperación del paper con todos los pesos a 1. El cálculo se hace en un módulo de código porque Make no tiene un operador de similitud vectorial.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Gemini · Generate embeddings | Vector de la consulta, mismo modelo y dimensión que Escribir |
| 3 | Airtable · Search records | Hasta 300 registros del agente y chat, los más recientes primero |
| 4 | Tools · Text aggregator | Serializa cada registro como una línea JSON |
| 5 | Make Code · Execute code | Calcula las puntuaciones y elige el top-k |
| 6 | Webhooks · Webhook response | Devuelve el resultado |
| 7 | Iterator | Recorre los IDs elegidos |
| 8 | Airtable · Update a record | Pone Ultimo_acceso = ahora en cada recuerdo recuperado |

### La fórmula

```latex
\text{score}(m) = \alpha_{rec}\,\widehat{\text{rec}}(m) + \alpha_{imp}\,\widehat{\text{imp}}(m) + \alpha_{rel}\,\widehat{\text{rel}}(m), \qquad \alpha_{rec} = \alpha_{imp} = \alpha_{rel} = 1
```

```latex
\text{rec}(m) = 0.995^{\,h(m)}, \qquad \text{rel}(m) = \frac{q \cdot e_m}{\lVert q \rVert\,\lVert e_m \rVert}, \qquad \widehat{x} = \frac{x - \min x}{\max x - \min x}
```

h(m) son las horas desde el último acceso al recuerdo (o desde su creación si no tiene acceso), imp(m) es su importancia, q el embedding de la consulta y e_m el del recuerdo. Cada componente se normaliza min-max sobre el conjunto evaluado, para que los tres pesen igual aunque sus escalas sean distintas; si todos los valores de un componente coinciden, ese componente vale 0 y no influye.

Con un decaimiento de 0,995 por hora, un recuerdo conserva el 89 % de su recencia a las 24 horas, el 43 % a la semana y el 3 % al mes. Como se mide desde el último acceso, un recuerdo antiguo que se recupera a menudo se mantiene fresco, como en el paper. Allí las horas eran del mundo simulado; aquí son reales.

### Por qué los datos viajan como texto

El módulo 4 construye una línea JSON por registro con `escapeJSON()`. El código extrae los números de los vectores con una expresión regular en lugar de `JSON.parse`, porque Make no siempre serializa los arrays como JSON válido.

### El código del módulo 5

Entradas: `consulta_emb` = `[2.embedding.values]`, `registros` = `4.text`, `k` = `ifempty(1.k; 10)`, `ahora` = `now`.

```javascript
// [Human_proxy] Recuperación de memorias (Park et al., 2023, §4.1)
// score = recencia + importancia + relevancia (alfa = 1), cada componente normalizado min-max a [0,1]
const { consulta_emb, registros, k, ahora } = input;

// Los vectores llegan como texto; se extraen los números con regex
const vec = t => (String(t || '').match(/-?\d+(?:\.\d+)?(?:e[-+]?\d+)?/gi) || []).map(Number);
const q = vec(consulta_emb);
const lineas = String(registros || '').split('\n').map(l => l.trim()).filter(Boolean);
const recs = [];
for (const l of lineas) { try { recs.push(JSON.parse(l)); } catch (e) { /* línea corrupta: se ignora */ } }
const K = parseInt(k, 10) > 0 ? parseInt(k, 10) : 10;
const now = ahora ? new Date(ahora) : new Date();

const coseno = (a, b) => {
  if (!a.length || a.length !== b.length) return 0;
  let p = 0, na = 0, nb = 0;
  for (let i = 0; i < a.length; i++) { p += a[i] * b[i]; na += a[i] * a[i]; nb += b[i] * b[i]; }
  return na && nb ? p / (Math.sqrt(na) * Math.sqrt(nb)) : 0;
};

const filas = recs.map(r => {
  const horas = Math.max(0, (now - new Date(r.acc)) / 3.6e6);
  return {
    id: r.id, memoria: r.txt, tipo: r.tipo, creado: r.creado,
    importancia: Number(r.imp) || 0,
    recencia: isFinite(horas) ? Math.pow(0.995, horas) : 0,
    relevancia: coseno(q, vec(r.emb)),
  };
});

const norm = campo => {
  const v = filas.map(f => f[campo]);
  const min = Math.min(...v), max = Math.max(...v);
  filas.forEach(f => { f[campo + '_n'] = max > min ? (f[campo] - min) / (max - min) : 0; });
};
['recencia', 'importancia', 'relevancia'].forEach(norm);
filas.forEach(f => { f.score = f.recencia_n + f.importancia_n + f.relevancia_n; });
filas.sort((a, b) => b.score - a.score);

const top = filas.slice(0, K).map(f => ({
  id: f.id, memoria: f.memoria, tipo: f.tipo, creado: f.creado,
  score: +f.score.toFixed(4), recencia: +f.recencia_n.toFixed(4),
  importancia: +f.importancia_n.toFixed(4), relevancia: +f.relevancia_n.toFixed(4),
}));
const texto = top.map((m, i) => `${i + 1}. ${m.memoria}`).join('\n');
const texto_ids = top.map(m => `(${m.id}) ${m.memoria}`).join('\n');

return {
  ids: top.map(m => m.id), texto, texto_ids, total_evaluadas: filas.length,
  respuesta: JSON.stringify({ memorias: top, texto, texto_ids, total_evaluadas: filas.length }),
};
```

La salida trae el mismo resultado en tres formas: `memorias`, con las puntuaciones desglosadas, para depurar; `texto`, una lista numerada lista para un prompt (la usa Responder); y `texto_ids`, la misma lista con el ID de cada recuerdo (la usa Reflexionar para citar evidencias). Los módulos 7 y 8 se ejecutan después de responder, así que actualizar los accesos no añade latencia.

## 8. (5) Planificar · obsoleto

Planificar generaba cada día el plan del agente al estilo del paper: bloques de hasta una hora para cualquier agente y, para el agente financiero, la agenda de eventos del día en bloques de 30 minutos, guardados como recuerdos de tipo `plan`. Lo llamaban el Despertador y el Gateway, y lo leían Responder, para saber qué estaba haciendo el agente en cada momento, y la versión anterior de Percibir, que estudiaba cada hora el punto del plan que tocaba.

Desde que la iniciativa del agente pasó a los compromisos (sección 17), el escenario está desactivado y lleva el prefijo «[OBSOLETO]»; nadie lo llama. La búsqueda de la agenda financiera vive ahora en la acción «agendar» de Cumplir (sección 14), con el mismo prompt. Su rama genérica, el plan narrativo de un personaje que no es Borja («7:30 desayuna, 9:00 abre la tienda»), no tiene equivalente, porque los compromisos solo recogen acciones que el bot sabe ejecutar.

## 9. (6) Responder

Responder genera lo que dice el agente con el prompt de diálogo del paper: quién es, qué hora es, qué compromisos tiene pendientes, qué sabe de su interlocutor, qué recuerda y qué se ha dicho. Antes percibe el mensaje: si pide algo o libera al agente de algo, obtiene la decisión de Comprometer. Suma además tres capacidades de su oficio, cada una en una ruta que solo se activa cuando el mensaje la necesita: registrar la cartera, consultar cotizaciones y hacer cuentas. No escribe en la memoria de la conversación; eso lo hace el Gateway después de enviar la respuesta, para que Responder pueda usarse desde otros canales o en pruebas.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 9 | HTTP · POST a Resumen del agente | Filtro «Token correcto»; resumen del agente, vacío si el modo no es ON |
| 2 | HTTP · POST a Recuperar | El mensaje como consulta, k = 10 |
| 12 | Data store · Get a record | Lo que el agente sabe del interlocutor (`rel\|agente\|chat_id`) |
| 3 | Airtable · Search records | Las 12 observaciones más recientes del chat |
| 4 | Tools · Text aggregator | Una línea por observación: `- [autor] texto` |
| 14 | Data store · Search records | Los compromisos vigentes del agente que vencen hoy, hasta 10 |
| 15 | Tools · Text aggregator | La línea de cada uno: los propios y los de este chat, nunca los de otros chats |
| 54 | Data store · Search records | Los compromisos vigentes con este interlocutor, de cualquier fecha, hasta 20 |
| 55 | Tools · Text aggregator | Una línea por compromiso, con su clave: `clave · línea` |
| 21 | Data store · Get a record | La cartera del usuario (`agente\|chat_id` en Cartera) |
| 20 | Router | Cinco rutas, que Make ejecuta en este orden |
| R0 · 50 | Gemini (salida JSON con esquema) | Percibir: clasifica el mensaje en REQUEST, UNREQUEST o NINGUNO; extrae acción, día, hora, fin, condición, símbolos y contenido, y señala por su clave los compromisos que libera, el que repite o el veto que lo impide |
| R0 · 51 | HTTP · POST a Comprometer | Solo si es REQUEST o UNREQUEST; reenvía el token recibido |
| R0 · 52 | Tools · Set variable | `decision`: la respuesta de Comprometer convertida a texto, o el aviso de reserva |
| R1 · 30 | Gemini (salida JSON con esquema) | Solo si el mensaje habla de posiciones propias: decide si las comunica y devuelve la cartera completa resultante |
| R1 · 31 | Data store · Add a record | Si hay cambio, guarda resumen y posiciones en Cartera |
| R1 · 32 y 33 | Tools · Set variable | `cartera_nueva` y `posiciones_nuevas`, para que las rutas siguientes usen la cartera ya actualizada |
| R2 · 22 | Gemini (salida JSON con esquema) | Solo si el mensaje pregunta por precios o mercado: los símbolos de Yahoo Finance que hacen falta, hasta 5 |
| R2 · 23 | HTTP · POST a Cotizar | Si hay símbolos; la respuesta llega sin interpretar, como texto |
| R2 · 28 | Tools · Get variable | `posiciones_nuevas` |
| R2 · 27 | Make Code (JavaScript) | Formatea las cotizaciones y valora la cartera |
| R2 · 24 | Tools · Set variable | `cotizaciones` |
| R3 · 40 | Gemini (salida JSON con esquema) | Solo si el mensaje pide o implica una cuenta: plantea la cuenta y escribe el código que la resuelve |
| R3 · 41 | Make Code (JavaScript) | Ejecuta ese código si pasa el filtro de seguridad |
| R3 · 42 | Tools · Set variable | `calculo`: planteamiento y resultado |
| R4 · 25, 26, 29 y 53 | Tools · Get variable | `cotizaciones`, `cartera_nueva`, `calculo` y `decision` |
| R4 · 5 | Gemini · Generate a response | Genera la respuesta |
| R4 · 6 | Webhooks · Webhook response | `{"respuesta": "…"}` |
| R4 · 10 | HTTP · POST a Recuperar | Solo si la relación no existe o tiene más de 24 h: qué sabe el agente de esa persona, k = 10 |
| R4 · 11 | Gemini | La resume en una o dos frases |
| R4 · 13 | Data store · Add a record | La guarda, salvo que diga que aún no sabe nada concreto |

La ruta 0 no tiene filtro: cada mensaje pasa por Percibir. Las rutas 1 a 3 empiezan con un filtro por expresión regular sobre el mensaje, sin distinguir mayúsculas, para no gastar una llamada a Gemini en cada mensaje. Si el filtro no pasa, la ruta no hace nada y su variable queda vacía. La ruta 4 no tiene filtro y siempre responde.

| Ruta | Expresión que la activa |
| --- | --- |
| R1 · cartera | `compr[eéoaá]\|vend[iíoaá]\|tengo .*(acci\|particip\|t[ií]tulo\|fondo\|etf\|bono\|letra\|euros en\|d[oó]lares en)\|mi cartera\|invierto\|he invertido\|he metido\|he puesto\|liquid\|deshec\|ampli[eéoa]\|reduj\|reduzc\|deshacerme\|me he salido\|he salido de` |
| R2 · cotizaciones | `cotiz\|precio\|cu[aá]nto (vale\|est[aá]\|cotiza)\|c[oó]mo (va\|est[aá]\|ha cerrado\|ha abierto\|cerr[oó]\|abri[oó])\|bolsa\|ibex\|acci[oó]n\|acciones\|cartera\|mercado\|sub(e\|en\|i[oó]\|ido)\|baj(a\|an\|[oó]\|ado)\|ca(e\|en\|[ií]do\|y[oó])\|nasdaq\|s&p\|dow jones\|[íi]ndice\|euro stoxx\|etf` |
| R3 · cálculo | `calcul\|cu[aá]nto (ser[ií]a\|tendr[ií]a\|ganar[ií]a\|ganar[eé]\|pagar[ií]a\|pagar[eé]\|me queda\|me quedar[ií]a\|me costar[ií]a\|necesito\|hace falta\|rinde\|renta)\|inter[eé]s\|hipoteca\|cuota\|rentabilidad\|anualizad\|compuest\|amortiz\|\btae\b\|\btin\b\|plazo fijo\|jubila\|inflaci[oó]n\|[0-9] ?%\|[0-9] ?a[nñ]os` |

Las expresiones son deliberadamente amplias: un falso positivo cuesta una llamada a Gemini que devuelve «no hace falta», mientras que un falso negativo deja al agente sin el dato.

### Ruta 0: percibir el mensaje

Es la primera fase del ciclo de Shoham: leer el mensaje y actualizar los compromisos. El módulo 50 recibe el mensaje y la conversación reciente (`4.text`), que es donde están los detalles cuando el usuario acepta algo que Borja le ofreció:

```text
Lee el mensaje que el interlocutor acaba de enviar a {{1.agente}} y decide qué acto de habla es, según el modelo de compromisos de Shoham.

{{1.agente}} solo puede comprometerse a estas acciones:
informar: escribir más adelante al interlocutor por iniciativa propia, por ejemplo para avisarle de algo o darle un dato, con cotizaciones si hacen falta;
preguntar: hacerle más adelante una pregunta;
investigar: buscar información sobre un tema;
cotizar: consultar precios de valores para sí mismo, sin escribir a nadie;
estudiar: analizar un evento o un dato económico cuando se publique;
abstenerse: no hacer algo que el interlocutor no quiere (informar o preguntar).

Actos posibles:
REQUEST: el interlocutor pide a {{1.agente}} que haga una de esas acciones más adelante, a una hora, en una fecha o cuando se cumpla una condición; o acepta algo que {{1.agente}} le ofreció en su último mensaje. Si le pide que no le escriba o no le pregunte por algo, es REQUEST con la acción abstenerse.
UNREQUEST: el interlocutor libera a {{1.agente}} de algo que le había pedido («ya no hace falta que me avises», «olvida lo de Inditex») o retira una prohibición anterior; en ese caso la acción es abstenerse.
NINGUNO: cualquier otro mensaje, incluidas las preguntas que {{1.agente}} puede contestar en este mismo momento.

Campos:
accion: una de las acciones anteriores; en UNREQUEST, vacía si le libera de todo.
dia (AAAA-MM-DD) y hora (HH:mm), en hora de Madrid; vacíos si es para ahora o no se dice.
fin (AAAA-MM-DD): hasta cuándo vale el encargo si el interlocutor lo dice; si no, vacío.
condicion: la condición de la que depende, escrita de forma que se pueda comprobar con datos (por ejemplo «Inditex por debajo de 50 euros»); si no hay, vacía.
simbolos: símbolos de Yahoo Finance de los valores implicados, separados por comas (por ejemplo ITX.MC, SAN.MC, ^IBEX); si no hay, vacío.
contenido: qué hará {{1.agente}}, en una frase y con los datos concretos que haya en la conversación; en abstenerse, solo la acción de la que se abstiene (informar o preguntar); en UNREQUEST, de qué le libera.
libera: en UNREQUEST, las claves de los compromisos de la lista de los que le libera, separadas por comas; si le libera de todo, todas las claves; si no reconoces ninguno, vacía. En una abstención, las claves de los compromisos de la lista que dejan de tener sentido por ella. En los demás casos, vacía.
repite: en REQUEST, la clave del compromiso de la lista que ya pide lo mismo; si no hay ninguno, vacía.
vetado_por: en REQUEST, la clave de la abstención de la lista que prohíbe lo que ahora pide; si no hay ninguna, vacía.
En una abstención sobre valores concretos, pon sus símbolos; sobre un tema sin valores, escribe el tema en condicion; si es sobre todo, deja ambos vacíos.
Usa solo claves que aparezcan en la lista.
En NINGUNO, deja vacíos todos los campos salvo acto.

Ahora es {{día de la semana}}, {{AAAA-MM-DD a las HH:mm}}, hora de Madrid.

Conversación reciente, de lo más nuevo a lo más antiguo:
{{4.text}}

Compromisos vigentes de {{1.agente}} con este interlocutor, con su clave:
{{ifempty(trim(55.text); "ninguno")}}

Mensaje que debes clasificar: «{{1.mensaje}}»
```

El esquema de salida es `{acto, accion, dia, hora, fin, condicion, simbolos, contenido, libera, repite, vetado_por}`, con `acto` limitado a los tres valores.

**La lista de compromisos.** La preparan los módulos 54 y 55: los vigentes cuyo destinatario o `comprometido_con` es este chat, cada uno con su clave. Con ella Percibir señala de cuáles libera el usuario, cuál repite o qué veto impide la petición, y Comprometer decide con esas claves, no con el texto.

**La decisión.** Make entrega la respuesta de Comprometer como dato binario, porque la llamada no la interpreta (`parseResponse` desactivado). El módulo 52 la convierte en texto con `toString` y la guarda solo si empieza por ACEPTADO, RECHAZADO, LIBERADO o NADA. Si no, guarda el aviso de reserva:

```text
Ahora mismo no has podido consultar tu agenda de compromisos. Di que en este momento no puedes apuntarlo y pide que te lo repita más tarde. No prometas nada ni te niegues.
```

> [!NOTE]
> El aviso de reserva salta, por ejemplo, cuando Make se queda sin créditos: la llamada a Comprometer queda en cola y responde «Accepted».

### Ruta 1: registrar la cartera

El módulo 30 recibe las posiciones registradas, el resumen y el mensaje:

```text
Posiciones registradas de {{ifempty(1.usuario; "esta persona")}} (JSON; vacío si no hay ninguna): {{ifempty(21.posiciones; "[]")}}
Resumen registrado de su cartera e intereses: {{ifempty(21.cartera; "(ninguno)")}}

Mensaje que acaba de escribir: {{1.mensaje}}

¿Comunica el mensaje, de forma explícita y en primera persona, que tiene, ha comprado, ha vendido o ha cambiado alguna posición propia? Las preguntas, las hipótesis («¿y si compro…?», «estoy pensando en…») y lo que hacen otras personas NO cuentan.
Si no la comunica, devuelve cambia=false y repite las posiciones y el resumen sin tocarlos.
Si la comunica, devuelve cambia=true y la lista COMPLETA de posiciones tras el cambio: conserva las que no menciona, suma o resta cantidades en compras y ventas parciales, elimina las vendidas por completo y recalcula el precio medio ponderado en las compras. Usa símbolos de Yahoo Finance (Bolsa de Madrid con sufijo .MC, Estados Unidos sin sufijo, ETF por su ticker de Yahoo solo si lo conoces con certeza; si no, deja el símbolo vacío). Si no da cantidad o precio, pon 0 en ese campo; no inventes cifras.
En resumen escribe en tercera persona, en una o dos frases, la cartera resultante con cantidades y precios medios, y conserva lo que el resumen anterior dijera de sus intereses, horizonte o preocupaciones.
posiciones_json debe ser exactamente la lista de posiciones como texto JSON.
```

Su esquema de salida es `{cambia, posiciones[], posiciones_json, resumen}`, con cada posición como `{simbolo, nombre, cantidad, precio_medio, moneda}`. La lista va dos veces, como objeto y como texto, porque Make guarda en el almacén el texto tal cual y no tiene una función fiable para convertir una lista en JSON.

### Ruta 2: cotizaciones y valoración

El módulo 22 elige los símbolos con el mensaje, la cartera en texto y las posiciones en JSON:

```text
Mensaje de {{ifempty(1.usuario; "su interlocutor")}} a {{1.agente}}: {{1.mensaje}}

Cartera e intereses registrados de esa persona: {{ifempty(21.cartera; "no hay cartera registrada")}}
Posiciones registradas (JSON, con su símbolo de Yahoo): {{ifempty(21.posiciones; "[]")}}

¿Necesita {{1.agente}} conocer cotizaciones actuales para responder bien a este mensaje? Si es así, devuelve como máximo 5 símbolos de Yahoo Finance de los valores o índices concretos que hacen falta: acciones de la Bolsa de Madrid con el sufijo .MC (ITX.MC, SAN.MC), acciones de Estados Unidos sin sufijo (AAPL), índices con su símbolo de Yahoo (^IBEX, ^STOXX50E, ^GSPC, ^IXIC) y ETF por su ticker de Yahoo solo si lo conoces con certeza. Si el mensaje habla de la cartera en general, usa las posiciones de la cartera registrada. Si no hacen falta cotizaciones, devuelve la lista vacía.
Devuelve exclusivamente un JSON: {"simbolos": ["..."]}
```

El módulo 27 es la calculadora de la cartera. Recibe la respuesta de Cotizar y las posiciones (las recién registradas o, si no hay, las del almacén) y devuelve un texto con una línea por cotización y, si alguna posición tiene precio, su valor, su coste y su resultado, con totales por moneda. El modelo no hace ninguna de esas cuentas: recibe las cifras hechas con la orden de usarlas tal cual.

```javascript
const { precios_json, posiciones_json } = input;
const num = (n, d) => {
  if (n === null || n === undefined || !isFinite(n)) return "s/d";
  const neg = n < 0;
  let [e, dec] = Math.abs(n).toFixed(d).split(".");
  e = e.replace(/\B(?=(\d{3})+(?!\d))/g, ".");
  return (neg ? "-" : "") + e + (d > 0 ? "," + dec : "");
};
const decs = x => Math.min(4, Math.max(2, ((String(x).split(".")[1]) || "").length));
const sig = n => (n > 0 ? "+" : "") + num(n, 2);
let precios = [];
try { precios = JSON.parse(precios_json || "{}").precios || []; } catch (e) { precios = []; }
let posiciones = [];
try { posiciones = JSON.parse(posiciones_json || "[]"); } catch (e) { posiciones = []; }
if (!Array.isArray(posiciones)) posiciones = [];

const lineas = precios.map(p => {
  if (p.precio === null || p.precio === undefined) {
    return `${p.simbolo}: sin datos${p.error ? " (" + p.error + ")" : ""}`;
  }
  const dec = decs(p.precio);
  const variacion = p.cierre_anterior ? (p.precio / p.cierre_anterior - 1) * 100 : null;
  return `${p.simbolo} (${p.nombre}): ${num(p.precio, dec)} ${p.moneda}` +
    (variacion === null ? "" : `, ${sig(variacion)} % frente al cierre anterior de ${num(p.cierre_anterior, dec)}`) +
    (p.hora ? `, dato de ${p.hora}` : "");
});

const porSimbolo = {};
for (const p of precios) {
  if (p.precio !== null && p.precio !== undefined) porSimbolo[String(p.simbolo).toUpperCase()] = p;
}
const filas = [];
const tot = {};
for (const q of posiciones) {
  const p = porSimbolo[String(q.simbolo || "").toUpperCase()];
  const cantidad = Number(q.cantidad) || 0;
  if (!p || !cantidad) continue;
  const moneda = p.moneda || q.moneda || "";
  const valor = cantidad * p.precio;
  const t = tot[moneda] || (tot[moneda] = { valor: 0, valorConCoste: 0, coste: 0 });
  t.valor += valor;
  let fila = `${q.nombre || q.simbolo}: ${num(cantidad, cantidad % 1 ? 2 : 0)} títulos a ${num(p.precio, decs(p.precio))} = ${num(valor, 2)} ${moneda}`;
  const pm = Number(q.precio_medio) || 0;
  if (pm > 0) {
    const coste = cantidad * pm;
    const pl = valor - coste;
    t.valorConCoste += valor;
    t.coste += coste;
    fila += `; coste ${num(coste, 2)} ${moneda}; resultado ${sig(pl)} ${moneda} (${sig(pl / coste * 100)} %)`;
  }
  filas.push(fila);
}
let valoracion = "";
if (filas.length) {
  valoracion = "Valoración de su cartera, calculada con exactitud (usa estas cifras tal cual, no hagas cuentas propias):\n" + filas.join("\n");
  for (const [moneda, t] of Object.entries(tot)) {
    valoracion += `\nTotal en ${moneda}: ${num(t.valor, 2)} ${moneda}`;
    if (t.coste > 0) {
      const pl = t.valorConCoste - t.coste;
      valoracion += `; resultado de las posiciones con precio de compra conocido ${sig(pl)} ${moneda} (${sig(pl / t.coste * 100)} %)`;
    }
  }
}
return { texto: lineas.join("\n") + (valoracion ? "\n\n" + valoracion : ""), valoradas: filas.length };
```

Un ejemplo de su salida, con 50 Inditex a 45 € y 200 Santander a 9,80 €:

```text
ITX.MC (Industria de Diseño Textil, S.A.): 57,24 EUR, +0,88 % frente al cierre anterior de 56,74, dato de 26/10/2026 17:44
SAN.MC (Banco Santander, S.A.): 11,154 EUR, -3,09 % frente al cierre anterior de 11,510, dato de 26/10/2026 17:44

Valoración de su cartera, calculada con exactitud (usa estas cifras tal cual, no hagas cuentas propias):
Inditex: 50 títulos a 57,24 = 2.862,00 EUR; coste 2.250,00 EUR; resultado +612,00 EUR (+27,20 %)
Banco Santander: 200 títulos a 11,154 = 2.230,80 EUR; coste 1.960,00 EUR; resultado +270,80 EUR (+13,82 %)
Total en EUR: 5.092,80 EUR; resultado de las posiciones con precio de compra conocido +882,80 EUR (+20,97 %)
```

Los totales se separan por moneda porque el sistema no convierte divisas, y el resultado total solo suma las posiciones cuyo precio de compra se conoce.

### Ruta 3: cálculos libres

Para cualquier otra cuenta (interés compuesto, cuota de un préstamo, rentabilidad anualizada, ahorro necesario), el módulo 40 escribe el código y el 41 lo ejecuta:

```text
Mensaje de {{ifempty(1.usuario; "su interlocutor")}} a {{1.agente}}, asesor financiero: {{1.mensaje}}

Cartera registrada de esa persona: {{ifempty(21.cartera; "no hay cartera registrada")}}

¿Necesita {{1.agente}} hacer alguna cuenta numérica para responder bien (interés compuesto, cuota de un préstamo, rentabilidad, conversión, porcentajes, ahorro necesario…)? Las cotizaciones y la valoración de la cartera se calculan aparte: no las incluyas.
Si no hace falta ninguna cuenta, devuelve necesita=false y deja codigo y planteamiento vacíos.
Si hace falta, devuelve necesita=true, en planteamiento una frase con la cuenta que se hace y los supuestos que tomas (si falta un dato imprescindible, toma un supuesto razonable y dilo), y en codigo el cuerpo de una función de JavaScript que haga la cuenta solo con aritmética y Math, sin bucles de más de 1.000 vueltas, sin funciones asíncronas y sin acceder a nada externo, y que termine con return { resultado: "..." }, donde resultado es un texto en español con las cifras redondeadas a dos decimales, con coma decimal y sus unidades.
```

El módulo 41 usa el modo de código mapeado de Make Code (`inputFormat: string`), que ejecuta el texto recibido. Antes, un filtro exige `necesita = true` y rechaza el código si contiene alguno de estos patrones: `require|import|process|fetch|eval|Function|global|constructor|prototype|__|while\s*\(\s*true|for\s*\(\s*;\s*;|setTimeout|setInterval|XMLHttp|Buffer|child_|\bfs\b|http|module|exports|async|await|Promise`. El código corre además en el entorno aislado de Make, con 30 segundos como máximo. Si falla, el manejador Ignore deja la variable vacía y el agente responde sin la cifra.

### El prompt de respuesta

La instrucción de sistema fija el personaje y las reglas de conversación:

```text
{{1.descripcion}}

Lo que {{1.agente}} ha llegado a saber de sí mismo por lo vivido en esta conversación (si está vacío, ignóralo):
{{9.data.resumen}}

Eres {{1.agente}} y estás hablando por Telegram. Mantente siempre en tu personaje. Responde en el idioma del interlocutor, con la extensión propia de un chat, y no inventes recuerdos que no aparezcan en tu memoria. No inventes precios ni cotizaciones: cita solo las cifras que aparezcan en las cotizaciones consultadas y, si no las tienes, dilo. Si en este mensaje te comunica un cambio en su cartera, ya ha quedado registrado: confírmalo con naturalidad. Si recibes una valoración de la cartera ya calculada, usa sus cifras exactamente como vienen y no hagas cuentas por tu cuenta.
```

El mensaje de usuario sigue la estructura del paper, con los datos de las rutas antes del historial:

```text
Son las {{formatDate(now; "HH:mm [del] DD/MM/YYYY"; "Europe/Madrid")}}.

Compromisos que {{1.agente}} tiene pendientes para hoy, en notación de compromisos (menciónalos solo si vienen al caso): {{ifempty(trim(15.text); "ninguno")}}
Decisión sobre los compromisos que pide este mensaje: {{ifempty(53.decision; "este mensaje no pide ningún compromiso")}} Atente a ella: si se ha aceptado, confírmalo diciendo cuándo lo harás; si se ha rechazado, explica el motivo con tu estilo; si te han liberado, dilo; y no prometas nada que no esté aceptado aquí o en tus compromisos.

Lo que {{1.agente}} sabe de {{ifempty(1.usuario; "su interlocutor")}} y de su relación: {{ifempty(12.resumen; "todavía no hay un resumen; guíate por el historial")}}

Resumen del contexto relevante de la memoria de {{1.agente}} (ordenado por relevancia):
{{2.data.texto}}

Cartera e intereses registrados de {{ifempty(1.usuario; "su interlocutor")}} (si está vacío, no ha contado nada de su cartera; no le atribuyas posiciones que no figuren aquí): {{ifempty(26.cartera_nueva; 21.cartera)}}

Cotizaciones consultadas ahora mismo en Yahoo Finance (si está vacío, no se han consultado; si la fecha del dato es de una sesión anterior, ese mercado aún no ha abierto o ya cerró):
{{25.cotizaciones}}

Cálculos hechos ahora para este mensaje (si está vacío, no se ha calculado nada; si hay cifras, son exactas: úsalas tal cual y menciona los supuestos si importan):
{{29.calculo}}

Historial reciente de la conversación (del más reciente al más antiguo):
{{4.text}}

{{ifempty(1.usuario; "El interlocutor")}} le dice a {{1.agente}}: {{1.mensaje}}

¿Cómo respondería {{1.agente}}? Escribe únicamente su respuesta, sin comillas ni prefijos.
```

El prompt separa dos memorias. Los recuerdos relevantes son la memoria a largo plazo, elegidos por puntuación y que pueden venir de semanas atrás o ser reflexiones. El historial de 12 observaciones es la memoria de trabajo, el hilo literal de la conversación, sin el cual el agente perdería la continuidad inmediata. El historial va del más reciente al más antiguo, y el prompt lo declara para que el modelo no lo lea al revés.

El resumen de la relación sigue el paper, que antes de cada diálogo recupera lo que el agente sabe de su interlocutor. Se regenera después de responder, como mucho una vez cada 24 horas por chat: la respuesta usa siempre la versión anterior y la nueva vale desde el mensaje siguiente, así que el usuario no espera por ella.

Los compromisos se buscan en el almacén por agente, con momento anterior a la medianoche siguiente (hora de Madrid) y sin los estados finales: cumplido, fallido, liberado y caducado. El agregador escribe la línea de cada orden con `switch(ifempty(14.data.destinatario; "SELF"); "SELF"; 14.data.linea; 1.chat_id; 14.data.linea; "")`: las órdenes sin destinatario son del propio agente y se muestran, las de este chat también, y las de otros chats quedan en blanco, para que el agente no cuente a una persona lo que ha prometido a otra. La búsqueda continúa aunque no haya resultados, para que Responder conteste igual. La hora se formatea en Europe/Madrid porque el paper siempre sitúa al agente en su hora actual.

Las variables pasan de una ruta a otra con Set variable y Get variable de alcance «roundtrip», porque una ruta de un router no puede leer los módulos de otra. Cada lectura cuesta un crédito.

## 10. (7) Resumen del agente

El Resumen del agente reconstruye quién es el agente a partir de su memoria con las tres consultas del paper: sus rasgos esenciales, su ocupación diaria actual y cómo se siente respecto a su progreso reciente. Responder lo añade a la instrucción de sistema, detrás de la descripción fija.

| Entrada | Comportamiento | Tiempo |
| --- | --- | --- |
| `modo` distinto de ON, o ausente | Devuelve `{"resumen": "", "estado": "OFF"}` sin consultar nada | < 1 s |
| ON, con caché de menos de 7 días | Devuelve el resumen guardado, estado cache | < 1 s |
| ON, sin caché o caducada | Devuelve el resumen anterior (o vacío) y lo regenera después | < 1 s; la regeneración, unos 35 s |

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Router | Ruta OFF o ruta ON según `modo` |
| R1 · 3 | Webhooks · Webhook response | Modo OFF: `{"resumen": "", "estado": "OFF"}` |
| R2 · 4 | Data store · Get a record | Lee la caché `agente\|chat_id` |
| R2 · 5 | Router | Caché vigente o caché ausente o caducada (7 días) |
| R2 · 6 | Webhooks · Webhook response | Caché vigente: devuelve el resumen guardado, estado cache |
| R2 · 19 | Webhooks · Webhook response | Sin caché: devuelve el resumen anterior o vacío, estado regenerando |
| R2 · 10, 12, 14 | HTTP · POST a Recuperar | Las tres consultas del paper, k = 15 cada una |
| R2 · 11, 13, 15 | Gemini | Resume cada consulta |
| R2 · 16 | Tools · Set variables | Une los tres apartados con la hora de actualización |
| R2 · 18 | Data store · Add a record | Guarda el resumen, salvo que los tres sean «sin datos» |

Las tres consultas a Recuperar son «rasgos esenciales de la personalidad de {agente}», «ocupación diaria actual de {agente}» y «cómo se siente {agente} respecto a su progreso reciente en la vida». Cada una se resume con el mismo patrón; este es el de la primera:

```text
¿Cómo describirías los rasgos esenciales de la personalidad de {{1.agente}} a partir de las siguientes afirmaciones?
{{10.data.texto}}

Responde en una o dos frases, en tercera persona y en texto plano (sin formato), sin inventar nada que no se deduzca de las afirmaciones. Si no hay afirmaciones suficientes, responde únicamente: sin datos.
```

La regeneración lanza las tres consultas contra Recuperar (k = 15) y resume cada una con Gemini en una o dos frases, o «sin datos» si no hay afirmaciones suficientes. Guarda el resultado con clave `agente|chat_id`, salvo que los tres apartados sean «sin datos»: así un chat nuevo no se queda una semana sin resumen. El interruptor está en el Gateway, que envía `"modo": "ON"`. Como la memoria es por chat, el agente puede verse distinto en cada conversación.

## 11. (8) Estudiar

Estudiar decide si un mensaje contiene una pregunta o un requerimiento profesional concreto que exija información verificable y, si es así, investiga en Google y guarda en memoria una pieza de conocimiento práctica. El Gateway le pasa cada mensaje después de responder, sin esperar.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 2 | Gemini (salida JSON con esquema) | Clasifica: investigar, aclarar, duda, tema y pregunta reformulada |
| 3 | JSON · Parse JSON | Convierte la clasificación |
| 8 | Router | Ruta «Investigar» o ruta «Falta un dato: interpelar» |
| 4 | HTTP · POST a Recuperar | Qué recuerda ya el agente del tema (k = 5) |
| 5 | Gemini con búsqueda de Google | Si ya lo sabe responde YA_SABIDO; si no, redacta hasta 60 palabras con la fuente al final |
| 6 | HTTP · POST a Escribir | Guarda «{agente} se ha informado sobre {tema}: {texto}» como conocimiento, autor sistema |
| 7 | HTTP · POST a Interpelar | Pasa la duda (el dato que falta), el tema y el mensaje original |

El clasificador del módulo 2 es la pieza que decide casi todo:

```text
Mensaje que {{ifempty(1.usuario; "el interlocutor")}} le ha enviado a {{1.agente}}:
«{{1.mensaje}}»

Perfil de {{1.agente}}: {{ifempty(1.descripcion; "(sin descripción)")}}

¿Contiene una pregunta o un requerimiento concreto sobre un tema del ámbito profesional (técnico, legal, fiscal, económico, de negocio, de una herramienta, una norma, un dato o un procedimiento) cuya respuesta exige información verificable que un profesional tendría que consultar? No cuentan los saludos, la charla, las opiniones, los asuntos personales o emocionales ni lo que se responde con sentido común.
Si es una cuestión profesional pero falta un dato imprescindible para poder investigarla con fiabilidad (por ejemplo el país, la herramienta, la versión, el sector o el tipo de contrato) y no se deduce del mensaje, marca aclarar como true e indica en duda qué dato falta.
Devuelve exclusivamente un JSON: {"investigar": true o false, "aclarar": true o false, "duda": "el dato que falta, o vacío", "tema": "el tema en pocas palabras", "pregunta": "la pregunta concreta, reformulada para buscarla en internet"}
```

Su esquema de salida:

```json
{"type": "object", "properties": {"investigar": {"type": "boolean"}, "tema": {"type": "string"}, "pregunta": {"type": "string"}, "aclarar": {"type": "boolean"}, "duda": {"type": "string"}}, "required": ["investigar", "aclarar", "duda", "tema", "pregunta"]}
```

La investigación del módulo 5, con la búsqueda de Google activada:

```text
{{1.agente}} necesita saber esto para su trabajo: {{3.pregunta}}

Lo que ya recuerda al respecto:
{{ifempty(4.data.texto; "(nada)")}}

Si lo que ya recuerda responde la pregunta de forma suficiente, responde exactamente YA_SABIDO y nada más.
Si no, investiga con búsquedas en internet y redacta una pieza de conocimiento muy pragmática, de 60 palabras como máximo, en texto plano: la respuesta concreta, el dato o el procedimiento clave y, si aplica, la cifra, el plazo o la condición que importa. Sin introducción, sin opiniones y sin relleno. Termina con «(Fuente: nombre del sitio)». Si las fuentes no permiten una respuesta fiable, dilo en una sola frase.
```

El filtro del módulo 6 descarta la respuesta YA_SABIDO, así que solo se guarda conocimiento nuevo.

No cuentan los saludos, la charla, las opiniones, los asuntos personales ni lo que se responde con sentido común. Falta un dato cuando la cuestión es profesional pero no se puede investigar con fiabilidad sin saber, por ejemplo, el país, la herramienta, la versión, el sector o el tipo de contrato. La respuesta del momento no usa la investigación; la usan los mensajes siguientes, a través de Recuperar.

## 12. (9) Interpelar

Interpelar escribe al usuario por iniciativa propia, en el personaje del agente, para resolver una duda concreta. Lo llaman Estudiar, cuando a una petición profesional le falta un dato imprescindible, y Cumplir, cuando al estudiar un evento le falta un dato de la cartera o cuando cumple un compromiso de preguntar.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 2 | Airtable · Search records | Interpelaciones de este agente en este chat en los últimos 30 minutos |
| 3 | Tools · Numeric aggregator | Las cuenta |
| 7 | Airtable · Search records | Interpelaciones de hoy en este chat |
| 8 | Tools · Numeric aggregator | Las cuenta |
| 4 | Gemini | Si no hubo ninguna en 30 minutos y van menos de dos hoy, redacta una sola pregunta breve, en el personaje |
| 5 | Telegram Bot · Send a Text Message | La envía al chat |
| 6 | HTTP · POST a Escribir | Guarda «{agente} le planteó una duda a {usuario}: «{pregunta}»» como observación del agente |

La pregunta se redacta con este prompt, con la descripción del personaje como instrucción de sistema:

```text
{{1.agente}} tiene una duda que necesita aclarar con {{ifempty(1.usuario; "su interlocutor")}} antes de seguir.
Qué necesita saber: {{1.motivo}}
Contexto: {{ifempty(1.contexto; "(sin contexto adicional)")}}

Escribe el mensaje de Telegram que {{1.agente}} le enviaría por iniciativa propia: una sola pregunta concreta y breve que permita resolver la duda, en su personaje y en texto plano. Escribe únicamente el mensaje, sin comillas ni prefijos.
```

Los dos límites evitan que el agente encadene preguntas, también en un día con muchos eventos. La pregunta queda en el historial: cuando el usuario contesta, Responder la ve en la conversación reciente y Estudiar recibe la respuesta como un mensaje nuevo que ya puede investigar.

## 13. (10) Despertador

El Despertador arranca el día del agente aunque nadie escriba. Cada día a las 5:00, hora de Madrid, lee el perfil de Borja y, si es laborable, deja en Compromisos la orden «agendar» para las 8:00. Cumplir la ejecuta en su primera vuelta.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Data store · Get a record | Lee `perfil\|Borja` |
| 5 | Data store · Add/replace a record | Si hay perfil y el día no es sábado ni domingo, guarda la orden con la clave `agendar\|<agente>\|<fecha>` |

La clave fija por día evita duplicados si el escenario se ejecuta dos veces. Escribe directamente en el almacén, sin pasar por Comprometer, porque es una regla fija del sistema. La orden se crea sin estado; un compromiso sin estado cuenta como pendiente. Cuesta 2 créditos al día.

## 14. (11) Cumplir: ejecución de compromisos

Cumplir es la segunda fase del ciclo de AGENT-0: a intervalos fijos, ejecuta lo que el agente se ha comprometido a hacer y ya ha vencido. Se llamó Percibir hasta noviembre de 2026; el nombre cambió porque no percibe nada, solo ejecuta. La percepción está en Responder (sección 9).

> [!IMPORTANT]
> Cumplir quedará como reloj cuando se active la ejecución por pasos de la [sección 18](#18-ejecución-en-desarrollo-15-actuar-16-observar-y-17-refinar). Hasta entonces ejecuta los compromisos con las acciones fijas que se describen aquí.

Se ejecuta cada 30 minutos los laborables de 8:00 a 20:30, hora de Madrid. Cada vuelta toma hasta 3 compromisos, del más antiguo al más reciente. Una vuelta sin nada vencido cuesta 1 crédito.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 20 | Data store · Search records | Compromisos con momento ya pasado, que no sean abstenerse ni estén cumplidos, fallidos, liberados o caducados; hasta 3 |
| 21 | Router | Tres rutas excluyentes |
| R1 · 70 | Data store · Add/replace a record | Si ha pasado su fecha de fin: estado caducado |
| R1 · 71 | Data store · Add a record | Si era una promesa a un chat: crea el compromiso de decírselo |
| R2 · 72 | Data store · Add/replace a record | Si lleva tres intentos: estado fallido |
| R2 · 73 | Data store · Add a record | Si era una promesa a un chat: crea el compromiso de avisar de que no ha podido cumplirla |
| R3 · 80 | Data store · Add/replace a record | Si está vigente y lleva menos de tres intentos: lo reserva (estado en_curso, momento dentro de 30 minutos, un intento más) |
| R3 · 1 | Data store · Get a record | El perfil del agente |
| R3 · 23 | Router | Una ruta por acción |

### Reservar en lugar de borrar

Antes de actuar, Cumplir no borra el compromiso: lo reserva. Al aplazarlo media hora, la vuelta siguiente no lo coge mientras se ejecuta. Si la acción termina bien, el final de su ruta lo marca como cumplido. Si falla, el manejador Ignore detiene la ruta, el compromiso no se marca y vuelve a salir en la vuelta siguiente como un reintento. Con tres intentos fallidos pasa a fallido.

Los avisos de los módulos 71 y 73 son compromisos de informar con `origen` `cumplir:caducado` o `cumplir:fallido`. Un aviso que falla no genera otro aviso.

Los contadores se suman como número: `parseNumber(ifempty(20.data.intentos; "0"); ".") + 1`. Con `ifempty(…) + 1`, Make concatena texto («0» + 1 = «01»).

### Las acciones

| Acción | Tipo | Módulos | Qué hace | Cuándo queda cumplido |
| --- | --- | --- | --- | --- |
| agendar | DO | 40, 82, 41–45 | Gemini con búsqueda de Google encuentra los eventos del día con hora confirmada; crea una orden de estudiar por evento y guarda la agenda en la memoria | En cuanto Gemini responde (82) |
| estudiar | DO | 3, 24, 25, 81 y 4–11 | Busca el resultado del evento. Si no está publicado, lo reprograma para dentro de 30 minutos. Si lo está, calcula su implicación para la cartera de cada chat, la guarda y, si falta un dato, llama a Interpelar | Al saberse publicado (81) |
| informar | INFORM | 50–55 y 83 con símbolos; 56–60 y 84 sin ellos | Con símbolos, pide cotizaciones. Gemini redacta el mensaje o responde `NO_CUMPLE` si la condición no se da, y entonces se reprograma para dentro de 2 horas | Al enviarse por Telegram (83, 84) |
| preguntar | REQUEST | 61 y 85 | Llama a Interpelar con el contenido como motivo | Tras la llamada (85) |
| investigar | DO | 62 y 86 | Llama a Estudiar con el contenido como mensaje | Tras la llamada (86) |
| cotizar | DO | 63, 64 y 87 | Pide cotizaciones y las guarda como conocimiento | Tras guardarlas (87) |

Abstenerse nunca se ejecuta: solo veta en Comprometer esa acción para ese chat, hasta que el usuario la libere o pase su fecha de fin.

Las reprogramaciones de «no publicado» (25) y de «no se cumple» (53, 58) devuelven el estado a pendiente y los intentos a 0, porque no son fallos. La de «no publicado» fija la fecha de fin, si no la tenía, 18 horas después del momento del estudio.

Cada orden de estudiar que crea agendar (43) lleva estado pendiente, nivel 1, momento una hora después del evento y fin 18 horas después de ese momento. La hora de margen da tiempo a que el dato esté publicado; las 18 horas cubren los eventos de la tarde de Estados Unidos, que se estudian a la mañana siguiente.

### Prompts

La investigación del resultado, en el módulo 3, con la búsqueda de Google activada:

```text
Evento financiero previsto: {{20.data.contenido}}

Busca el resultado de este evento: el dato publicado, la decisión tomada o las cifras presentadas, y la reacción inmediata del mercado si la hay. Si todavía no se ha publicado, responde únicamente: NO_PUBLICADO
Si se ha publicado, responde en un máximo de 60 palabras, con las cifras exactas, y termina con la fuente entre paréntesis.
```

La implicación para cada chat, en el módulo 8, con salida JSON (`conocimiento` y `duda`):

```text
Evento: {{20.data.contenido}}
Resultado: {{3.result}}

Cartera e intereses de su interlocutor: {{ifempty(7.cartera; "NO HAY CARTERA REGISTRADA")}}

Si hay cartera registrada, redacta en tercera persona y en un máximo de 60 palabras qué implica este resultado para esa cartera en concreto (qué posiciones o intereses se ven afectados, en qué sentido y con qué intensidad), manteniendo la fuente; si ninguna posición se ve afectada, dilo en una frase. Si no hay cartera registrada, redacta un comentario genérico de {{1.agente}} sobre qué implica este resultado para un inversor particular, en un máximo de 60 palabras y con la fuente.
En duda, formula en una frase el dato que a {{1.agente}} le falta y que solo su interlocutor puede darle para valorar el impacto (por ejemplo, si mantiene una posición o su horizonte); si no le falta nada o no hay cartera, deja duda vacía.
```

La agenda del día, en el módulo 40, con la búsqueda de Google activada y la descripción del personaje como instrucción de sistema:

```text
Hoy es {{día de la semana}}, {{DD/MM/YYYY}}. Busca la agenda de eventos financieros de hoy que seguiría un asesor financiero en Madrid: datos macroeconómicos de España, la zona euro y Estados Unidos, decisiones o comparecencias de bancos centrales, subastas de deuda relevantes y resultados empresariales del Ibex 35 y de las grandes cotizadas de Estados Unidos.
Incluye solo eventos de hoy con hora confirmada por una fuente, con la hora convertida a la de Madrid. Entre 2 y 5 eventos, los de mayor impacto.
Responde únicamente con una línea por evento, ordenadas por hora, con este formato exacto y sin nada más:
HH:mm|evento en una frase|ámbito (España, Zona euro, EE. UU. o empresa)|fuente
Si hoy no hay eventos relevantes, responde únicamente: NINGUNO
```

El mensaje de informar, en los módulos 51 y 56:

```text
Son las {{hora}}. {{1.agente}} se comprometió a escribir por iniciativa propia a su interlocutor con este fin: «{{20.data.contenido}}».
Solo debe escribirle si se cumple esta condición: «{{20.data.condicion}}».   ← solo si tiene condición
Cotizaciones actuales de Yahoo Finance (JSON): {{50.data}}                    ← solo en la ruta con símbolos
Si hay una condición y no se cumple, responde únicamente: NO_CUMPLE
En caso contrario, redacta el mensaje de Telegram que {{1.agente}} le envía, con su voz, en un máximo de 80 palabras, con las cifras exactas que procedan y sin firma. Responde solo con el mensaje.
```

La ruta sin símbolos activa la búsqueda de Google, para comprobar condiciones o datos que no son precios.

El estudio usa el campo `cartera` de cada chat, que Responder mantiene al día (sección 9).

## 15. (12) Noticias

Noticias lee la prensa económica dos veces cada día laborable, a las 8:00 y a las 16:00, y guarda en la memoria de cada chat los titulares que afectan a la cartera de ese usuario, con su implicación. Es la percepción del entorno que el paper resuelve con un mundo simulado: aquí el entorno es lo que se publica.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | RSS · Watch RSS feed items | Titulares nuevos del feed, hasta 20 por ejecución |
| 2 | Tools · Text aggregator | Una línea por titular: `- {título}` |
| 3 | Data store · Get a record | El perfil del agente (`perfil\|Borja`) |
| 4 | Airtable · Search records | Solo si hay titulares y perfil: los chats donde algún usuario ha escrito al agente |
| 5 | Tools · Text aggregator | Lista de chats, uno por línea |
| 6 | Iterator | Recorre los chats sin duplicados |
| 7 | Data store · Get a record | La cartera del chat |
| 8 | Gemini (salida JSON con esquema) | Elige los titulares que afectan a esa cartera y redacta su implicación |
| 9 | JSON · Parse JSON | Convierte la respuesta |
| 10 | Iterator | Recorre las noticias elegidas |
| 11 | HTTP · POST a Escribir | Guarda «{agente} leyó en prensa: «{titular}». {implicación}» como conocimiento, autor sistema |

El feed es una búsqueda de Google News limitada a tres diarios económicos y al último día:

```text
https://news.google.com/rss/search?q=(Ibex+OR+BCE+OR+%22bolsa+espa%C3%B1ola%22+OR+mercados)+(site:expansion.com+OR+site:cincodias.elpais.com+OR+site:eleconomista.es)+when:1d&hl=es&gl=ES&ceid=ES:es
```

El módulo RSS solo entrega los titulares que no vio en la ejecución anterior, así que cada titular se estudia una vez. La primera vez que se guarda el escenario, Make pregunta desde dónde empezar a leer.

El prompt del módulo 8, con la descripción del personaje como instrucción de sistema:

```text
Titulares de prensa económica publicados en las últimas horas:
{{2.text}}

Cartera e intereses de su interlocutor: {{ifempty(7.cartera; "NO HAY CARTERA REGISTRADA")}}

Si hay cartera registrada, elige como máximo 3 titulares que afecten de forma concreta a alguna de sus posiciones o intereses, y para cada uno explica en tercera persona, en un máximo de 40 palabras, qué implica para esa cartera y en qué sentido.
Si no hay cartera registrada, elige como máximo 1 titular, y solo si tiene un impacto general alto para un inversor particular en España, con su implicación en un máximo de 40 palabras.
No elijas titulares que no afecten de verdad; si ninguno lo hace, devuelve la lista vacía. Copia el titular tal como aparece, con su fuente.
Devuelve exclusivamente un JSON: {"noticias": [{"titular": "...", "implicacion": "..."}]}
```

Noticias no escribe al usuario: lo que aprende queda en la memoria del chat y Responder lo recupera cuando la conversación lo hace relevante. Un ejemplo real: con Santander en cartera, guardó la caída de Telefónica y Santander tras las elecciones en Brasil y el peso de la exposición de Santander a ese país.

## 16. (13) Cotizar

Cotizar devuelve el último precio de hasta 5 valores o índices. Lo llaman Responder, cuando un mensaje pregunta por precios o mercado, y Cumplir, en los compromisos de informar o cotizar que llevan símbolos.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Recibe `token` y `simbolos`, separados por comas |
| 2 | Iterator | Filtro «Token correcto»; `slice(split(replace(1.simbolos; " "; ""); ","); 0; 5)`: como mucho 5 símbolos |
| 3 | HTTP · GET a Yahoo Finance | `https://query1.finance.yahoo.com/v8/finance/chart/{símbolo}?interval=1d&range=1d`, con cabecera `User-Agent` y sin parar si Yahoo devuelve error |
| 4 | Tools · Text aggregator | Un objeto JSON por símbolo, separados por comas |
| 5 | Webhooks · Webhook response | `{"precios": [ … ]}` |

Cada objeto sale de los metadatos de la respuesta de Yahoo (`chart.result[1].meta`):

```text
{"simbolo": "{{escapeJSON(2.value)}}", "nombre": "{{escapeJSON(get(3.data.chart.result; "1.meta.longName"))}}", "precio": {{ifempty(get(3.data.chart.result; "1.meta.regularMarketPrice"); "null")}}, "cierre_anterior": {{ifempty(get(3.data.chart.result; "1.meta.chartPreviousClose"); "null")}}, "moneda": "{{get(3.data.chart.result; "1.meta.currency")}}", "hora": "{{if(get(3.data.chart.result; "1.meta.regularMarketTime"); formatDate(parseDate(get(3.data.chart.result; "1.meta.regularMarketTime"); "X"); "DD/MM/YYYY HH:mm"; "Europe/Madrid"); "")}}", "error": "{{escapeJSON(3.data.chart.error.description)}}"}
```

Un símbolo inexistente devuelve `precio: null` y el motivo en `error`, sin romper la respuesta. Los símbolos siguen la convención de Yahoo: Bolsa de Madrid con el sufijo `.MC` (ITX.MC), Estados Unidos sin sufijo (AAPL) e índices con `^` (^IBEX).

Yahoo Finance se eligió porque no necesita clave y cubre la Bolsa de Madrid. Alpha Vantage, la alternativa con clave gratuita, se descartó tras probarla: desde Make responde que se ha superado el límite diario ya en la primera petición, porque limita también por IP y Make comparte sus IPs de salida entre muchos usuarios. El endpoint de Yahoo no es una API oficial y puede cambiar o bloquearse sin aviso; si falla, Responder recibe «sin datos» y el agente dice que no tiene la cifra.

## 17. Compromisos: el modelo de Shoham y (14) Comprometer

### El modelo

En la programación orientada a agentes de Shoham, el estado mental de un agente son sus creencias, sus capacidades y sus compromisos. Un compromiso es la obligación de hacer algo en un momento dado; una decisión es un compromiso con uno mismo. Los compromisos nacen de reglas de compromiso, que combinan una condición sobre el mensaje recibido y otra sobre el estado mental. El intérprete de AGENT-0 repite un ciclo de dos fases: lee los mensajes y actualiza el estado mental, y ejecuta los compromisos que han vencido.

| AGENT-0 | Human_proxy |
| --- | --- |
| Creencias | La memoria (Memory Stream), la cartera y el perfil |
| Capacidades | agendar, estudiar, cotizar e investigar (DO), informar (INFORM), preguntar (REQUEST) y abstenerse (REFRAIN), más el horario de trabajo |
| Mensajes INFORM, REQUEST, UNREQUEST | Lo que escribe el usuario, clasificado por Percibir en Responder (sección 9) |
| Reglas de compromiso | Comprometer, deterministas; el Despertador y la acción agendar, fijas |
| Compromiso con otro | Registro cuyo `comprometido_con` es un chat |
| Decisión | Registro con `comprometido_con` «yo» |
| Compromiso condicional, `IF (B φ)` | Registro con `condicion`; Cumplir la evalúa antes de actuar |
| Fase 1 del ciclo | Percibir en Responder, y Comprometer |
| Fase 2 del ciclo | Cumplir, cada 30 minutos (sección 14) |

Los compromisos nacen de:

1. **Peticiones del usuario.** REQUEST desde Responder: Borja se compromete con ese chat.
2. **Reflexiones.** DECISION desde Reflexionar: Borja se compromete consigo mismo.
3. **Reglas fijas.** El Despertador (agendar) y la acción agendar (estudiar), que escriben directamente en el almacén.

Lo que Borja dice en una respuesta no crea compromisos: solo comunica decisiones ya tomadas. Un INFORM del usuario («he comprado Inditex») cambia creencias, no compromisos; puede llevar después a una decisión si una reflexión lo pide.

### La tabla Compromisos

| Campo | Contenido |
| --- | --- |
| agente | Agente que se compromete |
| comprometido_con | «yo» o el chat al que se prometió |
| destinatario | Chat al que va dirigida la acción, en informar, preguntar y abstenerse |
| accion | Una de las capacidades |
| momento | Cuándo vence |
| condicion | Condición de la que depende, o vacío |
| simbolos | Símbolos de Yahoo Finance implicados, o vacío |
| contenido | Qué hará, en una frase; en abstenerse, la acción vetada |
| linea | El compromiso en notación de Shoham, el que ve Responder |
| origen | `percibir`, `reflexion`, `despertador`, `agendar`, `cumplir:caducado`, `cumplir:fallido` |
| intentos | Ejecuciones empezadas y no terminadas |
| nivel | Profundidad en la cadena de compromisos; 0 en lo que pide el usuario |
| creado | Cuándo se creó |
| estado | pendiente, en_curso, cumplido, fallido, liberado o caducado; vacío cuenta como pendiente |
| fin | Fecha a partir de la cual caduca, o vacío |

La línea sigue la notación del artículo: `(INFORM 05/11 10:00 informar chat:123456789 «contarle cómo cierra Inditex»)` o, con condición, `(IF (B «Inditex por debajo de 40 €») (INFORM … ))`.

### Ciclo de vida

```text
pendiente ──Cumplir lo reserva──▶ en_curso ──la acción termina bien──▶ cumplido
    ▲                                 │
    │                                 ├── la acción falla: vuelve en 30 min; al tercer intento ──▶ fallido
    └── no publicado / no se cumple ──┘
pendiente o en_curso ──UNREQUEST o veto nuevo──▶ liberado
pendiente o en_curso ──pasa la fecha de fin──▶ caducado
```

Nada se borra. Los estados finales quedan como historia y las búsquedas los excluyen. El almacén tiene 1 MB: con unos 0,7 KB por registro caben unos 1.400. Conviene archivarlo de vez en cuando.

### (14) Comprometer

Comprometer aplica las reglas de compromiso. Siempre responde con la decisión, en texto.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 3 | Tools · Set variables | Filtro «Token correcto»; normaliza acto, acción, chat, contenido, símbolos, condición y nivel; calcula día y hora |
| 4 | Tools · Set variables | Momento ajustado al horario, si se ha aplazado, destinatario, comprometido_con, tipo y firmas de duplicado y de veto |
| 5 | Tools · Set variables | Fecha de fin, hora legible y línea en notación de Shoham |
| 6 | Data store · Search records | Los compromisos vigentes del agente, hasta 100 |
| 7 | Router | Tres rutas |
| R1 · 20 | Data store · Add/replace a record | Marca como liberado cada compromiso que se suelta: los que señala Percibir por su clave, los del valor que nombra el usuario si no hay claves, y lo pendiente que choca con un veto nuevo |
| R2 · 21 | Tools · Text aggregator | UNREQUEST o veto nuevo: las líneas liberadas |
| R2 · 23 | Tools · Set variable | `liberados`, para la respuesta de alta |
| R2 · 22 | Webhooks · Webhook response | UNREQUEST: LIBERADO o NADA |
| R3 · 30 | Tools · Text aggregator | REQUEST o DECISION: por cada vigente, su firma (`accion\|destinatario\|contenido\|primer valor o tema`), una marca si es de este chat y no es un veto, y su clave |
| R3 · 38 | Tools · Get variable | `liberados` |
| R3 · 31 | Tools · Set variables | Comprobaciones |
| R3 · 32 | Router | Aceptar o rechazar |
| 32 R1 · 33 | Data store · Add a record | Guarda el compromiso con estado pendiente |
| 32 R1 · 34 | Webhooks · Webhook response | ACEPTADO |
| 32 R2 · 35 | Webhooks · Webhook response | RECHAZADO, con el motivo |

**Reglas para aceptar** (REQUEST y DECISION):

1. La acción es una capacidad. Agendar solo se admite como DECISION.
2. Hay contenido.
3. Informar, preguntar y abstenerse tienen destinatario: un chat que no sea `GLOBAL`.
4. Cotizar tiene símbolos.
5. El nivel es 2 o menos.
6. No repite un compromiso vigente: ni uno con la misma acción, destinatario, contenido y primer valor, ni el que Percibir señala como repetido.
7. Ningún veto vigente lo impide (ver la tabla de vetos más abajo).
8. Hay menos de **10 compromisos pendientes con ese chat**, sin contar los vetos. Un veto nunca se rechaza por este motivo.

**Vetos.** Solo se vetan informar y preguntar, las acciones que llegan al usuario.

| Veto | Ejemplo | Qué impide | Cómo se comprueba |
| --- | --- | --- | --- |
| De una acción entera | «No me escribas más» | Toda esa acción hacia ese chat | Por su firma, sin modelo |
| De un valor | «No me avises más de Inditex» | Esa acción con ese valor como primer símbolo | Por su firma, sin modelo |
| De un tema | «No me preguntes por mi cartera» | Lo que trate de ese tema | Por la clave que señala Percibir |

Un veto se guarda como compromiso `abstenerse`, con la acción vetada como contenido, el valor en `simbolos` y el tema en `condicion`.

**Momento.** Sin día ni hora, es ahora; con día y sin hora, las 8:00 de ese día. Si cae fuera de los laborables de 8:00 a 20:30, pasa al primer momento en que Borja trabaja y la respuesta lo dice. El cálculo usa `formatDate(…; "d")` para el día de la semana, un `switch` sobre la hora y los minutos, y `formatDate(…; "YYYY-MM-DD [08:00]")` para construir la fecha nueva, sin comparaciones ni concatenaciones dentro de las funciones.

**Fecha de fin.** La que diga el usuario, a las 20:30 de ese día. Si no la dice: 7 días después del momento en los condicionales y las 20:30 del mismo día en los estudios. Los vetos y el resto de compromisos no caducan.

**Liberar.** Un compromiso se libera de tres maneras, y siempre solo si es de ese chat:

1. **Por clave.** En un UNREQUEST, libera los compromisos que Percibir señala. Si el usuario lo suelta todo, Percibir las señala todas. Retirar un veto es liberar su clave.
2. **Por valor.** Si Percibir no señala claves y el usuario nombra un valor, libera los que contienen ese símbolo y coinciden en acción si se indica.
3. **Por veto nuevo.** Al aceptar un veto de acción o de valor, libera lo pendiente que choca con él. En un veto de tema, libera lo que Percibir señala.

La respuesta de alta de un veto enumera lo liberado tras «Deja sin efecto». Una clave de otro chat no libera nada.

**Respuestas:**

```text
ACEPTADO. Borja se ha comprometido: (INFORM 05/11 10:00 informar chat:… «…»). Lo hará el 05/11 a las 10:00.
ACEPTADO. Borja se ha comprometido: (IF (B «Santander por debajo de 4 euros») (INFORM 09/11 08:00 informar chat:… «…»)). Vigilará la condición desde el 09/11 a las 08:00, el primer momento en que trabaja, porque lo pedido cae fuera de su horario (laborables de 8:00 a 20:30) y lo hará en cuanto se cumpla. Vale hasta el 15/11.
ACEPTADO. Borja se compromete a no hacerlo: (REFRAIN 09/11 08:00 abstenerse chat:… «informar»). Deja sin efecto: (INFORM 09/11 10:00 informar chat:… «informar de cómo abre Inditex»)
RECHAZADO. Borja no se compromete porque ya lo tenía apuntado.
LIBERADO. Borja queda libre de estos compromisos con este chat: (INFORM …) / (IF …)
NADA. Borja no tenía ningún compromiso así con este chat.
```

Los motivos de rechazo, en este orden: no sabe hacer eso o no le corresponde; no está claro qué tiene que hacer; no sabe a quién tendría que escribir; no sabe qué valores tiene que consultar; sería una cadena de compromisos demasiado larga; se comprometió a no hacerlo; ya lo tenía apuntado; tiene demasiados compromisos pendientes.

## 18. Ejecución en desarrollo: (15) Actuar, (16) Observar y (17) Refinar

> [!IMPORTANT]
> Los tres escenarios están creados en Make y **desactivados**. Hasta que se activen, Cumplir sigue ejecutando con sus acciones fijas ([sección 14](#14-11-cumplir-ejecución-de-compromisos)).

La ejecución nueva sigue el patrón **actuar, observar y refinar**. En lugar de una ruta fija por acción, el agente cumple cada compromiso paso a paso con sus herramientas y no lo da por cumplido hasta comprobarlo. Así, un compromiso puede pedir cualquier cosa que esas herramientas resuelvan.

```mermaid
flowchart LR
    CUM["(11) Cumplir<br/>reserva el compromiso"] --> ACT["(15) Actuar<br/>un paso con una herramienta"]
    ACT --> OBS["(16) Observar<br/>comprueba el paso"]
    OBS -- avanza --> ACT
    OBS -- falla --> REF["(17) Refinar<br/>decide qué cambiar"]
    REF -- corregir --> ACT
    OBS -- cumplido --> CS[("Compromisos")]
    OBS -- condición aún no --> CS
    REF -- aplazar o fallido --> CS
```

### Cómo se encadenan

- Se llaman en cadena y **sin esperarse**: cada uno responde «Accepted» y escribe su resultado en el almacén o llama al siguiente.
- Todos llevan el filtro «Token correcto» con el marcador `PON_AQUI_EL_TOKEN` y reenvían el token que reciben.
- **Límites por ejecución:** 5 pasos y 2 correcciones. Si una llamada de la cadena se pierde, el compromiso sigue reservado y Cumplir lo retoma a la media hora; al tercer intento, fallido.
- **Cuerpo común:** `token`, `clave` (la del compromiso), `agente`, `comprometido_con`, `destinatario`, `contenido`, `criterio`, `condicion`, `simbolos`, `origen`, `nivel`, `paso`, `correcciones`, `historial` y `correccion`. Si falta el criterio, se usa el contenido.

### (15) Actuar

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Data store · Get a record | Filtro «Token correcto»; el perfil del agente, que es la instrucción de sistema |
| 3 | Gemini (salida JSON con esquema) | Elige un solo paso: herramienta, parámetros y motivo |
| 4 | Router | Una ruta por herramienta y una final |

| Herramienta | Módulos | Qué hace |
| --- | --- | --- |
| `cotizar` | 10, 11 | Pide a Cotizar los precios de los símbolos |
| `recordar` | 12, 13 | Pide a Recuperar los 5 recuerdos más relevantes |
| `buscar` | 14, 15 | Gemini con búsqueda de Google: hasta 80 palabras con la fuente, o NO_ENCONTRADO |
| `calcular` | 16, 17 | Make Code ejecuta el código si pasa el mismo filtro de seguridad que la calculadora de Responder |
| `cartera` | 18, 19 | Lee la cartera del destinatario |
| `preguntar` | 20–25 | Busca los vetos de preguntar de ese chat. Si lo impiden, lo anota; si no, pregunta con Interpelar |
| `enviar` | 26–32 | Busca los vetos de informar de ese chat. Si lo impiden, lo anota; si no, envía el mensaje por Telegram y lo guarda en la memoria |
| *(final)* | 33, 34 | Pasa el resultado a Observar |

Un veto impide el paso si es de la acción entera o del primer valor del compromiso. Los vetos de tema ya los aplica Comprometer al crear el compromiso.

<details>
<summary>Prompt del módulo 3</summary>

```text
Eres {{1.agente}} y estás cumpliendo un compromiso. Decide el siguiente paso: uno solo.

Compromiso: «{{1.contenido}}»
Comprometido con: {{1.comprometido_con}}
Destinatario de los mensajes: {{ifempty(1.destinatario; "ninguno")}}
Condición: {{ifempty(1.condicion; "ninguna")}}
Valores implicados: {{ifempty(1.simbolos; "ninguno")}}
Criterio de cumplimiento: «{{ifempty(1.criterio; 1.contenido)}}»

Pasos ya dados y su resultado:
{{ifempty(1.historial; "ninguno")}}

Corrección que debes aplicar: {{ifempty(1.correccion; "ninguna")}}

Herramientas:
cotizar: consulta precios en Yahoo Finance. Rellena simbolos, símbolos de Yahoo Finance separados por comas (por ejemplo ITX.MC, SAN.MC, ^IBEX).
recordar: busca en tu memoria. Rellena consulta.
buscar: busca en internet. Rellena consulta.
calcular: hace una cuenta exacta. Rellena codigo con JavaScript que termine en return { resultado: ... }, sin acceso a red ni a ficheros.
cartera: lee la cartera del destinatario. No necesita parámetros.
preguntar: hace una pregunta al destinatario. Rellena texto con la pregunta.
enviar: escribe un mensaje al destinatario. Rellena texto con el mensaje, con tu voz, en un máximo de 80 palabras, con las cifras exactas y sin firma.

Reglas: si hay condición, comprueba primero con datos si se cumple. Usa preguntar o enviar solo si hay destinatario y solo cuando tengas lo necesario. No inventes datos: usa solo los de los pasos anteriores. Deja vacíos los campos que no use la herramienta. En motivo, explica en una frase por qué das este paso.

Ahora es {{AAAA-MM-DD a las HH:mm}}, hora de Madrid.
```

Esquema de salida: `{herramienta, simbolos, consulta, codigo, texto, motivo}`, con `herramienta` limitada a las siete.

</details>

### (16) Observar

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | El cuerpo común más `herramienta`, `parametros`, `motivo_paso` y `resultado` |
| 2 | Tools · Set variables | Filtro «Token correcto»; la comprobación sin modelo, el historial con el paso nuevo y el número del paso siguiente |
| 3 | Router | Vetado, fallo de la herramienta o juicio |
| R1 · 4, 5 | Data store · Get / Add/replace | Vetado por el destinatario: liberado, sin aviso |
| R2 · 6 | HTTP a Refinar | La herramienta no devolvió lo esperado |
| R3 · 7 | Gemini (salida JSON con esquema) | La herramienta respondió bien: Gemini da el veredicto |
| R3 · 8 | Router | Una ruta por veredicto |

**Comprobación sin modelo.** Una llamada bloqueada por el token cuenta como fallo, no como éxito.

| Herramienta | Vale si… |
| --- | --- |
| `cotizar` | Hay precios y no un «Accepted» |
| `recordar` | Hay memorias |
| `buscar` | Hay texto y no es NO_ENCONTRADO |
| `calcular` | Hay resultado |
| `cartera` | Siempre |
| `preguntar` | Interpelar responde «enviada» |
| `enviar` | Telegram devuelve el mensaje |

**Veredictos.**

| Veredicto | Módulos | Qué pasa |
| --- | --- | --- |
| `cumplido` | 9, 10, 11 | Cierra el compromiso. Si era una decisión propia, guarda lo conseguido como conocimiento |
| `avanza` | 12 / 13 | Con menos de 5 pasos, vuelve a Actuar a por el siguiente; con 5, pasa a Refinar |
| `condicion` | 14, 15 | Lo deja pendiente dentro de 2 horas, sin contarlo como fallo |
| `falla` | 16 | Pasa a Refinar |

<details>
<summary>Prompt del módulo 7</summary>

```text
Compromiso de {{1.agente}}: «{{1.contenido}}»
Condición: {{ifempty(1.condicion; "ninguna")}}
Criterio de cumplimiento: «{{ifempty(1.criterio; 1.contenido)}}»
Destinatario: {{ifempty(1.destinatario; "ninguno")}}

Pasos dados y sus resultados:
{{2.ob_hist}}

Decide, solo con estos datos:
cumplido: lo hecho ya cumple el criterio.
avanza: el último paso ha ido bien, pero falta algún paso para cumplir el criterio.
condicion: el compromiso depende de una condición y los datos muestran que todavía no se da.
falla: el último paso no sirve o ha salido mal.
En motivo, explica la decisión en una frase. En resumen, di en una frase lo conseguido, con las cifras.
```

</details>

### (17) Refinar

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | El cuerpo común más `motivo`, el fallo |
| 2 | Tools · Set variables | Filtro «Token correcto»; los contadores siguientes y el historial con el fallo |
| 3 | Router | Sin margen o con margen |
| R1 · 4, 5, 6 | Data store · Get / Add/replace / Add a record | **Sin margen** (2 correcciones o 5 pasos): fallido y, si era una promesa a un chat, el compromiso de avisarle |
| R2 · 7 | Gemini (salida JSON con esquema) | **Con margen:** corregir, aplazar o fallido |
| R2 · 8 | Router | Una ruta por decisión |

| Decisión | Módulos | Qué pasa |
| --- | --- | --- |
| `corregir` | 9 | Vuelve a Actuar con la instrucción como corrección |
| `aplazar` | 10, 11 | Lo deja pendiente dentro de los minutos que diga, entre 30 y 240 |
| `fallido` | 12, 13, 14 | Fallido y, si era una promesa a un chat, el aviso |

<details>
<summary>Prompt del módulo 7</summary>

```text
{{1.agente}} intenta cumplir este compromiso y algo ha fallado.
Compromiso: «{{1.contenido}}»
Criterio de cumplimiento: «{{ifempty(1.criterio; 1.contenido)}}»
Destinatario: {{ifempty(1.destinatario; "ninguno")}}

Pasos dados y sus resultados:
{{1.historial}}

Fallo: {{1.motivo}}
Correcciones ya intentadas: {{ifempty(1.correcciones; "0")}} de 2.

Decide:
corregir: el fallo tiene arreglo ahora. En instruccion, di exactamente qué hacer en el siguiente paso: otro símbolo, otra búsqueda, otra herramienta o preguntar al destinatario el dato que falta.
aplazar: conviene volver a intentarlo más tarde, por ejemplo porque el dato aún no está publicado. En minutos, cuántos, entre 30 y 240.
fallido: no tiene arreglo.
Deja vacíos los campos que no correspondan.
```

</details>

### Pendiente antes de activarlos

- [ ] **Interpelar** responde al final con lo que ha enviado. Hoy contesta «Accepted» y Observar no puede saber si la pregunta salió.
- [ ] **Cumplir** pasa a Actuar todo lo que no sea agendar o estudiar, con `paso` 1, `correcciones` 0 y el historial vacío, y deja de ejecutarlo él.
- [ ] **Compromisos** tiene el campo `criterio`, que redactan Percibir y Reflexionar y valida Comprometer. Con él, Comprometer deja de limitar los compromisos a una lista de acciones y rechaza lo que no pueda resolverse con las herramientas.
- [ ] **Token:** el real en los filtros de los tres y en la llamada de Cumplir a Actuar. Después, activarlos y probarlos.

## 19. Contratos de los webhooks

Cada escenario con webhook se invoca con un POST JSON (`Content-Type: application/json`). Todos exigen el campo `token`; sin él, o con uno distinto, el escenario no hace nada y Make responde 200 «Accepted».

| Función | URL | Entrada | Salida |
| --- | --- | --- | --- |
| Escribir (2) | `https://hook.eu1.make.com/<WEBHOOK_ESCRIBIR>` | `token`, `memoria`, `tipo`, `agente`, `chat_id`, `autor`; opcional `evidencias[]` | 200 `{"id", "importancia", "tipo"}` |
| Reflexionar (3) | `https://hook.eu1.make.com/<WEBHOOK_REFLEXIONAR>` | `token`, `agente`, `chat_id` | 202 `{"estado": "reflexion en curso", "agente"}` |
| Recuperar (4) | `https://hook.eu1.make.com/<WEBHOOK_RECUPERAR>` | `token`, `agente`, `chat_id`, `consulta`; opcional `k` (10) | 200 `{"memorias": [...], "texto", "texto_ids", "total_evaluadas"}` |
| Responder (6) | `https://hook.eu1.make.com/<WEBHOOK_RESPONDER>` | `token`, `agente`, `chat_id`, `mensaje`, `descripcion`; opcionales `usuario`, `modo` | 200 `{"respuesta"}` |
| Resumen del agente (7) | `https://hook.eu1.make.com/<WEBHOOK_RESUMEN>` | `token`, `agente`, `chat_id`, `modo` (ON u OFF) | 200 `{"resumen", "estado"}` |
| Estudiar (8) | `https://hook.eu1.make.com/<WEBHOOK_ESTUDIAR>` | `token`, `agente`, `chat_id`, `mensaje`; opcionales `usuario`, `descripcion` | «Accepted» |
| Interpelar (9) | `https://hook.eu1.make.com/<WEBHOOK_INTERPELAR>` | `token`, `agente`, `chat_id`, `motivo`; opcionales `usuario`, `contexto`, `descripcion` | «Accepted»; envía el mensaje por Telegram |
| Cotizar (13) | `https://hook.eu1.make.com/<WEBHOOK_COTIZAR>` | `token`, `simbolos` (hasta 5, separados por comas) | 200 `{"precios": [{"simbolo", "nombre", "precio", "cierre_anterior", "moneda", "hora", "error"}]}` |
| Comprometer (14) | `https://hook.eu1.make.com/<WEBHOOK_COMPROMETER>` | `token`, `acto` (REQUEST, UNREQUEST o DECISION), `agente`, `chat`, `accion`, `contenido`; opcionales `dia` (AAAA-MM-DD), `hora` (HH:mm), `fin` (AAAA-MM-DD), `condicion`, `simbolos`, `libera` (claves separadas por comas), `repite`, `vetado_por`, `origen`, `nivel` | 200, texto: ACEPTADO, RECHAZADO, LIBERADO o NADA |
| Actuar (15) | `https://hook.eu1.make.com/<WEBHOOK_ACTUAR>` | El cuerpo común (sección 18) | «Accepted»; desactivado |
| Observar (16) | `https://hook.eu1.make.com/<WEBHOOK_OBSERVAR>` | El cuerpo común más `herramienta`, `parametros`, `motivo_paso` y `resultado` | «Accepted»; desactivado |
| Refinar (17) | `https://hook.eu1.make.com/<WEBHOOK_REFINAR>` | El cuerpo común más `motivo` | «Accepted»; desactivado |

Ejemplos:

```json
POST Escribir
{"token": "<TOKEN>", "memoria": "Ana le dijo a Borja: «Tengo una reunión con el banco en mayo»", "tipo": "observacion", "agente": "Borja", "chat_id": "123456789", "autor": "usuario"}

POST Recuperar
{"token": "<TOKEN>", "agente": "Borja", "chat_id": "123456789", "consulta": "¿Qué sabe Borja de los planes de Ana?", "k": 5}

POST Responder
{"token": "<TOKEN>", "agente": "Borja", "chat_id": "123456789", "mensaje": "¿Qué tal el día?", "usuario": "Ana", "descripcion": "Borja es un asesor financiero..."}

POST Cotizar
{"token": "<TOKEN>", "simbolos": "ITX.MC,SAN.MC,^IBEX"}

POST Comprometer
{"token": "<TOKEN>", "acto": "REQUEST", "agente": "Borja", "chat": "123456789", "accion": "informar", "dia": "2026-11-05", "hora": "10:00", "simbolos": "ITX.MC", "contenido": "Contarle cómo ha abierto Inditex", "origen": "prueba", "nivel": 0}

POST Comprometer
{"token": "<TOKEN>", "acto": "UNREQUEST", "agente": "Borja", "chat": "123456789", "accion": "informar", "simbolos": "ITX.MC", "libera": "a1b2c3d4e5f6"}
```

## 20. Fidelidad al paper

La memoria sigue el paper en lo que la define: el registro único, la importancia asignada por el LLM, la fórmula de recuperación y el ciclo de reflexión con citas. Las desviaciones vienen de pasar de un mundo simulado a una conversación de chat y de los límites de Make y Airtable.

| Elemento | Paper | Human_proxy | Valoración |
| --- | --- | --- | --- |
| Objeto de memoria | Descripción, creación, último acceso | Memoria, Creado, Ultimo_acceso + metadatos | Fiel |
| Importancia | LLM, escala 1–10 anclada | Mismo prompt traducido | Fiel |
| Recencia | 0,995 por hora de juego desde el último acceso | 0,995 por hora real | Fiel en la forma; cambia la escala temporal |
| Relevancia | Coseno entre embeddings | Coseno, Gemini Embedding 2, 768 dimensiones | Fiel |
| Combinación | Min-max a [0, 1] y suma con α = 1 | Igual | Fiel |
| Candidatos | Toda la memoria del agente | Los 300 registros más recientes del chat | Desviación por límite técnico (sección 22) |
| Cuántos se recuperan | Los que caben en el contexto | k fijo, 10 por defecto | Simplificación razonable |
| Disparo de reflexión | Suma de importancia reciente > 150 | Suma de observaciones no reflejadas > 150 | Fiel |
| Preguntas focales | 3, a partir de los 100 registros más recientes | Igual | Fiel |
| Insights | 5, con «(because of 1, 5, 3)» | 5, con IDs reales en un JSON con esquema | Fiel y más robusto |
| Árbol de reflexiones | Las reflexiones pueden citar reflexiones | Igual, vía Evidencias / Citada_por | Fiel |
| Resumen del agente | Generado desde la memoria y cacheado | Las tres consultas del paper, caché de 7 días | Fiel; la descripción fija sigue siendo la base |
| Diálogo | Recupera lo que sabe del interlocutor antes de hablar | Resumen de la relación, caché de 24 h; además, sus compromisos y la decisión sobre el mensaje | Fiel |
| Planificación | Plan diario, por horas y por tramos de 5–15 minutos | No hay plan: la agenda del día se convierte en compromisos de estudiar cada evento | Sustituida por los compromisos de Shoham (sección 17) |
| Percibir y reaccionar | El agente percibe su entorno en cada paso y decide si reacciona y replanifica | Percibe cada mensaje; Cumplir ejecuta cada media hora lo vencido; la prensa se lee dos veces al día | Parcial: reacciona a mensajes y compromisos, no a cada cambio del entorno |
| Ámbito de la memoria | Un agente, un mundo | Un agente por chat | Decisión deliberada: el agente no mezcla lo que le cuentan usuarios distintos |
| Entorno y otros agentes | Mundo simulado con espacios y varios agentes | No existe | Inherente al canal |

Estudiar, Interpelar, Noticias, Cotizar, la cartera y la calculadora no están en el paper; son extensiones para un agente que conversa con personas reales y necesita información verificable y cifras exactas.

Respecto a Shoham, el sistema es fiel en lo esencial. El ciclo tiene sus dos fases: percibir y comprometerse (Responder y Comprometer) y ejecutar (Cumplir). Las reglas de compromiso son explícitas y deterministas. Los compromisos tienen momento, las decisiones son compromisos con uno mismo y hay compromisos condicionales y de abstención. Solo se acepta lo que el agente sabe hacer, incluido su horario. El usuario puede liberar al agente (UNREQUEST), y lo prometido a otro no se abandona en silencio: si falla o caduca, el agente avisa.

Se aparta en tres cosas:

1. Los mensajes los clasifica un LLM, no un analizador formal.
2. Las condiciones las evalúa Gemini con las cotizaciones o la búsqueda de Google delante, no un operador lógico sobre las creencias.
3. El ciclo va cada 30 minutos y solo en horario de trabajo, y no hay mensajes entre agentes.

## 21. Costes

Con un usuario, el sistema gasta unos 3.400–3.900 créditos de Make al mes antes de que nadie converse, según una estimación hecha con los créditos medidos de cada pieza: Cumplir (≈ 550 en vueltas sin nada que hacer, ≈ 480 en la agenda diaria y ≈ 1.600 en el estudio de eventos), Noticias (≈ 750–1.250) y el Despertador (≈ 60). Hasta noviembre de 2026, con el plan diario, Percibir cada hora y Noticias cuatro veces al día, eran unos 6.400–7.400. En el coste manda Make, no Gemini: los créditos son el 85–90 % de cada ejecución.

Precios aplicados: Make Core, 10.000 créditos por 10,59 $ al mes (0,00106 $ por crédito; 0,0009 $ en anual); Gemini 3.8 Flash, 0,75 $ por millón de tokens de entrada y 3,75 $ por millón de salida, razonamiento incluido, hasta el 31 de diciembre de 2026 y el doble desde el 1 de enero de 2027; búsqueda de Google gratis hasta 5.000 consultas al mes. Conversión a euros al cambio del Banco de España del 19 de octubre de 2026, 1,1225 $ por euro. Los créditos se midieron en el historial de cada escenario; los tokens, en el inspector de Make para Responder, Escribir y Estudiar, y el resto se estima por el tamaño del prompt.

### Coste unitario por escenario

Solo lo que gasta el propio escenario; las llamadas a otros escenarios se suman en la tabla siguiente.

| Escenario | Créditos | Make (€) | Gemini (€) | Gemini |
| --- | --- | --- | --- | --- |
| (1) Gateway | 9–10 | 0,009 | 0 | — |
| (2) Escribir | 7 | 0,0066 | 0,0012 | medido |
| (3) Reflexionar | 25 | 0,024 | ≈ 0,009 | estimado |
| (4) Recuperar k = 5 / 10 / 15 | 12 / 17 / 18 | 0,011 / 0,016 / 0,017 | < 0,0001 | medido |
| (6) Responder, sin petición / con petición o liberación | 20 / 22 | 0,019 / 0,021 | ≈ 0,003 | medido el 8/11, con la relación en caché |
| (6) Responder, más ruta de cartera / cotizaciones / cálculo | + 4 / + 6 / + 4 | + 0,004 / + 0,006 / + 0,004 | ≈ 0,001 cada una | medido en créditos |
| (7) Resumen OFF / caché / regenerando | 2 / 3 / 10 | 0,002 / 0,003 / 0,009 | 0 / 0 / ≈ 0,006 | estimado |
| (8) Estudiar sin investigar / ya sabido / investigando | 3 / 5 / 6 | 0,003 / 0,005 / 0,006 | 0,001 / ≈ 0,003 / 0,0047 | medido |
| (9) Interpelar | 8 | 0,0076 | ≈ 0,002 | estimado |
| (10) Despertador | 2 | 0,0019 | 0 | — |
| (11) Cumplir, vuelta sin nada vencido | 1 | 0,0009 | 0 | medido |
| (11) Cumplir, por compromiso: reservar, leer el perfil y marcar cumplido | 3 | 0,0028 | 0 | medido |
| (11) Cumplir, agendar con 4 eventos | 13 | 0,012 | ≈ 0,004 | estimado |
| (11) Cumplir, estudiar: por evento + por chat | 5 + 4 | 0,005 + 0,004 | ≈ 0,004 + ≈ 0,0025 | medido: 22 créditos con dos estudios, uno con dos chats |
| (11) Cumplir, informar con cotizaciones / cotizar | 3 / 2 | 0,003 / 0,002 | ≈ 0,001 / 0 | medido |
| (12) Noticias, por ejecución + por chat + por noticia | 5 + 5 + 1 | 0,005 + 0,005 + 0,001 | ≈ 0,0015 por chat | medido en créditos |
| (13) Cotizar, por llamada + por símbolo | 4 + 1 | 0,004 + 0,001 | 0 | medido |
| (14) Comprometer, REQUEST o DECISION / UNREQUEST | 11–12 / 13 | 0,011 / 0,012 | 0 | medido el 8/11 |

La calculadora usa el módulo Make Code, que cobra 2 créditos por segundo de ejecución; las dos calculadoras tardan menos de un segundo, así que cuestan 2 créditos cada vez. En las llamadas medidas manda el razonamiento interno de Gemini: la importancia de Escribir devuelve 1 token y razona 331; la investigación de Estudiar recibe 521 tokens, escribe 120 y razona 880.

### Coste por operación completa

| Operación | Escenarios que encadena | Créditos | Total (€) |
| --- | --- | --- | --- |
| Mensaje de Telegram con cachés vigentes | Gateway, Responder con Percibir, Recuperar k = 5, Resumen en caché, 2 × Escribir, Estudiar | ≈ 62 | ≈ 0,065 |
| Pregunta de cotizaciones o de cartera | + ruta de cotizaciones y Cotizar con 2–3 símbolos | + 12–13 | + 0,013 |
| Mensaje que registra posiciones o pide un cálculo | + su ruta | + 4 | + 0,005 |
| Regenerar el resumen (una vez a la semana por chat) | Resumen con 3 × Recuperar k = 15 | ≈ 55 | ≈ 0,06 |
| Regenerar la relación (una vez al día por chat) | Recuperar k = 10, Gemini | ≈ 19 | ≈ 0,02 |
| Estudiar con investigación | + Recuperar, Escribir | + 22 | + 0,026 |
| Interpelación | Interpelar, Escribir | ≈ 15 | ≈ 0,017 |
| Reflexión | Reflexionar, 3 × Recuperar k = 10, 5 × Escribir | ≈ 111 | ≈ 0,12 |
| Agenda diaria del agente financiero | Despertador, Cumplir (agendar), Escribir | ≈ 22 | ≈ 0,026 |
| Estudio de un evento, primer usuario | Cumplir, Escribir | ≈ 18 | ≈ 0,026 |
| Cada usuario más por evento | Cumplir, Escribir | + 11 | + 0,014 |
| Encargo aceptado y cumplido con un mensaje | Responder (+2), Comprometer, Cumplir, Cotizar, Escribir | ≈ 33 | ≈ 0,038 |
| Noticia guardada | Noticias, Escribir | ≈ 8 | ≈ 0,009 |
| Conversación de 20 mensajes | 20 mensajes, regeneración diaria, 2 investigaciones, 1 interpelación, 0–1 reflexión | 1.290–1.400 | 1,35–1,48 |

### Coste mensual

| Concepto | Créditos al mes | Coste (€) |
| --- | --- | --- |
| Cumplir, ≈ 550 vueltas al mes | ≈ 550 | ≈ 0,52 |
| Agenda diaria, 22 laborables | ≈ 480 | ≈ 0,52 |
| Estudio de eventos, 1 usuario, ≈ 90 eventos | ≈ 1.600 | ≈ 1,90 |
| Cada usuario más en el estudio de eventos | ≈ 1.000 | ≈ 1,20 |
| Noticias, 44 ejecuciones al mes con uno o dos usuarios | ≈ 750–1.250 | ≈ 0,80–1,35 |
| Cada conversación de 20 mensajes | ≈ 1.350 | ≈ 1,40 |

Un plan de 10.000 créditos da para el gasto fijo y unas cuantas conversaciones al mes. Lo que más pesa del gasto fijo es el estudio de eventos, que crece con cada usuario: limitar la agenda a 2 o 3 eventos lo recorta en proporción, y llevar Cumplir a una vuelta por hora ahorra unos 275 créditos al mes a cambio de reaccionar más tarde. Juntar las tres lecturas de variables de Responder en un único módulo Get multiple variables ahorra 2 créditos por mensaje. Si Gemini duplica su precio en 2027, un mensaje pasa de ≈ 0,058 € a ≈ 0,063 €.

Fuentes: [precios de la API de Gemini](https://ai.google.dev/gemini-api/docs/pricing), [Gemini 3.8 Flash, Layer3 Labs](https://www.layer3labs.io/guides/gemini-3-8-flash-pricing), [Gemini Embedding 2, Maxim](https://www.getmaxim.ai/bifrost/llm-cost-calculator/provider/gemini/model/gemini-embedding-2), [precios de Make](https://www.make.com/en/pricing), [Make Code](https://apps.make.com/code.md), [precios de Make, Jetadmin](https://www.jetadmin.io/blog/make-pricing/), [cambio oficial del euro, BOE 20/10/2026](https://www.boe.es/boe/dias/2026/10/20/pdfs/BOE-A-2026-21865.pdf).

## 22. Limitaciones y riesgos

### Escala de la memoria

Recuperar solo evalúa los 300 registros más recientes del chat. A partir de unos 150 intercambios más sus reflexiones, los recuerdos más antiguos dejan de ser candidatos aunque sean importantes, cosa que el paper no hace. El techo existe porque todos los embeddings viajan como texto al módulo de código, del orden de 2,7 MB por llamada. Las salidas, de menor a mayor esfuerzo: subir el techo y vigilar tiempos; prefiltrar por importancia (por ejemplo, las 200 más recientes más las 100 más importantes); o mover los vectores a una base vectorial y dejar Airtable como registro legible.

El volumen total lo limita el plan de Airtable: 1.000 registros por base en Free, 50.000 en Team y 125.000 en Business ([Jetadmin, julio de 2026](https://www.jetadmin.io/blog/airtable-pricing-in-2026-plans-real-costs-and-when-to-upgrade/)). Con dos registros por turno, más cinco por reflexión y lo que guardan Cumplir y Noticias, el límite de Free se alcanza pronto.

### Errores

Make desactiva un escenario secuencial en cuanto una ejecución falla, y uno en paralelo tras varios errores seguidos. Gateway, Estudiar, Interpelar y Cumplir llevan un manejador Ignore en todos los módulos que pueden fallar. Noticias y Cotizar lo llevan en sus módulos externos, y Responder en los de sus rutas de percepción, cartera, cotizaciones y cálculo, de modo que un fallo ahí deja al agente sin ese dato pero no sin respuesta. Escribir, Reflexionar, Recuperar, el núcleo de Responder, el Resumen del agente, el Despertador y Comprometer no lo llevan, con estas consecuencias:

1. Si Gemini o Airtable fallan durante Responder, el usuario no recibe respuesta; el Gateway sigue activo.
2. Si Gemini devuelve la importancia con texto alrededor en la primera línea («Puntuación: 7»), la conversión de Airtable puede fallar y ese recuerdo se pierde.
3. Si el LLM inventa un ID en las evidencias de una reflexión, falla la actualización del módulo 14 de Reflexionar y no se escriben las reflexiones restantes de esa tanda.
4. Como el Gateway responde antes de memorizar, un fallo en Escribir no lo ve el usuario: la conversación sigue, con huecos en la memoria.
5. Si Comprometer no da una decisión reconocible, Borja dice que en ese momento no puede apuntarlo y pide que se lo repitan; no promete nada ni se niega. Si falla Percibir o la llamada, contesta sin decisión y tampoco promete nada.
6. Cumplir no pierde compromisos: reserva antes de actuar y marca después. Pero una llamada a otro escenario que el filtro del token bloquea devuelve 200, así que Cumplir la da por buena y la marca como cumplida.

### Latencia

Un mensaje tarda unos 10–20 segundos en recibir respuesta: embedding y búsqueda en Recuperar, código y generación. Con el procesamiento secuencial, los mensajes en ráfaga se encolan.

> [!WARNING]
> **Sin créditos.** Si Make se queda sin créditos, los escenarios se paran y las llamadas a sus webhooks quedan en cola con la respuesta «Accepted». Borja dice entonces que no puede apuntarlo. Al volver los créditos, Make procesa la cola: algo pedido durante el corte puede quedar registrado o liberado después, sin que Borja lo confirme.

### Seguridad y privacidad

**Token.** Cada llamada entre escenarios lleva un campo `token` en el cuerpo, y cada escenario con webhook lo compara en el filtro «Token correcto», justo después del webhook. Si no coincide, el escenario no ejecuta nada. Responder y Reflexionar reenvían a Comprometer el token que recibieron, sin escribirlo.

El token es el mismo en todas las llamadas. Si se expone, cámbialo a la vez en todos los filtros y en todos los cuerpos. No lo guardes en documentos ni en blueprints: los de este repositorio lo sustituyen por `<TOKEN>`. Make permite además exigir una clave por cabecera en cada webhook (`x-make-apikey`), una alternativa más robusta si el sistema se abre a terceros.

**URLs.** Quien tenga la URL de un webhook puede invocarlo; con el token, ya no puede hacer nada sin conocerlo. Aun así, no publiques las URLs: si se filtra una, regenera ese webhook y actualiza los módulos que lo llaman.

**Código.** La ruta de cálculo ejecuta código que escribe el modelo. Lo contienen el filtro de patrones, el entorno aislado de Make Code y su límite de 30 segundos, pero el filtro es una lista negra, no una garantía. Si el bot se abre a desconocidos, vigila el historial de ese módulo o desactiva la ruta.

**Datos.** Airtable guarda literalmente lo que cada usuario dice al bot; Cartera, sus inversiones; Compromisos, lo que el agente le ha prometido. Si lo usan otras personas, deben saberlo.

### Otros

En un grupo de Telegram todos los miembros comparten chat y, por tanto, memoria; las observaciones sí distinguen a cada uno por su nombre. Telegram limita los mensajes a 4.096 caracteres. El límite de 10 mensajes en 10 minutos cuenta solo mensajes ya guardados.

La cartera depende de que el modelo interprete bien lo que cuenta el usuario. Una frase ambigua («he metido algo más en Inditex») registra la posición con cantidad 0, y una corrección («no, eran 40») se aplica como un cambio más. El resumen y el JSON están a la vista en el almacén Cartera para revisarlos o corregirlos a mano. Las rutas se activan por expresiones regulares: un mensaje que pregunta por la cartera sin ninguna de las palabras previstas («¿y lo mío qué tal?») no consulta cotizaciones.

Las cifras del cálculo libre son exactas para el planteamiento que elige el modelo, pero el planteamiento puede no ser el que el usuario tenía en mente (capitalización mensual o anual, aportaciones al principio o al final del periodo). El planteamiento llega al agente, aunque con el límite de 20 palabras de Telegram no siempre lo menciona.

Percibir depende de que Gemini clasifique bien el mensaje. Si no ve una petición, Borja no se compromete; si ve una donde no la hay, Comprometer la acepta o la rechaza con las reglas, y Borja lo dice en su respuesta. Al liberar, Percibir señala claves: si se equivoca de clave, libera otro compromiso del mismo chat, nunca uno de otro chat. Las condiciones de precio las evalúa Gemini con las cotizaciones delante y se revisan cada 2 horas, así que un aviso del tipo «si baja de 40» puede llegar tarde o no llegar si el precio cruza y vuelve entre dos revisiones. A los 7 días caduca, y Borja lo dice.

Fuera de los laborables de 8:00 a 20:30 Cumplir no actúa. Comprometer aplaza lo que cae fuera y Borja lo avisa al aceptarlo.

El conocimiento que guardan Estudiar, Cumplir y Noticias compite en desventaja en la recuperación: la importancia mide la carga emocional, como en el paper, y un dato fiscal o de mercado puntúa bajo. Además, ese conocimiento es por chat. Interpelar envía siempre por el mismo bot; con varios agentes y bots habría que pasarle cuál usar.

### Mejoras posibles

1. Terminar la ejecución por pasos (sección 18). Con Observar, ninguna llamada bloqueada por el token se da por cumplida.
2. Evaluar las condiciones de precio con una comparación numérica en lugar de con Gemini, y revisarlas con más frecuencia.
3. Archivar los compromisos en estado final para no llenar el almacén.
4. Dar una importancia fija alta al tipo conocimiento para que compita en la recuperación.
5. Añadir manejadores de error al resto de escenarios y extraer el número de la importancia con una expresión regular.
6. Convertir divisas en la valoración de la cartera para dar un total único.

## 23. Operación

### Pruebas aisladas

Cada función se puede probar sola con un POST a su webhook, con el token, sin pasar por Telegram. Conviene usar un chat_id de prueba, como `PRUEBA`, para no mezclar esos recuerdos con conversaciones reales. Las pruebas de Responder con posiciones dejan además un registro en Cartera y otro en la caché de relaciones, que hay que borrar después. El estudio de eventos de Cumplir y Noticias recorren los chats que tienen mensajes de usuario en Airtable, así que un chat de prueba que solo haya pasado por Responder no les afecta.

Para probar los compromisos, escribe a Borja una petición («avísame mañana a las 10 de cómo abre el Ibex») y mira en el historial de Responder qué devolvió el módulo 50, en el de Comprometer qué decidió, y en el almacén el registro nuevo. También se puede llamar a Comprometer directamente con un agente de prueba que tenga perfil en la caché (`perfil|PRUEBA`) y pulsar Run once en Cumplir. Un destinatario inventado, como `PRUEBA`, hace que el envío por Telegram falle sin escribir a nadie; con tres fallos el compromiso queda fallido. Para retirar compromisos de prueba, pon su estado en liberado desde el almacén, sin borrarlos.

**Prueba de referencia.** Recorre todas las reglas de compromiso. Manda cada mensaje después de la respuesta del anterior:

| # | Mensaje | Resultado esperado |
| --- | --- | --- |
| 1 | Avísame el lunes a las 10 de cómo abre Inditex | Lo acepta |
| 2 | Recuérdame el lunes a las 10 lo de Inditex | Lo rechaza: ya lo tenía apuntado |
| 3 | Si el Santander baja de 4 euros esta semana, avísame | Lo acepta con condición y fecha de fin |
| 4 | No me avises más de Inditex | Acepta el veto y anula el aviso del lunes |
| 5 | Olvida lo del Santander | Lo libera |
| 6 | Ya puedes volver a avisarme de Inditex | Retira el veto |

Al final no queda nada pendiente. Para ver qué decidió Comprometer, abre su ejecución y el módulo de respuesta que se ejecutó: 34 o 35 en un alta, 22 en una liberación.

> [!CAUTION]
> El detalle de la llamada 51 de Responder muestra el token en el cuerpo. No compartas esa pantalla.

Para probar la recuperación, llama a Recuperar y revisa en `memorias` las tres componentes de cada resultado: si la relevancia es 0 en todos, los embeddings no se están leyendo bien; si la recencia manda siempre, hay pocos recuerdos o todos tienen la misma importancia. Para provocar una reflexión sin esperar al umbral, llama a Reflexionar con `{"agente", "chat_id"}`. Para probar la voz del personaje sin escribir memoria, llama a Responder. Para probar las cotizaciones, llama a Cotizar con varios símbolos, incluido uno inexistente.

### Dónde mirar cuando algo falla

Sigue el orden del flujo. Primero el historial del Gateway: si la ejecución no aparece, Telegram no está entregando; si se para en un filtro, era un comando, un mensaje sin texto o el límite de frecuencia. Después Responder y Recuperar, y por último Escribir y Reflexionar. Si el agente no hace algo que prometió, mira el registro en Compromisos (su estado e intentos) y después los historiales de Responder, Comprometer y Cumplir. Si un escenario no hace nada y su historial muestra que el filtro «Token correcto» no pasó, el token de quien llama no coincide. Make guarda el detalle de cada módulo con sus datos de entrada y salida.

### Al modificar escenarios

Al pegar un blueprint completo (Paste blueprint from clipboard), Make restablece la programación de los escenarios con webhook a «Every 15 minutes»: hay que volver a poner Immediately antes de guardar. Los escenarios programados conservan su programación. Pegar un blueprint conserva los filtros y los cuerpos tal como estaban, incluido el token. Un blueprint no crea webhooks: se crean a mano en el módulo disparador y, para los genéricos, hay que enviar una llamada de ejemplo para que Make detecte la estructura.

Si se añade una llamada nueva entre escenarios, hay que actualizar a mano la nota del webhook de destino.

Al reactivar un escenario que Make desactivó por un error, aparece la opción de procesar o borrar los datos que quedaron en la cola del webhook. Si el error estaba en esos datos, procésalos solo después de corregir el escenario.

### Mantenimiento de la memoria

Borrar filas de Memory Stream es seguro: no hay contadores derivados salvo Reflejada, y Citada_por se recalcula solo. Para reiniciar a un agente en un chat basta con borrar sus filas por Agente y Chat_ID. Cambiar el modelo o la dimensión de los embeddings obliga a recalcular el campo Embedding de todas las filas. Para trasladar recuerdos de una conversación a otra se cambia su Chat_ID. Los compromisos se pueden editar a mano en el almacén. Para anular uno, cambia su estado a liberado en lugar de borrarlo.
