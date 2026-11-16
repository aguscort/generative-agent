# 09 · Clasificar

**Escenario:** `[Human_proxy]:[03-Planning] (09/20) | Clasificar`
**Blueprint:** [`blueprints/09-clasificar.blueprint.json`](../blueprints/09-clasificar.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_CLASIFICAR__`).
**Lo llama:** el Loop (módulo 80), en cada mensaje.

## Función

Interpreta el mensaje del usuario y redacta la respuesta del agente. En una sola llamada a Gemini hace tres cosas:

1. **Examina el mensaje.** Determina qué dice el usuario, qué quiere y qué datos aporta.
2. **Decide el curso de acción y lo clasifica como acto de habla,** según el modelo de compromisos de Shoham (1993):

   | Acto | Curso de acción |
   |---|---|
   | `REQUEST` | Comprometerse a hacer algo más adelante. |
   | `UNREQUEST` | Dejar de ocuparse de algo. |
   | `INFORM` | Contestar ahora y nada más. |
   | `NINGUNO` | No hacer nada. |

3. **Extrae los datos del compromiso y redacta la respuesta,** en el papel del agente, que es la que recibe el usuario.

Clasificar no decide si el compromiso se acepta: eso lo hace Comprometer. Lo que sí hace es dar a Gemini todo el contexto que el Loop ha reunido, más los compromisos vigentes con este usuario, para que la clasificación y la respuesta sean coherentes con lo que el agente ya sabe y ya ha prometido.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat_id` | Chat. |
| `usuario` | Nombre del usuario. |
| `mensaje` | Mensaje que se clasifica. |
| `descripcion` | Quién es el agente (perfil). |
| `resumen` | Resumen del agente en este chat (escenario Resumen). |
| `recuerdos` | Recuerdos relacionados con el mensaje (Recuperar). |
| `mensajes` | Mensajes anteriores del chat (Recuperar). |

## Respuesta

200, en JSON, con todos los campos de la clasificación:

```json
{"acto": "REQUEST", "examen": "…", "curso_accion": "…", "accion": "informar",
 "dia": "2026-11-17", "hora": "09:00", "fin": "", "condicion": "…", "simbolos": "SAN.MC",
 "contenido": "…", "libera": "", "repite": "", "vetado_por": "", "informe": "…", "respuesta": "…"}
```

Si falla un paso, responde 500 con `{"error": "Clasificar › <paso> (<servicio>)"}`.

## Flujo

```
1 Webhook
54 Almacén Compromisos: compromisos vigentes con este chat
55 Agregador de texto: lista de compromisos
50 Gemini: clasificación y respuesta
3 Respuesta 200
```

## Módulos

### 1. Webhook

Atiende las llamadas de una en una.

### 54. Compromisos vigentes (almacén Compromisos, *Search records*)

Busca los compromisos del agente con este chat que no están cumplidos, fallidos, liberados ni caducados. Un compromiso es «con este chat» si su `destinatario` o su `comprometido_con` es el chat. Los ordena por momento y trae como mucho 20. Si no hay ninguno, sigue igualmente.

### 55. Lista de compromisos (agregador de texto)

Una línea por compromiso, con su clave y su línea en notación AGENT0: `<clave> · <linea>`.

### 50. Clasificación y respuesta (Gemini 3.8 Flash, salida JSON)

Instrucción de sistema:

> Clasificas mensajes para un agente que funciona con el modelo de compromisos de Shoham. Respondes solo con el JSON pedido y no inventas datos que la conversación no da.

El prompt tiene cinco bloques.

**`<task_instructions>`.** Pide examinar el mensaje y, después, proponer el curso de acción y clasificarlo en un acto.

**`<agent_capabilities>`.** Las únicas acciones a las que el agente puede comprometerse:

| Acción | Qué es |
|---|---|
| informar | Escribir más adelante al interlocutor por iniciativa propia, con cotizaciones si hacen falta. |
| preguntar | Hacerle más adelante una pregunta. |
| investigar | Buscar información sobre un tema. |
| cotizar | Consultar precios de valores para sí mismo, sin escribir a nadie. |
| estudiar | Analizar un evento o un dato económico cuando se publique. |
| abstenerse | No hacer algo que el interlocutor no quiere (informar o preguntar). |

**`<speech_acts_definitions>`.** Cuándo corresponde cada acto:

| Acto | Cuándo |
|---|---|
| REQUEST | El interlocutor pide algo para más adelante (a una hora, en una fecha o cuando se cumpla una condición), o acepta algo que el agente le ofreció. Si pide que no le escriba o no le pregunte por algo, el compromiso es abstenerse. |
| UNREQUEST | El interlocutor libera al agente de algo que le había pedido, o retira una prohibición; en este último caso, la acción es abstenerse. |
| INFORM | El interlocutor comunica un hecho, saluda, comenta, hace una pregunta que se puede contestar en el momento, o pide algo que no está entre las capacidades del agente, que entonces le dice que no puede. |
| NINGUNO | No hay que hacer nada, ni siquiera contestar. |

**`<field_definitions>`.** Los campos de la salida y las reglas para rellenarlos:

| Campo | Contenido |
|---|---|
| `examen` | Qué dice, qué quiere y qué datos aporta el interlocutor, en dos o tres frases. |
| `curso_accion` | Lo que debe hacer el agente, con libertad y con datos. |
| `acto` | REQUEST, UNREQUEST, INFORM o NINGUNO. |
| `accion` | Una de las capacidades. En UNREQUEST, vacía si le libera de todo. |
| `dia`, `hora` | AAAA-MM-DD y HH:mm, hora de Madrid. |
| `fin` | AAAA-MM-DD, si el interlocutor dice hasta cuándo vale. |
| `condicion` | Comprobable con datos («Inditex por debajo de 50 euros»). |
| `simbolos` | Tickers de Yahoo Finance separados por comas. |
| `contenido` | Qué hará el agente, en una frase y con datos. |
| `libera`, `repite`, `vetado_por` | Claves de compromisos de la lista: los que se liberan, el que ya pide lo mismo y la abstención que lo prohíbe. |
| `informe` | Lo decidido, en tercera persona, como registro. |
| `respuesta` | El mensaje que el agente envía ahora: en su personaje, en el idioma del interlocutor, en una o dos frases y sin pasar de 20 palabras, sin inventar recuerdos ni cotizaciones. |

Además, el prompt fija qué campos se rellenan en cada acto:

- `examen`, `curso_accion` y `acto` se rellenan siempre.
- En INFORM, se rellenan además `informe` y `respuesta`.
- En REQUEST y UNREQUEST, se rellenan los datos del compromiso, `informe` y `respuesta`.
- En NINGUNO, todo lo demás va vacío.

**`<context>`.** Todo el contexto que recibe Gemini:

- el día de la semana, la fecha y la hora de Madrid;
- quién es el agente (`descripcion`);
- lo que sabe de sí mismo y del interlocutor (`resumen`);
- lo que recuerda relacionado con el mensaje (`recuerdos`);
- la conversación reciente (`mensajes`);
- los compromisos vigentes con su clave (módulo 55), o «ninguno»;
- el mensaje que hay que clasificar.

El módulo tiene fijado un esquema de respuesta JSON con los 15 campos, todos obligatorios y en el orden de la tabla. `acto` está limitado a los cuatro valores posibles. El orden de los campos hace que Gemini examine y razone el curso de acción antes de clasificar y de redactar.

### 3. Respuesta

Devuelve los 15 campos de Gemini en JSON, escapados.

## Gestión de errores

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 54 | 954 → 955 | `Clasificar › lectura de los compromisos (almacén Compromisos)` |
| 50 | 950 → 951 | `Clasificar › clasificación del mensaje (Gemini)` |
| 3 | 903 | Solo Commit. |

En los dos casos con cuerpo, el manejador responde 500 con ese cuerpo y hace Commit.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_CLASIFICAR__` | Un webhook nuevo. Su URL es `__WEBHOOK_CLASIFICAR__` en el Loop. |
| `__ALMACEN_COMPROMISOS__` | El almacén de compromisos (módulo 54). |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulo 50). |

Después de importar hay que fijar la programación en *Immediately*.
