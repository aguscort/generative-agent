# Arquitectura
La arquitectura del agente **Human\_proxy** ("Borja") puede conceptualizarse bajo el marco del **modelo BDI (** **Belief-Desire-Intention** **)**, el cual establece que los modelos de lenguaje no actúan como meros generadores de texto, sino como simuladores de **comunicación intencional** capaces de inferir y mantener representaciones sobre las creencias, deseos e intenciones del usuario[1].

A continuación se reescribe la arquitectura del agente integrando las bases teóricas de la literatura científica, identificando los detalles clave de su diseño actual y proponiendo vías concretas de expansión basadas en el estado del arte[6].

---

### 1\. **Perfil del Agente y Coherencia de Personalidad (** **Agent Profiling** **)**

* **Diseño actual**: El agente define su identidad ("Borja", asesor financiero de los años 90) mediante una combinación de *prompts* descriptivos (*Descriptive Personas*) e inyección periódica de resúmenes de memoria en el contexto[10].
* **Fundamentación teórica**:
  * **Ubicación latente de la personalidad**: Las representaciones semánticas de alto nivel sobre rasgos de personalidad (modelo OCEAN), ética e ideología se codifican de forma diferenciada en el **último tercio de las capas del** **decoder** del LLM[13].
  * **Auto-conciencia conductual**: Los LLMs demuestran una capacidad de *razonamiento fuera de contexto* (*Out-of-Context Reasoning* \- OOCR), lo que les permite reconocer y describir sus propias políticas de conducta y diferenciar la personalidad propia (*self*) de la de otros interlocutores sin confundirlas[16].
  * **Deriva en el discurso dinámico**: La adherencia a rasgos de personalidad en conversaciones dinámicas tiende a fluctuar debido a los cambios de contexto[19]. Asimismo, las descripciones puramente narrativas de la personalidad (*Descriptive Personas*) introducen sesgos sistemáticos hacia respuestas excesivamente positivas[12].
* **Propuestas de expansión**:
  1. **Tipificación mediante** **Tabular Personas**: Reestructurar el perfil de Borja desde descripciones narrativas libres hacia esquemas estructurados (*Tabular Personas* con atributos objetivos y subjetivos delimitados) para evitar sesgos de simulación[24][25].
  2. **Monitoreo de consistencia mediante** **Judge Agents**: Incorporar un evaluador automatizado basado en el inventario BFI (*Big Five Inventory*) para medir la deriva de personalidad durante interacciones extensas[26].

---

### 2\. **Módulo de Memoria Persistente (** **Memory-Augmented Systems** **)**

* **Diseño actual**: Implementa la arquitectura de **Stream de Memoria** (*Generative Agents*, Park et al., 2023) sobre Airtable[10][29]. Cada experiencia se almacena y se recupera calculando una puntuación combinada de **Recencia**, **Relevancia** (vía *embeddings*) e **Importancia** (evaluada de 1 a 10)[10]. Además, ejecuta un flujo periódico de **autorreflexión** para extraer conclusiones de alto nivel[10][31].
* **Fundamentación teórica**: Dentro de la taxonomía de planificación y arquitectura de agentes, este enfoque corresponde a los sistemas de **memoria augmentada basada en RAG** (*RAG-based Memory*)[32].
* **Propuestas de expansión**:
  1. **Jerarquía estilo** **MemGPT**: Implementar una gestión de memoria inspirada en los sistemas operativos, distinguiendo entre memoria principal de contexto (RAM) y almacenamiento externo persistente (Disco), permitiendo al LLM decidir cuándo recuperar o guardar información de forma autónoma[31].
  2. **Consolidación y Olvido Activo**: Integrar la **Curva del Olvido de Ebbinghaus** (*MemoryBank*) para atenuar o podar recuerdos episódicos antiguos y consolidar múltiples hechos en nodos abstractos, reduciendo el ruido en las búsquedas vectoriales[31][35].
  3. **Memoria Procedimental (** **Skill Library** **)**: Construir un banco persistente de rutinas exitosas (*Skill Memory* como en *Voyager* o *AdaPlanner*) para guardar secuencias de código de Make.com o estrategias de análisis financiero aprobadas, evitando reevaluar el procedimiento ante solicitudes idénticas[35][36].

---

### 3\. **Módulo de Planificación y Gestión de Compromisos (** **Task Planning** **)**

* **Diseño actual**: Sigue el principio de que **«planificar es comprometerse»**, derivado de la *Agent-Oriented Programming* (Shoham, 1993)[10]. Clasifica los mensajes entrantes en actos de habla (`REQUEST`, `UNREQUEST`, `INFORM`, `NINGUNO`) y registra compromisos rígidos con metadatos explícitos (acción, fecha/hora, fin, condición, símbolos y vetos)[10].
* **Fundamentación teórica**: El marco estándar de planificación exige desacoplar formalmente el **Planificador de Tareas** (*Task Planner*), el **Ejecutor del Plan** (*Plan Executor*) y el **Entorno**[8]. La literatura clasifica la planificación en cinco direcciones estratégicas[39][40]:

```
                     ┌────────────────────────────────────────┐
                     │          Evolución del Módulo          │
                     │            de Planificación            │
                     └───────────────────┬────────────────────┘
                                         │
 ┌──────────────────────┬────────────────┼──────────────────────┬──────────────────────┐
 │                      │                │                      │                      │
 ▼                      ▼                ▼                      ▼                      ▼
Task Decomposition   Multi-Plan      External Planners      Reflection &amp;          Memory-Augmented
(Divide &amp; Conquer)    Selection      (PDDL / Solvers)        Refinement               Planning
 [41, 42]          [43, 44]         [33, 45]          [33, 46]             [33, 34]

```

* **Propuestas de expansión**:
  1. **Descomposición Jerárquica (** **Task Decomposition** **)**: Evolucionar desde la clasificación puntual hacia enfoques *Decomposition-First* (como *Plan-and-Solve* o *HuggingGPT*) o descomposiciones intercaladas (*Interleaved* / *ReAct*) que desglosen peticiones complejas en un grafo de sub-objetivos antes de aceptar la tarea[36].
  2. **Búsqueda y Selección Multi-Ruta (** **Multi-Plan Selection** **)**: Implementar estructuras de pensamiento en árbol (*Tree of Thoughts* \- ToT) o en grafo (*Graph of Thoughts* \- GoT) para explorar múltiples alternativas de inversión o análisis financiero y aplicar algoritmos de búsqueda (MCTS, BFS) para seleccionar la ruta óptima antes de comprometerse[36].
  3. **Planificadores Simbólicos Externos (** **External Planners** **)**: Conectar el LLM con resolvedores lógicos formales (como lenguajes PDDL o ASP en sistemas *LLM+P*) para verificar matemáticamente que las condiciones temporales y restricciones de cartera son realizables antes de guardar el compromiso[33].

---

### 4\. **Módulo de Ejecución y Autocorrección (** **Execution &amp; Refinement** **)**

* **Diseño actual**: El escenario en segundo plano `Cumplir` se ejecuta cada 30 minutos aplicando la arquitectura **Reflexion** (Shinn et al., 2023)[11]:
  * **(15) Actuar (** **Actor** **)**: Ejecuta un único paso invocando una herramienta (`cotizar`, `calcular`, `buscar`, `consultar_cartera`, `enviar`)[11].
  * **(16) Observar (** **Evaluator** **)**: Inspecciona la respuesta obtenida del entorno o servicio externo[11].
  * **(17) Refinar (** **Self-Reflection** **)**: Genera una instrucción verbal de autocorrección si la herramienta devuelve un fallo[11].
* **Fundamentación teórica**: El ciclo de refinamiento mediante retroalimentación verbal actúa como un aprendizaje por refuerzo cualitativo en el espacio de texto, donde el agente ajusta sus salidas sin modificar los pesos del modelo[33].
* **Propuestas de expansión**:
  1. **Supervisión de Trazas de Razonamiento (** **CoT Monitoring** **)**: Al utilizar modelos de razonamiento (como DeepSeek-R1 u o3-mini), es fundamental auditar la traza interna (*Chain-of-Thought*) para verificar que las explicaciones intermedias coincidan con las acciones ejecutadas, previniendo justificaciones engañosas o sesgos de alineación emergente[54].
  2. **Banco de Reflexiones Persistente (** **Long-Term Reflexion Pool** **)**: Guardar las instrucciones verbales de corrección generadas por el escenario *Refinar* en la base de datos de largo plazo de Airtable. De este modo, ante fallos recurrentes de cotización o búsqueda, el agente consultará sus errores pasados durante el paso *Actuar* y no repetirá el mismo intento fallido[31].

---

### 5\. **Orquestación y Arquitectura Multi-Agente (** **Multi-Agent Systems** **)**

* **Diseño actual**: SISTEMA Monolítico/Centralizado basado en 18 escenarios de Make.com organizados de forma secuencial mediante *webhooks*[11][59].
* **Fundamentación teórica**: Los sistemas multi-agente basados en LLM se estructuran según cuatro patrones de comunicación: *Jerárquico/Layered*, *Descentralizado*, *Centralizado* y *Pool de Mensajes Compartido* (*Shared Message Pool*)[59].

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Estructuras de Comunicación                      │
├───────────────────┬──────────────────┬─────────────────┬───────────────┤
│   Hierarchical    │   Decentralized  │   Centralized   │ Shared Message│
│     (Layered)     │   (Peer-to-Peer) │   (Coordinator) │      Pool     │
│       [61]       │       [61]      │      [61]      │     [63]     │
└───────────────────┴──────────────────┴─────────────────┴───────────────┘

```

* **Propuestas de expansión**:
  1. **Implementación de un** **Shared Message Pool** **(estilo MetaGPT)**: Sustituir las llamadas circulares entre escenarios por un bus central de eventos donde distintos sub-agentes especializados (p. ej., *Agente Analista Macro*, *Agente de Gestión de Riesgos*, *Agente de Ejecución*) publiquen y suscriban mensajes según sus perfiles[61][62].
  2. **Mecanismos de Debate Multilateral (** **Multi-Agent Debate** **)**: Introducir discusiones entre un agente proponente de inversiones y un agente auditor de riesgos antes de emitir recomendaciones financieras al usuario, lo que incrementa la factibilidad de los análisis y reduce las alucinaciones[63][64].
