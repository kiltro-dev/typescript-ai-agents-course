# Documentación

Documentación de este repo organizada para **comprender el código de arriba abajo**: primero cómo
ejecutarlo, después los conceptos base, y luego los subsistemas del agente. No sigue el orden de
las lecciones del curso; sigue el orden en que conviene leer el repo.

## 📁 Estructura

```
docs/
├── README.md                  ← índice (este archivo)
├── 01-getting-started.md      ← setup, .env, cómo ejecutar los ejemplos
├── 02-messages-and-prompting.md ← Message, roles, llm.generate(), system prompts
├── 03-llm-abstraction.md      ← por qué existe LLM.ts y cómo se configura
├── 04-memory-and-conversation-history.md ← LLMs stateless, historial de conversación
├── 05-quasi-agents.md         ← prompt chaining, manipulación de memoria
├── 06-from-text-to-actions.md ← acciones ejecutables: parsing vs function calling
├── 07-the-agent-loop.md       ← el loop paso a paso, agent rules, orquestación
└── GUIA-DEL-REPO.md           ← referencia técnica: shared/, módulos, framework GAME
```

## 🗺️ Ruta de lectura

| # | Documento | Qué cubre |
|---|---|---|
| 01 | [Getting Started](01-getting-started.md) | Requisitos, clonar, `.env`, scripts npm, estructura del proyecto, troubleshooting |
| 02 | [Messages & Programmatic Prompting](02-messages-and-prompting.md) | Unidades `Message` y roles, `LLM.generate()`, system messages para "programar" al agente, JSON como entrada y salida |
| 03 | [The LLM Abstraction](03-llm-abstraction.md) | Capa `LLM`: independencia de proveedor, configuración (`BASE_URL`, `OPENAI_MODEL`), contrato de `generate()` |
| 04 | [Memory & Conversation History](04-memory-and-conversation-history.md) | LLMs sin memoria, rehistorial con roles `assistant`, control de qué recuerda el modelo |
| 05 | [Quasi-Agents](05-quasi-agents.md) | Ejercicio + solución: prompt chaining (código → docs → tests), manipulación de memoria, code review multi-turno, por qué aún no es un agente |
| 06 | [From Text to Actions](06-from-text-to-actions.md) | Acciones ejecutables desde un LLM: método 1 (prompts + parsing), método 2 (function calling), el ciclo con feedback |
| 07 | [The Agent Loop](07-the-agent-loop.md) | El loop paso a paso (prompt → generar → parsear → ejecutar → memoria), agent rules, orquestación vs. ejecución |
| — | *(continuará)* | Tools & function calling en profundidad · Framework GAME |

Cuando el camino llegue a los conceptos de agente, la referencia completa de cada pieza ya está en
la guía técnica:

## 📘 Referencia técnica

**[GUIA-DEL-REPO.md](GUIA-DEL-REPO.md)** — consulta rápida del código de este repo:

- Qué es el repo y el framework GAME (Goals, Actions, Memory, Environment)
- Funcionalidades principales (agent loop, agent languages, tools con Zod)
- La capa `shared/` explicada archivo por archivo
- Guía de cada módulo (`module1`–`module4`) con su correspondencia en el curso

## 🎓 Fuente del curso

**[AI Agents in Typescript/Javascript with Generative AI](https://www.coursera.org/learn/ai-agents-javascript-typescript)** — Vanderbilt University (Dr. Jules White).
Repo oficial: [github.com/juleswhite/typescript-ai-agents-course](https://github.com/juleswhite/typescript-ai-agents-course)

Los materiales que se van incorporando se reordenan aquí por tema (no por lección) y se mantienen
en su idioma original. Cada documento cita su lección de origen.

| Documento | Lección de origen |
|---|---|
| [01 — Getting Started](01-getting-started.md) | Module 1 — "Running the Code in the Course" |
| [02 — Messages & Prompting](02-messages-and-prompting.md) | Module 1 — "Sending Prompts Programmatically & Managing Memory" |
| [03 — LLM Abstraction](03-llm-abstraction.md) | Module 1 — "Sending Prompts Programmatically & Managing Memory" |
| [04 — Memory & Conversation History](04-memory-and-conversation-history.md) | Module 1 — "LLMs Do Not Have Memory" |
| [05 — Quasi-Agents](05-quasi-agents.md) | Module 1 — "Building a Quasi-Agent" + "Possible Solution for Building a Quasi-Agent" |
| [06 — From Text to Actions](06-from-text-to-actions.md) | Video ~14 min — obtención y ejecución de acciones desde modelos generativos |
| [07 — The Agent Loop](07-the-agent-loop.md) | Module 1 — "Building a Simple AI Agent, Parts 1–3" |
