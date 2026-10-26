# Human_proxy · Documentación técnica

Human_proxy implementa en Make.com la arquitectura de *Generative Agents: Interactive Simulacra of Human Behavior* (Park et al., 2023, [arXiv 2304.03442](https://arxiv.org/abs/2304.03442)) para agentes que conversan por Telegram. Son 11 escenarios: cinco de memoria y diálogo, la planificación y el bucle percibir-reaccionar del paper, y tres extensiones propias.

## 1. Visión general

El paper describe un agente con capacidades encadenadas sobre un único registro de memoria, el *memory stream*. Percibe y almacena cada experiencia como una observación en lenguaje natural, con su marca temporal y una puntuación de importancia. Recupera los recuerdos pertinentes combinando lo reciente, lo importante y lo relevante. Reflexiona periódicamente, sintetizando observaciones en conclusiones que vuelven al mismo registro. Y planifica su día, reaccionando a lo que percibe.

El repositorio incluye como ejemplo un agente, Borja: un asesor financiero ficticio de una firma de hedge funds del Madrid de los 90. Todo lo específico de él está en dos sitios: su descripción, guardada en un almacén de datos, y la rama de eventos financieros de Planificar. Cambiar de agente es cambiar esas dos cosas.

| Escenario | Disparo | Función |
| --- | --- | --- |
| (1) Gateway Telegram | Mensaje en Telegram | Aplica el límite de frecuencia, pide la respuesta, la envía, la memoriza y lanza Estudiar y Planificar |
| (2) Memoria: Escribir | Webhook | Puntúa, vectoriza y guarda un recuerdo; dispara la reflexión si toca |
| (3) Memoria: Reflexionar | Webhook | Genera preguntas focales, recupera evidencias y guarda 5 reflexiones |
| (4) Memoria: Recuperar | Webhook | Devuelve los k recuerdos con mejor puntuación para una consulta |
| (5) Planificar | Webhook | Genera el plan de tareas del día y, para el agente financiero, la agenda de eventos |
| (6) Responder | Webhook | Construye el prompt de diálogo y genera la respuesta |
| (7) Resumen del agente | Webhook | Resume quién es el agente según su memoria, con caché de 24 h e interruptor ON/OFF |
| (8) Estudiar | Webhook | Detecta peticiones profesionales, investiga en Google y guarda conocimiento |
| (9) Interpelar | Webhook | Escribe al usuario por iniciativa propia cuando le falta un dato |
| (10) Despertador | Programado, diario a las 5:00 | Lanza la planificación aunque nadie escriba |
| (11) Percibir y reaccionar | Programado, cada hora de 9:00 a 23:00 en laborables | Estudia el evento que tocaba, calcula su implicación para cada usuario y pregunta si falta un dato |

Los modelos son Gemini 3.8 Flash para todo el razonamiento en lenguaje natural y Gemini Embedding 2, con 768 dimensiones, para los vectores. La memoria vive en Airtable y las cachés en almacenes de datos de Make.

## 2. Arquitectura

Cada escenario es una función: un webhook de entrada, un cuerpo JSON con sus parámetros y una respuesta JSON, o un «Accepted» inmediato si trabaja en segundo plano. Solo el Gateway recibe de Telegram. Solo Escribir crea recuerdos y solo Recuperar hace la búsqueda puntuada; los demás hacen operaciones simples en la tabla, como contar mensajes, leer el historial o leer el plan vigente.

Un mensaje recorre el sistema así:

1. El Gateway lee el perfil del agente, comprueba el límite de frecuencia y llama a Responder.
2. Responder pide el resumen del agente, lee lo que sabe del interlocutor, busca el bloque del plan vigente, recupera los recuerdos relevantes y el historial, y genera la respuesta.
3. El Gateway la envía a Telegram y guarda el mensaje y la respuesta como dos observaciones.
4. El Gateway pasa el mensaje a Estudiar en segundo plano y, si el agente aún no tiene plan para hoy, llama a Planificar.
5. Cada escritura suma la importancia de las observaciones aún no reflejadas y, por encima de 150, lanza Reflexionar.

Sin que nadie escriba, el Despertador planifica el día a las 5:00 y Percibir revisa cada hora los eventos planificados. Esta descomposición permite probar cada función sola enviando un JSON a su URL, añadir canales con un Gateway nuevo y mantener las reglas de la memoria en un único sitio.

| Escenario | Respuesta al llamante | Procesamiento |
| --- | --- | --- |
| (1) Gateway | No aplica | Secuencial, para respetar el orden de los mensajes |
| (2) Escribir | Síncrona, en cuanto el registro existe | Paralelo |
| (3) Reflexionar | 202 inmediato | Paralelo |
| (4) Recuperar | Síncrona; actualiza accesos después | Paralelo |
| (5) Planificar | «Accepted» inmediato | Secuencial, para no generar dos planes el mismo día |
| (6) Responder | Síncrona | Paralelo |
| (7) Resumen del agente | Síncrona, menos de 1 s; regenera después | Paralelo |
| (8) Estudiar | «Accepted» inmediato | Paralelo |
| (9) Interpelar | «Accepted» inmediato | Paralelo |
| (10) Despertador | No aplica | Paralelo |
| (11) Percibir y reaccionar | No aplica | Paralelo |

Make no admite el módulo de respuesta de webhook en un escenario secuencial y, además, desactiva un escenario secuencial en cuanto una ejecución falla. Por eso Planificar no devuelve nada y los escenarios que no deben pararse llevan un manejador de error Ignore en sus módulos externos (sección 18).

Cada escenario lleva notas de Make en sus webhooks y en cada llamada a webhook, con el origen y el destino:

| Webhook de destino | Lo llaman (escenario y módulo) |
| --- | --- |
| (2) Escribir | Gateway 4 y 5, Reflexionar 11, Planificar 13, 27 y 106, Estudiar 6, Interpelar 6, Percibir 10 |
| (3) Reflexionar | Escribir 8 |
| (4) Recuperar | Reflexionar 16, Responder 2 y 10, Resumen 10, 12 y 14, Estudiar 4 |
| (5) Planificar | Gateway 13, Despertador 2 |
| (6) Responder | Gateway 2 |
| (7) Resumen del agente | Responder 9 |
| (8) Estudiar | Gateway 15 |
| (9) Interpelar | Estudiar 7, Percibir 11 |

## 3. Modelo de datos

### La tabla Memory Stream

Toda la memoria vive en una sola tabla de Airtable, igual que en el paper, donde observaciones, reflexiones y planes son objetos del mismo registro y se recuperan con el mismo mecanismo. Cada fila es un recuerdo y el campo Tipo distingue su naturaleza.

| Campo | Tipo Airtable | Quién lo escribe | Significado |
| --- | --- | --- | --- |
| Memoria | Texto largo (campo primario) | Escribir | Descripción en lenguaje natural del recuerdo |
| Tipo | Selección única: observacion, reflexion, plan, evento, conocimiento | Escribir | Naturaleza del recuerdo |
| Agente | Texto de una línea | Escribir | Agente propietario del recuerdo |
| Chat_ID | Texto de una línea | Escribir | Chat de Telegram; segmenta la memoria por conversación. Planes y eventos usan `GLOBAL` |
| Autor | Selección única: usuario, agente, sistema | Escribir | Quién originó el contenido |
| Importancia | Número | Escribir | Intensidad emocional de 1 a 10 asignada por el LLM |
| Creado | Fecha de creación (automático) | Airtable | Marca temporal de creación |
| Ultimo_acceso | Fecha con hora | Escribir y Recuperar | Última vez que el recuerdo se creó o se recuperó; base de la recencia |
| Embedding | Texto largo | Escribir | Vector de 768 dimensiones serializado como `[n1, n2, …]` |
| Evidencias | Vínculo a registros de la misma tabla | Reflexionar | Recuerdos que justifican una reflexión |
| Citada_por | Vínculo recíproco de Evidencias | Airtable | Reflexiones que citan este recuerdo |
| Reflejada | Casilla | Reflexionar | La observación ya se ha contado para una reflexión |
| Plan_ubicacion | Texto de una línea | Planificar | Lugar del bloque de plan, o ámbito del evento |
| Plan_inicio | Fecha con hora | Planificar | Inicio del bloque o hora del evento |
| Plan_duracion_min | Número | Planificar | Duración en minutos |

Tres decisiones conviene tenerlas presentes. El paper guarda para cada objeto la descripción, la marca de creación y la de último acceso; aquí son Memoria, Creado y Ultimo_acceso, y el resto son añadidos para operar con varios agentes y conversaciones. El embedding se guarda como texto porque Airtable no tiene un tipo vectorial; ocupa unos 8–10 KB por fila y es la razón de que la recuperación tenga un techo de 300 registros. Evidencias es un vínculo real entre filas, así que el árbol de reflexiones del paper se puede navegar desde Airtable siguiendo Citada_por.

Reflejada no está en el paper. Allí la reflexión se dispara cuando «la suma de importancia de los eventos más recientes» supera un umbral; sin un estado explícito, cada observación nueva volvería a sumar las anteriores y el umbral se superaría en todos los mensajes a partir del primero que lo cruza. Reflejada funciona como un contador que se pone a cero.

Escribir crea los registros con la conversión de tipos de Airtable (typecast) activada, para que acepte valores nuevos en Tipo e intente convertir a número una importancia que llegue como texto.

Algunos textos siguen una convención porque otros escenarios buscan por ellos. Las observaciones que nacen de Interpelar empiezan por «{agente} le planteó una duda a». Los planes empiezan por «Plan de {agente} para el» y los eventos por «Evento financiero del».

### Almacenes de datos de Make

| Almacén | Clave | Contenido |
| --- | --- | --- |
| Caché (campos agente, chat_id, resumen, actualizado) | `perfil|<agente>` | Descripción del personaje, en el campo resumen, con chat_id `GLOBAL` |
| | `<agente>|<chat_id>` | Resumen del agente en ese chat (escenario 7) |
| | `rel|<agente>|<chat_id>` | Lo que el agente sabe de su interlocutor (escenario 6) |
| Cartera (campos agente, chat_id, cartera, actualizado) | `<agente>|<chat_id>` | Posiciones e intereses del usuario en texto libre (escenario 11) |

El perfil es la única fuente de la personalidad. Lo leen el Gateway, el Despertador y Percibir, y el Gateway lo pasa a Responder, Estudiar y Planificar. El límite de longitud de las respuestas no está en el perfil, porque solo aplica a Telegram: el Gateway añade «Limita la respuesta a una o dos frases, sin pasar de 20 palabras» únicamente en la llamada a Responder.

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
| R1 · 960 | Data store · Get a record | Lee `perfil|<agente>` |
| R1 · 2 | HTTP · POST a Responder | agente, `modo` (ON/OFF del resumen), chat_id, mensaje, usuario y descripción |
| R1 · 957 | Slack · Create a message | Registro opcional de cada intercambio en un canal |
| R1 · 3 | Telegram Bot · Send a Text Message | Envía la respuesta al chat |
| R1 · 4 | HTTP · POST a Escribir | Guarda «{nombre} le dijo a {agente}: «{mensaje}»», autor usuario |
| R1 · 5 | HTTP · POST a Escribir | Guarda «{agente} le respondió a {nombre}: «{respuesta}»», autor agente |
| R1 · 15 | HTTP · POST a Estudiar | Pasa el mensaje para que decida si investigar; no espera |
| R1 · 11 | Airtable · Search records | ¿Tiene el agente algún plan con fecha de hoy (Europe/Madrid)? |
| R1 · 12 | Tools · Numeric aggregator | Cuenta los planes de hoy |
| R1 · 13 | HTTP · POST a Planificar | Solo si no hay plan de hoy; no espera |

Todos los módulos que pueden fallar llevan un manejador Ignore: si uno falla, esa ejecución termina y el Gateway sigue activo para el siguiente mensaje. Sin ellos, Make desactivaría el escenario al primer error porque es secuencial.

El límite de frecuencia protege el consumo frente a ráfagas. Cuenta solo los mensajes del usuario ya guardados; como el mensaje en curso se guarda después de responder, no se cuenta a sí mismo. La condición de fecha, `IS_AFTER({Creado}, DATEADD(NOW(), -10, 'minutes'))`, compara instantes absolutos y no depende de la zona horaria.

La búsqueda de Airtable en Make devuelve un paquete vacío (`__IMTLENGTH__` = 0) cuando no encuentra nada. Por eso los contadores suman `if(id; 1; 0)` en lugar de 1.

Las observaciones se redactan en tercera persona y con el nombre del interlocutor porque el paper guarda las percepciones como frases narrativas («Isabella Rodriguez is setting out the pastries»). Así el texto que luego lee el LLM es autoexplicativo.

El procesamiento secuencial hace que tres mensajes seguidos se ejecuten uno detrás de otro. Sin él, el segundo podría recuperar memoria antes de que se guardara el primero. El coste es la latencia: cada mensaje espera al anterior, unos 10–20 segundos.

Todos los textos que entran en un cuerpo JSON pasan por `escapeJSON()`, para que unas comillas o un salto de línea no rompan la llamada. Telegram solo admite un webhook por bot: si el bot ya lo usaba otro escenario, este deja de recibir al crear el del Gateway.

Para tener varios agentes basta con duplicar el Gateway y cambiar el nombre del agente, la clave del perfil y la conexión de Telegram: la memoria ya está separada por el campo Agente.

## 5. (2) Memoria: Escribir

Escribir es la única puerta de entrada a la memoria: toda observación, reflexión, plan o pieza de conocimiento pasa por aquí, así que todos quedan puntuados y vectorizados con el mismo criterio. Responde en cuanto el registro existe y después comprueba si hay que reflexionar.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 2 | Gemini · Generate a response | Puntúa la importancia del recuerdo de 1 a 10 |
| 3 | Gemini · Generate embeddings | Vector de la memoria, 768 dimensiones |
| 4 | Airtable · Create a record | Crea la fila con typecast, Importancia, Ultimo_acceso = ahora y Embedding |
| 5 | Webhooks · Webhook response | 200 con `{"id", "importancia", "tipo"}` |
| 6 | Airtable · Search records | Observaciones del mismo agente y chat sin reflejar (hasta 500) |
| 7 | Tools · Numeric aggregator | Suma su importancia |
| 8 | HTTP · POST a Reflexionar | Solo si la suma supera 150 y el recuerdo nuevo es una observación |

### Importancia

El paper pide al LLM que puntúe la intensidad del recuerdo con una escala anclada en dos ejemplos extremos. El prompt es su traducción, con la instrucción añadida de devolver solo el número:

```text
En una escala del 1 al 10, donde 1 es puramente mundano (p. ej., cepillarse los dientes, hacer la cama) y 10 es extremadamente conmovedor (p. ej., una ruptura, ser admitido en la universidad), puntúa la probable intensidad emocional del siguiente recuerdo de {{1.agente}}.
Responde únicamente con un número entero del 1 al 10.
Recuerdo: {{1.memoria}}
Puntuación:
```

La importancia se calcula una vez y no cambia. Es lo que permite que un acontecimiento importante siga apareciendo semanas después aunque su recencia haya caído.

### Embedding

El vector se guarda como texto entre corchetes. Su dimensión debe coincidir con la de la consulta en Recuperar (768 en ambos). Si se cambia el modelo o la dimensión en uno de los dos, hay que cambiarla en el otro y recalcular los embeddings existentes: la similitud entre vectores de distinta longitud se trata como 0.

### Disparo de la reflexión

El filtro del módulo 8 exige dos condiciones: suma mayor que 150 y tipo observacion. La segunda evita un bucle, porque las reflexiones también se guardan llamando a Escribir. Con mensajes de chat corrientes, que el modelo suele puntuar entre 2 y 4, el umbral se alcanza tras unas 40–60 observaciones, es decir, 20–30 intercambios.

### Entrada y salida

Entrada: `memoria` (obligatorio), `tipo`, `agente`, `chat_id`, `autor` y, opcionales, `evidencias` (IDs de registro), `plan_ubicacion`, `plan_inicio` y `plan_duracion_min`. Salida: `{"id": "rec…", "importancia": "4", "tipo": "observacion"}`.

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

## 8. (5) Planificar

Planificar genera una vez al día el plan del agente y lo guarda como recuerdos con Chat_ID `GLOBAL`, porque el agente tiene un único día para todos sus interlocutores. Lo llaman el Despertador a las 5:00 y el Gateway si al llegar un mensaje todavía no hay plan de hoy.

Tras el webhook, un router reparte en tres rutas:

| Ruta | Para quién | Qué genera |
| --- | --- | --- |
| Es cualquier otro | Cualquier agente salvo el financiero | Un plan de 5 a 8 puntos coherente con su personaje, su ciudad, su época y el día de la semana |
| Es Borja, nuestro Financial Adviser | El agente financiero | Su plan de tareas, de 3 a 5 puntos |
| Es Borja · capa de eventos | El agente financiero, a la vez que la anterior | La agenda de eventos financieros del día |

Las dos rutas de tareas siguen los mismos pasos: calculan la fecha, comprueban si ya hay plan de hoy, leen el plan de ayer para dar continuidad, piden a Gemini el plan en trazos gruesos y luego su descomposición en bloques de hasta una hora (JSON con inicio, duración, actividad y lugar), y guardan cada bloque mediante Escribir con Tipo `plan`. Un bloque queda como «Plan de Borja para el viernes 23/10/2026, de 14:30 a 15:30: …».

La capa de eventos (módulos 100 a 109) comprueba si ya hay eventos de hoy y, si no, pide a Gemini con búsqueda de Google la agenda financiera del día: de 3 a 8 eventos con hora confirmada, en hora de Madrid, de datos macroeconómicos de España, la zona euro y Estados Unidos, bancos centrales, subastas de deuda y resultados del Ibex 35 y de grandes empresas americanas. Gemini devuelve una línea por evento con el formato `HH:mm|evento|ámbito|fuente`. Cada línea válida se guarda con Tipo `evento`, la hora en Plan_inicio, 30 minutos de duración y el ámbito en Plan_ubicacion. Responder no confunde los eventos con lo que el agente está haciendo, porque solo busca registros de tipo `plan`.

Plan_inicio se escribe en ISO con el desfase de Madrid, así que Airtable guarda el instante correcto. La instrucción de sistema de Gemini es la descripción del personaje, o su nombre si no llega descripción. El escenario es secuencial: si dos llamadas piden plan a la vez, la segunda espera y, al llegar su turno, encuentra el plan escrito y no hace nada.

## 9. (6) Responder

Responder genera lo que dice el agente con el prompt de diálogo del paper: quién es, qué hora es, qué está haciendo, qué sabe de su interlocutor, qué recuerda y qué se ha dicho. No escribe en la memoria de la conversación; eso lo hace el Gateway después de enviar la respuesta, para que Responder pueda usarse desde otros canales o en pruebas.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Webhooks · Custom webhook | Entrada |
| 9 | HTTP · POST a Resumen del agente | Resumen del agente; vacío si el modo no es ON |
| 2 | HTTP · POST a Recuperar | El mensaje como consulta, k = 10 |
| 12 | Data store · Get a record | Lo que el agente sabe del interlocutor (`rel|agente|chat_id`) |
| 3 | Airtable · Search records | Las 12 observaciones más recientes del chat |
| 4 | Tools · Text aggregator | Una línea por observación: `- [autor] texto` |
| 7 | Airtable · Search records | El bloque del plan que abarca este momento |
| 8 | Tools · Text aggregator | Su texto, o vacío |
| 5 | Gemini · Generate a response | Genera la respuesta |
| 6 | Webhooks · Webhook response | `{"respuesta": "…"}` |
| 10 | HTTP · POST a Recuperar | Solo si la relación no existe o tiene más de 24 h: qué sabe el agente de esa persona, k = 10 |
| 11 | Gemini | La resume en una o dos frases |
| 13 | Data store · Add a record | La guarda, salvo que diga que aún no sabe nada concreto |

La instrucción de sistema fija el personaje y las reglas de conversación:

```text
{{1.descripcion}}

Lo que {{1.agente}} ha llegado a saber de sí mismo por lo vivido en esta conversación (si está vacío, ignóralo):
{{9.data.resumen}}

Eres {{1.agente}} y estás hablando por Telegram. Mantente siempre en tu personaje. Responde en el idioma del interlocutor, con la extensión propia de un chat, y no inventes recuerdos que no aparezcan en tu memoria.
```

El mensaje de usuario sigue la estructura del paper:

```text
Son las {{formatDate(now; "HH:mm [del] DD/MM/YYYY"; "Europe/Madrid")}}.

Lo que {{1.agente}} tiene planificado para este momento: {{ifempty(8.text; "nada en concreto")}}

Lo que {{1.agente}} sabe de {{ifempty(1.usuario; "su interlocutor")}} y de su relación: {{ifempty(12.resumen; "todavía no hay un resumen; guíate por el historial")}}

Resumen del contexto relevante de la memoria de {{1.agente}} (ordenado por relevancia):
{{2.data.texto}}

Historial reciente de la conversación (del más reciente al más antiguo):
{{4.text}}

{{ifempty(1.usuario; "El interlocutor")}} le dice a {{1.agente}}: {{1.mensaje}}

¿Cómo respondería {{1.agente}}? Escribe únicamente su respuesta, sin comillas ni prefijos.
```

El prompt separa dos memorias. Los recuerdos relevantes son la memoria a largo plazo, elegidos por puntuación y que pueden venir de semanas atrás o ser reflexiones. El historial de 12 observaciones es la memoria de trabajo, el hilo literal de la conversación, sin el cual el agente perdería la continuidad inmediata. El historial va del más reciente al más antiguo, y el prompt lo declara para que el modelo no lo lea al revés.

El resumen de la relación sigue el paper, que antes de cada diálogo recupera lo que el agente sabe de su interlocutor. Se regenera después de responder, como mucho una vez cada 24 horas por chat: la respuesta usa siempre la versión anterior y la nueva vale desde el mensaje siguiente, así que el usuario no espera por ella.

El plan vigente se busca entre los planes `GLOBAL` con `IS_BEFORE({Plan_inicio}, DATEADD(NOW(), 1, 'minutes'))` y `IS_AFTER(DATEADD({Plan_inicio}, {Plan_duracion_min}, 'minutes'), NOW())`. La hora se formatea en Europe/Madrid porque el paper siempre sitúa al agente en su hora actual.

## 10. (7) Resumen del agente

El Resumen del agente reconstruye quién es el agente a partir de su memoria con las tres consultas del paper: sus rasgos esenciales, su ocupación diaria actual y cómo se siente respecto a su progreso reciente. Responder lo añade a la instrucción de sistema, detrás de la descripción fija.

| Entrada | Comportamiento | Tiempo |
| --- | --- | --- |
| `modo` distinto de ON, o ausente | Devuelve `{"resumen": "", "estado": "OFF"}` sin consultar nada | < 1 s |
| ON, con caché de menos de 24 h | Devuelve el resumen guardado, estado cache | < 1 s |
| ON, sin caché o caducada | Devuelve el resumen anterior (o vacío) y lo regenera después | < 1 s; la regeneración, unos 35 s |

La regeneración lanza las tres consultas contra Recuperar (k = 15) y resume cada una con Gemini en una o dos frases, o «sin datos» si no hay afirmaciones suficientes. Guarda el resultado con clave `agente|chat_id`, salvo que los tres apartados sean «sin datos»: así un chat nuevo no se queda 24 horas sin resumen. El interruptor está en el Gateway, que envía `"modo": "ON"`. Como la memoria es por chat, el agente puede verse distinto en cada conversación.

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

No cuentan los saludos, la charla, las opiniones, los asuntos personales ni lo que se responde con sentido común. Falta un dato cuando la cuestión es profesional pero no se puede investigar con fiabilidad sin saber, por ejemplo, el país, la herramienta, la versión, el sector o el tipo de contrato. La respuesta del momento no usa la investigación; la usan los mensajes siguientes, a través de Recuperar.

## 12. (9) Interpelar

Interpelar escribe al usuario por iniciativa propia, en el personaje del agente, para resolver una duda concreta. Lo llaman Estudiar, cuando a una petición profesional le falta un dato imprescindible, y Percibir, cuando para valorar el impacto de un evento le falta un dato de la cartera.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 2 | Airtable · Search records | Interpelaciones de este agente en este chat en los últimos 30 minutos |
| 3 | Tools · Numeric aggregator | Las cuenta |
| 7 | Airtable · Search records | Interpelaciones de hoy en este chat |
| 8 | Tools · Numeric aggregator | Las cuenta |
| 4 | Gemini | Si no hubo ninguna en 30 minutos y van menos de dos hoy, redacta una sola pregunta breve, en el personaje |
| 5 | Telegram Bot · Send a Text Message | La envía al chat |
| 6 | HTTP · POST a Escribir | Guarda «{agente} le planteó una duda a {usuario}: «{pregunta}»» como observación del agente |

Los dos límites evitan que el agente encadene preguntas, también en un día con muchos eventos. La pregunta queda en el historial: cuando el usuario contesta, Responder la ve en la conversación reciente y Estudiar recibe la respuesta como un mensaje nuevo que ya puede investigar.

## 13. (10) Despertador

El Despertador lanza la planificación todos los días a las 5:00, hora de Madrid, aunque nadie escriba. Sin él, el plan solo nacía con el primer mensaje del día, y un día sin mensajes no había agenda de eventos para Percibir.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Data store · Get a record | Lee `perfil|<agente>` |
| 2 | HTTP · POST a Planificar | Si el perfil existe, envía agente y descripción |

Planificar lanza a la vez las rutas de tareas y de eventos, y ninguna duplica nada si después llega un mensaje, porque las dos comprueban si ya hay registros de hoy.

## 14. (11) Percibir y reaccionar

Percibir es el bucle percibir-reaccionar del paper adaptado a un agente sin mundo simulado: cada hora mira qué evento financiero tocaba, lo investiga, calcula su implicación para la cartera de cada usuario y, si le falta un dato, pregunta. Se ejecuta 15 veces cada día laborable, a cada hora en punto de 9:00 a 23:00. Si en la franja no hay evento, termina tras gastar unos 2 créditos.

| # | Módulo | Qué hace |
| --- | --- | --- |
| 1 | Data store · Get a record | Lee el perfil del agente |
| 2 | Airtable · Search records | Eventos del agente cuya hora cayó entre hace 2 h y hace 1 h |
| 3 | Gemini con búsqueda de Google | Resultado del evento en 60 palabras con cifras y fuente, o `NO_PUBLICADO` |
| 4 | Airtable · Search records | Todos los chats en los que algún usuario ha escrito al agente |
| 5 | Tools · Text aggregator | Lista de chats, uno por línea |
| 6 | Iterator | Recorre los chats sin duplicados |
| 7 | Data store · Get a record | La cartera del chat, clave `agente|chat_id` |
| 8 | Gemini (salida JSON con esquema) | Con cartera: la implicación para cada posición afectada y la duda si falta un dato. Sin cartera: un comentario general para un inversor particular, sin duda |
| 9 | JSON · Parse JSON | Separa conocimiento y duda |
| 10 | HTTP · POST a Escribir | Guarda el conocimiento en ese chat |
| 11 | HTTP · POST a Interpelar | Solo si hay duda |

El resultado del evento se investiga una vez y la implicación se calcula para cada chat. Lo aprendido se guarda en cada chat porque Responder solo busca en la memoria de la conversación en curso.

La franja de una a dos horas da margen para que un dato publicado esté indexado: uno de las 14:30 se estudia a las 16:00. Cada evento cae en una sola franja, así que no hace falta llevar registro de lo estudiado. Si el dato aún no está publicado, el evento se pierde, porque no hay reintento.

La cartera es texto libre: posiciones, pesos, horizonte y preocupaciones, por ejemplo «Inditex 30 %, fondo indexado MSCI World 40 %, liquidez 30 %; horizonte largo; le preocupa la inflación». Se rellena desde Make.

## 15. Contratos de los webhooks

Cada escenario con webhook se invoca con un POST JSON (`Content-Type: application/json`). Cualquier canal nuevo, como WhatsApp, un formulario web u otro agente, puede usar la memoria llamando a estas URLs sin tocar los escenarios 2 a 4 y 6.

| Función | URL | Entrada | Salida |
| --- | --- | --- | --- |
| Escribir (2) | `https://hook.eu1.make.com/<WEBHOOK_ESCRIBIR>` | `memoria`, `tipo`, `agente`, `chat_id`, `autor`; opcionales `evidencias[]`, `plan_ubicacion`, `plan_inicio`, `plan_duracion_min` | 200 `{"id", "importancia", "tipo"}` |
| Reflexionar (3) | `https://hook.eu1.make.com/<WEBHOOK_REFLEXIONAR>` | `agente`, `chat_id` | 202 `{"estado": "reflexion en curso", "agente"}` |
| Recuperar (4) | `https://hook.eu1.make.com/<WEBHOOK_RECUPERAR>` | `agente`, `chat_id`, `consulta`; opcional `k` (10) | 200 `{"memorias": [...], "texto", "texto_ids", "total_evaluadas"}` |
| Planificar (5) | `https://hook.eu1.make.com/<WEBHOOK_PLANIFICAR>` | `agente`, `descripcion`; opcional `fecha` (AAAA-MM-DD, hoy por defecto) | «Accepted» |
| Responder (6) | `https://hook.eu1.make.com/<WEBHOOK_RESPONDER>` | `agente`, `chat_id`, `mensaje`, `descripcion`; opcionales `usuario`, `modo` | 200 `{"respuesta"}` |
| Resumen del agente (7) | `https://hook.eu1.make.com/<WEBHOOK_RESUMEN>` | `agente`, `chat_id`, `modo` (ON u OFF; OFF si falta) | 200 `{"resumen", "estado"}` |
| Estudiar (8) | `https://hook.eu1.make.com/<WEBHOOK_ESTUDIAR>` | `agente`, `chat_id`, `mensaje`; opcionales `usuario`, `descripcion` | «Accepted» |
| Interpelar (9) | `https://hook.eu1.make.com/<WEBHOOK_INTERPELAR>` | `agente`, `chat_id`, `motivo`; opcionales `usuario`, `contexto`, `descripcion` | «Accepted»; envía el mensaje por Telegram |

Ejemplos de llamada:

```json
POST Escribir
{"memoria": "Ana le dijo a Borja: «Tengo una reunión con el banco en mayo»", "tipo": "observacion", "agente": "Borja", "chat_id": "123456789", "autor": "usuario"}

POST Recuperar
{"agente": "Borja", "chat_id": "123456789", "consulta": "¿Qué sabe Borja de los planes de Ana?", "k": 5}

POST Responder
{"agente": "Borja", "chat_id": "123456789", "mensaje": "¿Qué tal el día?", "usuario": "Ana", "descripcion": "Borja es un asesor financiero..."}
```

## 16. Fidelidad al paper

La memoria sigue el paper en lo que la define: el registro único, la importancia asignada por el LLM, la fórmula de recuperación y el ciclo de reflexión con citas. Las desviaciones vienen de pasar de un mundo simulado a una conversación de chat y de los límites de Make y Airtable.

| Elemento | Paper | Human_proxy | Valoración |
| --- | --- | --- | --- |
| Objeto de memoria | Descripción, creación, último acceso | Memoria, Creado, Ultimo_acceso + metadatos | Fiel |
| Importancia | LLM, escala 1–10 anclada | Mismo prompt traducido | Fiel |
| Recencia | 0,995 por hora de juego desde el último acceso | 0,995 por hora real | Fiel en la forma; cambia la escala temporal |
| Relevancia | Coseno entre embeddings | Coseno, Gemini Embedding 2, 768 dimensiones | Fiel |
| Combinación | Min-max a [0, 1] y suma con α = 1 | Igual | Fiel |
| Candidatos | Toda la memoria del agente | Los 300 registros más recientes del chat | Desviación por límite técnico (sección 18) |
| Cuántos se recuperan | Los que caben en el contexto | k fijo, 10 por defecto | Simplificación razonable |
| Disparo de reflexión | Suma de importancia reciente > 150 | Suma de observaciones no reflejadas > 150 | Fiel |
| Preguntas focales | 3, a partir de los 100 registros más recientes | Igual | Fiel |
| Insights | 5, con «(because of 1, 5, 3)» | 5, con IDs reales en un JSON con esquema | Fiel y más robusto |
| Árbol de reflexiones | Las reflexiones pueden citar reflexiones | Igual, vía Evidencias / Citada_por | Fiel |
| Resumen del agente | Generado desde la memoria y cacheado | Las tres consultas del paper, caché de 24 h | Fiel; la descripción fija sigue siendo la base |
| Diálogo | Recupera lo que sabe del interlocutor antes de hablar | Resumen de la relación, caché de 24 h | Fiel |
| Planificación | Plan diario, por horas y por tramos de 5–15 minutos | Plan diario y bloques de hasta una hora, más agenda de eventos | Parcial: sin tramos de 5–15 minutos |
| Percibir y reaccionar | El agente percibe su entorno en cada paso y decide si reacciona y replanifica | Revisión horaria de los eventos planificados, con reacción por usuario | Parcial: no replanifica |
| Ámbito de la memoria | Un agente, un mundo | Un agente por chat | Decisión deliberada: el agente no mezcla lo que le cuentan usuarios distintos |
| Entorno y otros agentes | Mundo simulado con espacios y varios agentes | No existe | Inherente al canal |

Estudiar, Interpelar y la cartera de Percibir no están en el paper; son extensiones para un agente que conversa con personas reales y necesita información verificable.

## 17. Costes

Con un usuario, el sistema gasta unos 9.400 créditos de Make al mes antes de que nadie converse: el plan diario (≈ 6.600) y Percibir (≈ 2.800). En el coste manda Make, no Gemini: los créditos son el 85–90 % de cada ejecución.

Precios aplicados: Make Core, 10.000 créditos por 10,59 $ al mes (0,00106 $ por crédito; 0,0009 $ en anual); Gemini 3.8 Flash, 0,75 $ por millón de tokens de entrada y 3,75 $ por millón de salida, razonamiento incluido, hasta el 31 de diciembre de 2026 y el doble desde el 1 de enero de 2027; búsqueda de Google gratis hasta 5.000 consultas al mes. Conversión a euros al cambio del Banco de España del 19 de octubre de 2026, 1,1225 $ por euro. Los créditos se midieron en el historial de cada escenario; los tokens, en el inspector de Make para Responder, Escribir y Estudiar, y el resto se estima por el tamaño del prompt.

### Coste unitario por escenario

Solo lo que gasta el propio escenario; las llamadas a otros escenarios se suman en la tabla siguiente.

| Escenario | Créditos | Make (€) | Gemini (€) | Gemini |
| --- | --- | --- | --- | --- |
| (1) Gateway | 11–12 | 0,011 | 0 | — |
| (2) Escribir | 7 | 0,0066 | 0,0012 | medido |
| (3) Reflexionar | 25 | 0,024 | ≈ 0,009 | estimado |
| (4) Recuperar k = 5 / 10 / 15 | 12 / 17 / 18 | 0,011 / 0,016 / 0,017 | < 0,0001 | medido |
| (5) Planificar, tareas | ≈ 48 | 0,045 | ≈ 0,013 | estimado |
| (5) Planificar, eventos (≈ 5 eventos) | ≈ 15 | 0,014 | ≈ 0,004 | estimado |
| (6) Responder, con caché / regenerando la relación | 10 / 13 | 0,009 / 0,012 | 0,0017 / 0,0045 | medido |
| (7) Resumen OFF / caché / regenerando | 2 / 3 / 10 | 0,002 / 0,003 / 0,009 | 0 / 0 / ≈ 0,006 | estimado |
| (8) Estudiar sin investigar / ya sabido / investigando | 3 / 5 / 6 | 0,003 / 0,005 / 0,006 | 0,001 / ≈ 0,003 / 0,0047 | medido |
| (9) Interpelar | 8 | 0,0076 | ≈ 0,002 | estimado |
| (10) Despertador | 1 | 0,0009 | 0 | — |
| (11) Percibir sin evento | 2 | 0,0019 | 0 | — |
| (11) Percibir por evento + por chat | 6 + 4 | 0,006 + 0,004 | ≈ 0,004 + ≈ 0,0025 | estimado |

En las llamadas medidas manda el razonamiento interno de Gemini: la importancia de Escribir devuelve 1 token y razona 331; la investigación de Estudiar recibe 521 tokens, escribe 120 y razona 880.

### Coste por operación completa

| Operación | Escenarios que encadena | Créditos | Total (€) |
| --- | --- | --- | --- |
| Mensaje de Telegram con cachés vigentes | Gateway, Responder, Recuperar k = 5, Resumen en caché, 2 × Escribir, Estudiar | ≈ 54 | ≈ 0,056 |
| Regenerar resumen y relación (una vez al día por chat) | Resumen con 3 × Recuperar k = 15, relación con Recuperar k = 10 | ≈ 74 | ≈ 0,08 |
| Estudiar con investigación | + Recuperar, Escribir | + 22 | + 0,026 |
| Interpelación | Interpelar, Escribir | ≈ 15 | ≈ 0,017 |
| Reflexión | Reflexionar, 3 × Recuperar k = 10, 5 × Escribir | ≈ 111 | ≈ 0,12 |
| Plan diario completo | Despertador, Planificar (tareas y eventos), ≈ 22 × Escribir | ≈ 220 | ≈ 0,25 |
| Evento en Percibir, primer usuario | Percibir, Escribir | ≈ 17 | ≈ 0,025 |
| Cada usuario más por evento | Percibir, Escribir | + 11 | + 0,014 |
| Conversación de 20 mensajes | 20 mensajes, regeneración diaria, 2 investigaciones, 1 interpelación, 0–1 reflexión | 1.210–1.320 | 1,27–1,40 |

### Coste mensual

| Concepto | Créditos al mes | Coste (€) |
| --- | --- | --- |
| Plan diario, 30 días | ≈ 6.600 | ≈ 7,40 |
| Percibir, 1 usuario: 330 vueltas, ≈ 120 eventos, hasta 44 preguntas | ≈ 2.800 | ≈ 3,30 |
| Cada usuario más en Percibir | ≈ 1.300–2.000 | ≈ 1,70–2,50 |
| Cada conversación de 20 mensajes | ≈ 1.300 | ≈ 1,35 |

Un plan de 10.000 créditos no da para el sistema en marcha con conversaciones. Tres cambios recortan el gasto fijo. Guardar los bloques del plan y los eventos con un Airtable · Create record directo (1 crédito, sin importancia ni embedding) en lugar de Escribir (7) ahorra unos 130 créditos al día, porque Responder y Percibir los buscan por fecha y no por similitud. Programar el Despertador solo en laborables ahorra unos 1.700 al mes. Bajar el nivel de razonamiento de Gemini en la importancia de Escribir recorta 4–5 segundos por memoria, aunque apenas dinero. Si Gemini duplica su precio en 2027, un mensaje pasa de ≈ 0,056 € a ≈ 0,061 €.

Fuentes: [precios de la API de Gemini](https://ai.google.dev/gemini-api/docs/pricing), [Gemini 3.8 Flash, Layer3 Labs](https://www.layer3labs.io/guides/gemini-3-8-flash-pricing), [Gemini Embedding 2, Maxim](https://www.getmaxim.ai/bifrost/llm-cost-calculator/provider/gemini/model/gemini-embedding-2), [precios de Make](https://www.make.com/en/pricing), [precios de Make, Jetadmin](https://www.jetadmin.io/blog/make-pricing/), [cambio oficial del euro, BOE 20/10/2026](https://www.boe.es/boe/dias/2026/10/20/pdfs/BOE-A-2026-21865.pdf).

## 18. Limitaciones y riesgos

### Escala de la memoria

Recuperar solo evalúa los 300 registros más recientes del chat. A partir de unos 150 intercambios más sus reflexiones, los recuerdos más antiguos dejan de ser candidatos aunque sean importantes, cosa que el paper no hace. El techo existe porque todos los embeddings viajan como texto al módulo de código, del orden de 2,7 MB por llamada. Las salidas, de menor a mayor esfuerzo: subir el techo y vigilar tiempos; prefiltrar por importancia (por ejemplo, las 200 más recientes más las 100 más importantes); o mover los vectores a una base vectorial y dejar Airtable como registro legible.

El volumen total lo limita el plan de Airtable: 1.000 registros por base en Free, 50.000 en Team y 125.000 en Business ([Jetadmin, julio de 2026](https://www.jetadmin.io/blog/airtable-pricing-in-2026-plans-real-costs-and-when-to-upgrade/)). Con dos registros por turno, más cinco por reflexión y los planes diarios, el límite de Free se alcanza pronto.

### Errores

Make desactiva un escenario secuencial en cuanto una ejecución falla, y uno en paralelo tras varios errores seguidos. Gateway, Planificar, Estudiar, Interpelar, Despertador y Percibir llevan un manejador Ignore en todos los módulos que pueden fallar. Escribir, Reflexionar, Recuperar, Responder y el Resumen del agente no lo llevan, con estas consecuencias:

1. Si Gemini o Airtable fallan durante Responder, el usuario no recibe respuesta; el Gateway sigue activo.
2. Si Gemini devuelve la importancia con texto alrededor («Puntuación: 7»), la conversión de Airtable puede fallar y ese recuerdo se pierde.
3. Si el LLM inventa un ID en las evidencias de una reflexión, falla la actualización del módulo 14 de Reflexionar y no se escriben las reflexiones restantes de esa tanda.
4. Como el Gateway responde antes de memorizar, un fallo en Escribir no lo ve el usuario: la conversación sigue, con huecos en la memoria.

### Latencia

Un mensaje tarda unos 10–20 segundos en recibir respuesta: embedding y búsqueda en Recuperar, código y generación. Con el procesamiento secuencial, los mensajes en ráfaga se encolan.

### Seguridad y privacidad

Make no protege las URLs de los webhooks: quien tenga una puede invocar el escenario, leer o escribir la memoria y gastar créditos. No las publiques y, si se filtra alguna, regenera ese webhook y actualiza los módulos que lo llaman. Si el sistema se abre a terceros, conviene añadir una cabecera secreta en las llamadas entre escenarios.

La tabla guarda literalmente lo que cada usuario dice al bot, segmentado por su chat. Si lo usan otras personas, deben saberlo.

### Otros

En un grupo de Telegram todos los miembros comparten chat y, por tanto, memoria; las observaciones sí distinguen a cada uno por su nombre. Telegram limita los mensajes a 4.096 caracteres. El límite de 10 mensajes en 10 minutos cuenta solo mensajes ya guardados.

El conocimiento que guardan Estudiar y Percibir compite en desventaja en la recuperación: la importancia mide la carga emocional, como en el paper, y un dato fiscal o de mercado puntúa bajo. Además, ese conocimiento es por chat. Interpelar envía siempre por el mismo bot; con varios agentes y bots habría que pasarle cuál usar.

### Mejoras posibles

1. Replanificar el día cuando una conversación cambie la agenda del agente.
2. Reintentar en la vuelta siguiente los eventos cuyo dato aún no estaba publicado.
3. Dar una importancia fija alta al tipo conocimiento para que compita en la recuperación.
4. Añadir manejadores de error al resto de escenarios y extraer el número de la importancia antes de guardarla.
5. Guardar planes y eventos con escritura directa en Airtable para reducir créditos.

## 19. Operación

### Pruebas aisladas

Cada función se puede probar sola con un POST a su webhook, sin pasar por Telegram. Conviene usar un chat_id de prueba para no mezclar esos recuerdos con conversaciones reales, y excluirlo después de Percibir, que recorre todos los chats.

Para probar la recuperación, llama a Recuperar y revisa en `memorias` las tres componentes de cada resultado: si la relevancia es 0 en todos, los embeddings no se están leyendo bien; si la recencia manda siempre, hay pocos recuerdos o todos tienen la misma importancia. Para provocar una reflexión sin esperar al umbral, llama a Reflexionar con `{"agente", "chat_id"}`. Para probar la voz del personaje sin escribir memoria, llama a Responder.

### Dónde mirar cuando algo falla

Sigue el orden del flujo. Primero el historial del Gateway: si la ejecución no aparece, Telegram no está entregando; si se para en un filtro, era un comando, un mensaje sin texto o el límite de frecuencia. Después Responder y Recuperar, y por último Escribir y Reflexionar. Make guarda el detalle de cada módulo con sus datos de entrada y salida.

### Al modificar escenarios

Al pegar un blueprint completo (Paste blueprint from clipboard), Make restablece la programación de los escenarios con webhook a «Every 15 minutes»: hay que volver a poner Immediately antes de guardar. Los escenarios programados conservan su programación. Un blueprint no crea webhooks: se crean a mano en el módulo disparador y, para los genéricos, hay que enviar una llamada de ejemplo para que Make detecte la estructura.

Si se añade una llamada nueva entre escenarios, hay que actualizar a mano la nota del webhook de destino.

Al reactivar un escenario que Make desactivó por un error, aparece la opción de procesar o borrar los datos que quedaron en la cola del webhook. Si el error estaba en esos datos, procésalos solo después de corregir el escenario.

### Mantenimiento de la memoria

Borrar filas de Memory Stream es seguro: no hay contadores derivados salvo Reflejada, y Citada_por se recalcula solo. Para reiniciar a un agente en un chat basta con borrar sus filas por Agente y Chat_ID. Cambiar el modelo o la dimensión de los embeddings obliga a recalcular el campo Embedding de todas las filas. Para trasladar recuerdos de una conversación a otra se cambia su Chat_ID.
