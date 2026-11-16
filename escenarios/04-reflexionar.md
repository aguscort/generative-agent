# 04 · Reflexionar

**Escenario:** `[Human_proxy]:[02-Memoria] (04/20) | Reflexionar`
**Blueprint:** [`blueprints/04-reflexionar.blueprint.json`](../blueprints/04-reflexionar.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_REFLEXIONAR__`).
**Lo llama:** el Despertador, cada mañana, una vez por cada chat que tenga observaciones sin reflejar.

## Función

Implementa la reflexión de Park et al. (2023): a partir de las experiencias recientes, el agente saca conclusiones de nivel superior y las guarda en la memoria como recuerdos nuevos.

El proceso tiene cinco pasos:

1. **Umbral.** Suma la importancia de las observaciones que todavía no se han reflejado. Si no llega a 30, no reflexiona.
2. **Preguntas.** Con esas observaciones, Gemini formula hasta tres preguntas sobre la actuación del agente.
3. **Recuperación.** Por cada pregunta, Recuperar trae los recuerdos relacionados, sean de cuando sean.
4. **Conclusiones.** Con lo recuperado, Gemini escribe como máximo dos conclusiones nuevas y decide si hace falta actuar.
5. **Cierre.** Guarda las conclusiones con sus evidencias y, si hay acción, la pide a Comprometer. Al final marca como reflejadas las observaciones de las que ha partido.

Cada conclusión es un hecho que las evidencias dan por comprobado, o un criterio para actuar mejor. Nunca es una tarea: las tareas van como acción a Comprometer. Para no repetirse, Gemini sabe cuáles de los recuerdos recuperados son conclusiones anteriores y tiene la instrucción de no volver a escribirlas.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat_id` | Chat sobre el que se reflexiona. |

## Respuesta

| Caso | Código | Cuerpo |
|---|---|---|
| Por debajo del umbral | 200 | `{"estado": "sin material suficiente", "agente": "<agente>"}` |
| Sin preguntas | 200 | `{"estado": "reflexion completada", "agente": "<agente>", "reflexiones": 0}` |
| Reflexión hecha | 200 | `{"estado": "reflexion completada", "agente": "<agente>", "reflexiones": <n>}` |
| Fallo | 500 | `{"error": "Reflexionar › <paso> (<servicio>)"}` |

## Llamadas a otros escenarios

| Módulo | Destino | Cuerpo | Espera |
|---|---|---|---|
| 16 | Recuperar | `agente`, `chat_id`, `consultas` (lista de preguntas), `k: 10` | 60 s |
| 11 | Escribir | `memoria` (la conclusión), `tipo: "reflexion"`, `agente`, `chat_id`, `autor: "sistema"` | 120 s |
| 31 | Comprometer | `acto: "DECISION"`, `agente`, `chat_id`, `accion`, `dia`, `hora`, `fin: ""`, `condicion`, `simbolos`, `contenido`, `origen: "reflexion"`, `nivel: 1` | 120 s |

Todas usan la conexión de API key de Make.

## Flujo

```
1 Webhook
12 Airtable: observaciones sin reflejar
21 Agregador (id, Importancia, Memoria)
22 Router del umbral
├─ Ruta 1  [No llega al umbral]
│  23 Respuesta «sin material suficiente»
└─ Ruta 2  [Llega al umbral]
   7 Gemini: preguntas
   15 Iterador de líneas
   24 Agregador de preguntas  [Es una pregunta]
   25 Router de preguntas
   ├─ Ruta 1  [Hay preguntas]
   │  16 HTTP → Recuperar
   │  26 Iterador de resultados
   │  17 Agregador de texto: recuerdos por pregunta
   │  8 Gemini: conclusiones y acción
   │  9 Lectura del JSON
   │  30 Router de cierre
   │  ├─ Ruta 1: 10 Iterador de conclusiones → 11 HTTP → Escribir → 14 Airtable: evidencias
   │  ├─ Ruta 2: 31 HTTP → Comprometer  [La reflexión exige actuar]
   │  ├─ Ruta 3: 40 Iterador de observaciones → 13 Airtable: marcar reflejada
   │  └─ Ruta 4: 32 Respuesta «reflexion completada»
   ├─ Ruta 2: 41 Iterador de observaciones → 42 Airtable: marcar reflejada  [No hay preguntas]
   └─ Ruta 3: 43 Respuesta «reflexion completada», 0 reflexiones  [No hay preguntas]
```

## Módulos

### 1. Webhook

Recibe el agente y el chat. Atiende las llamadas de una en una.

### 12. Observaciones sin reflejar (Airtable, *Search records*)

```
AND({Agente}="<agente>", {Chat_ID}="<chat>", {Tipo}="observacion", NOT({Reflejada}))
```

Las ordena de la más reciente a la más antigua y trae de cada una la importancia y el texto. Como mucho, 500. Las que pasen de ese número quedan para la siguiente reflexión.

### 21. Agregador de observaciones

Junta las observaciones en una lista con su identificador, su importancia y su texto. Esa misma lista se usa tres veces: para sumar el umbral, para formular las preguntas y para marcar las observaciones al final. Así, las observaciones que se cuentan, las que se analizan y las que se marcan son siempre las mismas. Si la búsqueda no encuentra nada, la lista llega vacía.

### 22. Router del umbral

Los dos filtros calculan la suma de las importancias de la lista, que vale 0 si está vacía:

```
ifempty(sum(map(21.array; Importancia)); 0)
```

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | No llega al umbral | suma < 30 |
| 2 | Llega al umbral | suma ≥ 30 |

### 23. Respuesta «sin material suficiente»

Responde 200 y termina, sin escribir ni marcar nada. Las observaciones se vuelven a sumar en la siguiente llamada, junto con las nuevas.

### 7. Preguntas (Gemini 3.8 Flash)

```
<role>
Eres el módulo de autoevaluación y reflexión estratégica de {{agente}}. Tu función es analizar sus observaciones nuevas para formular las preguntas clave que guiarán la extracción de nuevas conclusiones en el sistema de memoria.
</role>

<context>
Observaciones nuevas de {{agente}} desde su última reflexión, de la más reciente a la más antigua:
<observaciones_nuevas>
{{una observación por línea}}
</observaciones_nuevas>
</context>

<instructions>
{{agente}} quiere ser más eficaz en su labor profesional y en su desempeño con su interlocutor. Basándote únicamente en la información de <observaciones_nuevas>, formula una pregunta útil para revisar su actuación reciente por cada uno de estos aspectos en el que las observaciones muestren algún indicio:
1. Qué hizo que no dio resultado o se puede hacer mejor.
2. Qué oportunidades económicas dejó pasar por ignorancia o desatención.
3. Qué necesitaba su interlocutor que no recibió.

Si en un aspecto las observaciones no muestran ningún indicio, no formules pregunta para él.
</instructions>

<output_format>
Responde únicamente con las preguntas, una por línea, como máximo 3. NO incluyas números (ej. 1., 2.), ni prefijos/viñetas (ej. -, *), ni introducciones conversacionales ni texto adicional. Si ningún aspecto muestra indicios, responde únicamente: NINGUNA
</output_format>
```

Las preguntas solo se formulan cuando hay indicios. Así no dan por hecho que hubo fallos, y Gemini no se ve empujado a inventárselos.

### 15. Iterador de líneas

Parte la respuesta de Gemini por saltos de línea y entrega cada línea por separado.

### 24. Agregador de preguntas

**Filtro de entrada: «Es una pregunta».** La línea tiene que tener texto y ser distinta de `NINGUNA`, sin distinguir mayúsculas.

Junta las preguntas en una lista y guarda de cada una dos versiones:

| Campo | Contenido | Uso |
|---|---|---|
| `pregunta` | El texto tal cual. | El prompt del módulo 8. |
| `json` | El texto entre comillas y escapado. | El cuerpo de la llamada a Recuperar. |

### 25. Router de preguntas

| Ruta | Filtro | Qué hace |
|---|---|---|
| 1 | Hay preguntas | Sigue la reflexión. |
| 2 | No hay preguntas | Marca las observaciones como reflejadas (41, 42). |
| 3 | No hay preguntas | Responde con 0 reflexiones (43). |

### 16. Recuperación (HTTP → Recuperar)

Hace una sola llamada con todas las preguntas:

```json
{"agente": "…", "chat_id": "…", "consultas": ["pregunta 1", "pregunta 2"], "k": 10}
```

Recuperar devuelve en `resultados` un bloque por pregunta, en el mismo orden. Cada bloque trae los 10 recuerdos mejor puntuados de dos formas: como texto con sus identificadores (`texto_ids`) y como lista con el tipo de cada uno (`memorias`).

### 26. Iterador de resultados

Recorre los bloques. Su número de orden indica a qué pregunta de la lista del módulo 24 corresponde cada uno.

### 17. Recuerdos por pregunta (agregador de texto)

Monta un bloque por pregunta:

```
Pregunta: <pregunta>
(rec…) <recuerdo>
(rec…) <recuerdo>
…
Conclusiones anteriores entre estas afirmaciones: rec…, rec…
```

La última línea lista los identificadores de los recuerdos de tipo `reflexion`, sacados de `memorias`:

```
join(map(26.memorias; id; tipo; reflexion); ", ")
```

### 8. Conclusiones y acción (Gemini 3.8 Flash, salida JSON)

```
<role>
Eres el módulo de síntesis de conclusiones y toma de decisiones estratégicas de {{agente}}. Tu objetivo es formular como máximo 2 conclusiones nuevas a partir de los recuerdos recuperados y evaluar si lo que muestran exige ejecutar una acción inmediata.
</role>

<time_context>
Ahora es {{fecha y hora}}, hora de Madrid.
</time_context>

<retrieved_memories>
Para cada pregunta sobre la actuación de {{agente}}, estas son las afirmaciones relevantes que recuerda (cada una con su identificador entre paréntesis):
{{bloques del módulo 17}}
</retrieved_memories>

<synthesis_rules>
1. Una conclusión es una afirmación que {{agente}} debe tener presente a partir de ahora: un hecho que las evidencias dan por comprobado (sobre su interlocutor, sobre el mundo o sobre su propio funcionamiento) o un criterio para actuar mejor.
2. Una conclusión no es una tarea: algo que hacer en un momento concreto o cuando se cumpla una condición (avisar, consultar, investigar algo determinado). Eso se describe en `accion`.
3. Cada conclusión es una sola frase afirmativa.
4. Las conclusiones deben estar justificadas por las afirmaciones de <retrieved_memories>, sin inventar nada que no se deduzca de ellas.
5. Da prioridad a las conclusiones que tengan mayor trascendencia económica para {{agente}} o para su interlocutor.
6. Algunas afirmaciones de <retrieved_memories> son conclusiones a las que {{agente}} ya llegó antes; al final de cada bloque se indican sus identificadores. Si una conclusión dice lo mismo que una de ellas, no la escribas.
7. En `evidencias`, copia literalmente únicamente los identificadores entre paréntesis que empiezan por "rec" (ej. recA1b2C3d4E5f6G7h).
</synthesis_rules>

<action_trigger_rules>
Después de formular las conclusiones, piensa como {{agente}} y evalúa si lo que muestran los recuerdos exige realizar una acción concreta de un solo paso dentro de sus capacidades:
- Capacidades disponibles:
  - informar: escribir por iniciativa propia a su interlocutor.
  - preguntar: hacerle una pregunta.
  - investigar: buscar información sobre un tema.
  - cotizar: consultar precios de valores para sí mismo.
  - estudiar: analizar un evento o un dato económico cuando se publique.

Si EXIGE actuar:
- Pon `necesaria` en true.
- Describe la acción más necesaria (solo una y de un solo paso) en los campos `accion`, `dia`, `hora`, `condicion`, `simbolos` y `contenido`.

Si no exige actuar:
- Pon `necesaria` en false.
- Deja vacíos los campos `accion`, `dia`, `hora`, `condicion`, `simbolos` y `contenido`. (Lo normal es que no haga falta actuar).
</action_trigger_rules>

<output_format>
Responde únicamente con este objeto JSON, sin texto antes ni después. Cada conclusión es un elemento de `insights`; si no hay ninguna conclusión nueva, `insights` es []. Un campo de texto vacío se escribe "" y una lista vacía, [].
{ "insights": [ {"insight": "…", "evidencias": ["rec…"]} ],
  "accion": { "necesaria": true|false, "accion": "…", "dia": "…", "hora": "…",
              "condicion": "…", "simbolos": "…", "contenido": "…" } }
</output_format>
```

El módulo tiene además fijado un esquema de respuesta JSON con esos mismos campos.

### 9. Lectura del JSON

Convierte la respuesta de Gemini en datos: la lista `insights` y el objeto `accion`.

### 30. Router de cierre

Las cuatro rutas se ejecutan en orden.

| Ruta | Filtro | Módulos | Qué hace |
|---|---|---|---|
| 1 | — | 10 → 11 → 14 | Por cada conclusión: la guarda con Escribir como `reflexion` en el chat y después le añade las evidencias (campo Evidencias del registro creado). |
| 2 | La reflexión exige actuar | 31 | Si `necesaria` es true y hay `accion` y `contenido`, pide a Comprometer el compromiso como DECISION. Comprometer aplica sus reglas: capacidades, chat, tope y duplicados. |
| 3 | — | 40 → 13 | Marca como reflejadas las observaciones de la lista del módulo 21. |
| 4 | — | 32 | Responde con el número de conclusiones. |

Si una ruta falla, la ejecución se detiene y las siguientes no se ejecutan. En particular, si falla el guardado de una conclusión, las observaciones no se marcan y entran de nuevo en la siguiente reflexión.

## Gestión de errores

Cada módulo que puede fallar responde 500 con el origen del fallo y hace Commit. Las respuestas solo hacen Commit.

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 12 | 957 → 958 | `Reflexionar › búsqueda de observaciones sin reflejar (Airtable)` |
| 7 | 943 → 944 | `Reflexionar › preguntas de alto nivel (Gemini)` |
| 16 | 945 → 946 | `Reflexionar › recuperación de recuerdos (Recuperar)` |
| 8 | 947 → 948 | `Reflexionar › conclusiones (Gemini)` |
| 9 | 949 → 950 | `Reflexionar › lectura de las conclusiones (JSON)` |
| 11 | 951 → 952 | `Reflexionar › guardado de la reflexión (Escribir)` |
| 14 | 953 → 954 | `Reflexionar › guardado de las evidencias (Airtable)` |
| 31 | 955 → 956 | `Reflexionar › decisión de actuar (Comprometer)` |
| 13 | 959 → 960 | `Reflexionar › marcado de observaciones reflejadas (Airtable)` |
| 42 | 969 → 970 | `Reflexionar › marcado de observaciones reflejadas (Airtable)` |
| 23, 32, 43 | 962, 961, 971 | Solo Commit. |

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_REFLEXIONAR__` | Un webhook nuevo. Su URL es la que usa el Despertador. |
| `__CONEXION_AIRTABLE__`, `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | Conexión, base y tabla de la memoria (módulos 12, 14, 13 y 42). |
| `__CAMPO_Evidencias__`, `__CAMPO_Reflejada__` | Los campos Evidencias (enlace a la misma tabla) y Reflejada (casilla). |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulos 7 y 8). |
| `__CONEXION_API_KEY_MAKE__` | La conexión de API key para las llamadas a otros escenarios. |
| `__WEBHOOK_RECUPERAR__`, `__WEBHOOK_ESCRIBIR__`, `__WEBHOOK_COMPROMETER__` | Las URL de esos escenarios. |

Después de importar:

- fija la programación en *Immediately*;
- comprueba que los agregadores 21 y 24 muestran sus campos: el 21, id, Importancia y Memoria; el 24, pregunta y json.
