# Generative Agent

Se trata de diseñar un bot de Telegram para conversar con **Borja**, un asesor financiero. Borja recuerda lo que le cuentas, se forma una opinión sobre ti con el tiempo, sigue la actualidad financiera y cumple lo que acepta hacer. No hay comandos: se le escribe con naturalidad, y cada conversación tiene su propia memoria.

Está construido en Make.com sobre tres ideas, cada una tomada de un trabajo de investigación:

| Parte | Pregunta que resuelve | Se basa en |
| --- | --- | --- |
| [Memoria](#memoria) | ¿Qué sabe Borja y qué recuerda en cada momento? | Park *et al.* (2023), *Generative Agents* |
| [Planificación](#planificación) | ¿A qué se compromete y cuándo? | Shoham (1993), *Agent-oriented programming* |
| [Ejecución](#ejecución-en-desarrollo) | ¿Cómo cumple lo que ha prometido? | Shinn *et al.* (2023), *Reflexion* |

La memoria vive en Airtable, los compromisos en un almacén de datos de Make y el lenguaje lo pone Gemini.

> [!NOTE]
> La documentación técnica, con cada escenario, sus prompts y sus costes, está en [DOCUMENTACION.md](DOCUMENTACION.md).

## Qué hace

| | Qué hace Borja |
| --- | --- |
| **Recuerda** | Guarda cada mensaje y, antes de responder, recupera lo que viene al caso. |
| **Reflexiona** | De vez en cuando saca conclusiones de lo vivido y las guarda como recuerdos nuevos. |
| **Se compromete** | Si le pides algo («avísame mañana de cómo abre Inditex») o le liberas de algo («olvida lo de Inditex»), decide con reglas fijas si acepta, y te lo dice. |
| **Cumple** | Cada media hora, en su horario, hace lo que tiene pendiente. Si no puede cumplir algo que te prometió, te avisa. |
| **Lee la prensa** | Dos veces al día guarda los titulares que afectan a tu cartera, con lo que implican. |
| **Lleva tu cartera, cotiza y calcula** | Apunta las posiciones que le cuentas, consulta precios en Yahoo Finance y resuelve las cuentas con código, no a ojo. |
| **Estudia** | Si le haces una pregunta profesional que no sabe responder, investiga en la web y guarda lo aprendido. |

Las tres partes trabajan juntas: lo que dices entra en la memoria y en la planificación; la planificación deja compromisos; la ejecución los cumple, y lo que hace vuelve a la memoria.

```mermaid
flowchart LR
    U([Tu mensaje]) --> M["Memoria<br/>recordar y reflexionar"]
    U --> P["Planificación<br/>comprometerse"]
    M -- conclusiones que exigen actuar --> P
    P --> C[("Compromisos")]
    C --> E["Ejecución<br/>cumplir"]
    E -- lo que hace y aprende --> M
    E --> T([Mensaje de Borja])
```

## Memoria

**Para qué sirve.** Para que Borja se comporte de forma coherente con lo que ha vivido: que no te pregunte dos veces lo mismo, que recuerde tu cartera y que su opinión sobre ti cambie con el tiempo.

**La idea.** Toda la experiencia del agente se guarda en un único registro, como un diario: cada entrada es un recuerdo escrito en lenguaje natural. Lo difícil no es guardar, sino elegir qué recordar en cada momento. Para eso cada recuerdo lleva tres medidas:

- **Recencia:** cuánto hace que se usó por última vez. Se desvanece con las horas.
- **Importancia:** de 1 (cepillarse los dientes) a 10 (una ruptura). La puntúa Gemini al guardar el recuerdo.
- **Relevancia:** cuánto se parece a lo que se está hablando. Se calcula con su *vector*, una representación numérica del significado que permite buscar por parecido.

**Cómo funciona.**

1. **Escribir.** Todo entra por un único escenario, (2) Escribir: tus mensajes, las respuestas de Borja, sus reflexiones y lo que aprende. Cada recuerdo se guarda en Airtable con su importancia y su vector.
2. **Recuperar.** Ante una pregunta, (4) Recuperar suma las tres medidas y devuelve los recuerdos con mejor puntuación. La cuenta la hace código, no el modelo.
3. **Reflexionar.** Cuando se acumula bastante importancia sin procesar, (3) Reflexionar se pregunta qué ha aprendido, busca los recuerdos que lo prueban y guarda cinco conclusiones que los citan. Las conclusiones son también recuerdos, así que pueden alimentar reflexiones posteriores.
4. **Resumir.** (7) Resumen del agente reconstruye quién es Borja a partir de su memoria.

**Referencia.** Park, J. S., O'Brien, J. C., Cai, C. J., Morris, M. R., Liang, P. y Bernstein, M. S. (2023). *Generative Agents: Interactive Simulacra of Human Behavior*. UIST '23. [arXiv:2304.03442](https://arxiv.org/abs/2304.03442). Apartados 4.1 (memoria y recuperación) y 4.2 (reflexión).

## Planificación

**Para qué sirve.** Para que Borja decida qué va a hacer y cuándo, y para que solo prometa lo que sabe hacer.

**La idea.** Planificar es comprometerse. Un compromiso es la obligación de hacer algo en un momento dado, ante alguien o ante uno mismo; cuando es ante uno mismo, es una decisión. «Te aviso el lunes a las 10 de cómo abre Inditex» es un compromiso con un momento, un destinatario y un contenido. Borja no improvisa promesas: unas reglas fijas deciden si acepta cada una.

**De dónde salen los compromisos.**

| Fuente | Cómo |
| --- | --- |
| **Lo que pides** | Antes de contestarte, (6) Responder clasifica tu mensaje: petición, liberación o ninguna de las dos. Para hacerlo ve tus compromisos pendientes, cada uno con su clave, y señala de cuál te libera, cuál repites o qué veto choca con lo que pides. Después consulta a (14) Comprometer. |
| **Sus reflexiones** | Si una conclusión le exige actuar, (3) Reflexionar le pide a Comprometer que se comprometa consigo mismo. |
| **Rutinas** | Cada laborable, (10) Despertador le ordena preparar la agenda del día, y la agenda le ordena estudiar cada evento. |

**Las reglas.** Comprometer acepta un compromiso solo si se cumplen todas:

1. Borja sabe hacerlo y, si es un mensaje, sabe a quién enviarlo.
2. No repite un compromiso que ya tiene.
3. No contradice un veto.
4. No encadena demasiados compromisos.
5. Hay menos de **10 pendientes con ese usuario**, sin contar los vetos.

Un **veto** es un compromiso de no hacer algo, y puede ser de tres clases:

| Veto | Ejemplo | Qué impide |
| --- | --- | --- |
| De una acción | «No me escribas más» | Toda esa acción hacia ti |
| De un valor | «No me avises más de Inditex» | Esa acción, solo para ese valor |
| De un tema | «No me preguntes por mi cartera» | Lo que trate de ese tema |

Al aceptar un veto, Borja abandona lo que tuviera pendiente y choque con él.

Comprometer también ajusta el momento al horario de Borja (laborables de 8:00 a 20:30) y fija hasta cuándo vale el compromiso. Su respuesta es una de cuatro (aceptado, rechazado con el motivo, liberado o nada) y Borja contesta ateniéndose a ella: no promete lo que no se ha aceptado. Los compromisos se guardan con su estado y nunca se borran.

**Referencia.** Shoham, Y. (1993). *Agent-oriented programming*. Artificial Intelligence, 60(1), 51–92. [doi:10.1016/0004-3702(93)90034-9](https://doi.org/10.1016/0004-3702(93)90034-9). De ahí salen el estado mental del agente (creencias, capacidades y compromisos), las reglas de compromiso, los actos de pedir (REQUEST) y liberar (UNREQUEST), la abstención (REFRAIN) y el ciclo del intérprete AGENT-0.

## Ejecución (en desarrollo)

> [!IMPORTANT]
> (15) Actuar, (16) Observar y (17) Refinar están creados en Make y **desactivados**. Mientras tanto, (11) Cumplir ejecuta los compromisos con acciones fijas.

**Para qué sirve.** Para que Borja cumpla lo que ha prometido y compruebe que lo ha logrado antes de darlo por hecho.

**Hoy.** Cada media hora, en horario de trabajo, (11) Cumplir toma los compromisos vencidos y ejecuta cada uno con una de seis acciones fijas: agendar, estudiar, informar, preguntar, investigar o cotizar. Antes de actuar reserva el compromiso, para no hacerlo dos veces, y al terminar lo marca como cumplido. Si falla tres veces, lo marca como fallido; si pasa su fecha de fin, como caducado. En los dos casos te avisa si te lo había prometido.

**La idea de la versión nueva.** En lugar de una acción fija por compromiso, Borja trabaja por pasos, como lo haría una persona: hace algo, mira el resultado y corrige si hace falta. Así un compromiso puede pedir cualquier cosa que sus herramientas resuelvan.

| Escenario | Papel | Ejemplo con «Mañana a las 10 dime cómo abre Inditex» |
| --- | --- | --- |
| **(15) Actuar** | Elige un solo paso y lo da con una herramienta: cotizar, recordar, buscar, calcular, consultar la cartera, preguntar o enviar un mensaje | Primero cotiza Inditex; después te envía el mensaje |
| **(16) Observar** | Comprueba sin modelo que la herramienta respondió bien y luego, con Gemini, si ya se cumple el compromiso | Ve que hay precio y pide el paso siguiente; luego ve que el mensaje salió y lo cierra |
| **(17) Refinar** | Si algo falla, decide: corregir el paso, aplazarlo o darlo por fallido | Si el símbolo no existe, ordena cotizar con el correcto |

Cada compromiso admite como mucho 5 pasos y 2 correcciones. Los mensajes solo van al destinatario del compromiso y respetan sus vetos, y ningún paso puede crear compromisos nuevos ni decidir qué entra en la memoria.

**Pendiente antes de activarlos:**

- [ ] Que Interpelar responda con lo que envía, para que Observar sepa si la pregunta salió.
- [ ] Que Cumplir pase los compromisos a Actuar en lugar de ejecutarlos él.
- [ ] Que cada compromiso lleve un criterio de cumplimiento: qué tiene que haber pasado para darlo por hecho.

**Referencia.** Shinn, N. *et al.* (2023). *Reflexion: Language Agents with Verbal Reinforcement Learning*. NeurIPS 2023. [arXiv:2303.11366](https://arxiv.org/abs/2303.11366). Su arquitectura tiene los mismos tres papeles:

| Reflexion | Human_proxy |
| --- | --- |
| Actor: da los pasos | (15) Actuar |
| Evaluador: juzga el resultado | (16) Observar |
| Autorreflexión: convierte el fallo en una instrucción para el intento siguiente | (17) Refinar |

## Escenarios

| Archivo | Escenario | Disparo |
| --- | --- | --- |
| `blueprints/01_gateway_telegram.blueprint.json` | (1) Gateway | Mensaje en Telegram |
| `blueprints/02_memoria_escribir.blueprint.json` | (2) Escribir | Webhook |
| `blueprints/03_memoria_reflexionar.blueprint.json` | (3) Reflexionar | Webhook |
| `blueprints/04_memoria_recuperar.blueprint.json` | (4) Recuperar | Webhook |
| `blueprints/05_planificar.blueprint.json` | (5) Planificar · obsoleto | Desactivado |
| `blueprints/06_responder.blueprint.json` | (6) Responder | Webhook |
| `blueprints/07_resumen_agente.blueprint.json` | (7) Resumen del agente | Webhook |
| `blueprints/08_estudiar.blueprint.json` | (8) Estudiar | Webhook |
| `blueprints/09_interpelar.blueprint.json` | (9) Interpelar | Webhook |
| `blueprints/10_despertador.blueprint.json` | (10) Despertador | Diario, 5:00 |
| `blueprints/11_cumplir.blueprint.json` | (11) Cumplir | Cada 30 minutos, laborables de 8:00 a 20:30 |
| `blueprints/12_noticias.blueprint.json` | (12) Noticias | Laborables, 8:00 y 16:00 |
| `blueprints/13_cotizar.blueprint.json` | (13) Cotizar | Webhook |
| `blueprints/14_comprometer.blueprint.json` | (14) Comprometer | Webhook |

Planificar y la versión anterior de Percibir siguen en Make, desactivados y con el prefijo «[OBSOLETO]». No hacen falta para que el sistema funcione.

Cada blueprint lleva notas de Make en sus webhooks y en sus llamadas, con el origen y el destino de cada una.

## Requisitos

- Una cuenta de Make.com con plan de pago, porque la calculadora usa Make Code. Con un usuario, el sistema gasta unos 3.400–3.900 créditos al mes antes de conversar (sección 21 de la documentación).
- Una base de Airtable, una clave de la API de Gemini y un bot de Telegram.
- Opcional: Slack, para registrar las conversaciones en un canal.

Las cotizaciones y la prensa no necesitan clave.

> [!WARNING]
> Si Make se queda sin créditos, todos los escenarios se paran hasta que se renuevan o se compran más. Lo que se escriba al bot mientras tanto queda en cola y se procesa después.

## Instalación

1. Crea en Airtable la tabla Memory Stream con los campos de la sección 3.
2. Crea en Make tres almacenes de datos:
   - **Caché:** agente, chat_id, resumen y actualizado. Guarda en ella el registro `perfil|<agente>` con la descripción del personaje.
   - **Cartera:** agente, chat_id, cartera, posiciones y actualizado.
   - **Compromisos:** los quince campos de la sección 17, incluidos `estado` y `fin`.
3. Importa los blueprints. Crea primero todos los webhooks y después rellena las llamadas, porque hay dependencias circulares: Escribir llama a Reflexionar y Reflexionar a Escribir.
4. Elige un token secreto y ponlo en lugar de `<TOKEN>`, en los filtros «Token correcto» y en los cuerpos de las llamadas. Es el mismo en todos.
5. Sustituye el resto de marcadores (tabla siguiente). Las notas de cada webhook dicen qué módulos lo llaman.
6. Elige en cada módulo las conexiones y los almacenes de datos, que llegan vacíos.
7. Fija la programación, toda en hora de Madrid:

   | Escenario | Programación |
   | --- | --- |
   | Escenarios con webhook | Inmediata |
   | Despertador | Diaria, a las 5:00 |
   | Cumplir | Cada 30 minutos, laborables de 8:00 a 20:30 |
   | Noticias | Laborables, a las 8:00 y a las 16:00 |

| Marcador | Qué poner |
| --- | --- |
| `<TOKEN>` | El token secreto común |
| `<WEBHOOK_ESCRIBIR>`, `<WEBHOOK_RESPONDER>`… | El identificador del webhook de cada escenario |
| `<AIRTABLE_BASE_ID>`, `<AIRTABLE_TABLE_ID>`, `<AIRTABLE_VIEW_ID>` | La base, la tabla Memory Stream y su vista |
| `<FLD_MEMORIA>`, `<FLD_TIPO>`… | El ID de cada campo de la tabla; el nombre indica cuál |
| `<SLACK_CHANNEL_ID>` | El canal de Slack, si lo usas |
| `<USUARIO>` | Tu nombre, donde aparezca en un prompt o en un filtro |

## Seguridad

> [!CAUTION]
> No publiques las URLs de los webhooks ni el token. Si se expone alguno, regenera el webhook o cambia el token en todos los escenarios a la vez.

- **Token.** Cada llamada entre escenarios lleva el token, y cada escenario con webhook lo comprueba antes de hacer nada. Un webhook bloqueado por el token responde igualmente 200, así que el error no se ve desde fuera: se ve en el historial del escenario.
- **Datos.** Airtable guarda literalmente lo que cada usuario dice al bot; Cartera, sus inversiones; Compromisos, lo que el agente le ha prometido. Si el bot lo usan otras personas, deben saberlo.
- **Código.** La ruta de cálculo ejecuta el código que escribe el modelo, en el entorno aislado de Make y tras un filtro de patrones. Si el bot va a hablar con desconocidos, lee la sección 22 de la documentación.
