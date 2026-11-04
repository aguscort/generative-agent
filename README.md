# Human_proxy

Human_proxy es un bot de Telegram con el que se conversa con Borja, un asesor financiero ficticio del Madrid de los 90, como si fuera una persona: recuerda lo que le cuentas, forma su propia opinión sobre ti y sobre sí mismo con el tiempo, tiene iniciativa propia y sigue la actualidad financiera aunque nadie le escriba. Si le cuentas qué inversiones tienes («he comprado 50 acciones de Inditex a 45 euros»), las apunta en tu cartera; si le preguntas cómo van («¿cuánto vale hoy mi cartera?»), consulta las cotizaciones y te da el valor y el resultado calculados al céntimo; si le planteas una cuenta («si ahorro 300 euros al mes durante 20 años al 5 %, ¿cuánto tendría?»), la resuelve con código en lugar de estimarla. Cada mañana revisa la agenda de eventos financieros del día, estudia cada uno cuando se publica, lee la prensa económica y relaciona lo que pasa con tu cartera; cuando le falta un dato para valorar algo, te escribe él para preguntártelo. Y cumple lo que promete: si le pides «avísame mañana de cómo abre Inditex» y te dice que sí, lo apunta como compromiso y te escribe a su hora. Para usarlo basta con abrir el bot en Telegram y escribirle con naturalidad: no hay comandos, y todo lo que dice y aprende se guarda por conversación, de modo que cada usuario tiene su propia relación con él.

Por debajo, el bot implementa en Make.com la memoria, la recuperación y la reflexión de *Generative Agents: Interactive Simulacra of Human Behavior* (Park et al., 2023, [arXiv 2304.03442](https://arxiv.org/abs/2304.03442)). Su iniciativa no sale de un plan diario, como en ese paper, sino de los compromisos de la programación orientada a agentes de Shoham (*Agent-oriented programming*, Artificial Intelligence 60, 1993): órdenes que el agente se da a sí mismo o promete a otros, cada una con su momento, y que un intérprete ejecuta cuando vencen. Guarda lo que vive en Airtable y razona con Gemini.

## Qué hace

- **Recuerda.** Cada mensaje se guarda como una memoria con su importancia y su embedding. Antes de responder, el agente recupera lo más relevante combinando recencia, importancia y similitud, como en el paper.
- **Reflexiona.** Cuando la importancia acumulada supera un umbral, extrae conclusiones de alto nivel y las vuelve a guardar como memorias, con las que las justifican.
- **Se compromete.** Cada recuerdo nuevo pasa por una regla de compromiso: si contiene una promesa o una decisión del agente, se convierte en una orden con su momento, que puede depender de una condición («si Inditex baja de 40, te aviso»).
- **Cumple sus compromisos.** Cada media hora, los laborables de 8:00 a 20:30, un intérprete ejecuta las órdenes que han vencido: revisa la agenda del día a las 8:00, estudia cada evento una hora después de publicarse y lo cruza con la cartera de cada usuario, avisa a quien se lo prometió y pregunta si le falta un dato. A las 20:00 cierra el día: anula lo que decidió por su cuenta y no hizo, y pasa al día siguiente lo que prometió a otros.
- **Lee la prensa.** Dos veces al día revisa los titulares de la prensa económica y guarda los que afectan a la cartera de cada usuario, con su implicación.
- **Lleva tu cartera.** Registra las posiciones que le cuentas en la conversación, con cantidad y precio medio.
- **Consulta cotizaciones y calcula.** Pide precios a Yahoo Finance cuando hacen falta, valora la cartera y resuelve cuentas financieras con código ejecutado en Make, sin dejar la aritmética al modelo.
- **Estudia.** Ante una pregunta profesional que no sabe responder, investiga en la web y guarda lo aprendido.

La documentación técnica completa, con el detalle de cada escenario y sus costes, está en [DOCUMENTACION.md](DOCUMENTACION.md).

## Escenarios

| Archivo | Escenario | Disparo |
| --- | --- | --- |
| `blueprints/01_gateway_telegram.blueprint.json` | (1) Gateway de Telegram | Mensaje en Telegram |
| `blueprints/02_memoria_escribir.blueprint.json` | (2) Memoria: Escribir | Webhook |
| `blueprints/03_memoria_reflexionar.blueprint.json` | (3) Memoria: Reflexionar | Webhook |
| `blueprints/04_memoria_recuperar.blueprint.json` | (4) Memoria: Recuperar | Webhook |
| `blueprints/05_planificar.blueprint.json` | (5) Planificar · obsoleto | Desactivado |
| `blueprints/06_responder.blueprint.json` | (6) Responder | Webhook |
| `blueprints/07_resumen_agente.blueprint.json` | (7) Resumen del agente | Webhook |
| `blueprints/08_estudiar.blueprint.json` | (8) Estudiar | Webhook |
| `blueprints/09_interpelar.blueprint.json` | (9) Interpelar | Webhook |
| `blueprints/10_despertador.blueprint.json` | (10) Despertador | Diario, 5:00 |
| `blueprints/11_percibir_reaccionar.blueprint.json` | (11) Percibir: intérprete de compromisos | Cada 30 minutos, laborables de 8:00 a 20:30 |
| (sin blueprint en esta versión) | (12) Noticias | Laborables a las 8:00 y a las 16:00 |
| (sin blueprint en esta versión) | (13) Cotizar | Webhook |
| (sin blueprint en esta versión) | (14) Comprometer | Webhook |

Los blueprints de esta versión corresponden a una revisión anterior de los escenarios: no incluyen Noticias, Cotizar ni Comprometer, y los de Gateway, Escribir, Responder, Despertador y Percibir no reflejan aún los cambios descritos en la documentación (cartera estructurada, cotizaciones, calculadora y sistema de compromisos). El de Percibir es el del antiguo bucle que leía el plan, no el del intérprete. Planificar está obsoleto: en Make sigue con el prefijo «[OBSOLETO]», desactivado, y no hace falta para que el sistema funcione. La documentación describe el sistema actual con el detalle suficiente para reconstruirlo.

Cada blueprint lleva notas de Make en sus webhooks y en cada llamada a webhook con el origen y el destino.

## Requisitos

- Una cuenta de Make.com con un plan de pago, porque la calculadora usa el módulo Make Code. Con un usuario activo, el sistema consume, según una estimación, unos 3.200–3.700 créditos al mes antes de conversar; la sección 20 de la documentación desglosa el coste.
- Una base de Airtable.
- Una clave de la API de Gemini.
- Un bot de Telegram.
- Opcional: Slack, si quieres un registro de las conversaciones en un canal.

Las cotizaciones y la prensa no necesitan clave: salen de Yahoo Finance y de un feed RSS de Google News.

## Instalación

1. Crea en Airtable la tabla Memory Stream con los campos de la sección 3 de la documentación.
2. Crea en Make tres almacenes de datos: la caché (agente, chat_id, resumen, actualizado), Cartera (agente, chat_id, cartera, posiciones, actualizado) y Compromisos, con los campos de la sección 17 de la documentación. Guarda en la caché el registro `perfil|<agente>` con la descripción del personaje.
3. Importa los blueprints y crea el webhook de cada escenario que lo tenga. Crea todos los webhooks antes de rellenar ninguna llamada, porque hay dependencias circulares: Escribir llama a Reflexionar y Reflexionar a Escribir. Monta Noticias, Cotizar, Comprometer y el intérprete, y actualiza Gateway, Escribir, Responder y Despertador, siguiendo sus secciones de la documentación. Planificar no hace falta.
4. Sustituye los marcadores de la tabla siguiente. Las notas de cada webhook dicen qué módulos lo llaman.
5. Elige en cada módulo las conexiones y los almacenes de datos, que llegan vacíos.
6. Fija la programación: inmediata en los escenarios con webhook, diaria a las 5:00 en el Despertador, cada 30 minutos de 8:00 a 20:30 en laborables en el intérprete y a las 8:00 y 16:00 en laborables en Noticias.

| Marcador | Qué poner |
| --- | --- |
| `<WEBHOOK_ESCRIBIR>`, `<WEBHOOK_RESPONDER>`… | El identificador del webhook de cada escenario |
| `<AIRTABLE_BASE_ID>`, `<AIRTABLE_TABLE_ID>` | La base y la tabla Memory Stream |
| `<FLD_MEMORIA>`, `<FLD_TIPO>`… | El ID de cada campo de la tabla; el nombre indica cuál |
| `<SLACK_CHANNEL_ID>` | El canal de Slack, si lo usas |

## Seguridad

Por defecto, Make no protege las URLs de los webhooks: quien tenga una puede invocar el escenario, leer o escribir la memoria del agente y gastar tus créditos. No publiques tus URLs ni las incluyas en una copia de este repositorio. Make permite exigir una clave en cada webhook; si abres el bot a terceros, actívala.

La tabla de Airtable guarda literalmente lo que cada usuario dice al bot; el almacén Cartera, las inversiones que le cuenta, y Compromisos, lo que el agente le ha prometido. Si lo abres a terceros, avísales.

La ruta de cálculo ejecuta en el entorno aislado de Make el código que escribe el modelo, tras un filtro que rechaza cualquier acceso externo. Si el bot va a hablar con desconocidos, revisa la sección 21 de la documentación.
