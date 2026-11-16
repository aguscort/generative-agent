# 01 · Telegram (Gateway)

**Escenario:** `[Human_proxy]:[00-Gateway] (01/20) | Telegram`
**Blueprint:** [`blueprints/01-telegram.blueprint.json`](../blueprints/01-telegram.blueprint.json)
**Disparador:** instantáneo, por el webhook del bot de Telegram.
**Procesamiento:** secuencial (`sequential: true`). Los mensajes se atienden de uno en uno y en orden de llegada.

## Función

Es la puerta de entrada del sistema. Recibe los mensajes que los usuarios escriben al bot de Telegram y hace tres comprobaciones:

1. que el mensaje sea texto y no un comando;
2. que el remitente sea el usuario autorizado;
3. que no haya superado el límite de mensajes recientes.

Si pasa las tres, entrega el mensaje al Loop y devuelve al usuario el texto que el Loop responda. No interpreta el mensaje ni escribe en la memoria: esas tareas son del Loop.

## Entrada

La actualización de Telegram que recibe el módulo *Watch Updates*. Se usan estos campos:

| Campo | Uso |
|---|---|
| `message.text` | Texto del mensaje. |
| `message.chat.id` | Chat al que se responde y que identifica la conversación. |
| `message.from.id` | Identificador del remitente, para la autorización. |
| `message.from.first_name` | Nombre del remitente, que se pasa al Loop como `usuario`. |

## Llamadas a otros escenarios

| Módulo | Destino | Cuerpo |
|---|---|---|
| 70 | Loop (`__WEBHOOK_LOOP__`) | `{"agente": "Borja", "chat_id": <chat>, "usuario": <nombre>, "mensaje": <texto>}` |

La llamada usa la conexión de API key de Make y espera la respuesta del Loop un máximo de 120 segundos. De la respuesta se usa el campo `respuesta`, que es el texto que se envía al usuario.

## Flujo

```
1 Watch Updates
└─ 965 Router  [Mensaje de texto (no comandos)]
   ├─ Ruta 1
   │  6 Airtable: mensajes recientes  [Es Usuario Identificado]
   │  7 Agregador numérico (suma)
   │  8 Router
   │  ├─ Ruta 1: 70 HTTP → Loop  [Límite: menos de 10 mensajes en 10 min]
   │  │          71 Telegram: respuesta del Loop
   │  └─ Ruta 2: 10 Telegram: aviso de límite  [Límite: mas de 10 mensajes en 10 min]
   └─ Ruta 2
      967 Telegram: aviso de usuario no autorizado  [No es Usuario Identificado]
```

## Módulos

### 1. Watch Updates (Telegram)

Recibe cada actualización del bot por webhook.

### 965. Router de entrada

Lleva el filtro **«Mensaje de texto (no comandos)»** en su entrada: el mensaje tiene que tener texto y no puede empezar por `/`. Las actualizaciones que no lo cumplen (fotos, stickers, comandos, ediciones…) se descartan sin respuesta.

Tiene dos rutas, que dependen del remitente.

### 6. Mensajes recientes (Airtable, *Search records*)

**Filtro de entrada: «Es Usuario Identificado».** El identificador del remitente (`message.from.id`) tiene que ser igual a `__ID_USUARIO_TELEGRAM_AUTORIZADO__`.

Busca en la tabla Memory Stream los mensajes que el usuario ha escrito en este chat en los últimos 10 minutos. Como mucho devuelve 10.

```
AND({Agente}="Borja", {Chat_ID}="<chat>", {Tipo}="observacion", {Autor}="usuario",
    IS_AFTER({Creado}, DATEADD(NOW(), -10, 'minutes')))
```

El recuento se hace sobre los mensajes ya guardados en la memoria, que guarda el Loop. Por eso el mensaje que se está procesando todavía no cuenta.

### 7. Recuento (agregador numérico, `SUM`)

Suma 1 por cada registro que devuelve el módulo 6. El resultado es el número de mensajes recientes. Si la búsqueda no devuelve nada, los filtros siguientes lo tratan como 0 (`ifempty(7.result; 0)`).

### 8. Router del límite

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Límite: menos de 10 mensajes en 10 min | recuento < 10 |
| 2 | Límite: mas de 10 mensajes en 10 min | recuento ≥ 10 |

### 70. Llamada al Loop (HTTP)

Envía el mensaje al Loop. Tiene desactivado *Return error if HTTP request fails* (`stopOnHttpError: false`). Si el Loop responde con un código de error, eso no se trata como un fallo de este módulo: su respuesta se entrega igualmente al módulo 71. Así llegan al usuario los avisos que el Loop redacta cuando falla alguno de sus pasos.

### 71. Respuesta al usuario (Telegram, *Send a Text Message or a Reply*)

Envía al chat el campo `respuesta` del Loop. Si viene vacío, envía:

> Perdona, no puedo atenderte: el Loop ha respondido sin texto. Escríbeme de nuevo en un momento.

### 10. Aviso de límite (Telegram)

Si se ha alcanzado el límite, envía:

> Descansamos un poco. Intentalo en un par de minutos

El mensaje no llega al Loop.

### 967. Aviso de usuario no autorizado (Telegram)

**Filtro de entrada: «No es Usuario Identificado».** El remitente es distinto del usuario autorizado. En ese caso envía:

> No eres un usuario autentificado. Si necesitas un agente, puedes informarte de como despliegar tu propia infraestructura en: https://github.com/aguscort/generative-agent

## Gestión de errores

En este escenario el canal de aviso es el propio chat de Telegram. Por eso, cuando falla un paso, el manejador avisa al usuario y después hace Commit. Commit detiene la ejecución sin marcarla como fallida, así que el escenario no se desactiva.

| Módulo | Manejador | Mensaje al usuario |
|---|---|---|
| 6 | 974 Telegram → 975 Commit | «Perdona, no puedo atenderte: ha fallado el control de mensajes recientes (Telegram › Airtable). Escríbeme de nuevo en un momento.» |
| 70 | 972 Telegram → 973 Commit | «Perdona, no puedo atenderte: no he podido comunicarme con el Loop (no responde o rechaza la conexión). Escríbeme de nuevo en un momento.» |
| 71 | 977 Commit | — |
| 10 | 976 Commit | — |
| 967 | 978 Commit | — |

Si falla el propio envío del aviso (974 o 972), su manejador anidado (980 o 979) solo hace Commit. El fallo queda en el historial de ejecuciones de Make.

Fallar, en el módulo 70, significa que el Loop no responde, que la conexión se rechaza o que se agota el tiempo de espera. Un código de error devuelto por el Loop no es un fallo, como se explica en el módulo 70.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_TELEGRAM__` | El webhook del bot. Se crea desde el módulo 1 al elegir la conexión de Telegram. |
| `__CONEXION_TELEGRAM__` | La conexión con el bot de Telegram (token de BotFather). Se usa en 1, 71, 10, 967, 974 y 972. |
| `__CONEXION_AIRTABLE__` | La conexión con Airtable (módulo 6). |
| `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | La base y la tabla Memory Stream. |
| `__CONEXION_API_KEY_MAKE__` | La conexión de tipo API key con la que se llaman los webhooks de los escenarios (módulo 70). |
| `__WEBHOOK_LOOP__` | La URL del webhook del escenario Loop. |
| `__ID_USUARIO_TELEGRAM_AUTORIZADO__` | El identificador numérico de Telegram del usuario autorizado, en los filtros de los módulos 6 y 967. |

Después de importar hay que fijar la programación en *Immediately*, porque es un escenario instantáneo.

La tabla Memory Stream tiene que tener, al menos, los campos `Agente`, `Chat_ID`, `Tipo`, `Autor` y `Creado`. `Creado` es de tipo *Created time*.

El nombre del agente, `Borja`, está escrito literalmente en la fórmula del módulo 6 y en el cuerpo del módulo 70.
