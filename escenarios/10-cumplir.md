# 10 · Cumplir

**Escenario:** `[Human_proxy]:[03-Planning] (10/20) | Cumplir`
**Blueprint:** [`blueprints/10-cumplir.blueprint.json`](../blueprints/10-cumplir.blueprint.json)
**Disparador:** programado. Según su nota, cada 30 minutos, los laborables de 8:00 a 20:30.

## Función

Es la segunda fase del ciclo de AGENT0 (Shoham, 1993): recorre los compromisos cuyo momento ha llegado y gestiona su ciclo de vida. Cumplir no ejecuta ninguna acción: las ejecuta Actuar. Cumplir decide qué hacer con cada compromiso vencido:

| Situación | Qué hace Cumplir |
|---|---|
| Ha pasado su fecha de fin | Lo borra y, si era una promesa a un chat, pide a Comprometer el compromiso de avisar de que ha caducado. |
| Lleva tres intentos sin completarse | Lo borra y, si era una promesa a un chat, pide a Comprometer el compromiso de avisar de que no se ha podido cumplir. |
| Está vigente y con menos de tres intentos | Lo reserva media hora, le suma un intento y lo envía a Actuar. |

Reservarlo significa aplazar su momento media hora. Si Actuar lo completa, debe borrarlo del almacén. Si no lo hace, el compromiso vuelve a salir en la vuelta siguiente, con un intento más.

Los avisos al chat no se escriben directamente en el almacén: se piden a Comprometer, para que pasen por sus reglas (tope, duplicados, horario y longitud de la cadena). Un aviso ya no genera otro aviso, porque su origen empieza por `cumplir:`.

## Flujo

```
20 Almacén Compromisos: compromisos vencidos
21 Router (por cada compromiso)
├─ Ruta 1  [Ha pasado su fecha de fin]
│  70 Almacén: borrado
│  71 HTTP → Comprometer: aviso de caducidad  [Era una promesa a un chat y no es ya un aviso]
├─ Ruta 2  [Ha fallado tres veces]
│  72 Almacén: borrado
│  73 HTTP → Comprometer: aviso de fallo  [Era una promesa a un chat y no es ya un aviso]
└─ Ruta 3  [Vigente y con menos de tres intentos: reservar]
   80 Almacén: reserva (en_curso, +30 min, +1 intento)
   1197 HTTP → Actuar
```

## Módulos

### 20. Compromisos vencidos (almacén Compromisos, *Search records*)

Busca los compromisos que cumplen todas estas condiciones:

- su momento es anterior o igual a ahora;
- no son de `abstenerse`, porque una abstención no se ejecuta;
- su estado no es `cumplido`, `fallido`, `liberado` ni `caducado`.

Los ordena por momento, del más antiguo al más reciente, y toma como mucho 100 por vuelta. Si no hay ninguno, la ejecución termina. Cada compromiso encontrado sigue su propio camino por el router.

Salen también los compromisos atrasados de vueltas anteriores. Si han pasado su fecha de fin, van por la ruta de caducado.

### 21. Router

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Ha pasado su fecha de fin | Tiene `fin` y `fin` es anterior a ahora. |
| 2 | Ha fallado tres veces | `intentos` ≥ 3, y no tiene `fin` o `fin` aún no ha pasado. |
| 3 | Vigente y con menos de tres intentos: reservar | `intentos` < 3, y no tiene `fin` o `fin` aún no ha pasado. |

Las tres condiciones son excluyentes: cada compromiso toma una sola ruta.

### 70 y 72. Borrado (almacén, *Delete a record*)

Borra el compromiso por su clave.

### 71 y 73. Aviso al chat (HTTP → Comprometer)

**Filtro: «Era una promesa a un chat y no es ya un aviso».** `comprometido_con` no es `yo` (es decir, era una petición del usuario) y `origen` no contiene `cumplir:`.

Pide a Comprometer una DECISION:

| Campo | Valor |
|---|---|
| `chat_id` | El chat del compromiso: `chat` o, en registros antiguos, `destinatario` o `comprometido_con`. |
| `accion` | `informar` |
| `contenido` | 71: «Decir a su interlocutor que ha vencido el plazo de este encargo sin que se diera la condición: «…»». 73: «Decir a su interlocutor que no ha podido cumplir este compromiso: «…»». |
| `nivel` | Nivel del compromiso + 1. |
| `origen` | `cumplir:caducado` o `cumplir:fallido`. |

La respuesta de Comprometer no se usa. Si el nivel resultante pasa de 2, Comprometer rechaza el aviso por cadena demasiado larga.

### 80. Reserva (almacén, *Update a record*)

Pone el estado `en_curso`, aplaza el momento a dentro de 30 minutos y suma 1 a `intentos`. Los demás campos se conservan.

### 1197. Envío a Actuar (HTTP)

Envía el compromiso a Actuar (`__WEBHOOK_ACTUAR__`), con la conexión de API key y una espera de 120 segundos. Cuerpo:

| Campo | Valor |
|---|---|
| `clave` | Clave del compromiso en el almacén. |
| `agente`, `comprometido_con`, `destinatario` | Del compromiso. |
| `chat_id` | `chat` o, si falta, `destinatario`. |
| `contenido`, `condicion`, `simbolos`, `origen` | Del compromiso. |
| `nivel` | Del compromiso, o 0. |
| `criterio`, `historial`, `correccion` | Vacíos. |
| `paso`, `correcciones` | `1` y `0`. |

Los campos `paso`, `correcciones`, `historial`, `criterio` y `correccion` son el estado inicial de la ejecución por pasos de Actuar.

## Gestión de errores

Cumplir es un escenario programado: no tiene a quién responder. Por eso, cuando falla un paso, el manejador deja constancia del error en la memoria y después hace Commit. El error queda como una observación de autor `sistema`, que las reflexiones posteriores pueden analizar:

> `Error en Cumplir › <paso> (<servicio>) con el compromiso «<línea>».`

La observación se guarda en el chat del compromiso, o en GLOBAL si no tiene chat.

| Módulo | Registro | Paso | Commit |
|---|---|---|---|
| 20 | 1120, directo en Airtable, sin agente, en GLOBAL | búsqueda de compromisos vencidos (almacén Compromisos) | 920 |
| 70 | 1170, por Escribir | borrado del compromiso caducado (almacén Compromisos) | 970 |
| 71 | 1171, por Escribir | aviso de caducidad al chat (Comprometer) | 971 |
| 72 | 1172, por Escribir | borrado del compromiso fallido (almacén Compromisos) | 972 |
| 73 | 1173, por Escribir | aviso de fallo al chat (Comprometer) | 973 |
| 80 | 1180, por Escribir | reserva del compromiso (almacén Compromisos) | 980 |
| 1197 | 1198, por Escribir | envío del compromiso a Actuar (Actuar) | 1199 |

Si falla la búsqueda inicial, todavía no se sabe de qué agente es el error. Por eso ese registro se escribe directamente en Airtable, sin agente.

Si falla también el registro del error, ese segundo fallo no tiene manejador: queda como error del escenario y cuenta para su desactivación.

Commit detiene la vuelta. Los compromisos que quedaban por tratar no se pierden: salen en la vuelta siguiente.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__ALMACEN_COMPROMISOS__` | El almacén de compromisos (módulos 20, 70, 72 y 80). |
| `__CONEXION_API_KEY_MAKE__` | La conexión de API key para las llamadas a otros escenarios. |
| `__WEBHOOK_COMPROMETER__`, `__WEBHOOK_ACTUAR__`, `__WEBHOOK_ESCRIBIR__` | Las URL de esos escenarios. |
| `__CONEXION_AIRTABLE__`, `__BASE_AIRTABLE__`, `__TABLA_MEMORY_STREAM__`, `__CAMPO_<nombre>__` | Conexión, base, tabla y campos de la memoria, para el registro directo del módulo 1120. |

Después de importar hay que programar el escenario: cada 30 minutos, los laborables entre las 8:00 y las 20:30. La programación no se guarda en el blueprint.
