# LangGraph 101

[Los LLMs](https://python.langchain.com/docs/concepts/chat_models/) hacen posible integrar inteligencia en una nueva clase de aplicaciones. [LangGraph](https://langchain-ai.github.io/langgraph/) es un framework para ayudar a construir aplicaciones con LLMs. Aquí, revisaremos los conceptos básicos de LangGraph, explicaremos sus beneficios, mostraremos cómo usarlo para construir flujos de trabajo / agentes, y mostraremos cómo funciona con [LangChain](https://www.langchain.com/) / [LangSmith](https://docs.smith.langchain.com/).

![ecosystem](./img/ecosystem.png)

## Modelos de chat

[Los modelos de chat](https://python.langchain.com/docs/concepts/chat_models/) son la base de las aplicaciones LLM. Típicamente se accede a ellos a través de una interfaz de chat que toma una lista de [mensajes](https://python.langchain.com/docs/concepts/messages/) como entrada y devuelve un [mensaje](https://python.langchain.com/docs/concepts/messages/) como salida. LangChain proporciona [una interfaz estandarizada para modelos de chat](https://python.langchain.com/api_reference/langchain/chat_models/langchain.chat_models.base.init_chat_model.html), facilitando el [acceso a muchos proveedores diferentes](https://python.langchain.com/docs/integrations/chat/).

## Ejecutando el modelo

La interfaz `init_chat_model` proporciona métodos [estandarizados](https://python.langchain.com/docs/concepts/runnables/) para usar modelos de chat, que incluyen:
- `invoke()`: Una sola entrada se transforma en una salida.
- `stream()`: Las salidas se [transmiten en streaming](https://python.langchain.com/docs/concepts/streaming/#stream-and-astream) a medida que se producen.

## Herramientas

[Las herramientas](https://python.langchain.com/docs/concepts/tools/) son utilidades que pueden ser llamadas por un modelo de chat. En LangChain, crear herramientas se puede hacer usando el decorador `@tool`, que transforma funciones de Python en herramientas llamables. Automáticamente inferirá el nombre de la herramienta, descripción y argumentos esperados desde la definición de la función. También puedes usar [servidores del Protocolo de Contexto del Modelo (MCP)](https://github.com/langchain-ai/langchain-mcp-adapters) como herramientas compatibles con LangChain.

## Llamadas a herramientas

Las herramientas pueden ser [llamadas](https://python.langchain.com/docs/concepts/tool_calling/) por LLMs. Cuando una herramienta está vinculada al modelo, el modelo puede elegir llamar la herramienta devolviendo una salida estructurada con argumentos de la herramienta. Usamos el método `bind_tools` para aumentar un LLM con herramientas.

![tool-img](img/tool_call_detail.png)

Los proveedores a menudo tienen [parámetros como `tool_choice`](https://python.langchain.com/docs/how_to/tool_choice/) para forzar la llamada a herramientas específicas. `any` seleccionará al menos una de las herramientas.

Además, podemos [establecer `parallel_tool_calls=False`](https://python.langchain.com/docs/how_to/tool_calling_parallel/) para asegurar que el modelo solo llame una herramienta a la vez.

Podemos usar tool_choice para forzar la llamada a herramientas específicas. Por ejemplo, podemos establecer `tool_choice="any"` para seleccionar al menos una de las herramientas.

![basic_prompt](img/tool_call.png)

## Flujos de trabajo

Hay muchos patrones para construir aplicaciones con LLMs.

[Podemos integrar llamadas LLM en flujos de trabajo predefinidos](https://langchain-ai.github.io/langgraph/tutorials/workflows/), dando al sistema más agencia para tomar decisiones.

Como ejemplo, podríamos agregar un paso de enrutamiento para determinar si escribir un email o no.

![workflow_example](img/workflow_example.png)

## Agentes

Podemos aumentar aún más la agencia, permitiendo que el LLM dirija dinámicamente su propio uso de herramientas.

[Los agentes](https://langchain-ai.github.io/langgraph/tutorials/workflows/) típicamente se implementan como llamadas a herramientas en un bucle, donde la salida de cada llamada a herramienta se usa para informar la siguiente acción.

![agent_example](img/agent_example.png)

Los agentes son adecuados para problemas abiertos donde es difícil predecir los pasos *exactos* necesarios de antemano.

Los flujos de trabajo a menudo son apropiados cuando el flujo de control se puede definir fácilmente de antemano.

![workflow_v_agent](img/workflow_v_agent.png)

## ¿Qué es LangGraph?

[LangGraph](https://langchain-ai.github.io/langgraph/concepts/high_level/) proporciona infraestructura de soporte de bajo nivel que se encuentra debajo de *cualquier* flujo de trabajo o agente.

No abstrae prompts o arquitectura, y proporciona algunos beneficios:

- **Control**: Hace fácil definir y/o combinar agentes y flujos de trabajo.
- **Persistencia**: Proporciona una forma de persistir el estado de un grafo, lo que permite tanto memoria como humano-en-el-bucle.
- **Pruebas, Depuración y Despliegue**: Proporciona una entrada fácil para probar, depurar y desplegar aplicaciones.

### Control

LangGraph te permite definir tu aplicación como un grafo con:

1. *Estado*: ¿Qué información necesitamos rastrear durante el curso de la aplicación?
2. *Nodos*: ¿Cómo queremos actualizar esta información durante el curso de la aplicación?
3. *Aristas*: ¿Cómo queremos conectar estos nodos juntos?

Podemos usar la [clase `StateGraph`](https://langchain-ai.github.io/langgraph/concepts/low_level/#graphs) para inicializar un grafo LangGraph con un [objeto `State`](https://langchain-ai.github.io/langgraph/concepts/low_level/#state).

`State` define el esquema para la información que queremos rastrear durante el curso de la aplicación.

Esto puede ser cualquier objeto con `getattr()` en python, como un diccionario, dataclass, u objeto Pydantic:

- TypeDict es más rápido pero no soporta valores por defecto
- Dataclass es básicamente tan rápido, soporta sintaxis de punto `state.foo`, y tiene valores por defecto.
- Pydantic es más lento (especialmente con validadores personalizados) pero da validación de tipos.

Cada nodo es simplemente una función de python o código typescript. Esto nos da control total sobre la lógica dentro de cada nodo.

Reciben el estado actual y devuelven un diccionario para actualizar el estado.

Por defecto, [las claves de estado se sobrescriben](https://langchain-ai.github.io/langgraph/how-tos/state-reducers/).

Sin embargo, puedes [definir lógica de actualización personalizada](https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers).

![nodes_edges](img/nodes_edges.png)

Las aristas conectan nodos juntos.

Especificamos el flujo de control agregando aristas y nodos a nuestro grafo de estado.

El enrutamiento entre nodos se puede hacer [condicionalmente](https://langchain-ai.github.io/langgraph/concepts/low_level/#conditional-edges) usando una función simple.

El valor de retorno de esta función se usa como el nombre del nodo (o lista de nodos) al que enviar el estado a continuación.

Opcionalmente puedes proporcionar un diccionario que mapee la salida `should_continue` al nombre del siguiente nodo.

Con estos componentes de bajo nivel, puedes construir muchos flujos de trabajo y agentes diferentes. ¡Mira [este tutorial](https://langchain-ai.github.io/langgraph/tutorials/workflows/)!

Debido a que los agentes son un patrón tan común, [LangGraph](https://langchain-ai.github.io/langgraph/tutorials/workflows/#pre-built) tiene [una abstracción de agente pre-construida](https://langchain-ai.github.io/langgraph/agents/overview/?ref=blog.langchain.dev#what-is-an-agent).

Con el [método pre-construido de LangGraph](https://langchain-ai.github.io/langgraph/tutorials/workflows/#pre-built), solo pasamos el LLM, herramientas y prompt.

### Persistencia

#### Hilos

Puede ser muy útil permitir que los agentes se pausen durante tareas de larga duración.

LangGraph tiene una capa de persistencia integrada, implementada a través de checkpointers, para habilitar esto.

Cuando compilas un grafo con un checkpointer, el checkpointer guarda un [checkpoint](https://langchain-ai.github.io/langgraph/concepts/persistence/#checkpoints) del estado del grafo en cada paso.

Los checkpoints se guardan en un hilo, que puede ser accedido después de que se complete la ejecución del grafo.

![checkpointer](img/checkpoints.png)

Compilamos el grafo con un [checkpointer](https://langchain-ai.github.io/langgraph/concepts/persistence/#checkpointer-libraries).

#### Interrupciones

En LangGraph, también podemos usar [interrupciones](https://langchain-ai.github.io/langgraph/how-tos/human_in_the_loop/wait-user-input/) para detener la ejecución del grafo en puntos específicos.

A menudo esto se usa para recopilar entrada de un usuario y continuar la ejecución con la entrada recopilada.

Para reanudar desde una interrupción, podemos usar [el objeto `Command`](https://langchain-ai.github.io/langgraph/how-tos/command/).

Lo usaremos para reanudar el grafo desde el estado interrumpido, pasando el valor a devolver desde la llamada de interrupción a `resume`.

### Trazado

Cuando estamos usando LangChain o LangGraph, el registro de LangSmith [funcionará automáticamente](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langgraph) con las siguientes variables de entorno configuradas:

```
export LANGSMITH_TRACING=true
export LANGSMITH_API_KEY="<tu-langsmith-api-key>"
```

Aquí está la traza de LangSmith de la ejecución del agente anterior:

https://smith.langchain.com/public/6f77014f-d054-44ed-aa2c-8b06ceab689f/r

Podemos ver que el agente puede continuar la conversación desde el estado anterior porque usamos un checkpointer.

### Despliegue

También podemos desplegar nuestro grafo usando [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/).

Esto crea un servidor [con una API](https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html) que podemos usar para interactuar con nuestro grafo y un IDE interactivo, LangGraph [Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/).

Simplemente necesitamos asegurar que nuestro proyecto tenga [una estructura](https://langchain-ai.github.io/langgraph/concepts/application_structure/) como esta:

```
my-app/
├── src/email_assistant # todo el código del proyecto está aquí
│   └── langgraph101.py # código para construir tu grafo
├── .env # variables de entorno
├── langgraph.json  # archivo de configuración para LangGraph
└── pyproject.toml # dependencias para tu proyecto
```

El archivo `langgraph.json` especifica las dependencias, grafos, variables de entorno y otras configuraciones requeridas para iniciar un servidor LangGraph.

Para probar esto, vamos a desplegar `langgraph_101.py`. Lo tenemos en nuestro archivo `langgraph.json` en este repo:

```
 "langgraph101": "./src/email_assistant/langgraph_101.py:app",
```

Para LangGraph Platform, hay una gama de [opciones de despliegue](https://langchain-ai.github.io/langgraph/tutorials/deployment/):

* Los despliegues locales se pueden iniciar con `langgraph dev` desde el directorio raíz del repo. Los checkpoints se guardan en el sistema de archivos local.
* También hay varias [opciones auto-hospedadas](https://langchain-ai.github.io/langgraph/tutorials/deployment/#other-deployment-options).
* Para despliegues hospedados, los checkpoints se guardan en Postgres usando un [checkpointer](https://langchain-ai.github.io/langgraph/concepts/persistence/#checkpointer-libraries) de postgres.

Prueba:
```
Draft a response to my boss (boss@company.ai) confirming that I want to attend Interrupt!
```

Aquí podemos ver una visualización del grafo así como el estado del grafo en Studio.

![langgraph_studio](img/langgraph_studio.png)

También, puedes ver la documentación de la API para el despliegue local aquí:

http://127.0.0.1:2024/docs ****