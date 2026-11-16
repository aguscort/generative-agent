# 05 · Recuperar

**Escenario:** `[Human_proxy]:[02-Memoria] (05/20) | Recuperar`
**Blueprint:** [`blueprints/05-recuperar.blueprint.json`](../blueprints/05-recuperar.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_RECUPERAR__`).
**Lo llaman:** el Loop, Resumen y Reflexionar.

## Función

Implementa la recuperación de memoria de Park et al. (2023, §4.1). Dada una o varias consultas, puntúa los recuerdos del agente y devuelve los mejores. La puntuación de cada recuerdo suma tres componentes, cada uno normalizado entre 0 y 1:

| Componente | Cálculo |
|---|---|
| Recencia | 0,995 elevado a las horas que han pasado desde su último acceso, o desde su creación si nunca se ha recuperado. |
| Importancia | La nota de 1 a 10 que puso Escribir. |
| Relevancia | La similitud del coseno entre el vector del recuerdo y el de la consulta. |

Los tres pesan lo mismo, así que la puntuación va de 0 a 3. Los recuerdos devueltos se marcan con la hora actual como último acceso, como en Park: recuperar un recuerdo lo refresca.

Además, en las llamadas de conversación mantiene una ventana con los últimos mensajes del chat.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat_id` | Chat. Se buscan los recuerdos de este chat y los de `GLOBAL`. |
| `consulta` | Una frase de consulta. |
| `consultas` | Lista de frases. Si llega, sustituye a `consulta`. |
| `k` | Recuerdos que se devuelven por consulta. Por defecto, 10. |
| `n` | Mensajes anteriores que se devuelven. Por defecto, 0. Si es mayor que 0, la consulta se apunta en la ventana. |
| `usuario` | Nombre del usuario, para apuntar el mensaje en la ventana. |

Los usos actuales son estos:

| Llamante | Consulta | `k` | `n` |
|---|---|---|---|
| Loop | El mensaje del usuario | 5 | 6 |
| Reflexionar | Las preguntas de la reflexión (`consultas`) | 10 | 0 |
| Resumen | Cuatro consultas fijas sobre el agente y su interlocutor (`consultas`) | 15 | 0 |

## Respuesta

200, en JSON:

```json
{
  "memorias": [{"id": "rec…", "memoria": "…", "tipo": "…", "creado": "…",
                "score": 2.41, "recencia": 0.98, "importancia": 0.67, "relevancia": 0.76}],
  "texto": "1. …\n2. …",
  "texto_ids": "(rec…) …\n(rec…) …",
  "mensajes": "…\n…",
  "total_evaluadas": 277,
  "resultados": [{"memorias": […], "texto": "…", "texto_ids": "…"}]
}
```

`memorias`, `texto` y `texto_ids` corresponden a la primera consulta. `resultados` lleva un bloque por consulta, en el mismo orden en que llegaron. `total_evaluadas` es el número de recuerdos distintos que se han puntuado.

En caso de fallo responde 500 con `{"error": "Recuperar › <paso> (<servicio>)"}`.

## Flujo

```
1 Webhook
9 Iterador de consultas
2 Gemini: embedding de cada consulta
10 Agregador de vectores
3 Airtable: últimos 300 recuerdos
4 Agregador: una línea JSON por recuerdo
15 Airtable: reflexiones e importantes
16 Agregador: una línea JSON por recuerdo
12 Almacén Caché: ventana del chat
5 Código: puntuación, ventana y lotes de último acceso
13 Router
├─ Ruta 1: 14 Almacén Caché: guarda la ventana  [Llamada de conversación (n > 0)]
└─ Ruta 2: 6 Respuesta 200
           11 Iterador de lotes
           7 Airtable (API): marca el último acceso
```

## Módulos

### 1. Webhook

Atiende las llamadas de una en una. La estructura guardada del webhook no incluye `usuario`, pero el campo llega igualmente y el módulo 5 lo usa.

### 9 → 2 → 10. Vectores de las consultas

El iterador 9 entrega cada consulta (`consultas`, o una lista con `consulta`). El módulo 2 calcula su vector con Gemini Embedding 2, de 768 dimensiones, el mismo modelo que usa Escribir. El agregador 10 junta los vectores, uno por línea y en el orden de las consultas.

### 3 → 4. Primera lectura: lo reciente

Lee los 300 recuerdos más recientes del agente en el chat y en GLOBAL:

```
AND({Agente}="<agente>", OR({Chat_ID}="<chat>", {Chat_ID}="GLOBAL"))
```

El agregador 4 convierte cada registro en una línea JSON con el identificador, el tipo, el chat, el autor, la importancia, el último acceso (o la creación), la fecha de creación, el vector y el texto.

### 15 → 16. Segunda lectura: lo importante de cualquier fecha

Lee, de los mismos agente y chats, todas las reflexiones y todos los recuerdos de importancia 6 o más, hasta 1.000:

```
AND({Agente}="<agente>", OR({Chat_ID}="<chat>", {Chat_ID}="GLOBAL"),
    OR({Tipo}="reflexion", {Importancia}>=6))
```

Gracias a esta segunda lectura, una conclusión o un hecho importante antiguo puede salir aunque no esté entre los 300 recuerdos más recientes. El agregador 16 da el mismo formato que el 4.

### 12. Ventana del chat (almacén Caché)

Lee el registro `ventana|<agente>|<chat>`, cuyo campo `ventana` guarda como texto JSON los últimos mensajes del chat.

### 5. Cálculo (código JavaScript)

Recibe los vectores de las consultas, las líneas de las dos lecturas, `k`, `n`, la hora actual, la ventana y los datos para apuntar el mensaje. Hace cuatro cosas:

1. **Reúne los recuerdos sin repetir.** Un recuerdo que llega por las dos lecturas se cuenta una sola vez.
2. **Puntúa.** Para cada recuerdo calcula la recencia, la importancia y, por cada consulta, la relevancia. Normaliza cada componente con min-max entre todos los recuerdos evaluados y suma los tres. La recencia y la importancia son comunes a todas las consultas; la relevancia depende de cada una. Si todos los valores de un componente son iguales, ese componente vale 0 para todos.
3. **Ordena y corta.** Por cada consulta devuelve los `k` recuerdos con mejor puntuación, con su desglose.
4. **Prepara la ventana y el último acceso.**
   - Si `n` > 0, devuelve en `mensajes` los `n` últimos mensajes de la ventana, hasta 6, sin el actual. Prepara la ventana nueva añadiendo al final `«<usuario> le dijo a <agente>: «<consulta>»»` y conservando los 6 últimos.
   - Reúne los identificadores de todos los recuerdos devueltos y prepara lotes de 10, el máximo de Airtable por llamada, para marcarles el último acceso.

Los vectores se leen con una expresión regular en vez de interpretarlos como JSON, porque Make no siempre los guarda como JSON válido. Si una línea de recuerdo está corrupta, se ignora.

El código escribe el campo Ultimo_acceso por su identificador de campo (`__CAMPO_Ultimo_acceso__`).

### 13. Router

| Ruta | Filtro | Qué hace |
|---|---|---|
| 1 | Llamada de conversación (n > 0) | 14: guarda la ventana nueva (sobrescribe el registro). |
| 2 | — | 6: responde. Después, 11 → 7: por cada lote, marca el último acceso con una llamada PATCH a la API de Airtable. |

El marcado del último acceso se hace después de responder, así que no retrasa al que llama.

## Gestión de errores

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 2 | 901 → 902 | `Recuperar › embedding de la consulta (Gemini)` |
| 3 | 903 → 904 | `Recuperar › lectura de la memoria (Airtable)` |
| 15 | 915 → 916 | `Recuperar › lectura de reflexiones e importantes (Airtable)` |
| 12 | 911 → 912 | `Recuperar › lectura de la ventana (almacén Caché)` |
| 5 | 905 → 906 | `Recuperar › cálculo de relevancia (código)` |
| 14 | 913 → 914 | `Recuperar › guardado de la ventana (almacén Caché)` |
| 6, 7 | 909, 907 | Solo Commit, porque ya se ha respondido o la que falla es la propia respuesta. |

En todos los casos, el manejador responde 500 con ese cuerpo y hace Commit.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_RECUPERAR__` | Un webhook nuevo. Su URL es `__WEBHOOK_RECUPERAR__` en el Loop, Resumen y Reflexionar. |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulo 2). |
| `__CONEXION_AIRTABLE__`, `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | Conexión, base y tabla de la memoria (módulos 3, 15 y 7). La base y la tabla aparecen también en la URL del módulo 7. |
| `__ALMACEN_CACHE__` | El almacén Caché, con clave de texto y los campos `agente`, `chat_id` y `ventana` (módulos 12 y 14). |
| `__CAMPO_Ultimo_acceso__` | El identificador del campo Ultimo_acceso, dentro del código del módulo 5. |

Después de importar hay que fijar la programación en *Immediately*.
