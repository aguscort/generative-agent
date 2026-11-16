# 02 · Loop principal

**Escenario:** `[Human_proxy]:[01-Core] | (02/20) Loop Principal`
**Blueprint:** [`blueprints/02-loop.blueprint.json`](../blueprints/02-loop.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_LOOP__`).
**Lo llama:** [01 · Telegram](01-telegram.md), módulo 70.

## Función

Es el núcleo de la conversación. Por cada mensaje del usuario, el Loop hace cinco cosas:

1. Lee el perfil del agente.
2. Pide el resumen de la relación con el usuario.
3. Recupera los recuerdos relevantes y los mensajes anteriores.
4. Pide a Clasificar que interprete el mensaje y redacte la respuesta.
5. Según el tipo de acto que devuelva Clasificar, pasa la petición a Comprometer y guarda en la memoria lo que se ha dicho y lo que se ha decidido.

Al terminar, devuelve a Telegram el texto con el que responde Borja.

El Loop no razona por sí mismo. Toda la inteligencia está en los escenarios a los que llama. El Loop los encadena y escribe la memoria.

## Entrada

Cuerpo JSON del webhook:

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente (`Borja`). |
| `chat_id` | Identificador del chat de Telegram. |
| `usuario` | Nombre de pila del usuario. |
| `mensaje` | Texto del mensaje. |

## Respuesta

| Caso | Código | Cuerpo |
|---|---|---|
| Correcto | 200 | `{"respuesta": "<texto de Borja>"}` |
| Fallo en un paso | 500 | `{"respuesta": "Perdona, no puedo atenderte: … (escenario X) …"}` |

Los avisos de error van también en el campo `respuesta` y están redactados como si los dijera Borja. Telegram los envía al usuario igual que una respuesta normal.

## Llamadas a otros escenarios

Todas las llamadas usan la conexión de API key de Make y tienen activado *Return error if HTTP request fails*: si el escenario llamado responde con un código de error, el módulo falla y actúa su manejador.

| Módulo | Destino | Cuerpo | Espera |
|---|---|---|---|
| 61 | Resumen (`__WEBHOOK_RESUMEN__`) | `agente`, `chat_id`, `modo: "ON"` | 180 s |
| 2 | Recuperar (`__WEBHOOK_RECUPERAR__`) | `agente`, `chat_id`, `usuario`, `consulta` (el mensaje), `k: 5`, `n: 6` | 60 s |
| 80 | Clasificar (`__WEBHOOK_CLASIFICAR__`) | `agente`, `chat_id`, `usuario`, `mensaje`, `descripcion`, `resumen`, `recuerdos`, `mensajes` | 60 s |
| 96 | Comprometer (`__WEBHOOK_COMPROMETER__`) | `agente`, `chat_id`, `usuario`, `acto`, `accion`, `dia`, `hora`, `fin`, `condicion`, `simbolos`, `contenido`, `libera`, `repite`, `vetado_por` | 60 s |
| 97, 98, 100, 93, 94 | Escribir (`__WEBHOOK_ESCRIBIR__`) | `memoria`, `tipo`, `agente`, `chat_id`, `autor` | 120 s |

## Flujo

```
1 Webhook
960 Almacén Caché: perfil del agente
61 HTTP → Resumen
2 HTTP → Recuperar
80 HTTP → Clasificar
90 Router
├─ Ruta 1  [REQUEST o UNREQUEST]
│  96 HTTP → Comprometer
│  97 HTTP → Escribir: mensaje del usuario
│  98 HTTP → Escribir: respuesta de Borja
│  100 HTTP → Escribir: decisión de Comprometer
│  99 Respuesta 200
└─ Ruta 2  [INFORM]
   93 HTTP → Escribir: mensaje del usuario
   94 HTTP → Escribir: respuesta de Borja
   95 Respuesta 200
```

## Módulos

### 1. Webhook

Recibe el mensaje desde Telegram. Atiende las llamadas de una en una (`maxResults: 1`).

### 960. Perfil del agente (almacén Caché, *Get a record*)

Lee el registro con clave `perfil|<agente>`. De él se usa el campo `resumen`, que es la descripción del agente. Si el registro no existe o no tiene resumen, en su lugar se envía a Clasificar el nombre del agente.

### 61. Resumen de la relación (HTTP → Resumen)

Pide al escenario Resumen el resumen de lo que el agente sabe de sí mismo, de su trabajo y de su interlocutor en este chat. Se usa el campo `resumen` de la respuesta. Es la llamada que más puede tardar, porque puede tener que regenerar el resumen; por eso su espera máxima es de 180 segundos.

### 2. Recuerdos y mensajes anteriores (HTTP → Recuperar)

Envía el mensaje como consulta. Pide los 5 recuerdos más útiles (`k: 5`) y los 6 últimos mensajes de la conversación (`n: 6`). Como `n` es mayor que 0, Recuperar apunta además este mensaje en la ventana de la conversación. De la respuesta se usan `texto`, los recuerdos numerados, y `mensajes`.

### 80. Clasificación y respuesta (HTTP → Clasificar)

Envía a Clasificar el mensaje con todo el contexto reunido:

| Campo | Origen |
|---|---|
| `descripcion` | Perfil del agente (960) o, si falta, su nombre. |
| `resumen` | Resumen de la relación (61). |
| `recuerdos` | Recuerdos recuperados (2). |
| `mensajes` | Mensajes anteriores (2). |

Clasificar devuelve el acto comunicativo (`acto`), la respuesta redactada (`respuesta`) y, si el mensaje pide algo, los datos del posible compromiso: `accion`, `dia`, `hora`, `fin`, `condicion`, `simbolos`, `contenido`, `libera`, `repite` y `vetado_por`.

### 90. Router por acto

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | REQUEST o UNREQUEST | `acto` es REQUEST o UNREQUEST. |
| 2 | INFORM | `acto` es INFORM. |

Si Clasificar devuelve otro acto, no se toma ninguna ruta. El escenario termina sin ejecutar ningún módulo de respuesta.

### 96. Decisión sobre el compromiso (HTTP → Comprometer)

Solo en la ruta 1. Pasa a Comprometer el acto y los datos del compromiso tal como los devuelve Clasificar. Comprometer decide si lo acepta, lo rechaza o, en un UNREQUEST, qué compromisos libera, y responde con un texto que empieza por ACEPTADO, RECHAZADO, LIBERADO o NADA.

### 97 y 93. Mensaje del usuario (HTTP → Escribir)

Guarda en la memoria una observación con autor `usuario`:

> `<usuario> le dijo a <agente>: «<mensaje>»`

### 98 y 94. Respuesta de Borja (HTTP → Escribir)

Guarda una observación con autor `agente` con la respuesta redactada por Clasificar:

> `<agente> le respondió a <usuario>: «<respuesta>»`

### 100. Decisión de Comprometer (HTTP → Escribir)

Solo en la ruta 1. Guarda una observación con autor `agente`:

> `Decisión de <agente> sobre sus compromisos: <respuesta de Comprometer>`

### 99 y 95. Respuesta a Telegram

Responde 200 con `{"respuesta": "<respuesta de Clasificar>"}`. En las dos rutas, el texto que recibe el usuario es el que redactó Clasificar.

## Gestión de errores

El Loop responde a Telegram, así que cada fallo se convierte en un aviso para el usuario. Cada módulo que puede fallar tiene un manejador que responde 500 con un texto que dice qué ha fallado, y después hace Commit. Commit detiene la ejecución sin marcarla como fallida, de modo que el escenario no se desactiva.

| Módulo | Manejador | Aviso |
|---|---|---|
| 960 | 101 → 102 | «Perdona, no puedo atenderte: ha fallado la lectura de mi perfil (Loop › almacén Caché). Escríbeme de nuevo en un momento.» |
| 61 | 105 → 106 | «… ha fallado el resumen de nuestra conversación (escenario Resumen) …» |
| 2 | 107 → 108 | «… ha fallado la recuperación de mis recuerdos (escenario Recuperar) …» |
| 80 | 117 → 118 | «… ha fallado la clasificación de tu mensaje (escenario Clasificar) …» |
| 96 | 135 → 136 | «… ha fallado la decisión sobre lo que me pides (escenario Comprometer) …» |
| 97, 93 | 137 → 138, 131 → 132 | «Perdona, no he podido guardar lo que me has dicho (escenario Escribir). Repítemelo en un momento.» |
| 98, 94 | 139 → 140, 133 → 134 | «Perdona, no he podido guardar mi respuesta (escenario Escribir). Repíteme lo que me decías en un momento.» |
| 100 | 141 → 142 | «Perdona, no he podido guardar lo que he decidido (escenario Escribir). Repíteme lo que me decías en un momento.» |

Los módulos de respuesta 99 y 95 no tienen manejador.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_LOOP__` | Un webhook nuevo para este escenario. Su URL es la que usa Telegram (módulo 70) como `__WEBHOOK_LOOP__`. |
| `__ALMACEN_CACHE__` | El almacén de datos donde se guarda el perfil del agente con la clave `perfil|<agente>` y el campo `resumen`. |
| `__CONEXION_API_KEY_MAKE__` | La conexión de API key con la que se llaman los webhooks de los escenarios. |
| `__WEBHOOK_RESUMEN__`, `__WEBHOOK_RECUPERAR__`, `__WEBHOOK_CLASIFICAR__`, `__WEBHOOK_COMPROMETER__`, `__WEBHOOK_ESCRIBIR__` | Las URL de los webhooks de esos escenarios. |

Después de importar hay que fijar la programación en *Immediately*.
