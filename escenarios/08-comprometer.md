# 08 · Comprometer

**Escenario:** `[Human_proxy]:[03-Planning] (08/20) | Comprometer`
**Blueprint:** [`blueprints/08-comprometer.blueprint.json`](../blueprints/08-comprometer.blueprint.json)
**Disparador:** instantáneo, por webhook (`__HOOK_COMPROMETER__`).
**Lo llaman:** el Loop (peticiones y liberaciones del usuario), Reflexionar (decisiones propias), el Despertador y Cumplir (avisos de caducidad y de fallo).

## Función

Es la puerta de entrada al almacén de compromisos. Sigue la idea de AGENT0 (Shoham, 1993): el agente no hace las cosas en el momento en que se le piden, sino que se compromete a hacerlas en un momento dado, y un compromiso solo se acepta si cumple unas reglas. Comprometer aplica esas reglas y decide:

| Acto | Origen | Decisión posible |
|---|---|---|
| `REQUEST` | El usuario pide algo. | Aceptarlo o rechazarlo. |
| `DECISION` | El propio agente decide hacer algo. | Aceptarlo o rechazarlo. |
| `UNREQUEST` | El usuario libera al agente de algo. | Borrar los compromisos que queden anulados. |

Las reglas para aceptar un compromiso nuevo, en el orden en que se comprueban:

1. **Capacidad.** El agente sabe hacer esa acción, y el acto la permite.
2. **Contenido.** Está claro qué hay que hacer: hay contenido.
3. **Destinatario.** Si la acción es escribir al chat, hay un chat real al que escribir (no GLOBAL).
4. **Cadena.** El nivel es 0, 1 o 2. Un compromiso que nace de otro sube un nivel, y así se cortan las cadenas largas.
5. **Tope.** El agente no tiene ya 10 compromisos pendientes con ese chat.
6. **Duplicado.** No tiene ya apuntado uno que pida lo mismo. Lo juzga Gemini por el significado, no por las palabras.

Además, el momento se ajusta al horario de trabajo del agente: laborables de 8:00 a 20:30, hora de Madrid.

## Entrada

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat_id` | Chat al que pertenece el compromiso, o `GLOBAL`. |
| `acto` | `REQUEST`, `UNREQUEST` o `DECISION`. Por defecto, `DECISION`. |
| `accion` | `informar`, `preguntar`, `investigar`, `cotizar`, `estudiar`, `abstenerse` o `agendar`. |
| `contenido` | Qué hay que hacer, en una frase. |
| `condicion` | Condición de la que depende, si la hay. |
| `simbolos` | Tickers de Yahoo Finance separados por comas, si los hay. |
| `dia` | `AAAA-MM-DD`, hora de Madrid. Si falta, hoy. |
| `hora` | `HH:mm`, hora de Madrid. Si falta: las 08:00 cuando hay día, o la hora actual cuando no lo hay. |
| `fin` | `AAAA-MM-DD`. Fecha límite; vence ese día a las 20:30. |
| `nivel` | Longitud de la cadena de compromisos. Por defecto, 0. |
| `origen` | De dónde viene el compromiso. Si falta, se guarda el acto en minúsculas. |

## Respuesta

Siempre 200, en texto plano (`text/plain; charset=utf-8`). El texto empieza por una palabra que resume la decisión:

| Inicio | Cuándo | Ejemplo |
|---|---|---|
| `ACEPTADO.` | Se da de alta. | `ACEPTADO. Borja se ha comprometido: (DO 17/11 09:00 cotizar «…»). Lo hará el 17/11 a las 09:00.` |
| `RECHAZADO.` | No cumple alguna regla. | `RECHAZADO. Borja no se compromete porque ya lo tenía apuntado.` |
| `LIBERADO.` | Se borran compromisos. | `LIBERADO. Borja queda libre de estos compromisos con este chat: …` |
| `NADA.` | Una liberación que no anula nada. | `NADA. Borja no tenía ningún compromiso así con este chat.` |

Si se aplaza el compromiso por caer fuera del horario, el texto de ACEPTADO lo explica. Si el compromiso tiene condición o fecha de fin, también.

Si falla un paso, responde 500 con `{"error": "Comprometer › <paso> (<servicio>)"}`.

## El almacén Compromisos

Cada compromiso es un registro del almacén, con clave generada por Make:

| Campo | Contenido |
|---|---|
| `agente` | Nombre del agente. |
| `chat` | Chat del compromiso o `GLOBAL`. |
| `accion` | Acción. |
| `contenido`, `condicion`, `simbolos` | Datos de la acción. |
| `momento` | Cuándo hacerlo (ISO 8601, UTC). |
| `fin` | Cuándo caduca, si caduca. |
| `linea` | El compromiso en una línea legible (ver abajo). |
| `nivel` | Longitud de la cadena. |
| `origen` | De dónde viene. |
| `estado` | `pendiente` al darse de alta; `en_curso` mientras Cumplir lo ejecuta. |
| `intentos` | Veces que Cumplir lo ha intentado. |
| `creado` | Fecha del alta. |
| `destinatario` | Chat al que escribir, en las acciones que escriben al chat. |
| `comprometido_con` | El chat, si es una petición del usuario; `yo`, si es una decisión propia. |

`destinatario` y `comprometido_con` son campos anteriores a `chat`. Se siguen escribiendo porque los registros antiguos solo tienen esos campos y algunos escenarios todavía los leen.

La `linea` sigue la notación de AGENT0. Se usa para mostrar el compromiso a Gemini y en las respuestas:

```
(<TIPO> <DD/MM> <HH:mm> <acción>[ chat:<chat>] «<contenido>»)
(IF (B «<condición>») (<TIPO> …))
```

Donde `<TIPO>` es `DO` para las tareas propias, `INFORM` para informar, `REQUEST` para preguntar y `REFRAIN` para abstenerse. `(B «…»)` indica una creencia: el compromiso se activa cuando el agente cree que se cumple la condición.

## Flujo

```
1 Webhook
3 Código: preparación del compromiso
31 Variables: comprobaciones al recibir
80 Router de entrada
├─ Ruta 1: 81 Respuesta RECHAZADO (falta algo)  [Faltan datos o no sabe hacerlo]
└─ Ruta 2: 6 Almacén: compromisos vigentes  [Es una liberación o los datos están completos]
           50 Agregador: compromisos de este chat
           7 Router
           ├─ Ruta 1  [Es una liberación]
           │  60 Gemini: compromisos a liberar
           │  61 Router
           │  ├─ Ruta 1: 62 Iterador → 63 Almacén: borrado  [La línea está en la lista]
           │  └─ Ruta 2: 64 Respuesta LIBERADO o NADA
           └─ Ruta 2: 70 Router  [Es un compromiso nuevo]
              ├─ Ruta 1: 71 Respuesta RECHAZADO (tope)  [Ya tiene 10 pendientes con este chat]
              └─ Ruta 2  [Hay sitio]
                 43 Gemini: duplicados
                 32 Router
                 ├─ Ruta 1: 33 Almacén: alta  [No está duplicado]
                 │          34 Respuesta ACEPTADO
                 └─ Ruta 2: 35 Respuesta RECHAZADO (duplicado)  [Está duplicado]
```

## Módulos

### 1. Webhook

Atiende las llamadas de una en una.

### 3. Preparación del compromiso (código JavaScript)

Recibe los datos del webhook y la hora actual, y hace cuatro cosas:

1. **Normaliza la entrada.**
   - Quita los espacios sobrantes.
   - Pone el acto en mayúsculas, con `DECISION` si no llega.
   - Pone la acción en minúsculas.
   - Pone los símbolos en mayúsculas y sin espacios.
2. **Calcula el momento en hora de Madrid,** con el cambio de hora incluido. Si el momento pedido cae en un laborable entre las 8:00 y las 20:30, se queda tal cual. Si cae antes de las 8:00 de un laborable, pasa a las 8:00 de ese día. En cualquier otro caso pasa a las 8:00 del siguiente laborable, y se apunta que se ha aplazado.
3. **Decide destinatario y tipo.** Informar, preguntar y abstenerse escriben al chat, y su destinatario es el chat si no es GLOBAL. `comprometido_con` es el chat en un REQUEST, y `yo` en los demás casos.
4. **Calcula la fecha de fin:**
   - si llega `fin`, ese día a las 20:30;
   - si es una abstención, no tiene fin;
   - si hay condición, una semana después del momento;
   - si es un estudio, el mismo día a las 20:30;
   - en los demás casos, sin fin.

   Además escribe la `linea` y el momento legible (`DD/MM a las HH:mm`).

Devuelve, con prefijo `cm_`: acto, accion, chat, contenido, simbolos, condicion, nivel_txt, momento, aplazado, dest, con, tipo, fin, cuando y linea. Si la fecha o la hora no tienen el formato esperado, el código falla y actúa el manejador.

### 31. Comprobaciones al recibir (*Set multiple variables*)

Calcula cuatro respuestas de sí o no, que usa el router 80:

| Variable | Pregunta | Regla |
|---|---|---|
| `cm_ok_accion` | ¿Sabe hacerlo? | En un REQUEST valen informar, preguntar, estudiar, investigar, cotizar y abstenerse. En una DECISION valen esas y además agendar. |
| `cm_ok_dest` | ¿Hay a quién escribir? | Si la acción es informar, preguntar o abstenerse, tiene que haber destinatario. En otro caso, sí. |
| `cm_ok_nivel` | ¿La cadena es corta? | El nivel es 0, 1 o 2. |
| `cm_ok_cont` | ¿Hay contenido? | El contenido no está vacío. |

### 80. Router de entrada

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Faltan datos o no sabe hacerlo | No es UNREQUEST y alguna comprobación da «no». |
| 2 | Es una liberación o los datos están completos | Es UNREQUEST, o las cuatro comprobaciones dan «sí». |

Una liberación nunca se rechaza por falta de datos.

### 81. Respuesta RECHAZADO (falta algo)

Da el primer motivo que falle, en este orden:

1. «no sabe hacer eso o no le corresponde»;
2. «no está claro qué tiene que hacer»;
3. «no sabe a quién tendría que escribir»;
4. «sería una cadena de compromisos demasiado larga».

### 6. Compromisos vigentes (almacén, *Search records*)

Busca los compromisos del agente que no están cumplidos, fallidos, liberados ni caducados, ordenados por momento. Como mucho trae 100. Si no hay ninguno, sigue igualmente (`continueWhenNoRes`).

### 50. Compromisos de este chat (agregador de texto)

Junta en un texto, uno por línea, los compromisos cuyo chat coincide con el de la llamada, con el formato `<clave> · <linea>`. El chat de cada compromiso se toma de `chat` o, en los registros antiguos, de `destinatario` o `comprometido_con`. Este texto sirve para tres cosas: el tope, los duplicados y las liberaciones.

### 7. Router por tipo de llamada

| Ruta | Filtro |
|---|---|
| 1 | Es una liberación (UNREQUEST) |
| 2 | Es un compromiso nuevo (REQUEST o DECISION) |

### 60. Compromisos a liberar (Gemini 3.8 Flash, salida JSON)

Instrucción de sistema:

> Recibes lo que el interlocutor de un agente le pide que deje de hacer y los compromisos vigentes del agente con él. Juzgas por el significado, no por las palabras. Respondes solo con el JSON pedido.

Prompt:

```
<context>
Compromisos vigentes de {{agente}} con su interlocutor, uno por línea, cada uno precedido de su clave:
<lista_vigentes>
{{líneas del módulo 50, o "ninguno"}}
</lista_vigentes>

Petición del interlocutor para dejar de hacer algo:
- acción: {{accion}}
- contenido: {{contenido}}
- condición: {{condicion}}
- valores: {{simbolos}}
</context>

<instructions>
¿Qué compromisos de <lista_vigentes> quedan anulados por esta petición?
</instructions>

<output_format>
lineas:
<copia aquí las líneas de esos compromisos tal como aparecen en la lista. Si no hay ninguno, deja este espacio totalmente en blanco>
</output_format>
```

Esquema de respuesta: `{"lineas": [<texto>, …]}`.

### 62 → 63. Borrado de los liberados

El iterador 62 recorre las líneas que devuelve Gemini. El módulo 63 borra el compromiso cuya clave es la parte de la línea anterior a `·`.

**Filtro: «La línea está en la lista».** Solo se borra si la línea aparece literalmente en el texto del módulo 50. Así, una línea inventada por Gemini nunca borra nada.

Un compromiso liberado se borra, en vez de marcarse, porque deja de importar al sistema.

### 64. Respuesta LIBERADO o NADA

LIBERADO, con las líneas liberadas separadas por ` / `, o NADA si Gemini no ha devuelto ninguna.

### 70. Router del tope

La expresión de los filtros comprueba si existe una décima línea en el texto del módulo 50:

```
if(accion = abstenerse; "no"; if(get(remove(split(50.text; newline); ""); 10); "si"; "no"))
```

| Ruta | Filtro | Condición |
|---|---|---|
| 1 | Ya tiene 10 pendientes con este chat | La expresión da «si». |
| 2 | Hay sitio | La expresión da «no». |

Las abstenciones no cuentan para el tope.

### 71. Respuesta RECHAZADO (tope)

> RECHAZADO. Borja no se compromete porque tiene demasiados compromisos pendientes.

### 43. Duplicados (Gemini 3.8 Flash, salida JSON)

Instrucción de sistema:

> Comparas un compromiso nuevo de un agente con los compromisos que ya tiene. Juzgas por el significado, no por las palabras: dos compromisos son el mismo aunque estén redactados de forma distinta. Respondes solo con el JSON pedido.

Prompt:

```
<context>
Compromisos vigentes de {{agente}} con su interlocutor, uno por línea:
<lista_vigentes>
{{líneas del módulo 50, o "ninguno"}}
</lista_vigentes>

Nuevo compromiso recibido:
- acción: {{accion}}
- contenido: {{contenido}}
- condición: {{condicion}}
- valores: {{simbolos}}
- cuándo: {{cuando}}, hasta {{fin o "sin fecha de fin"}}
</context>

<instructions>
¿Hay en <lista_vigentes> un compromiso que ya pide lo mismo que el nuevo, aunque esté dicho de otra forma? Es el mismo si coinciden la acción, a quién va dirigida, lo que hay que hacer y la condición o los valores de los que depende.
</instructions>

<output_format>
duplicado: <si lo hay, copia aquí su línea tal como aparece en la lista. Solo puede haber uno. Si no lo hay, déjalo totalmente en blanco>
</output_format>
```

Esquema de respuesta: `{"duplicado": <texto>}`.

### 32. Router de duplicado

| Ruta | Filtro | Módulos |
|---|---|---|
| 1 | No está duplicado (`duplicado` vacío) | 33 → 34 |
| 2 | Está duplicado | 35 |

### 33. Alta (almacén, *Add a record*)

Guarda el compromiso con los campos descritos en «El almacén Compromisos», con estado `pendiente` y 0 intentos.

### 34. Respuesta ACEPTADO

Monta el texto a partir de la línea, el momento, el aplazamiento, la condición y la fecha de fin. Las abstenciones tienen un texto propio: «Borja se compromete a no hacerlo».

### 35. Respuesta RECHAZADO (duplicado)

> RECHAZADO. Borja no se compromete porque ya lo tenía apuntado.

## Gestión de errores

| Módulo | Manejador | Cuerpo |
|---|---|---|
| 3 | 903 → 1003 | `Comprometer › preparación del compromiso (código)` |
| 6 | 906 → 1006 | `Comprometer › lectura de los compromisos (almacén Compromisos)` |
| 60 | 960 → 1060 | `Comprometer › compromisos a liberar (Gemini)` |
| 63 | 963 → 1063 | `Comprometer › liberación de compromisos (almacén Compromisos)` |
| 43 | 943 → 1043 | `Comprometer › comprobación de duplicados (Gemini)` |
| 33 | 933 → 1033 | `Comprometer › alta del compromiso (almacén Compromisos)` |
| 81, 64, 71, 34, 35 | 981, 964, 971, 934, 935 | Solo Commit. |

En todos los casos con cuerpo, el manejador responde 500 con ese cuerpo y hace Commit.

## Configuración al importar

| Marcador | Qué poner |
|---|---|
| `__HOOK_COMPROMETER__` | Un webhook nuevo. Su URL es `__WEBHOOK_COMPROMETER__` en los escenarios que lo llaman. |
| `__ALMACEN_COMPROMISOS__` | El almacén de compromisos (módulos 6, 63 y 33), con los campos descritos arriba. |
| `__CONEXION_GEMINI__` | La conexión con Gemini (módulos 60 y 43). |

Después de importar hay que fijar la programación en *Immediately*.
