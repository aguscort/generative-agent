# 06 · Consolidar/Olvidar

**Escenario:** `[Human_proxy]:[02-Memoria] (06/20) | Consolidar/Olvidar`
**Blueprint:** [`blueprints/06-olvidar.blueprint.json`](../blueprints/06-olvidar.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_OLVIDAR__`).
**Lo llama:** el Despertador, cada día después de la reflexión.

## Función

Borra de la memoria los recuerdos que ya no aportan nada. Un recuerdo se olvida si cumple todas estas condiciones:

- es una observación del agente;
- ya se ha reflejado, es decir, su contenido ya ha pasado por una reflexión;
- tiene importancia, y es menor que 4;
- ninguna reflexión lo cita como evidencia;
- se creó hace más de 15 días.

Así la memoria no crece sin límite, y no se pierde nada que sostenga una conclusión ni nada importante. Al terminar, deja constancia en la memoria de cuántos recuerdos ha borrado.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |

## Respuesta

| Caso | Código | Cuerpo |
|---|---|---|
| Correcto | 200 | `{"estado": "olvido completado", "agente": "<agente>", "borrados": <n>}` |
| Fallo | 500 | `{"error": "Olvidar › <paso> (Airtable)"}` |

## Flujo

```
1 Webhook
2 Airtable: recuerdos a olvidar  [Llega el agente]
3 Airtable (API): borrado de cada recuerdo
4 Agregador de texto: identificadores borrados
5 Airtable: registro del olvido
6 Respuesta 200
```

## Módulos

### 1. Webhook

Recibe el agente. Atiende las llamadas de una en una.

### 2. Recuerdos a olvidar (Airtable, *Search records*)

**Filtro de entrada: «Llega el agente».** El campo `agente` tiene que venir con valor.

```
AND({Agente}="<agente>", {Tipo}="observacion", {Reflejada},
    LEN({Importancia}&"")>0, {Importancia}<4,
    NOT({Citada_por}),
    IS_BEFORE(CREATED_TIME(), DATEADD(TODAY(), -15, 'days')))
```

Como mucho trae 1.000 recuerdos. La condición `LEN({Importancia}&"")>0` deja fuera los registros sin importancia. Sin ella, un registro sin importancia contaría como menor que 4 y se borraría.

### 3. Borrado (Airtable, *Make an API Call*)

Por cada recuerdo encontrado hace `DELETE /v0/<base>/<tabla>/<id>`. El borrado es definitivo.

### 4. Identificadores borrados (agregador de texto)

Junta los identificadores de los recuerdos borrados, uno por línea. Se usa para contarlos.

### 5. Registro del olvido (Airtable, *Create a record*)

Crea una observación en GLOBAL, con autor `sistema`:

> `Olvido diario de <agente>: <n> recuerdos de poca importancia borrados (observaciones ya reflejadas, sin citar en ninguna reflexión y de hace más de 15 días).`

Se escribe directamente en Airtable, sin pasar por Escribir, y por eso no lleva importancia. Como no tiene importancia, el propio Olvidar nunca la borra.

### 6. Respuesta

Responde 200 con el número de recuerdos borrados.

## Gestión de errores

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 2 | 902 → 1002 | `Olvidar › búsqueda de recuerdos a olvidar (Airtable)` |
| 3 | 903 → 1003 | `Olvidar › borrado de un recuerdo (Airtable)` |
| 5 | 905 → 1005 | `Olvidar › registro del olvido en la memoria (Airtable)` |
| 6 | 1006 | Solo Commit. |

En todos los casos el manejador responde 500 con ese cuerpo y hace Commit. Si falla el borrado de un recuerdo, la ejecución se detiene: los recuerdos ya borrados no se recuperan y los demás se intentan borrar al día siguiente.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_OLVIDAR__` | Un webhook nuevo. Su URL es la que usa el Despertador. |
| `__CONEXION_AIRTABLE__`, `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | Conexión, base y tabla de la memoria. La base y la tabla aparecen también en la URL del módulo 3. |
| `__CAMPO_<nombre>__` | Los campos del módulo 5: Memoria, Tipo, Agente, Chat_ID, Autor y Ultimo_acceso. |

Después de importar hay que fijar la programación en *Immediately*.
