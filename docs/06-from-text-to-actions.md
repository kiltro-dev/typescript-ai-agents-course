# 06 — From Text to Actions: Acciones ejecutables

> Cómo lograr que un LLM produzca acciones claras, consistentes y ejecutables programáticamente.
> Es el puente entre el quasi-agente ([05](05-quasi-agents.md)) y el agent loop completo.
>
> Fuente: video del curso *AI Agents in Typescript/Javascript with Generative AI* (~14 min) —
> obtención y ejecución de acciones desde modelos generativos (resumen detallado del material).
> Implementación: [`src/module1/AgentLoop.ts`](../src/module1/AgentLoop.ts).
>
> Run: `npm run module1:agent`

---

## El problema central: obtener acciones consistentes y parseables

Los modelos generativos son muy verbosos y **no deterministas**: cada vez que se les consulta pueden responder de formas ligeramente diferentes, con variaciones en formato y contenido. Esta variabilidad dificulta que un sistema externo pueda interpretar automáticamente qué acción debe ejecutar el agente.

El agente, en este contexto, **no puede ejecutar acciones directamente** en el sistema; solo puede generar texto. Por lo tanto, el texto generado debe seguir un formato muy estricto y predecible para que un software tradicional pueda analizarlo (parsearlo) y ejecutar la acción correspondiente. Esto implica diseñar prompts que guíen al LLM para producir respuestas en un formato específico, **minimizando la "charla" o información irrelevante** que no es útil para la ejecución.

Esto se traduce en un ciclo autónomo donde el agente:

1. Recibe un problema o tarea.
2. Genera una acción en un formato específico.
3. El sistema ejecuta esa acción.
4. Se recibe feedback o resultado de la acción.
5. El feedback se retroalimenta al agente para decidir la siguiente acción.

Este ciclo se repite continuamente, permitiendo que el agente actúe de forma autónoma y adaptativa.

## Método 1: Ingeniería de prompts y parsing manual

El primer método para lograr esta salida consistente es diseñar cuidadosamente el prompt para que el modelo entienda **qué acciones puede tomar** y **cómo debe formatear su respuesta**.

Por ejemplo, se puede definir un conjunto limitado de acciones (`pickup`, `use`, `discard`) y objetos (`pan`, `butter`, `green bean`). El prompt instruye al modelo a responder siempre con una plantilla, por ejemplo:

```
Acción:Objeto
```

Así, si se le pide al agente que cocine ejotes sabrosos, el modelo podría responder:

```
Pickup:pan
```

Esto es mucho más fácil de parsear y ejecutar que una receta larga y narrativa.

**Espacio para el razonamiento.** Se puede incluir un espacio para que el modelo explique su razonamiento antes de emitir la acción — separando la parte "charlatana" de la acción concreta. Esto ayuda a mantener la salida estructura y facilita la interpretación.

**Flexibilidad y coste.** Este método funciona con **cualquier LLM**, ofrece control total y es útil cuando el modelo no soporta capacidades especiales como *function calling*. El precio: requiere un diseño cuidadoso del prompt y un parser robusto para manejar variaciones menores en la salida (mayúsculas, formato).

### Ciclo con feedback

El video simula un ciclo donde el agente emite una acción, el sistema la ejecuta y devuelve un resultado — por ejemplo, al intentar usar la sartén, el sistema responde `"handle breaks off pan"` — y el agente responde con la siguiente acción (`discard the pan`). Cada interacción se añade al **historial de la conversación**, que el modelo utiliza como contexto para decidir la siguiente acción. Esto simula memoria, ya que el LLM no tiene memoria persistente por sí mismo (ver [04 — Memory](04-memory-and-conversation-history.md)).

## Método 2: Function calling

El segundo método aprovecha capacidades especiales de algunos LLMs modernos que soportan *function calling*: el modelo no solo genera texto, sino que puede devolver directamente una **estructura de datos (JSON)** que indica qué función llamar y con qué argumentos.

Esto simplifica enormemente la interpretación — la salida es consistente y estructurada, eliminando la necesidad de diseñar prompts complejos para obtener un formato específico. Sin embargo:

- Depende de que el LLM soporte esta funcionalidad (no es adecuado para todos los casos).
- No elimina la necesidad de un buen prompt.
- No elimina la posibilidad de errores en la salida.

## Ejemplos prácticos y consideraciones

El video muestra cómo construir un **lenguaje interno** para que el modelo describa acciones programáticas, por ejemplo:

- Llamar funciones para obtener el texto de una página web y luego codificarlo en base64.
- Generar comandos Bash o Python para ejecutar en un entorno — con el **riesgo inherente** que esto implica (ejecutar comandos arbitrarios puede ser peligroso).

Se enfatiza que la interacción con el agente se modela como una **conversación donde cada mensaje incluye todo el historial**, para simular memoria y contexto.

## In this repo

[`AgentLoop.ts`](../src/module1/AgentLoop.ts) implementa el Método 1 completo (text-parsing agent, sin function calling):

| Pieza | Qué hace |
|---|---|
| `AGENT_RULES` | El prompt del sistema: lista los tools disponibles (`printMessage`, `listFiles`, `readFile`, `terminate`) con sus firmas, exige exactamente una acción por respuesta en un bloque ` ```action `, y define cuándo usar `terminate` |
| `parseTextAction()` | El parser: localiza el bloque ` ```action `, extrae el JSON y lo convierte en `{toolName, args}`. Si falla (sin bloque, bloque sin cerrar, JSON inválido), **degrada con gracia** a `terminate` con la respuesta cruda — evita que el agente quede atrapado. Tolera variantes (`toolName` / `tool_name` / `tool`) |
| `executeTextAction()` | Ejecuta la acción parseada vía switch sobre los tools, devolviendo `{success, result}` o `{success: false, error}` |
| `runTextParsingAgentLoop()` | **El loop core** que todo agente sigue: construir prompt (reglas + memoria) → generar respuesta → parsear acción → ejecutar → actualizar memoria (`addAssistant` + `addEnvironment`) → verificar terminación → repetir. Con tope de `MAX_ITERATIONS` |
| `runSimpleAgentLoop()` | Versión procedural mínima (sin clases, un `while` + array de mensajes), más cercana al equivalente en Python |

El formato exacto que pide el prompt:

````
```action
{"toolName": "readFile", "args": {"fileName": "package.json"}}
```
````

Notas del diseño, fieles al material:

- El formato con marcadores del Método 1 (`Acción:Objeto`) aquí es un bloque ` ```action ` con JSON — más robusto de parsear, misma idea: plantilla estricta + parser.
- El espacio para "charlar" antes de la acción: el bloque permite texto de razonamiento alrededor; el parser solo extrae la acción.
- El feedback del entorno (resultado de la tool) entra al historial como mensaje `environment` — el ciclo con feedback del video.
- El **Método 2** (function calling) es el tema de `src/module2/` (`FunctionCallingExample.ts`, `AgentLoopFunctionCalling.ts`).

## Resumen de los dos métodos

| | Método 1: prompts + parsing | Método 2: function calling |
|---|---|---|
| Salida del LLM | Texto con plantilla estricta (JSON en bloque) | JSON estructurado nativo |
| Requisitos del modelo | Cualquier LLM | Modelos con soporte de function calling |
| Control | Total (diseñas prompt y parser) | El formato lo garantiza la API |
| Riesgos | Variaciones en la salida, parser frágil | Errores de la API, buen prompt sigue siendo necesario |
| En este repo | `src/module1/AgentLoop.ts` | `src/module2/` |

## Por qué importa

Diseñar prompts y estructuras para que un modelo generativo produzca acciones claras, consistentes y ejecutables es **la clave para construir agentes autónomos** que puedan interactuar con entornos reales de forma programática y confiable. Los dos enfeques — parsing manual y function calling — tienen sus ventajas y limitaciones; la elección depende del contexto y de las capacidades del LLM utilizado.

---

**Next:** [07 — The Agent Loop](07-the-agent-loop.md)
