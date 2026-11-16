# 03 · Escribir

**Escenario:** `[Human_proxy]:[02-Memoria] (03/20) | Escribir`
**Blueprint:** [`blueprints/03-escribir.blueprint.json`](../blueprints/03-escribir.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_ESCRIBIR__`).
**Lo llaman:** el Loop, Reflexionar y Cumplir, y cualquier escenario que necesite guardar un recuerdo.

## Función

Es la única puerta de escritura en la memoria (*memory stream*, en la terminología de Park et al., 2023). Antes de guardar un recuerdo hace dos cosas:

1. le pone nota de importancia, de 1 a 10;
2. calcula su vector semántico (*embedding*).

Con esos dos datos lo guarda como un registro nuevo de la tabla Memory Stream. Recuperar usa después la importancia y el vector para puntuar los recuerdos.

## Entrada

Cuerpo JSON del webhook:

| Campo | Obligatorio | Contenido |
|---|---|---|
| `memoria` | sí | Texto del recuerdo. |
| `tipo` | sí | `observacion`, `reflexion`, `conocimiento` o `plan`. |
| `agente` | sí | Nombre del agente. |
| `chat_id` | sí | Chat al que pertenece el recuerdo, o `GLOBAL`. |
| `autor` | sí | `usuario`, `agente` o `sistema`. |
| `evidencias` | no | Lista de identificadores de registro (`rec…`) en los que se apoya el recuerdo. Lo usan las reflexiones. |
| `plan_ubicacion`, `plan_inicio`, `plan_duracion_min` | no | Datos de un plan. Están en la estructura del webhook, pero el escenario no los guarda. |

## Respuesta

| Caso | Código | Cuerpo |
|---|---|---|
| Correcto | 200 | `{"id": "<rec…>", "importancia": "<n>", "tipo": "<tipo>"}` |
| Fallo | 500 | `{"error": "Escribir › <paso> (<servicio>)"}` |

`id` es el identificador del registro creado. Reflexionar lo usa para añadir después las evidencias. `importancia` se devuelve como texto.

## Flujo

```
1 Webhook
2 Gemini: importancia
3 Gemini: embedding
4 Airtable: alta del recuerdo
5 Respuesta 200
```

## Módulos

### 1. Webhook

Recibe el recuerdo y atiende las llamadas de una en una (`maxResults: 1`).

### 2. Importancia (Gemini 3.8 Flash)

Puntúa la importancia económica del recuerdo. El modelo responde en JSON, según el esquema `{"importancia": <entero>}`.

```
<context>
Recuerdo (autor: {{autor}}): {{memoria}}
</context>

<instructions>
En una escala del 1 al 10, donde:
- 1 es económicamente irrelevante (p. ej., un saludo o una charla trivial).
- 10 es de máxima trascendencia económica (p. ej., una operación de gran volumen, una pérdida o ganancia importante, un movimiento brusco del mercado).

Puntúa la importancia económica del recuerdo anterior para {{agente}} o para su interlocutor.
Devuelve únicamente el valor solicitado en el formato exacto especificado.
</instructions>

<output_format>
importancia: <número entero del 1 al 10>
</output_format>
```

En Park et al., la importancia mide lo conmovedor (*poignancy*) del recuerdo. Aquí el criterio se ha cambiado por la trascendencia económica, porque el agente es un asesor financiero.

### 3. Embedding (Gemini Embedding 2)

Calcula el vector del texto del recuerdo, con 768 dimensiones.

### 4. Alta del recuerdo (Airtable, *Create a record*)

Crea el registro en la tabla Memory Stream, con *typecast* activado: si llega un valor nuevo en un campo de selección (Tipo, Autor), Airtable crea la opción.

| Campo | Valor |
|---|---|
| Memoria | `memoria` |
| Tipo | `tipo` |
| Agente | `agente` |
| Chat_ID | `chat_id` |
| Autor | `autor` |
| Importancia | La nota del módulo 2. |
| Ultimo_acceso | Fecha y hora del alta. |
| Embedding | El vector como texto: `[v1, v2, …]`. |
| Evidencias | `evidencias` |

### 5. Respuesta

Responde 200 con el identificador, la importancia y el tipo del registro creado.

## Gestión de errores

Escribir es un escenario llamado. Cada módulo que puede fallar responde 500 con el origen del fallo y hace Commit. Commit detiene la ejecución sin marcarla como fallida, de modo que el escenario no se desactiva.

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 2 | 901 → 902 | `{"error": "Escribir › puntuación de importancia (Gemini)"}` |
| 3 | 903 → 904 | `{"error": "Escribir › embedding del recuerdo (Gemini)"}` |
| 4 | 905 → 906 | `{"error": "Escribir › guardado del recuerdo (Airtable)"}` |
| 5 | 909 | Solo Commit. |

## Tabla Memory Stream

Escribir es el escenario que fija la estructura de la tabla. Los campos que usa el sistema son estos:

| Campo | Tipo en Airtable | Lo rellena |
|---|---|---|
| Memoria | Texto largo | Escribir |
| Tipo | Selección única | Escribir |
| Agente | Texto | Escribir |
| Chat_ID | Texto | Escribir |
| Autor | Selección única | Escribir |
| Importancia | Número | Escribir |
| Creado | *Created time* | Airtable |
| Ultimo_acceso | Fecha con hora | Escribir y Recuperar |
| Embedding | Texto largo | Escribir |
| Evidencias | Enlace a la misma tabla | Escribir y Reflexionar |
| Citada_por | Enlace inverso de Evidencias | Airtable |
| Reflejada | Casilla | Reflexionar |
| Plan_ubicacion, Plan_inicio, Plan_duracion_min | Texto, fecha y número | Nadie, por ahora |

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_ESCRIBIR__` | Un webhook nuevo. Su URL es `__WEBHOOK_ESCRIBIR__` en los escenarios que lo llaman. |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulos 2 y 3). |
| `__CONEXION_AIRTABLE__` | La conexión con Airtable (módulo 4). |
| `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | La base y la tabla Memory Stream. |
| `__CAMPO_<nombre>__` | Los campos de la tabla. En el módulo 4 hay que volver a elegir cada campo de la lista, porque los identificadores de campo cambian de una base a otra. |

Después de importar hay que fijar la programación en *Immediately*.
