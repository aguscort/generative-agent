# Human_proxy · Documentación técnica

Human_proxy implementa en Make.com la arquitectura de *Generative Agents: Interactive Simulacra of Human Behavior* (Park et al., 2023, [arXiv 2304.03442](https://arxiv.org/abs/2304.03442)) para agentes que conversan por Telegram, y les da iniciativa con los compromisos de la programación orientada a agentes de Shoham (*Agent-oriented programming*, Artificial Intelligence 60, 1993). Son 13 escenarios activos: los de memoria y diálogo del paper (Gateway, Escribir, Reflexionar, Recuperar, Responder y Resumen del agente), los tres del sistema de compromisos (Despertador, Comprometer y Percibir, que hace de intérprete) y cuatro extensiones propias para un asesor financiero que conversa con personas reales: estudio de dudas, preguntas por iniciativa propia, lectura de prensa y cotizaciones, además de la calculadora de Responder. Planificar y la versión anterior de Percibir siguen en Make, desactivados y marcados como obsoletos.

## 1. Visión general

El paper describe un agente con capacidades encadenadas sobre un único registro de memoria, el *memory stream*. Percibe y almacena cada experiencia como una observación en lenguaje natural, con su marca temporal y una puntuación de importancia. Recupera los recuerdos pertinentes combinando lo reciente, lo importante y lo relevante. Reflexiona periódicamente, sintetizando observaciones en conclusiones que vuelven al mismo registro. Y planifica su día, reaccionando a lo que percibe.

Human_proxy conserva la memoria, la recuperación y la reflexión del paper, pero no su planificación. La iniciativa del agente sale de la programación orientada a agentes de Shoham, donde el estado mental de un agente incluye sus compromisos y una decisión no es más que un compromiso consigo mismo. Cualquier actividad del bot puede crear un compromiso: una promesa a un usuario, la decisión de estudiar un dato cuando se publique o la orden diaria de revisar la agenda. Cada uno queda guardado con su momento en el almacén Compromisos, y un intérprete lo revisa cada media hora y ejecuta los que han vencido. Un plan diario es, en esos términos, un conjunto de decisiones con hora, así que no hace falta un sistema aparte (sección 17).

El repositorio incluye como ejemplo un agente, Borja: un asesor financiero ficticio de una firma de hedge funds del Madrid de los 90. Su personalidad está en un único sitio, su descripción, guardada en un almacén de datos. Lo propio de su oficio está en la acción «agendar» del intérprete, que cada mañana convierte la agenda de eventos financieros del día en compromisos de estudiar cada evento, en Noticias, en Cotizar y en las rutas de cartera, cotizaciones y cálculo de Responder. Para un agente de otro ámbito basta con su descripción: no tendrá agenda financiera, pero conserva la memoria, la reflexión y los compromisos que nazcan de sus conversaciones, y las rutas financieras de Responder solo se activan si el mensaje habla de dinero.

| Escenario | Disparo | Función |
| --- | --- | --- |
| (1) Gateway Telegram | Mensaje en Telegram | Aplica el límite de frecuencia, pide la respuesta, la envía, la memoriza y lanza Estudiar |
| (2) Memoria: Escribir | Webhook | Puntúa, vectoriza y guarda un recuerdo; si contiene una promesa o una decisión, la manda a Comprometer; dispara la reflexión si toca |
| (3) Memoria: Reflexionar | Webhook | Genera preguntas focales, recupera evidencias y guarda 5 reflexiones |
| (4) Memoria: Recuperar | Webhook | Devuelve los k recuerdos con mejor puntuación para una consulta |
| (6) Responder | Webhook | Construye el prompt de diálogo y genera la respuesta; registra la cartera, consulta cotizaciones y hace cálculos cuando el mensaje lo pide |
| (7) Resumen del agente | Webhook | Resume quién es el agente según su memoria, con caché de 7 días e interruptor ON/OFF |
| (8) Estudiar | Webhook | Detecta peticiones profesionales, investiga en Google y guarda conocimiento |
| (9) Interpelar | Webhook | Escribe al usuario por iniciativa propia cuando le falta un dato |
| (10) Despertador | Programado, diario a las 5:00 | Los laborables deja en Compromisos la orden «agendar» para las 8:00 |
| (11) Percibir: intérprete de compromisos | Programado, cada 30 minutos los laborables de 8:00 a 20:30 | Ejecuta los compromisos vencidos (agendar, estudiar un evento, informar, preguntar, investigar o cotizar) y a las 20:00 cierra el día |
| (12) Noticias | Programado, laborables a las 8:00 y a las 16:00 | Lee titulares de prensa económica y guarda los que afectan a la cartera de cada usuario, con su implicación |
| (13) Cotizar | Webhook | Devuelve precio, cierre anterior, moneda y hora de hasta 5 valores o índices |
| (14) Comprometer | Webhook | Valida una orden y la guarda en Compromisos con su momento |

Los números son los de esta documentación; en Make, el intérprete se llama «[Human_proxy] (07/14) | Percibir: intérprete de compromisos» y Comprometer, «(14/14)». Planificar y la versión anterior de Percibir, que leía el plan cada hora, conservan sus escenarios con el prefijo «[OBSOLETO]», desactivados; ningún escenario los llama.

Los modelos son Gemini 3.8 Flash para todo el razonamiento en lenguaje natural y Gemini Embedding 2, con 768 dimensiones, para los vectores. Las cuentas no las hace el modelo: las ejecuta el módulo Make Code en JavaScript. La memoria vive en Airtable, la cartera y las cachés en almacenes de datos de Make, y las cotizaciones salen de Yahoo Finance.

## 2. Arquitectura

Cada escenario es una función: un webhook de entrada, un cuerpo JSON con sus parámetros y una respuesta JSON, o un «Accepted» inmediato si trabaja en segundo plano. Solo el Gateway recibe de Telegram. Solo Escribir crea recuerdos y solo Recuperar hace la búsqueda puntuada; los demás hacen operaciones simples en la tabla, como contar mensajes, leer el historial o leer los compromisos pendientes.

Un mensaje recorre el sistema así:

1. El Gateway lee el perfil del agente, comprueba el límite de frecuencia y llama a Responder.
2. Responder pide el resumen del agente, lee lo que sabe del interlocutor, lee sus compromisos pendientes de hoy, recupera los recuerdos relevantes y el historial y lee la cartera del usuario. Si el mensaje comunica posiciones, las registra; si pregunta por precios, llama a Cotizar y valora la cartera; si pide una cuenta, la calcula. Con todo ello genera la respuesta.
3. El Gateway la envía a Telegram y guarda el mensaje y la respuesta como dos observaciones.
4. El Gateway pasa el mensaje a Estudiar en segundo plano.
5. Cada escritura suma la importancia de las observaciones aún no reflejadas y, por encima de 150, lanza Reflexionar.
6. Si el recuerdo guardado contiene una promesa o una decisión del agente («te aviso mañana de cómo cierra»), Escribir la manda a Comprometer, que la deja en Compromisos hasta que venza.

```mermaid
flowchart LR
    TG([Telegram · mensaje]) --> GW["(1) Gateway"]
    DSP["(10) Despertador<br/>diario 5:00"] -. agendar .-> CS
    CS[("Compromisos")] -.-> PER["(11) Percibir<br/>intérprete, cada 30 min"]
    PER --> COT
    PER --> EST
    PER --> INT
    PER --> ESC
    PER --> TGO
    GW --> RS["(6) Responder"]
    RS --> COT["(13) Cotizar"]
    COT -.-> YF([Yahoo Finance])
    NOT["(12) Noticias<br/>2 veces al día"] --> ESC
    RSS([Google News · RSS]) -.-> NOT
    GW --> EST["(8) Estudiar"]
    GW --> ESC
    RS --> RES["(7) Resumen<br/>del agente"]
    RS --> REC
    RS -.-> CS
    RES --> REC
    EST --> INT["(9) Interpelar"]
    EST --> REC
    EST --> ESC
    INT --> ESC
    INT --> TGO([Telegram · mensaje propio])
    ESC["(2) Escribir"] --> CMP["(14) Comprometer"]
    CMP -.-> CS
    ESC --> REF["(3) Reflexionar"]
    REF --> REC["(4) Recuperar"]
    REF -- reflexiones --> ESC
    ESC -.-> AT[(Airtable<br/>Memory Stream)]
    REC -.-> AT
```

Las flechas continuas son llamadas a webhook y las discontinuas, el acceso a la memoria o a fuentes externas. El diagrama solo muestra a Escribir y Recuperar tocando la tabla porque son los únicos que crean recuerdos y hacen la búsqueda puntuada; los demás escenarios también leen de Airtable para tareas simples, como contar mensajes. Compromisos es un almacén de datos de Make: lo escriben Comprometer, el Despertador y el intérprete, y lo leen el intérprete y Responder.

Sin que nadie escriba, el Despertador deja a las 5:00 la orden de revisar la agenda, el intérprete revisa los compromisos cada media hora y Noticias lee la prensa dos veces al día. Esta descomposición permite probar cada función sola enviando un JSON a su URL, añadir canales con un Gateway nuevo y mantener las reglas de la memoria en un único sitio.

| Escenario | Respuesta al llamante | Procesamiento |
| --- | --- | --- |
| (1) Gateway | No aplica | Secuencial, para respetar el orden de los mensajes |
| (2) Escribir | Síncrona, en cuanto el registro existe | Paralelo |
| (3) Reflexionar | 202 inmediato | Paralelo |
| (4) Recuperar | Síncrona; actualiza accesos después | Paralelo |
| (6) Responder | Síncrona | Paralelo |
| (7) Resumen del agente | Síncrona, menos de 1 s; regenera después | Paralelo |
| (8) Estudiar | «Accepted» inmediato | Paralelo |
| (9) Interpelar | «Accepted» inmediato | Paralelo |
| (10) Despertador | No aplica | Paralelo |
| (11) Percibir: intérprete | No aplica | Paralelo |
| (12) Noticias | No aplica | Paralelo |
| (13) Cotizar | Síncrona | Paralelo |
| (14) Comprometer | «Accepted» inmediato | Paralelo |

Make no admite el módulo de respuesta de webhook en un escenario secuencial y, además, desactiva un escenario secuencial en cuanto una ejecución falla. Por eso los escenarios que no deben pararse llevan un manejador de error Ignore en sus módulos externos (tabla de ajustes y sección 21).

### Ajustes de cada escenario

| Escenario | Programación | Procesamiento | Manejador Ignore |
| --- | --- | --- | --- |
| (1) Gateway | Inmediata, al llegar un mensaje | Secuencial | En los 11 módulos externos |
| (2) Escribir | Inmediata | Paralelo | No |
| (3) Reflexionar | Inmediata | Paralelo | No |
| (4) Recuperar | Inmediata | Paralelo | No |
| (6) Responder | Inmediata | Paralelo | En los 8 módulos de las rutas de cartera, cotizaciones y cálculo |
| (7) Resumen del agente | Inmediata | Paralelo | No |
| (8) Estudiar | Inmediata | Paralelo | En los 5 módulos externos |
| (9) Interpelar | Inmediata | Paralelo | En los 5 módulos externos |
| (10) Despertador | Diaria, 5:00 (Europe/Madrid) | Paralelo | No |
| (11) Percibir: intérprete | Cada 30 minutos, laborables de 8:00 a 20:30 (Europe/Madrid) | Paralelo | En todos sus módulos externos |
| (12) Noticias | Laborables a las 8:00 y 16:00 (Europe/Madrid) | Paralelo | En los 6 módulos externos |
| (13) Cotizar | Inmediata | Paralelo | En la consulta a Yahoo Finance |
| (14) Comprometer | Inmediata | Paralelo | No |

Todos usan el máximo de 3 errores seguidos de Make antes de desactivarse y confirman cada operación al terminarla (auto-commit).

Cada escenario lleva notas de Make en sus webhooks y en cada llamada a webhook, con el origen y el destino:

| Webhook de destino | Lo llaman (escenario y módulo) |
| --- | --- |
| (2) Escribir | Gateway 4 y 5, Reflexionar 11, Estudiar 6, Interpelar 6, Percibir 10, 45, 55, 60 y 64, Noticias 11 |
| (3) Reflexionar | Escribir 8 |
| (4) Recuperar | Reflexionar 16, Responder 2 y 10, Resumen 10, 12 y 14, Estudiar 4 |
| (6) Responder | Gateway 2 |
| (7) Resumen del agente | Responder 9 |
| (8) Estudiar | Gateway 15, Percibir 62 |
| (9) Interpelar | Estudiar 7, Percibir 11 y 61 |
| (13) Cotizar | Responder 23, Percibir 50 y 63 |
| (14) Comprometer | Escribir 11 |

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

Algunos textos siguen una convención porque otros escenarios buscan por ellos. Las observaciones que nacen de Interpelar empiezan por «{agente} le planteó una duda a». La agenda que guarda el intérprete empieza por «Agenda financiera de {agente} para hoy» y lo que escribe a un usuario por iniciativa propia, por «{agente} le escribió por iniciativa propia». Las lecturas de prensa de Noticias empiezan por «{agente} leyó en prensa».

Los registros de tipo `plan` que dejó Planificar, y los de tipo `evento` de versiones anteriores, siguen en la tabla como recuerdos, pero ningún escenario los busca ya. Los compromisos no viven en esta tabla sino en el almacén Compromisos: cambian cada media hora y no son memoria del agente, sino su trabajo pendiente. Cuando un compromiso se cumple, lo que el agente hizo o aprendió sí entra en la memoria a través de Escribir.

### Almacenes de datos de Make

| Almacén | Clave | Contenido |
| --- | --- | --- |
| Caché (campos agente, chat_id, resumen, actualizado) | `perfil\|<agente>` | Descripción del personaje, en el campo resumen, con chat_id `GLOBAL` |
| | `<agente>\|<chat_id>` | Resumen del agente en ese chat (escenario 7) |
| | `rel\|<agente>\|<chat_id>` | Lo que el agente sabe de su interlocutor (escenario 6) |
| Cartera (campos agente, chat_id, cartera, posiciones, actualizado) | `<agente>\|<chat_id>` | En `cartera`, un resumen en texto de las posiciones, los intereses y el horizonte del usuario; en `posiciones`, la lista estructurada en JSON. La escribe Responder y la leen Responder, el intérprete y Noticias |
| Compromisos (campos agente, comprometido_con, destinatario, accion, momento, condicion, simbolos, contenido, linea, origen, intentos, nivel, creado) | La genera Make; `agendar\|<agente>\|<fecha>` para la orden del Despertador | Una orden por registro, con el momento en que vence (sección 17) |

El perfil es la única fuente de la personalidad. Lo leen el Gateway, el Despertador y el intérprete, y el Gateway lo pasa a Responder y Estudiar. El límite de longitud de las respuestas no está en el perfil, porque solo aplica a Telegram: el Gateway añade «Limita la respuesta a una o dos frases, sin pasar de 20 palabras» únicamente en la llamada a Responder.

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

Todos los módulos que pueden fallar llevan un manejador Ignore: si uno falla, esa ejecución termina y el Gateway sigue activo para el siguiente mensaje. Sin ellos, Make desactivaría el escenario al primer error porque es secuencial.

El límite de frecuencia protege el consumo frente a ráfagas. Cuenta solo los mensajes del usuario ya guardados; como el mensaje en curso se guarda después de responder, no se cuenta a sí mismo. La condición de fecha, `IS_AFTER({Creado}, DATEADD(NOW(), -10, 'minutes'))`, compara instantes absolutos y no depende de la zona horaria.

La búsqueda de Airtable en Make devuelve un paquete vacío (`__IMTLENGTH__` = 0) cuando no encuentra nada. Por eso los contadores suman `if(id; 1; 0)` en lugar de 1.

Las observaciones se redactan en tercera persona y con el nombre del interlocutor porque el paper guarda las percepciones como frases narrativas («Isabella Rodriguez is setting out the pastries»). Así el texto que luego lee el LLM es autoexplicativo.

El procesamiento secuencial hace que tres mensajes seguidos se ejecuten uno detrás de otro. Sin él, el segundo podría recuperar memoria antes de que se guardara el primero. El coste es la latencia: cada mensaje espera al anterior, unos 10–20 segundos.

Todos los textos que entran en un cuerpo JSON pasan por `escapeJSON()`, para que unas comillas o un salto de línea no rompan la llamada. Telegram solo admite un webhook por bot: si el bot ya lo usaba otro escenario, este deja de recibir al crear el del Gateway.

Para tener varios agentes basta con duplicar el Gateway y cambiar el nombre del agente, la clave del perfil y la conexión de Telegram: la memoria ya está separada por el campo Agente.

## 5. (2) Memoria: Escribir

Escribir es la única puerta de entrada a la memoria: toda observación, reflexión o pieza de conocimiento pasa por aquí, así que todos quedan puntuados y vectorizados con el mismo criterio. Por eso es también donde se aplica la regla de compromiso: lo que entra en la memoria es lo que el agente ha hecho, dicho o aprendido, y ahí aparecen sus promesas y decisiones. Responde en cuanto el registro existe y después, en dos rutas, comprueba si hay que reflexionar y si hay que comprometerse.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Gemini · Generate a response | Puntúa la importancia del recuerdo de 1 a 10 y, si contiene una promesa o una decisión, la describe en una segunda línea |
| 3 | Gemini · Generate embeddings | Vector de la memoria, 768 dimensiones |
| 4 | Airtable · Create a record | Crea la fila con typecast, Importancia (la primera línea de la respuesta), Ultimo_acceso = ahora y Embedding |
| 5 | Webhooks · Webhook response | 200 con `{"id", "importancia", "tipo"}` |
| 9 | Router | Dos rutas independientes |
| R1 · 6 | Airtable · Search records | Observaciones del mismo agente y chat sin reflejar (hasta 500) |
| R1 · 7 | Tools · Numeric aggregator | Suma su importancia |
| R1 · 8 | HTTP · POST a Reflexionar | Solo si la suma supera 150 y el recuerdo nuevo es una observación |
| R2 · 10 | JSON · Parse JSON | Solo si la segunda línea describe un compromiso y el recuerdo no es del usuario |
| R2 · 11 | HTTP · POST a Comprometer | Manda la orden con un nivel más que el recuerdo que la origina |

### Importancia y regla de compromiso

El paper pide al LLM que puntúe la intensidad del recuerdo con una escala anclada en dos ejemplos extremos. El prompt es su traducción, y la misma llamada sirve de regla de compromiso: si el recuerdo contiene una promesa o una decisión, el modelo la describe en una segunda línea. Así detectar compromisos no cuesta ninguna llamada más.

```text
En una escala del 1 al 10, donde 1 es puramente mundano (p. ej., cepillarse los dientes, hacer la cama) y 10 es extremadamente conmovedor (p. ej., una ruptura, ser admitido en la universidad), puntúa la probable intensidad emocional del siguiente recuerdo de {{1.agente}}.
Además, si el recuerdo contiene una promesa o una decisión explícita de {{1.agente}} de hacer algo concreto más adelante (por ejemplo, «te aviso mañana», «lo miro y te digo», «revisaré ese dato a las 16:00»), añade después del número una segunda línea con un objeto JSON en una sola línea, sin bloques de código, con estas claves: accion, que es informar si se trata de escribir o contestar a su interlocutor (con cotizaciones si hacen falta), preguntar si va a hacerle una pregunta, estudiar si va a analizar un evento o un dato cuando se publique, investigar si va a buscar información, cotizar si solo va a consultar precios para sí mismo sin escribir a nadie, o abstenerse si se compromete a no hacer algo; a_quien («interlocutor» si se lo prometió a su interlocutor, «yo» si es una decisión propia); dia (AAAA-MM-DD) y hora (HH:mm) en hora de Madrid, vacías si es para ahora; contenido (qué hará, en una frase; si la acción es abstenerse, solo la acción de la que se abstiene: informar o preguntar); condicion (la condición de la que depende, o vacío); simbolos (símbolos de Yahoo Finance separados por comas, o vacío). Ahora son las {{formatDate(now; "HH:mm [del] YYYY-MM-DD"; "Europe/Madrid")}}. Lo normal es que no haya ninguna: entonces responde solo con el número.
Responde en la primera línea únicamente con un número entero del 1 al 10.
Recuerdo (autor: {{1.autor}}): {{1.memoria}}
Puntuación:
```

El módulo 4 guarda como importancia la primera línea, `trim(first(split(trim(2.result); newline)))`. La importancia se calcula una vez y no cambia: es lo que permite que un acontecimiento importante siga apareciendo semanas después aunque su recencia haya caído.

La ruta 2 lee la segunda línea y solo sigue si contiene `accion` y el recuerdo no es del usuario: lo que el usuario pide no compromete al agente hasta que él lo acepta, y su aceptación llega en su propia respuesta, que también pasa por Escribir. La llamada a Comprometer traduce la respuesta así:

| Campo de Comprometer | Valor |
| --- | --- |
| comprometido_con | El chat si `a_quien` es interlocutor y el chat no es `GLOBAL`; «yo» en otro caso |
| destinatario | El chat, solo en informar, preguntar y abstenerse, y nunca `GLOBAL` |
| accion, dia, hora, condicion, simbolos, contenido | Los de la respuesta |
| origen | `escribir:{tipo}` del recuerdo |
| nivel | El nivel del recuerdo más uno |

El nivel evita las cadenas sin fin: un compromiso que nace de lo que el agente hizo al cumplir otro tiene un nivel más, y Comprometer rechaza los de nivel superior a 2.

### Embedding

El vector se guarda como texto entre corchetes. Su dimensión debe coincidir con la de la consulta en Recuperar (768 en ambos). Si se cambia el modelo o la dimensión en uno de los dos, hay que cambiarla en el otro y recalcular los embeddings existentes: la similitud entre vectores de distinta longitud se trata como 0.

### Disparo de la reflexión

El filtro del módulo 8 exige dos condiciones: suma mayor que 150 y tipo observacion. La segunda evita un bucle, porque las reflexiones también se guardan llamando a Escribir. Con mensajes de chat corrientes, que el modelo suele puntuar entre 2 y 4, el umbral se alcanza tras unas 40–60 observaciones, es decir, 20–30 intercambios.

### Entrada y salida

Entrada: `memoria` (obligatorio), `tipo`, `agente`, `chat_id`, `autor` y, opcionales, `evidencias` (IDs de registro) y `nivel` (0 si falta; lo manda el intérprete). Los campos `plan_ubicacion`, `plan_inicio` y `plan_duracion_min` se siguen aceptando, aunque ya nadie los envía. Salida: `{"id": "rec…", "importancia": "4", "tipo": "observacion"}`.

## 6. (3) Memoria: Reflexionar

Reflexionar convierte una acumulación de observaciones en cinco conclusiones de nivel superior, cada una vinculada a los recuerdos que la justifican, con los tres pasos del paper: preguntas focales, recuperación de evidencias e inferencia de insights. Lo llama Escribir al cruzar el umbral; también se puede llamar a mano con `{"agente", "chat_id"}`.

### Respuesta inmediata y router

El módulo 2 responde 202 nada más recibir la llamada, porque la reflexión tarda alrededor de un minuto y Escribir no debe esperarla. Un router reparte después la ejecución en dos rutas que Make recorre en orden.

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
| 10 | Iterator | Recorre los insights |
| 11 | HTTP · POST a Escribir | Guarda el insight como reflexion, autor sistema |
| 14 | Airtable · Update a record | Rellena Evidencias con los IDs citados |

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

El módulo 8 fija `responseMimeType: application/json` y un `responseSchema`, de modo que Gemini no puede devolver prosa alrededor del JSON.

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

Desde que la iniciativa del agente pasó a los compromisos (sección 17), el escenario está desactivado y lleva el prefijo «[OBSOLETO]»; nadie lo llama. La búsqueda de la agenda financiera vive ahora en la acción «agendar» del intérprete (sección 14), con el mismo prompt. Su rama genérica, el plan narrativo de un personaje que no es Borja («7:30 desayuna, 9:00 abre la tienda»), no tiene equivalente, porque los compromisos solo recogen acciones que el bot sabe ejecutar.

## 9. (6) Responder

Responder genera lo que dice el agente con el prompt de diálogo del paper: quién es, qué hora es, qué compromisos tiene pendientes, qué sabe de su interlocutor, qué recuerda y qué se ha dicho. A eso suma tres capacidades de su oficio, cada una en una ruta que solo se activa cuando el mensaje la necesita: registrar la cartera, consultar cotizaciones con la valoración de la cartera, y hacer cuentas. No escribe en la memoria de la conversación; eso lo hace el Gateway después de enviar la respuesta, para que Responder pueda usarse desde otros canales o en pruebas.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 9 | HTTP · POST a Resumen del agente | Resumen del agente; vacío si el modo no es ON |
| 2 | HTTP · POST a Recuperar | El mensaje como consulta, k = 10 |
| 12 | Data store · Get a record | Lo que el agente sabe del interlocutor (`rel\|agente\|chat_id`) |
| 3 | Airtable · Search records | Las 12 observaciones más recientes del chat |
| 4 | Tools · Text aggregator | Una línea por observación: `- [autor] texto` |
| 14 | Data store · Search records | Los compromisos del agente que vencen hoy, hasta 10, en Compromisos |
| 15 | Tools · Text aggregator | La línea de cada uno: los propios y los de este chat, nunca los de otros chats |
| 21 | Data store · Get a record | La cartera del usuario (`agente\|chat_id` en Cartera) |
| 20 | Router | Cuatro rutas, que Make ejecuta en este orden |
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
| R4 · 25, 26 y 29 | Tools · Get variable | `cotizaciones`, `cartera_nueva` y `calculo` |
| R4 · 5 | Gemini · Generate a response | Genera la respuesta |
| R4 · 6 | Webhooks · Webhook response | `{"respuesta": "…"}` |
| R4 · 10 | HTTP · POST a Recuperar | Solo si la relación no existe o tiene más de 24 h: qué sabe el agente de esa persona, k = 10 |
| R4 · 11 | Gemini | La resume en una o dos frases |
| R4 · 13 | Data store · Add a record | La guarda, salvo que diga que aún no sabe nada concreto |

Las rutas 1 a 3 empiezan con un filtro por expresión regular sobre el mensaje, sin distinguir mayúsculas, para no gastar una llamada a Gemini en cada mensaje. Si el filtro no pasa, la ruta no hace nada y su variable queda vacía. La ruta 4 no tiene filtro y siempre responde.

| Ruta | Expresión que la activa |
| --- | --- |
| R1 · cartera | `compr[eéoaá]\|vend[iíoaá]\|tengo .*(acci\|particip\|t[ií]tulo\|fondo\|etf\|bono\|letra\|euros en\|d[oó]lares en)\|mi cartera\|invierto\|he invertido\|he metido\|he puesto\|liquid\|deshec\|ampli[eéoa]\|reduj\|reduzc\|deshacerme\|me he salido\|he salido de` |
| R2 · cotizaciones | `cotiz\|precio\|cu[aá]nto (vale\|est[aá]\|cotiza)\|c[oó]mo (va\|est[aá]\|ha cerrado\|ha abierto\|cerr[oó]\|abri[oó])\|bolsa\|ibex\|acci[oó]n\|acciones\|cartera\|mercado\|sub(e\|en\|i[oó]\|ido)\|baj(a\|an\|[oó]\|ado)\|ca(e\|en\|[ií]do\|y[oó])\|nasdaq\|s&p\|dow jones\|[íi]ndice\|euro stoxx\|etf` |
| R3 · cálculo | `calcul\|cu[aá]nto (ser[ií]a\|tendr[ií]a\|ganar[ií]a\|ganar[eé]\|pagar[ií]a\|pagar[eé]\|me queda\|me quedar[ií]a\|me costar[ií]a\|necesito\|hace falta\|rinde\|renta)\|inter[eé]s\|hipoteca\|cuota\|rentabilidad\|anualizad\|compuest\|amortiz\|\btae\b\|\btin\b\|plazo fijo\|jubila\|inflaci[oó]n\|[0-9] ?%\|[0-9] ?a[nñ]os` |

Las expresiones son deliberadamente amplias: un falso positivo cuesta una llamada a Gemini que devuelve «no hace falta», mientras que un falso negativo deja al agente sin el dato.

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

Los compromisos se buscan en el almacén por agente y con momento anterior a la medianoche siguiente, hora de Madrid. El agregador escribe la línea de cada orden con `switch(ifempty(14.data.destinatario; "SELF"); "SELF"; 14.data.linea; 1.chat_id; 14.data.linea; "")`: las órdenes sin destinatario son del propio agente y se muestran, las de este chat también, y las de otros chats quedan en blanco, para que el agente no cuente a una persona lo que ha prometido a otra. La búsqueda continúa aunque no haya resultados, para que Responder conteste igual. La hora se formatea en Europe/Madrid porque el paper siempre sitúa al agente en su hora actual.

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

Interpelar escribe al usuario por iniciativa propia, en el personaje del agente, para resolver una duda concreta. Lo llaman Estudiar, cuando a una petición profesional le falta un dato imprescindible, y el intérprete, cuando al estudiar un evento le falta un dato de la cartera o cuando cumple un compromiso de preguntar.

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

El Despertador arranca el día del agente aunque nadie escriba. Cada día a las 5:00, hora de Madrid, lee el perfil de Borja y, si es laborable, deja en Compromisos la orden «agendar» para las 8:00; el intérprete la ejecuta en su primera vuelta del día.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Data store · Get a record | Lee `perfil\|Borja` |
| 5 | Data store · Add/replace a record | Si hay perfil y el día no es sábado ni domingo, guarda la orden con la clave `agendar\|<agente>\|<fecha>` |

La clave fija por día evita que la orden se duplique si el escenario se ejecuta dos veces. Escribe directamente en el almacén, sin pasar por Comprometer, porque es una orden del propio sistema que no necesita validación. Cuesta 2 créditos al día.

## 14. (11) Percibir: intérprete de compromisos

Percibir es el intérprete de compromisos, el ciclo de AGENT-0 en el artículo de Shoham: a intervalos fijos, ejecuta lo que el agente se ha comprometido a hacer y ya ha vencido. Sustituye al antiguo bucle percibir-reaccionar, que leía el plan cada hora; lo que aquel hacía con cada evento financiero es ahora la acción «estudiar». Se ejecuta cada 30 minutos los laborables de 8:00 a 20:30, hora de Madrid. Cada vuelta lee hasta 3 órdenes vencidas, de la más antigua a la más reciente, para acotar el gasto de una vuelta; una vuelta sin órdenes vencidas cuesta 1 crédito.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 20 | Data store · Search records | Órdenes con momento ya pasado, salvo «abstenerse»; hasta 3, o todas en la vuelta de las 20:00 |
| 21 | Router | Ruta de ejecución o ruta de cierre, según la hora |
| R1 · 22 | Data store · Delete a record | Consume la orden antes de ejecutarla; solo si no es la vuelta de las 20:00 |
| R1 · 1 | Data store · Get a record | El perfil del agente de la orden |
| R1 · 23 | Router | Una ruta por acción |
| R2 · 30 | Router | Cierre del día, solo en la vuelta de las 20:00 |
| R2 · 31 | Data store · Delete a record | Anula las órdenes «abstenerse» y las decisiones propias que no sean estudiar |
| R2 · 32 | Data store · Add/replace a record | Pasa al siguiente laborable a las 8:00 las promesas a un chat y los estudios pendientes |

La búsqueda y el tope dependen de la hora: el valor con el que se compara la acción es `switch(formatDate(now; "HH"; "Europe/Madrid"); "20"; "___"; "abstenerse")`, de modo que a las 20:00 también entran las órdenes «abstenerse», y el tope es `switch(...; "20"; 50; 3)`. Una orden «abstenerse» nunca se ejecuta: solo veta, mientras dura el día, que Comprometer acepte la acción de la que se abstiene.

Las acciones son las capacidades del agente:

| Acción | Tipo | Módulos | Qué hace |
| --- | --- | --- | --- |
| agendar | DO | 40–45 | Gemini con búsqueda de Google encuentra los eventos financieros de hoy con hora confirmada; por cada uno crea en Compromisos la orden de estudiarlo una hora después, y guarda la agenda en la memoria como conocimiento `GLOBAL` |
| estudiar | DO | 3, 24, 25 y 4–11 | Busca el resultado del evento; si aún no está publicado, la vuelve a escribir para dentro de 30 minutos, hasta 4 veces. Si lo está, calcula su implicación para la cartera de cada chat, la guarda y, si falta un dato, llama a Interpelar |
| informar | INFORM | 50–55 con símbolos, 56–60 sin ellos | Con símbolos, pide cotizaciones a Cotizar. Gemini redacta el mensaje prometido o, si la orden tiene una condición que no se cumple, responde `NO_CUMPLE` y la orden se vuelve a escribir para dentro de 2 horas. Si se envía, sale por Telegram y se guarda como observación del agente en ese chat |
| preguntar | REQUEST | 61 | Llama a Interpelar con el contenido de la orden como motivo |
| investigar | DO | 62 | Llama a Estudiar con el contenido como mensaje |
| cotizar | DO | 63–64 | Pide cotizaciones a Cotizar y las guarda como conocimiento, sin escribir a nadie |

El estudio de un evento usa los prompts del antiguo Percibir, con el contenido de la orden en lugar del punto del plan. La investigación del resultado, en el módulo 3, con la búsqueda de Google activada:

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

La agenda del día, en el módulo 40, con la búsqueda de Google activada y la descripción del personaje como instrucción de sistema. Es el prompt que tenía Planificar:

```text
Hoy es {{día de la semana}}, {{DD/MM/YYYY}}. Busca la agenda de eventos financieros de hoy que seguiría un asesor financiero en Madrid: datos macroeconómicos de España, la zona euro y Estados Unidos, decisiones o comparecencias de bancos centrales, subastas de deuda relevantes y resultados empresariales del Ibex 35 y de las grandes cotizadas de Estados Unidos.
Incluye solo eventos de hoy con hora confirmada por una fuente, con la hora convertida a la de Madrid. Entre 2 y 5 eventos, los de mayor impacto.
Responde únicamente con una línea por evento, ordenadas por hora, con este formato exacto y sin nada más:
HH:mm|evento en una frase|ámbito (España, Zona euro, EE. UU. o empresa)|fuente
Si hoy no hay eventos relevantes, responde únicamente: NINGUNO
```

Cada línea válida se convierte en una orden «estudiar» con momento una hora después del evento, contenido «04/11 14:15 Publicación del informe de empleo… (EE. UU.; fuente)» y nivel 1. La hora de margen da tiempo a que el dato esté publicado e indexado; si no lo está, los reintentos de 30 minutos cubren dos horas más.

El mensaje de informar, en los módulos 51 y 56:

```text
Son las {{hora}}. {{1.agente}} se comprometió a escribir por iniciativa propia a su interlocutor con este fin: «{{20.data.contenido}}».
Solo debe escribirle si se cumple esta condición: «{{20.data.condicion}}».   ← solo si la orden tiene condición
Cotizaciones actuales de Yahoo Finance (JSON): {{50.data}}                    ← solo en la ruta con símbolos
Si hay una condición y no se cumple, responde únicamente: NO_CUMPLE
En caso contrario, redacta el mensaje de Telegram que {{1.agente}} le envía, con su voz, en un máximo de 80 palabras, con las cifras exactas que procedan y sin firma. Responde solo con el mensaje.
```

La ruta sin símbolos activa la búsqueda de Google, para que el agente pueda comprobar condiciones o datos que no son precios.

Tres decisiones conviene tenerlas presentes. La orden se borra antes de ejecutarse: si una vuelta tarda, la siguiente no la ejecuta dos veces, y un reintento consiste en volver a escribirla con la nueva hora y un intento más. Las llamadas a Escribir y a Interpelar son copias de las del antiguo Percibir, con el mismo cuerpo y los mismos campos; cada ruta acaba en su propia llamada porque en Make las rutas de un router no vuelven a juntarse. Y el cierre respeta la distinción de Shoham entre obligaciones y decisiones: lo que el agente decidió por su cuenta se puede anular, pero lo que prometió a otra persona no, así que pasa al día siguiente.

El estudio de eventos usa el resumen en texto de la cartera, el campo `cartera`, que Responder mantiene al día cuando el usuario cuenta sus posiciones (sección 9). También se puede rellenar a mano desde Make.

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

Cotizar devuelve el último precio de hasta 5 valores o índices. Lo llaman Responder, cuando un mensaje pregunta por precios o mercado, y el intérprete, en las órdenes de informar o cotizar que llevan símbolos.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Recibe `simbolos`, separados por comas |
| 2 | Iterator | `slice(split(replace(1.simbolos; " "; ""); ","); 0; 5)`: como mucho 5 símbolos |
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

En la programación orientada a agentes de Shoham, el estado mental de un agente está hecho de creencias, compromisos y capacidades. Un compromiso es la obligación de un agente con otro de hacer algo en un momento dado; una decisión es un compromiso con uno mismo. El intérprete de AGENT-0, el lenguaje del artículo, repite un ciclo a intervalos fijos: incorpora lo que ha recibido y ejecuta los compromisos que han vencido. Human_proxy aplica ese modelo así:

| AGENT-0 | Human_proxy |
| --- | --- |
| Creencias | La memoria (Memory Stream), la cartera y el perfil |
| Capacidades | Las acciones que sabe ejecutar el intérprete: agendar, estudiar, cotizar e investigar (DO), informar (INFORM), preguntar (REQUEST) y abstenerse (REFRAIN) |
| Compromiso con otro agente | Orden cuyo campo comprometido_con es el chat al que se prometió |
| Decisión | Orden con comprometido_con «yo» |
| Compromiso condicional, `IF (B φ)` | Orden con condicion; el intérprete la evalúa antes de actuar |
| Reglas de compromiso | La regla de Escribir (sección 5), el Despertador y la acción agendar |
| Ciclo del intérprete | Percibir cada 30 minutos (sección 14) |

Las órdenes nacen de cuatro sitios: de cualquier recuerdo que contenga una promesa o una decisión, a través de Escribir; del Despertador, que deja cada laborable la orden de agendar; de la acción agendar, que crea una orden de estudiar por evento; y de los reintentos del intérprete. Cualquier escenario nuevo puede comprometer al agente llamando a Comprometer.

### La tabla Compromisos

| Campo | Contenido |
| --- | --- |
| agente | Agente que se compromete |
| comprometido_con | «yo» o el chat al que se prometió |
| destinatario | Chat al que va dirigida la acción, en informar, preguntar y abstenerse |
| accion | Una de las capacidades |
| momento | Fecha y hora en que vence |
| condicion | Condición de la que depende, o vacío |
| simbolos | Símbolos de Yahoo Finance implicados, o vacío |
| contenido | Qué hará, en una frase; en abstenerse, la acción vetada |
| linea | La orden en notación de compromisos, la que ve Responder |
| origen | Quién la creó: `escribir:{tipo}`, `despertador`, `agendar`… |
| intentos | Reintentos y aplazamientos que lleva |
| nivel | Profundidad en la cadena de compromisos; 0 en las que nacen de fuera |
| creado | Cuándo se creó |

La línea sigue la notación del artículo: `(INFORM 05/11 10:00 informar chat:123456789 «contarle cómo cierra Inditex»)`, o con condición, `(IF (B «Inditex por debajo de 40 €») (INFORM … ))`.

La tabla guarda compromisos de cualquier fecha: el intérprete solo coge los que han vencido. Al terminar cada jornada no queda nada pendiente de ese día, porque el cierre de las 20:00 anula lo que se puede anular y pasa al siguiente laborable lo demás; las órdenes de días futuros siguen esperando su momento.

### (14) Comprometer

Comprometer es la única puerta de entrada de las órdenes que vienen de fuera del intérprete. Valida la orden y la guarda; no responde nada al llamante.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 3 | Tools · Set variables | Día y hora de la orden: sin día ni hora, ahora; con día y sin hora, las 8:00 |
| 4 | Tools · Set variables | Momento, hora para la línea, tipo (DO, INFORM, REQUEST, REFRAIN) y las firmas de duplicado y de veto |
| 6 | Data store · Search records | Las órdenes pendientes del agente, hasta 100 |
| 7 | Tools · Text aggregator | Su firma, `accion\|destinatario\|contenido`, una por línea |
| 8 | Tools · Set variables | Cuántas hay, si la nueva está duplicada o vetada, si la acción es una capacidad y si tiene destinatario cuando lo necesita |
| 10 | Data store · Add a record | La guarda si pasa todas las comprobaciones |

Una orden se rechaza si su acción no es una de las capacidades, si es informar, preguntar o abstenerse sin destinatario, si tiene nivel mayor que 2, si ya hay una igual pendiente, si una orden «abstenerse» pendiente veta esa acción para ese chat o si el agente tiene ya 30 órdenes pendientes. El rechazo es silencioso: la orden no se guarda y queda en el historial del escenario.


## 18. Contratos de los webhooks

Cada escenario con webhook se invoca con un POST JSON (`Content-Type: application/json`). Los webhooks de Make aceptan también los parámetros en la URL, lo que permite probar Responder o Cotizar abriendo la dirección en un navegador. Cualquier canal nuevo, como WhatsApp, un formulario web u otro agente, puede usar la memoria llamando a estas URLs sin tocar los escenarios 2 a 4 y 6.

| Función | URL | Entrada | Salida |
| --- | --- | --- | --- |
| Escribir (2) | `https://hook.eu1.make.com/<WEBHOOK_ESCRIBIR>` | `memoria`, `tipo`, `agente`, `chat_id`, `autor`; opcionales `evidencias[]`, `nivel` | 200 `{"id", "importancia", "tipo"}` |
| Reflexionar (3) | `https://hook.eu1.make.com/<WEBHOOK_REFLEXIONAR>` | `agente`, `chat_id` | 202 `{"estado": "reflexion en curso", "agente"}` |
| Recuperar (4) | `https://hook.eu1.make.com/<WEBHOOK_RECUPERAR>` | `agente`, `chat_id`, `consulta`; opcional `k` (10) | 200 `{"memorias": [...], "texto", "texto_ids", "total_evaluadas"}` |
| Responder (6) | `https://hook.eu1.make.com/<WEBHOOK_RESPONDER>` | `agente`, `chat_id`, `mensaje`, `descripcion`; opcionales `usuario`, `modo` | 200 `{"respuesta"}` |
| Resumen del agente (7) | `https://hook.eu1.make.com/<WEBHOOK_RESUMEN>` | `agente`, `chat_id`, `modo` (ON u OFF; OFF si falta) | 200 `{"resumen", "estado"}` |
| Estudiar (8) | `https://hook.eu1.make.com/<WEBHOOK_ESTUDIAR>` | `agente`, `chat_id`, `mensaje`; opcionales `usuario`, `descripcion` | «Accepted» |
| Interpelar (9) | `https://hook.eu1.make.com/<WEBHOOK_INTERPELAR>` | `agente`, `chat_id`, `motivo`; opcionales `usuario`, `contexto`, `descripcion` | «Accepted»; envía el mensaje por Telegram |
| Cotizar (13) | `https://hook.eu1.make.com/<WEBHOOK_COTIZAR>` | `simbolos` (hasta 5, separados por comas) | 200 `{"precios": [{"simbolo", "nombre", "precio", "cierre_anterior", "moneda", "hora", "error"}]}` |
| Comprometer (14) | `https://hook.eu1.make.com/<WEBHOOK_COMPROMETER>` | `agente`, `accion`, `contenido`; opcionales `comprometido_con` («yo» por defecto), `destinatario`, `dia` (AAAA-MM-DD), `hora` (HH:mm), `condicion`, `simbolos`, `origen`, `nivel`, `intentos` | «Accepted» |

Ejemplos de llamada:

```json
POST Escribir
{"memoria": "Ana le dijo a Borja: «Tengo una reunión con el banco en mayo»", "tipo": "observacion", "agente": "Borja", "chat_id": "123456789", "autor": "usuario"}

POST Recuperar
{"agente": "Borja", "chat_id": "123456789", "consulta": "¿Qué sabe Borja de los planes de Ana?", "k": 5}

POST Responder
{"agente": "Borja", "chat_id": "123456789", "mensaje": "¿Qué tal el día?", "usuario": "Ana", "descripcion": "Borja es un asesor financiero..."}

POST Cotizar
{"simbolos": "ITX.MC,SAN.MC,^IBEX"}

POST Comprometer
{"agente": "Borja", "comprometido_con": "123456789", "destinatario": "123456789", "accion": "informar", "dia": "2026-11-05", "hora": "10:00", "simbolos": "ITX.MC", "contenido": "Contarle cómo ha abierto Inditex", "origen": "prueba"}
```

## 19. Fidelidad al paper

La memoria sigue el paper en lo que la define: el registro único, la importancia asignada por el LLM, la fórmula de recuperación y el ciclo de reflexión con citas. Las desviaciones vienen de pasar de un mundo simulado a una conversación de chat y de los límites de Make y Airtable.

| Elemento | Paper | Human_proxy | Valoración |
| --- | --- | --- | --- |
| Objeto de memoria | Descripción, creación, último acceso | Memoria, Creado, Ultimo_acceso + metadatos | Fiel |
| Importancia | LLM, escala 1–10 anclada | Mismo prompt traducido | Fiel |
| Recencia | 0,995 por hora de juego desde el último acceso | 0,995 por hora real | Fiel en la forma; cambia la escala temporal |
| Relevancia | Coseno entre embeddings | Coseno, Gemini Embedding 2, 768 dimensiones | Fiel |
| Combinación | Min-max a [0, 1] y suma con α = 1 | Igual | Fiel |
| Candidatos | Toda la memoria del agente | Los 300 registros más recientes del chat | Desviación por límite técnico (sección 21) |
| Cuántos se recuperan | Los que caben en el contexto | k fijo, 10 por defecto | Simplificación razonable |
| Disparo de reflexión | Suma de importancia reciente > 150 | Suma de observaciones no reflejadas > 150 | Fiel |
| Preguntas focales | 3, a partir de los 100 registros más recientes | Igual | Fiel |
| Insights | 5, con «(because of 1, 5, 3)» | 5, con IDs reales en un JSON con esquema | Fiel y más robusto |
| Árbol de reflexiones | Las reflexiones pueden citar reflexiones | Igual, vía Evidencias / Citada_por | Fiel |
| Resumen del agente | Generado desde la memoria y cacheado | Las tres consultas del paper, caché de 7 días | Fiel; la descripción fija sigue siendo la base |
| Diálogo | Recupera lo que sabe del interlocutor antes de hablar | Resumen de la relación, caché de 24 h; además, sus compromisos pendientes | Fiel |
| Planificación | Plan diario, por horas y por tramos de 5–15 minutos | No hay plan: la agenda del día se convierte en compromisos de estudiar cada evento | Sustituida por los compromisos de Shoham (sección 17) |
| Percibir y reaccionar | El agente percibe su entorno en cada paso y decide si reacciona y replanifica | El intérprete ejecuta cada media hora lo que ha vencido y la prensa se lee dos veces al día, con reacción por usuario | Parcial: reacciona a lo que tiene comprometido, no a cada cambio del entorno |
| Ámbito de la memoria | Un agente, un mundo | Un agente por chat | Decisión deliberada: el agente no mezcla lo que le cuentan usuarios distintos |
| Entorno y otros agentes | Mundo simulado con espacios y varios agentes | No existe | Inherente al canal |

Estudiar, Interpelar, Noticias, Cotizar, la cartera y la calculadora no están en el paper; son extensiones para un agente que conversa con personas reales y necesita información verificable y cifras exactas.

Respecto a Shoham, el sistema es fiel en lo esencial: los compromisos tienen momento, las decisiones son compromisos con uno mismo, hay compromisos condicionales y de abstención, Comprometer solo acepta capacidades reales, y el cierre del día no anula por su cuenta lo prometido a otros. Se aparta en tres cosas. Las creencias no se consultan con un operador lógico: la condición de una orden la evalúa Gemini con las cotizaciones o la búsqueda de Google delante. Las reglas de compromiso no son reglas formales, sino la segunda línea de la llamada de importancia de Escribir. Y no hay mensajes entre agentes ni la posibilidad de que el otro libere al agente de un compromiso (el UNREQUEST del artículo); el ciclo, además, va cada 30 minutos y no a cada instante.

## 20. Costes

Con un usuario, el sistema gasta unos 3.200–3.700 créditos de Make al mes antes de que nadie converse, según una estimación hecha con los créditos medidos de cada pieza: el intérprete (≈ 550 en vueltas sin nada que hacer, ≈ 460 en la agenda diaria y ≈ 1.500 en el estudio de eventos), Noticias (≈ 750–1.250) y el Despertador (≈ 60). Hasta noviembre de 2026, con el plan diario, Percibir cada hora y Noticias cuatro veces al día, eran unos 6.400–7.400. En el coste manda Make, no Gemini: los créditos son el 85–90 % de cada ejecución.

Precios aplicados: Make Core, 10.000 créditos por 10,59 $ al mes (0,00106 $ por crédito; 0,0009 $ en anual); Gemini 3.8 Flash, 0,75 $ por millón de tokens de entrada y 3,75 $ por millón de salida, razonamiento incluido, hasta el 31 de diciembre de 2026 y el doble desde el 1 de enero de 2027; búsqueda de Google gratis hasta 5.000 consultas al mes. Conversión a euros al cambio del Banco de España del 19 de octubre de 2026, 1,1225 $ por euro. Los créditos se midieron en el historial de cada escenario; los tokens, en el inspector de Make para Responder, Escribir y Estudiar, y el resto se estima por el tamaño del prompt.

### Coste unitario por escenario

Solo lo que gasta el propio escenario; las llamadas a otros escenarios se suman en la tabla siguiente.

| Escenario | Créditos | Make (€) | Gemini (€) | Gemini |
| --- | --- | --- | --- | --- |
| (1) Gateway | 9–10 | 0,009 | 0 | — |
| (2) Escribir, sin compromiso / con compromiso | 7 / 9 | 0,0066 / 0,0085 | 0,0012 | medido |
| (3) Reflexionar | 25 | 0,024 | ≈ 0,009 | estimado |
| (4) Recuperar k = 5 / 10 / 15 | 12 / 17 / 18 | 0,011 / 0,016 / 0,017 | < 0,0001 | medido |
| (6) Responder, con caché / regenerando la relación | 14 / 17 | 0,013 / 0,016 | 0,0017 / 0,0045 | medido |
| (6) Responder, más ruta de cartera / cotizaciones / cálculo | + 4 / + 6 / + 4 | + 0,004 / + 0,006 / + 0,004 | ≈ 0,001 cada una | medido en créditos |
| (7) Resumen OFF / caché / regenerando | 2 / 3 / 10 | 0,002 / 0,003 / 0,009 | 0 / 0 / ≈ 0,006 | estimado |
| (8) Estudiar sin investigar / ya sabido / investigando | 3 / 5 / 6 | 0,003 / 0,005 / 0,006 | 0,001 / ≈ 0,003 / 0,0047 | medido |
| (9) Interpelar | 8 | 0,0076 | ≈ 0,002 | estimado |
| (10) Despertador | 2 | 0,0019 | 0 | — |
| (11) Intérprete, vuelta sin órdenes vencidas | 1 | 0,0009 | 0 | — |
| (11) Intérprete, por orden: consumirla y leer el perfil | 2 | 0,0019 | 0 | medido |
| (11) Intérprete, agendar con 4 eventos | 12 | 0,011 | ≈ 0,004 | medido |
| (11) Intérprete, estudiar: por evento + por chat | 4 + 4 | 0,004 + 0,004 | ≈ 0,004 + ≈ 0,0025 | estimado |
| (11) Intérprete, informar con cotizaciones / cotizar | 3 / 2 | 0,003 / 0,002 | ≈ 0,001 / 0 | medido |
| (12) Noticias, por ejecución + por chat + por noticia | 5 + 5 + 1 | 0,005 + 0,005 + 0,001 | ≈ 0,0015 por chat | medido en créditos |
| (13) Cotizar, por llamada + por símbolo | 4 + 1 | 0,004 + 0,001 | 0 | medido |
| (14) Comprometer | 7 | 0,0066 | 0 | medido |

La calculadora usa el módulo Make Code, que cobra 2 créditos por segundo de ejecución; las dos calculadoras tardan menos de un segundo, así que cuestan 2 créditos cada vez. En las llamadas medidas manda el razonamiento interno de Gemini: la importancia de Escribir devuelve 1 token y razona 331; la investigación de Estudiar recibe 521 tokens, escribe 120 y razona 880.

### Coste por operación completa

| Operación | Escenarios que encadena | Créditos | Total (€) |
| --- | --- | --- | --- |
| Mensaje de Telegram con cachés vigentes | Gateway, Responder, Recuperar k = 5, Resumen en caché, 2 × Escribir, Estudiar | ≈ 56 | ≈ 0,058 |
| Pregunta de cotizaciones o de cartera | + ruta de cotizaciones y Cotizar con 2–3 símbolos | + 12–13 | + 0,013 |
| Mensaje que registra posiciones o pide un cálculo | + su ruta | + 4 | + 0,005 |
| Regenerar el resumen (una vez a la semana por chat) | Resumen con 3 × Recuperar k = 15 | ≈ 55 | ≈ 0,06 |
| Regenerar la relación (una vez al día por chat) | Recuperar k = 10, Gemini | ≈ 19 | ≈ 0,02 |
| Estudiar con investigación | + Recuperar, Escribir | + 22 | + 0,026 |
| Interpelación | Interpelar, Escribir | ≈ 15 | ≈ 0,017 |
| Reflexión | Reflexionar, 3 × Recuperar k = 10, 5 × Escribir | ≈ 111 | ≈ 0,12 |
| Agenda diaria del agente financiero | Despertador, intérprete (agendar), Escribir | ≈ 21 | ≈ 0,025 |
| Estudio de un evento, primer usuario | Intérprete, Escribir | ≈ 17 | ≈ 0,025 |
| Cada usuario más por evento | Intérprete, Escribir | + 11 | + 0,014 |
| Promesa detectada y cumplida con un mensaje | Escribir (+2), Comprometer, intérprete, Cotizar, Escribir | ≈ 25 | ≈ 0,03 |
| Noticia guardada | Noticias, Escribir | ≈ 8 | ≈ 0,009 |
| Conversación de 20 mensajes | 20 mensajes, regeneración diaria, 2 investigaciones, 1 interpelación, 0–1 reflexión | 1.290–1.400 | 1,35–1,48 |

### Coste mensual

| Concepto | Créditos al mes | Coste (€) |
| --- | --- | --- |
| Intérprete, ≈ 550 vueltas al mes | ≈ 550 | ≈ 0,52 |
| Agenda diaria, 22 laborables | ≈ 460 | ≈ 0,50 |
| Estudio de eventos, 1 usuario, ≈ 90 eventos | ≈ 1.500 | ≈ 1,80 |
| Cada usuario más en el estudio de eventos | ≈ 1.000 | ≈ 1,20 |
| Noticias, 44 ejecuciones al mes con uno o dos usuarios | ≈ 750–1.250 | ≈ 0,80–1,35 |
| Cada conversación de 20 mensajes | ≈ 1.350 | ≈ 1,40 |

Un plan de 10.000 créditos da para el gasto fijo y unas cuantas conversaciones al mes. Lo que más pesa del gasto fijo es el estudio de eventos, que crece con cada usuario: limitar la agenda a 2 o 3 eventos lo recorta en proporción, y llevar el intérprete a una vuelta por hora ahorra unos 275 créditos al mes a cambio de reaccionar más tarde. Juntar las tres lecturas de variables de Responder en un único módulo Get multiple variables ahorra 2 créditos por mensaje. Si Gemini duplica su precio en 2027, un mensaje pasa de ≈ 0,058 € a ≈ 0,063 €.

Fuentes: [precios de la API de Gemini](https://ai.google.dev/gemini-api/docs/pricing), [Gemini 3.8 Flash, Layer3 Labs](https://www.layer3labs.io/guides/gemini-3-8-flash-pricing), [Gemini Embedding 2, Maxim](https://www.getmaxim.ai/bifrost/llm-cost-calculator/provider/gemini/model/gemini-embedding-2), [precios de Make](https://www.make.com/en/pricing), [Make Code](https://apps.make.com/code.md), [precios de Make, Jetadmin](https://www.jetadmin.io/blog/make-pricing/), [cambio oficial del euro, BOE 20/10/2026](https://www.boe.es/boe/dias/2026/10/20/pdfs/BOE-A-2026-21865.pdf).

## 21. Limitaciones y riesgos

### Escala de la memoria

Recuperar solo evalúa los 300 registros más recientes del chat. A partir de unos 150 intercambios más sus reflexiones, los recuerdos más antiguos dejan de ser candidatos aunque sean importantes, cosa que el paper no hace. El techo existe porque todos los embeddings viajan como texto al módulo de código, del orden de 2,7 MB por llamada. Las salidas, de menor a mayor esfuerzo: subir el techo y vigilar tiempos; prefiltrar por importancia (por ejemplo, las 200 más recientes más las 100 más importantes); o mover los vectores a una base vectorial y dejar Airtable como registro legible.

El volumen total lo limita el plan de Airtable: 1.000 registros por base en Free, 50.000 en Team y 125.000 en Business ([Jetadmin, julio de 2026](https://www.jetadmin.io/blog/airtable-pricing-in-2026-plans-real-costs-and-when-to-upgrade/)). Con dos registros por turno, más cinco por reflexión y lo que guardan el intérprete y Noticias, el límite de Free se alcanza pronto.

### Errores

Make desactiva un escenario secuencial en cuanto una ejecución falla, y uno en paralelo tras varios errores seguidos. Gateway, Estudiar, Interpelar y el intérprete llevan un manejador Ignore en todos los módulos que pueden fallar, y Escribir en su regla de compromiso. Noticias y Cotizar lo llevan en sus módulos externos, y Responder en los de sus rutas de cartera, cotizaciones y cálculo, de modo que un fallo ahí deja al agente sin ese dato pero no sin respuesta. Escribir, Reflexionar, Recuperar, el núcleo de Responder, el Resumen del agente, el Despertador y Comprometer no lo llevan, con estas consecuencias:

1. Si Gemini o Airtable fallan durante Responder, el usuario no recibe respuesta; el Gateway sigue activo.
2. Si Gemini devuelve la importancia con texto alrededor en la primera línea («Puntuación: 7»), la conversión de Airtable puede fallar y ese recuerdo se pierde.
3. Si el LLM inventa un ID en las evidencias de una reflexión, falla la actualización del módulo 14 de Reflexionar y no se escriben las reflexiones restantes de esa tanda.
4. Como el Gateway responde antes de memorizar, un fallo en Escribir no lo ve el usuario: la conversación sigue, con huecos en la memoria.
5. El intérprete borra cada orden antes de ejecutarla. Si la ejecución falla, la orden se pierde, salvo en los dos casos con reintento previsto: un dato aún no publicado y una condición que no se cumple.

### Latencia

Un mensaje tarda unos 10–20 segundos en recibir respuesta: embedding y búsqueda en Recuperar, código y generación. Con el procesamiento secuencial, los mensajes en ráfaga se encolan.

### Seguridad y privacidad

Por defecto, Make no protege las URLs de los webhooks: quien tenga una puede invocar el escenario, leer o escribir la memoria y gastar créditos. No las publiques y, si se filtra alguna, regenera ese webhook y actualiza los módulos que lo llaman. Make permite ahora exigir una clave en cada webhook (API key authentication, que se envía en la cabecera `x-make-apikey`); si el sistema se abre a terceros, conviene activarla en todos y añadir la cabecera en las llamadas entre escenarios.

La ruta de cálculo ejecuta código que escribe el modelo. Lo contienen el filtro de patrones, el entorno aislado de Make Code y su límite de 30 segundos, pero el filtro es una lista negra y no una garantía; si el bot se abre a desconocidos, conviene vigilar el historial de ese módulo o desactivar la ruta.

La tabla guarda literalmente lo que cada usuario dice al bot, segmentado por su chat. Si lo usan otras personas, deben saberlo.

### Otros

En un grupo de Telegram todos los miembros comparten chat y, por tanto, memoria; las observaciones sí distinguen a cada uno por su nombre. Telegram limita los mensajes a 4.096 caracteres. El límite de 10 mensajes en 10 minutos cuenta solo mensajes ya guardados.

La cartera depende de que el modelo interprete bien lo que cuenta el usuario. Una frase ambigua («he metido algo más en Inditex») registra la posición con cantidad 0, y una corrección («no, eran 40») se aplica como un cambio más. El resumen y el JSON están a la vista en el almacén Cartera para revisarlos o corregirlos a mano. Las rutas se activan por expresiones regulares: un mensaje que pregunta por la cartera sin ninguna de las palabras previstas («¿y lo mío qué tal?») no consulta cotizaciones.

Las cifras del cálculo libre son exactas para el planteamiento que elige el modelo, pero el planteamiento puede no ser el que el usuario tenía en mente (capitalización mensual o anual, aportaciones al principio o al final del periodo). El planteamiento llega al agente, aunque con el límite de 20 palabras de Telegram no siempre lo menciona.

La regla de compromiso depende de que Gemini reconozca las promesas. Si pasa una por alto, el agente no cumple lo que dijo; si ve una donde no la hay, gasta créditos en una orden innecesaria, acotada por el tope de 30 pendientes y el nivel máximo de 2. Las condiciones de precio las evalúa Gemini con las cotizaciones delante y se revisan cada 2 horas, así que un aviso del tipo «si baja de 40» puede llegar con retraso o no llegar si el precio cruza y vuelve entre dos revisiones.

El conocimiento que guardan Estudiar, el intérprete y Noticias compite en desventaja en la recuperación: la importancia mide la carga emocional, como en el paper, y un dato fiscal o de mercado puntúa bajo. Además, ese conocimiento es por chat. Interpelar envía siempre por el mismo bot; con varios agentes y bots habría que pasarle cuál usar.

### Mejoras posibles

1. Permitir que el usuario libere al agente de un compromiso («ya no hace falta que me avises»), el UNREQUEST de Shoham.
2. Evaluar las condiciones de precio con una comparación numérica en lugar de con Gemini, y revisarlas con más frecuencia.
3. Dar una importancia fija alta al tipo conocimiento para que compita en la recuperación.
4. Añadir manejadores de error al resto de escenarios y extraer el número de la importancia con una expresión regular.
5. Activar la autenticación por clave de los webhooks de Make, también en Comprometer.
6. Convertir divisas en la valoración de la cartera para dar un total único.

## 22. Operación

### Pruebas aisladas

Cada función se puede probar sola con un POST a su webhook, sin pasar por Telegram. Conviene usar un chat_id de prueba, como `PRUEBA`, para no mezclar esos recuerdos con conversaciones reales. Las pruebas de Responder con posiciones dejan además un registro en Cartera y otro en la caché de relaciones, que hay que borrar después. El estudio de eventos del intérprete y Noticias recorren los chats que tienen mensajes de usuario en Airtable, así que un chat de prueba que solo haya pasado por Responder no les afecta.

Para probar los compromisos sin esperar, crea una orden con Comprometer para un agente de prueba que tenga perfil en la caché (`perfil|PRUEBA`) y pulsa Run once en el intérprete: ejecuta hasta tres órdenes vencidas. Un destinatario inventado, como `PRUEBA`, hace que el envío por Telegram falle sin escribir a nadie. Para probar la regla de compromiso, manda a Escribir una observación del agente que contenga una promesa y mira si aparece la orden en Compromisos.

Para probar la recuperación, llama a Recuperar y revisa en `memorias` las tres componentes de cada resultado: si la relevancia es 0 en todos, los embeddings no se están leyendo bien; si la recencia manda siempre, hay pocos recuerdos o todos tienen la misma importancia. Para provocar una reflexión sin esperar al umbral, llama a Reflexionar con `{"agente", "chat_id"}`. Para probar la voz del personaje sin escribir memoria, llama a Responder. Para probar las cotizaciones, llama a Cotizar con varios símbolos, incluido uno inexistente.

### Dónde mirar cuando algo falla

Sigue el orden del flujo. Primero el historial del Gateway: si la ejecución no aparece, Telegram no está entregando; si se para en un filtro, era un comando, un mensaje sin texto o el límite de frecuencia. Después Responder y Recuperar, y por último Escribir y Reflexionar. Si el agente no hace algo que prometió, mira si la orden está en Compromisos y después los historiales de Escribir, Comprometer y el intérprete. Make guarda el detalle de cada módulo con sus datos de entrada y salida.

### Al modificar escenarios

Al pegar un blueprint completo (Paste blueprint from clipboard), Make restablece la programación de los escenarios con webhook a «Every 15 minutes»: hay que volver a poner Immediately antes de guardar. Los escenarios programados conservan su programación. Un blueprint no crea webhooks: se crean a mano en el módulo disparador y, para los genéricos, hay que enviar una llamada de ejemplo para que Make detecte la estructura.

Si se añade una llamada nueva entre escenarios, hay que actualizar a mano la nota del webhook de destino.

Al reactivar un escenario que Make desactivó por un error, aparece la opción de procesar o borrar los datos que quedaron en la cola del webhook. Si el error estaba en esos datos, procésalos solo después de corregir el escenario.

### Mantenimiento de la memoria

Borrar filas de Memory Stream es seguro: no hay contadores derivados salvo Reflejada, y Citada_por se recalcula solo. Para reiniciar a un agente en un chat basta con borrar sus filas por Agente y Chat_ID. Cambiar el modelo o la dimensión de los embeddings obliga a recalcular el campo Embedding de todas las filas. Para trasladar recuerdos de una conversación a otra se cambia su Chat_ID. Las órdenes de Compromisos se pueden editar o borrar a mano: el intérprete no lleva más registro que la propia tabla.
