# Human_proxy

Agentes conversacionales con memoria, reflexión y planificación, construidos en Make.com sobre la arquitectura de *Generative Agents: Interactive Simulacra of Human Behavior* (Park et al., 2023, [arXiv 2304.03442](https://arxiv.org/abs/2304.03442)). El agente habla por Telegram, guarda lo que vive en Airtable y razona con Gemini.

## Qué hace

- **Recuerda.** Cada mensaje se guarda como una memoria con su importancia y su embedding. Antes de responder, el agente recupera lo más relevante combinando recencia, importancia y similitud, como en el paper.
- **Reflexiona.** Cuando la importancia acumulada supera un umbral, extrae conclusiones de alto nivel y las vuelve a guardar como memorias, con las que las justifican.
- **Planifica.** Cada día genera un plan de tareas y una agenda de eventos de su dominio.
- **Percibe y reacciona.** Cada hora estudia el evento que tocaba, lo cruza con los intereses de cada usuario y, si le falta un dato, pregunta por iniciativa propia.
- **Estudia.** Ante una pregunta profesional que no sabe responder, investiga en la web y guarda lo aprendido.

La documentación técnica completa, con el detalle de cada escenario y sus costes, está en [DOCUMENTACION.md](DOCUMENTACION.md).

## Escenarios

| Archivo | Escenario | Disparo |
| --- | --- | --- |
| `blueprints/01_gateway_telegram.blueprint.json` | (1) Gateway de Telegram | Mensaje en Telegram |
| `blueprints/02_memoria_escribir.blueprint.json` | (2) Memoria: Escribir | Webhook |
| `blueprints/03_memoria_reflexionar.blueprint.json` | (3) Memoria: Reflexionar | Webhook |
| `blueprints/04_memoria_recuperar.blueprint.json` | (4) Memoria: Recuperar | Webhook |
| `blueprints/05_planificar.blueprint.json` | (5) Planificar, tareas y eventos | Webhook |
| `blueprints/06_responder.blueprint.json` | (6) Responder | Webhook |
| `blueprints/07_resumen_agente.blueprint.json` | (7) Resumen del agente | Webhook |
| `blueprints/08_estudiar.blueprint.json` | (8) Estudiar | Webhook |
| `blueprints/09_interpelar.blueprint.json` | (9) Interpelar | Webhook |
| `blueprints/10_despertador.blueprint.json` | (10) Despertador | Diario, 5:00 |
| `blueprints/11_percibir_reaccionar.blueprint.json` | (11) Percibir y reaccionar | Cada hora, laborables de 9:00 a 23:00 |

Cada blueprint lleva notas de Make en sus webhooks y en cada llamada a webhook con el origen y el destino.

## Requisitos

- Una cuenta de Make.com. Con un usuario activo el sistema consume unos 9.400 créditos al mes antes de conversar; la sección 17 de la documentación desglosa el coste.
- Una base de Airtable.
- Una clave de la API de Gemini.
- Un bot de Telegram.
- Opcional: Slack, si quieres un registro de las conversaciones en un canal.

## Instalación

1. Crea en Airtable la tabla Memory Stream con los campos de la sección 3 de la documentación.
2. Crea en Make dos almacenes de datos: la caché (agente, chat_id, resumen, actualizado) y Cartera (agente, chat_id, cartera, actualizado). Guarda en la caché el registro `perfil|<agente>` con la descripción del personaje.
3. Importa los 11 blueprints y crea el webhook de cada escenario que lo tenga. Crea todos los webhooks antes de rellenar ninguna llamada, porque hay dependencias circulares: Escribir llama a Reflexionar y Reflexionar a Escribir.
4. Sustituye los marcadores de la tabla siguiente. Las notas de cada webhook dicen qué módulos lo llaman.
5. Elige en cada módulo las conexiones y los almacenes de datos, que llegan vacíos.
6. Fija la programación: inmediata en los escenarios con webhook, diaria a las 5:00 en el Despertador y cada hora de 9:00 a 23:00 en laborables en Percibir.

| Marcador | Qué poner |
| --- | --- |
| `<WEBHOOK_ESCRIBIR>`, `<WEBHOOK_RESPONDER>`… | El identificador del webhook de cada escenario |
| `<AIRTABLE_BASE_ID>`, `<AIRTABLE_TABLE_ID>` | La base y la tabla Memory Stream |
| `<FLD_MEMORIA>`, `<FLD_TIPO>`… | El ID de cada campo de la tabla; el nombre indica cuál |
| `<SLACK_CHANNEL_ID>` | El canal de Slack, si lo usas |

## Seguridad

Make no protege las URLs de los webhooks: quien tenga una puede invocar el escenario, leer o escribir la memoria del agente y gastar tus créditos. No publiques tus URLs ni las incluyas en una copia de este repositorio.

La tabla de Airtable guarda literalmente lo que cada usuario dice al bot. Si lo abres a terceros, avísales.
