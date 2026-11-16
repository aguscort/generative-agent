# 07 · Resumen del agente

**Escenario:** `[Human_proxy]:[02-Memoria] (07/20) | Resumen del agente`
**Blueprint:** [`blueprints/07-resumen.blueprint.json`](../blueprints/07-resumen.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_RESUMEN__`).
**Lo llama:** el Loop (módulo 61), en cada mensaje.

## Función

Devuelve una descripción breve del agente tal como lo ve su propia memoria en este chat: cómo es, a qué se dedica, cómo va su progreso y qué sabe de su interlocutor. Es el *agent summary description* de Park et al. (2023), y el Loop la pasa a Clasificar como contexto.

Regenerar el resumen es caro: hace falta recuperar recuerdos y llamar a Gemini. Por eso se guarda en caché y solo se rehace cuando tiene más de 7 días o no existe.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat_id` | Chat. |
| `modo` | `ON` para obtener el resumen. Cualquier otro valor, o ninguno, lo desactiva. |
| `descripcion` | Está en la estructura del webhook, pero el escenario no lo usa. |

## Respuesta

200, en JSON, con el campo `estado`:

| `estado` | Cuándo | `resumen` |
|---|---|---|
| `OFF` | `modo` no es `ON`. | Vacío. |
| `cache` | Hay un resumen guardado con menos de 7 días. | El guardado. |
| `regenerado` | No había resumen o había caducado. | El nuevo. |

En caso de fallo responde 500 con `{"error": "Resumen: …"}`.

El resumen tiene esta forma:

```
Resumen de <agente> según su memoria de esta conversación (actualizado a las HH:mm del DD/MM/AAAA):
Rasgos esenciales: …
Ocupación diaria actual: …
Cómo se siente respecto a su progreso reciente: …
Lo que sabe de su interlocutor y su relación con él: …
```

## Flujo

```
1 Webhook
2 Router
├─ Ruta 1: 3 Respuesta «OFF»  [Resumen desactivado (OFF)]
└─ Ruta 2: 4 Almacén Caché: resumen guardado  [Resumen activado (ON)]
           5 Router
           ├─ Ruta 1: 6 Respuesta «cache»  [Caché vigente (< 7 días)]
           └─ Ruta 2  [Sin caché o caducada]
              10 HTTP → Recuperar
              11 Gemini: redacción del resumen
              16 Variable: resumen compuesto
              19 Respuesta «regenerado»
              18 Almacén Caché: guarda el resumen  [Hay algún dato que cachear]
```

## Módulos

### 1. Webhook

Atiende las llamadas de una en una.

### 2. Router de modo

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Resumen desactivado (OFF) | `upper(modo)` distinto de `ON`; si `modo` no llega, cuenta como `OFF`. |
| 2 | Resumen activado (ON) | `upper(modo)` igual a `ON`. |

### 3. Respuesta «OFF»

`{"resumen": "", "estado": "OFF"}`

### 4. Resumen guardado (almacén Caché, *Get a record*)

Lee el registro con clave `<agente>|<chat_id>`. Sus campos son `agente`, `chat_id`, `resumen` y `actualizado`.

### 5. Router de caché

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Caché vigente (< 7 días) | Hay `resumen` y `actualizado` es posterior a hace 7 días. |
| 2 | Sin caché o caducada | No hay `resumen`, o `actualizado` es de hace 7 días o más. |

### 6. Respuesta «cache»

`{"resumen": "<guardado>", "estado": "cache"}`

### 10. Recuperación (HTTP → Recuperar)

Una sola llamada con cuatro consultas fijas, y 15 recuerdos por consulta:

1. rasgos esenciales de la personalidad de `<agente>`;
2. ocupación diaria actual de `<agente>`;
3. cómo se siente `<agente>` respecto a su progreso reciente en la vida;
4. lo que `<agente>` sabe de su interlocutor y la relación que mantiene con él.

Son las tres consultas del resumen de Park et al., más una cuarta sobre el interlocutor. De la respuesta se usa el `texto` de cada bloque de `resultados`.

### 11. Redacción (Gemini 3.8 Flash, salida JSON)

```
<role>
Eres el módulo de síntesis de identidad y perfil de {{agente}}. Tu objetivo es redactar un resumen actualizado de su personalidad, ocupación, progreso y relación con el interlocutor basándote exclusivamente en las afirmaciones de memoria recuperadas.
</role>

<instructions>
Describe a {{agente}} en cuatro apartados obligatorios, evaluando únicamente la información aportada para cada uno:

1. rasgos: rasgos esenciales de su personalidad.
2. ocupacion: su ocupación diaria actual.
3. progreso: cómo se siente respecto a su progreso reciente en la vida.
4. interlocutor: qué sabe de su interlocutor y qué relación mantiene con él.

Reglas estrictas de redacción:
- Responde en una o dos frases por apartado.
- Escribe en tercera persona y en texto plano estricto (sin negritas, cursivas ni sintaxis Markdown).
- Queda prohibido inventar o deducir información que no esté respaldada explícitamente por las afirmaciones.
- Si un apartado no contiene afirmaciones o estas son insuficientes, responde en ese apartado ÚNICAMENTE con el texto literal: sin datos
</instructions>

<context>
<seccion_rasgos>
Afirmaciones sobre personalidad:
{{texto de la consulta 1}}
</seccion_rasgos>

<seccion_ocupacion>
Afirmaciones sobre ocupación diaria actual:
{{texto de la consulta 2}}
</seccion_ocupacion>

<seccion_progreso>
Afirmaciones sobre progreso reciente:
{{texto de la consulta 3}}
</seccion_progreso>

<seccion_interlocutor>
Afirmaciones sobre el interlocutor y la relación:
{{texto de la consulta 4}}
</seccion_interlocutor>
</context>

<output_format>
Devuelve la respuesta en texto plano respetando la estructura clave-valor exacta:

rasgos: <una o dos frases en texto plano, o 'sin datos'>
ocupacion: <una o dos frases en texto plano, o 'sin datos'>
progreso: <una o dos frases en texto plano, o 'sin datos'>
interlocutor: <una o dos frases en texto plano, o 'sin datos'>
</output_format>
```

El módulo tiene fijado un esquema de respuesta JSON con los campos `rasgos`, `ocupacion`, `progreso` e `interlocutor`, todos de texto y obligatorios.

### 16. Resumen compuesto (*Set variable*)

Monta el texto final a partir de los cuatro campos, con la fecha y la hora de Madrid.

### 19. Respuesta «regenerado»

`{"resumen": "<nuevo>", "estado": "regenerado"}`. Se responde antes de guardar en caché para no retrasar al Loop.

### 18. Guardado en caché (almacén Caché, *Add/replace a record*)

**Filtro: «Hay algún dato que cachear».** Pasa si al menos uno de los cuatro apartados no contiene «sin datos».

Sobrescribe el registro `<agente>|<chat_id>` con el resumen y la fecha actual. Si los cuatro apartados están vacíos, no se guarda nada, y el resumen se volverá a intentar en el siguiente mensaje.

## Gestión de errores

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 4 | 101 → 102 | `Resumen: no se pudo leer la caché` |
| 10 | 103 → 104 | `Resumen: no se pudo recuperar la memoria` |
| 11 | 105 → 106 | `Resumen: no se pudo redactar el resumen` |
| 16 | 115 → 116 | `Resumen: no se pudo componer el resumen` |
| 18 | 121 | Solo Commit (ya se ha respondido). |

En todos los casos con cuerpo, el manejador responde 500 y hace Commit. Las respuestas 3, 6 y 19 no tienen manejador.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_RESUMEN__` | Un webhook nuevo. Su URL es `__WEBHOOK_RESUMEN__` en el Loop. |
| `__ALMACEN_CACHE__` | El almacén Caché (módulos 4 y 18), con los campos `agente`, `chat_id`, `resumen` y `actualizado`. |
| `__CONEXION_API_KEY_MAKE__`, `__WEBHOOK_RECUPERAR__` | La conexión de API key y la URL de Recuperar (módulo 10). |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulo 11). |

Después de importar hay que fijar la programación en *Immediately*.

## Almacén Caché

Este almacén lo comparten varios escenarios. Cada uno usa su propio prefijo de clave:

| Clave | Contenido | Escribe | Lee |
|---|---|---|---|
| `perfil|<agente>` | Perfil del agente (`resumen`) | — (se crea a mano) | Loop |
| `<agente>|<chat_id>` | Resumen del agente en el chat (`resumen`, `actualizado`) | Resumen | Resumen |
| `ventana|<agente>|<chat_id>` | Últimos mensajes del chat (`ventana`) | Recuperar | Recuperar |
