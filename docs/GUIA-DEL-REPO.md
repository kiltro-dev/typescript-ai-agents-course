# Guía del Repo — AI Agents en TypeScript

> Material de apoyo del curso **[AI Agents in Typescript/Javascript with Generative AI](https://www.coursera.org/learn/ai-agents-javascript-typescript)** (Vanderbilt University, Dr. Jules White) — parte de la especialización *AI Agents in Typescript/Javascript*.
>
> Esta guía documenta **este repo en particular**: qué hace, cómo ejecutarlo, qué aporta cada pieza de `src/shared/` y qué se aprende en cada módulo.

---

## Tabla de contenidos

1. [Qué es este repo](#1-qué-es-este-repo)
2. [Cómo probarlo](#2-cómo-probarlo)
3. [Funcionalidades principales](#3-funcionalidades-principales)
4. [La capa `shared/` en detalle](#4-la-capa-shared-en-detalle)
5. [Guía de cada módulo](#5-guía-de-cada-módulo)
6. [Estructura del proyecto](#6-estructura-del-proyecto)
7. [Referencias](#7-referencias)

---

## 1. Qué es este repo

Este repo es la **implementación en TypeScript del framework de agentes del curso**. En lugar de usar un framework de moda (LangChain, etc.), se construye cada componente desde cero para aprender los principios que no cambian:

> *"Frameworks change weekly, but the core principles of agent design remain constant."* — Dr. Jules White

El resultado es un pequeño runtime de agentes con:

- Una **abstracción de LLM** desacoplada del proveedor (OpenAI-compatible).
- Un **loop de agente** (prompt → respuesta → parseo → ejecución → memoria → repetir).
- **Tools/function calling** con validación de tipos vía Zod y JSON Schema automático.
- El **framework GAME** (Goals, Actions, Memory, Environment) con *Agent Languages* intercambiables.

### El framework GAME

| Componente | Qué es | Clase en `shared/` |
|---|---|---|
| **G** — Goals | Qué intenta lograr el agente | `Goal` / `createGoal` |
| **A** — Actions | Tools que puede usar | `Tool`, `defineTool`, `ToolRegistry` |
| **M** — Memory | Contexto y estado de la conversación | `ConversationMemory`, `Memory` |
| **E** — Environment | El mundo donde ejecuta acciones | `Environment` |

Y una pieza transversal que el curso llama **Agent Language**: cómo se formatea el prompt y cómo se parsea la respuesta (lenguaje natural, bloques ```action, o function calling nativo).

---

## 2. Cómo probarlo

### Requisitos

- **Node.js 18+**
- Una API key de un proveedor compatible con la API de OpenAI (OpenAI, DeepSeek, etc.)

### Setup

```bash
# 1. Instalar dependencias
npm install

# 2. Configurar entorno
cp .env.example .env
# Editar .env y poner tu API key
```

### Variables de entorno (`.env`)

| Variable | Requerida | Descripción |
|---|---|---|
| `OPENAI_API_KEY` | ✅ | API key del proveedor (en este repo se usa una de DeepSeek) |
| `BASE_URL` | Opcional | Base URL de la API. Si se omite, se usa `https://api.openai.com/v1` |
| `OPENAI_MODEL` | Opcional | Modelo a usar (default: `gpt-5-nano`) |
| `DEBUG` | Opcional | `true` activa logging detallado del `LLM` |

> **Uso con DeepSeek (u otro proveedor compatible):** basta con setear `BASE_URL` (p. ej. `https://api.deepseek.com`) y un modelo soportado en `OPENAI_MODEL`. El SDK de OpenAI se usa como cliente HTTP contra cualquier endpoint compatible.

### Ejecutar los ejemplos

```bash
npm run module1:prompting        # Prompting básico
npm run module1:quasi            # Quasi-agente con memoria
npm run module1:agent            # Agent loop con parseo de texto
npm run module2:function-calling # Function calling básico
npm run module2:agent            # Agent loop con function calling
npm run module3:game             # Framework GAME completo
npm run module4:self-prompting   # Tools con Zod + filtros por tags
```

### Tests y calidad

```bash
npm test                 # Tests unitarios (Jest)
npm run test:watch       # Tests en watch mode
npm run test:coverage    # Con cobertura
npm run typecheck        # Chequeo de tipos sin emitir
npm run build            # Compilar a JS (dist/)
```

Los tests cubren las piezas de `shared/`: `Action`, `ActionResult`, `Goal`, `Memory`, `Message`, `Tool`.

---

## 3. Funcionalidades principales

### 3.1 Agent loop (el patrón central)

```
┌─────────────────────────────────────────────────────────────┐
│                      THE AGENT LOOP                         │
│                                                             │
│   1. Construir prompt (reglas + memoria + petición)         │
│                           ↓                                 │
│   2. Enviar al LLM, obtener respuesta                       │
│                           ↓                                 │
│   3. Parsear la respuesta → acción (tool + args)            │
│                           ↓                                 │
│   4. Ejecutar la acción en el Environment                   │
│                           ↓                                 │
│   5. Actualizar memoria con el resultado                    │
│                           ↓                                 │
│   6. ¿Terminó? → si no, volver al paso 1                    │
└─────────────────────────────────────────────────────────────┘
```

Terminación: existe un tool `terminal` (convencionalmente `terminate`) que corta el loop. Hay límite de iteraciones (`maxIterations`, default 50) y reintentos de parseo (`maxParseRetries`, default 3).

### 3.2 Tres formas de "hablar" con el LLM (Agent Languages)

| Lenguaje | Cómo decide el agente | Cuándo usarlo |
|---|---|---|
| `NaturalLanguage` | Cualquier respuesta termina con `terminate` | Chat simple sin tools |
| `JsonActionLanguage` | Parsea bloques ```action con JSON | Cualquier LLM (sin function calling) |
| `FunctionCallingLanguage` | Function calling nativo de la API OpenAI | Producción — el más robusto |

El mismo agente GAME funciona con cualquiera de los tres: eso es separar **qué hace el agente** de **cómo se comunica**.

### 3.3 Tools tipadas con Zod

```typescript
const readFileTool = defineTool({
  name: 'readFile',
  description: 'Reads the contents of a file',
  schema: z.object({
    fileName: z.string().describe('The name of the file to read'),
  }),
  tags: ['file_operations', 'read'],
  execute: async ({ fileName }) => fs.readFileSync(fileName, 'utf-8'),
});
```

- Validación de argumentos en runtime (Zod).
- JSON Schema generado automáticamente para la API del LLM.
- **Tags** para filtrar tools (`ToolRegistry({ tags: ['file_operations'] })`).
- Flag `terminal: true` para tools que terminan el loop.
- Registro global (`getGlobalTool`, `getAllGlobalTools`, `getToolNamesByTag`).

### 3.4 Function calling

El LLM devuelve llamadas estructuradas a tools en vez de texto libre: más confiable, sin regex ni extracción de markdown, con argumentos validados contra schema. La clase `LLM` las traduce a un JSON uniforme `{"tool": "...", "args": {...}}`.

---

## 4. La capa `shared/` en detalle

Todas las abstracciones reutilizables viven en [`src/shared/`](../src/shared/) y se exportan desde el barrel [`index.ts`](../src/shared/index.ts).

### Comunicación con el LLM

| Archivo | Responsabilidad |
|---|---|
| [`Message.ts`](../src/shared/Message.ts) | Mensaje con rol (`system`/`user`/`assistant`). Factories `Message.system()`, `.user()`, `.assistant()` + serialización JSON. |
| [`Prompt.ts`](../src/shared/Prompt.ts) | Mensajes + tools + metadata. API inmutable (`withMessage`, `withTools`) y factories (`Prompt.simple`, `Prompt.fromUser`). |
| [`LLM.ts`](../src/shared/LLM.ts) | Wrapper del SDK `openai` desacoplado del proveedor. `generate(messages \| Prompt)` → string. Soporta `apiKey`, `baseURL`, `model`, `maxTokens`, `debug` (config o env). Si hay tools y el modelo llama a una, devuelve `{"tool", "args"}`. |

### Acciones y tools

| Archivo | Responsabilidad |
|---|---|
| [`Tool.ts`](../src/shared/Tool.ts) | Definición de un tool con JSON Schema (`name`, `description`, `parameters`). Factories: `Tool.listFiles()`, `.readFile()`, `.terminate()`. |
| [`Action.ts`](../src/shared/Action.ts) | Invocación parseada de un tool: `toolName` + `args`. Acceso tipado a argumentos (`getArg`, `requireArg`), `isTerminate()`, `Action.terminate()`, `Action.error()`. |
| [`ActionResult.ts`](../src/shared/ActionResult.ts) | Resultado como *discriminated union* (`success` \| `error`). Helpers: `fromPromise`, `fromTry`, type guards. |
| [`defineTool.ts`](../src/shared/defineTool.ts) | Definición de tools con Zod (equivalente al decorador `@register_tool` de Python en el curso). Convierte Zod → JSON Schema, registra globalmente, indexa por tags. Incluye factories `listFilesDefinition`, `readFileDefinition`, `terminateDefinition`. |
| [`ToolRegistry.ts`](../src/shared/ToolRegistry.ts) | Registro filtrable por tags/nombres (equivalente a `PythonActionRegistry`). `getTools()` para el LLM, `execute()` con validación, `isTerminal()`. Factories: `createFileOperationsRegistry()`, `createFullRegistry()`. |

### Memoria y objetivos

| Archivo | Responsabilidad |
|---|---|
| [`Memory.ts`](../src/shared/Memory.ts) | Almacén clave-valor para estado entre iteraciones (`set`/`get`/`append`/`increment`). `toString()` para inyectar en contexto del LLM. |
| [`ConversationMemory.ts`](../src/shared/ConversationMemory.ts) | Historial tipado de conversación (`user`/`assistant`/`environment`/`system`). Filtros, copias, `toMessages()` para el LLM, serialización JSON. Es la memoria del framework GAME. |
| [`Goal.ts`](../src/shared/Goal.ts) | Objetivo con `name`, `description`, `priority` y estado de completado. Utilidades: `sortByPriority`, `nextGoal`, `formatForLLM`. |

### El runtime GAME

| Archivo | Responsabilidad |
|---|---|
| [`AgentLanguage.ts`](../src/shared/AgentLanguage.ts) | Abstracción abstracta + 3 implementaciones (`NaturalLanguage`, `JsonActionLanguage`, `FunctionCallingLanguage`). Define `constructPrompt`, `parseResponse` y `adaptPromptAfterParsingError` (retroalimentación al LLM cuando el parseo falla). |
| [`Environment.ts`](../src/shared/Environment.ts) | Ejecuta acciones vía `ToolRegistry` y envuelve el resultado en un `ActionResultEnvelope` (`toolExecuted`, `result`/`error`, `traceback`, `timestamp`). |
| [`Agent.ts`](../src/shared/Agent.ts) | El orquestador GAME: loop, reintentos de parseo, actualización de memoria, terminación. Incluye `AgentBuilder` (API fluida) y `createSimpleAgent()`. Callbacks observables (`onBeforePrompt`, `onActionExecuted`, `onTerminate`...). |

### Utilidades

| Archivo | Responsabilidad |
|---|---|
| [`FileTools.ts`](../src/shared/FileTools.ts) | Operaciones de FS compartidas: `listFiles`, `readFile`, `writeFile`, `exists`, `getStats`, etc. Versiones "safe" que devuelven `ActionResult`. |
| [`env.ts`](../src/shared/env.ts) | Carga de `.env` (`loadEnv`), `requireEnv`, `getEnv`. |

---

## 5. Guía de cada módulo

Los módulos del repo se alinean con los módulos del curso en Coursera.

### Módulo 1 — Conceptos de IA agéntica

**Archivos:** [`ProgrammaticPrompting.ts`](../src/module1/ProgrammaticPrompting.ts), [`QuasiAgent.ts`](../src/module1/QuasiAgent.ts), [`AgentLoop.ts`](../src/module1/AgentLoop.ts)
**Comandos:** `npm run module1:prompting` · `module1:quasi` · `module1:agent`
**En el curso:** *Agentic AI Concepts* (~4 h) — patrón de *flipped interaction*, agent loop, estructuración de salidas. Labs: programmatic prompting, dar memoria al agente, primer agente, interfaz agente/environment.

| Ejemplo | Qué demuestra |
|---|---|
| **Programmatic Prompting** | La interacción mínima con un LLM: mensajes con roles, request/response, salida estructurada (JSON), efecto del system prompt. Sin memoria, sin tools. |
| **Quasi-Agent** | Un "casi agente": mantiene historial de conversación pero no ejecuta acciones autónomas. Demuestra *prompt chaining*, manipulación de memoria (qué "ve" el LLM) y extracción de bloques de código. |
| **Agent Loop** | El loop completo con **parseo de texto** (bloques ```action): construir prompt → responder → parsear → ejecutar → memoria → repetir. Base antes de function calling. Usa `ConversationMemory`. |

**Conceptos clave:** roles de mensajes, memoria multi-turno, prompt chaining, el loop de agente como patrón fundamental.

---

### Módulo 2 — Tools, Actions & Function Calling

**Archivos:** [`FunctionCallingExample.ts`](../src/module2/FunctionCallingExample.ts), [`AgentLoopFunctionCalling.ts`](../src/module2/AgentLoopFunctionCalling.ts)
**Comandos:** `npm run module2:function-calling` · `module2:agent`
**En el curso:** *AI Agents, Tools, Actions, & Language* (~2 h) — GAIL (Goals, Actions, Information, Language), diseño de tools, resultados de tools. Labs: agent tools, function calling, buenas prácticas de diseño.

| Ejemplo | Qué demuestra |
|---|---|
| **Function Calling** | La API de function calling: tools con JSON Schema (p. ej. `getCurrentWeather`, `searchWeb`) y respuestas estructuradas del LLM. |
| **Agent Loop + Function Calling** | Clase `FunctionCallingAgent`: loop con llamadas estructuradas, registro dinámico de tools (`registerTool`), demos de explorador de archivos y tools custom. |

**Por qué function calling > parseo de texto:** confiabilidad (JSON estructurado), type safety vía schema, cero errores de parsing, el LLM ve el schema completo del tool.

---

### Módulo 3 — Framework GAME

**Archivo:** [`GAMEAgent.ts`](../src/module3/GAMEAgent.ts)
**Comando:** `npm run module3:game`
**En el curso:** *GAME Framework* (~2 h) — overview de GAME y simulación de agentes en ChatGPT. Labs: diseño modular, customización del loop, implementar GAME en código, agent language, agente generador de README.

Demuestra los 4 componentes trabajando juntos:

- **Goals** → `createGoal('Helper', 'Help the user with file operations')`
- **Actions** → tools registrados con `defineTool` + `ToolRegistry`
- **Memory** → `ConversationMemory` a lo largo del loop
- **Environment** → `Environment.executeAction(...)` con envelope de resultado

Y lo más importante: **el mismo agente corre con los 3 Agent Languages** (`NaturalLanguage`, `JsonActionLanguage`, `FunctionCallingLanguage`), probando que separar el *qué* del *cómo* funciona.

---

### Módulo 4 — Tools con Zod y filtrado por tags

**Archivo:** [`SelfPromptingAgent.ts`](../src/module4/SelfPromptingAgent.ts)
**Comando:** `npm run module4:self-prompting`
**En el curso:** patrón equivalente al decorador `@register_tool` de Python; complementa las labs de diseño de tools del Módulo 2/3.

Demuestra el approach idiomático de TypeScript:

1. **Definición con Zod** — `defineTool({ name, description, schema, tags, terminal, execute })`.
2. **Registro global** — descubrimiento de tools (`getAllGlobalTools`, `getToolNamesByTag`).
3. **Filtrado por tags** — registries especializados: `file_operations` vs `system`.
4. **Validación Zod** — errores claros cuando los args no cumplen el schema.
5. **Agente con tools filtrados** — un agente GAME que solo ve el subconjunto de tools que le corresponde.

Tools incluidos en la demo: `listFiles`, `readFile`, `writeFile`, `countLines`, `findPattern`, `printMessage`, `terminate`.

---

## 6. Estructura del proyecto

```
typescript-ai-agents-course/
├── docs/
│   └── GUIA-DEL-REPO.md      ← este documento
├── src/
│   ├── shared/               # Abstracciones reutilizables (ver sección 4)
│   │   ├── LLM.ts            #   Wrapper del LLM multi-proveedor
│   │   ├── Message.ts        #   Mensajes con roles
│   │   ├── Prompt.ts         #   Mensajes + tools + metadata
│   │   ├── Tool.ts           #   Tools con JSON Schema
│   │   ├── Action.ts         #   Invocaciones parseadas
│   │   ├── ActionResult.ts   #   Resultados success/error
│   │   ├── defineTool.ts     #   Tools con Zod + registro global
│   │   ├── ToolRegistry.ts   #   Registro filtrable por tags
│   │   ├── Memory.ts         #   Memoria clave-valor
│   │   ├── ConversationMemory.ts  # Historial tipado (GAME)
│   │   ├── Goal.ts           #   Objetivos (GAME)
│   │   ├── Environment.ts    #   Ejecución de acciones (GAME)
│   │   ├── AgentLanguage.ts  #   Cómo comunica el agente (3 lenguajes)
│   │   ├── Agent.ts          #   El loop GAME + builder
│   │   ├── FileTools.ts      #   Operaciones de archivos
│   │   └── env.ts            #   Carga de .env
│   │
│   ├── module1/              # Conceptos básicos: prompting → quasi-agente → loop
│   ├── module2/              # Function calling y agent loop estructurado
│   ├── module3/              # Framework GAME completo
│   └── module4/              # Zod tools + tag filtering
│
├── .env.example              # Plantilla de configuración
├── package.json              # Scripts de ejecución
├── tsconfig.json
└── jest.config.js
```

---

## 7. Referencias

- **Índice de documentación:** [docs/README.md](README.md) — ruta de lectura por temas (getting started → mensajes → LLM → …).
- **Curso:** [AI Agents in Typescript/Javascript with Generative AI](https://www.coursera.org/learn/ai-agents-javascript-typescript) — Vanderbilt University (Dr. Jules White), ~9 h, 4 módulos.
- **Especialización:** *AI Agents in Typescript/Javascript* (Coursera).
- **Repo oficial del curso:** [github.com/juleswhite/typescript-ai-agents-course](https://github.com/juleswhite/typescript-ai-agents-course)
- **Notas por tema:** [01 — Getting Started](01-getting-started.md), [02 — Messages & Prompting](02-messages-and-prompting.md), [03 — LLM Abstraction](03-llm-abstraction.md).
- **Módulo 4 del curso (teórico):** *Rethinking How Software is Built in the Age of AI Agents* — innovación, alucinación como "nueva forma de computación", nuevas formas de extraer información.

### Notas de implementación de este repo

- El cliente es el SDK oficial `openai`, pero funciona contra cualquier API compatible configurando `BASE_URL` (en este repo se usa DeepSeek).
- Si `BASE_URL` no está definida, se usa el default de OpenAI (`https://api.openai.com/v1`) sin configuración extra.
- La correspondencia con el curso en Python: `defineTool` ≈ `@register_tool`, `ToolRegistry` ≈ `PythonActionRegistry`, `AgentLanguage` ≈ la abstracción homónima.
