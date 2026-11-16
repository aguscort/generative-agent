# 11 · Despertador

**Escenario:** `[Human_proxy]:[03-Planning] (11/20) | Despertador`
**Blueprint:** [`blueprints/11-despertador.blueprint.json`](../blueprints/11-despertador.blueprint.json)
**Disparador:** programado. Según su nota, cada día a las 5:00.

## Función

Lanza cada mañana las tareas diarias del agente, sin hacer él ningún trabajo de lógica. Lee el perfil del agente y, si existe, hace tres llamadas:

1. **Agenda.** Los días laborables, pide a Comprometer el compromiso de hacer la agenda del día.
2. **Reflexión.** Llama a Reflexionar una vez por cada chat que tenga observaciones sin reflejar. Es Reflexionar quien decide, con su umbral, si hay material suficiente para reflexionar.
3. **Olvido.** Llama a Olvidar para que borre los recuerdos que ya no aportan nada.

## Flujo

```
1 Almacén Caché: perfil del agente
6 Router
├─ Ruta 1  [Hay perfil y es laborable]
│  11 HTTP → Comprometer: agenda del día
├─ Ruta 2  [Hay perfil]
│  7 Airtable: observaciones sin reflejar
│  8 Agregador de texto: chats
│  9 Iterador de chats sin repetir
│  10 HTTP → Reflexionar  [Chat no vacío]
└─ Ruta 3  [Hay perfil]
   12 HTTP → Olvidar
```

Las rutas se ejecutan en orden: primero la agenda, después la reflexión y por último el olvido.

## Módulos

### 1. Perfil del agente (almacén Caché, *Get a record*)

Lee el registro `perfil|Borja`. De él se usan `agente`, el nombre del agente que se pasa a los demás escenarios, y `resumen`, que indica si el perfil existe.

### 6. Router

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Hay perfil y es laborable | Hay `resumen` y el día de la semana en Madrid no es domingo (0) ni sábado (6). |
| 2 | Hay perfil | Hay `agente`. |
| 3 | Hay perfil | Hay `agente`. |

### 11. Agenda del día (HTTP → Comprometer)

Pide a Comprometer una DECISION:

```json
{"agente": "…", "chat_id": "GLOBAL", "acto": "DECISION", "accion": "agendar",
 "contenido": "agenda financiera del día", "dia": "<hoy>", "hora": "08:00",
 "nivel": "0", "origen": "despertador"}
```

La agenda queda como un compromiso del agente consigo mismo, a las 8:00 y en GLOBAL. Pasa por las mismas reglas que cualquier otro compromiso.

### 7. Observaciones sin reflejar (Airtable, *Search records*)

```
AND({Agente}="<agente>", {Tipo}="observacion", NOT({Reflejada}), {Chat_ID}!="GLOBAL")
```

Trae como mucho 1.000, y solo el campo Chat_ID.

### 8 → 9. Chats con material

El agregador 8 junta los chats, uno por línea. El iterador 9 los recorre sin repetir (`deduplicate`).

### 10. Reflexión (HTTP → Reflexionar)

**Filtro: «Chat no vacío».**

Llama a Reflexionar con `agente` y `chat_id`, una vez por chat. Espera hasta 300 segundos, porque una reflexión completa hace varias llamadas a Gemini y a Recuperar.

### 12. Olvido (HTTP → Olvidar)

Llama a Olvidar con `agente`. Espera hasta 300 segundos.

## Gestión de errores

El Despertador es un escenario programado y no tiene a quién responder. Todos sus manejadores solo hacen Commit:

| Módulo | Manejador |
|---|---|
| 1 | 101 |
| 11 | 111 |
| 7 | 107 |
| 10 | 110 |
| 12 | 112 |

Commit detiene la ejecución sin marcarla como fallida. Como las rutas se ejecutan en orden, un fallo en una de ellas impide las siguientes ese día: si falla la agenda, tampoco se reflexiona ni se olvida. El fallo queda solo en el historial de ejecuciones de Make.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__ALMACEN_CACHE__` | El almacén Caché, con el registro `perfil|<agente>` (campos `agente` y `resumen`). La clave tiene escrito el nombre del agente. |
| `__CONEXION_AIRTABLE__`, `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__` | Conexión, base y tabla de la memoria (módulo 7). |
| `__CONEXION_API_KEY_MAKE__` | La conexión de API key para las llamadas a otros escenarios. |
| `__WEBHOOK_COMPROMETER__`, `__WEBHOOK_REFLEXIONAR__`, `__WEBHOOK_OLVIDAR__` | Las URL de esos escenarios. |

Después de importar hay que programar el escenario: cada día a las 5:00. La programación no se guarda en el blueprint.
