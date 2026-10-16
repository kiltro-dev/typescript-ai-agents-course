# 01 — Getting Started

> Configuración del entorno y ejecución de los ejemplos.
> Fuente: curso [AI Agents in Typescript/Javascript with Generative AI](https://www.coursera.org/learn/ai-agents-javascript-typescript) — *"Running the Code in the Course"*.
> Repositorio oficial: [github.com/juleswhite/typescript-ai-agents-course](https://github.com/juleswhite/typescript-ai-agents-course)

---

Para empezar a construir agentes, necesitamos entender cómo enviar prompts a los LLM. Los agentes requieren dos capacidades clave que veremos en esta guía:

1. **Envío programático de prompts** — automatizar el ciclo prompt-respuesta que los humanos hacemos manualmente en una conversación. Esto forma la base del *Agent Loop*.
2. **Gestión de memoria** — controlar qué información persiste entre iteraciones (como llamadas a APIs y sus resultados).

## Cómo obtener el código

Todos los materiales del curso están disponibles en GitHub. Empieza clonando el repositorio en tu equipo local:

| Paso | Comando |
|---|---|
| 1 | `git clone https://github.com/juleswhite/typescript-ai-agents-course.git` |
| 2 | `cd typescript-ai-agents-course` |

Esto crea una copia local de todos los ejemplos de código organizados por módulos, junto con las utilidades compartidas que utilizarás a lo largo del curso.

## Requisitos previos

| Requisito | Notas |
|---|---|
| Node.js | Se recomienda la versión 18 o posterior |
| npm | Se incluye con Node.js |
| API key de OpenAI | Necesaria para ejecutar los ejemplos del agente |

Si no tienes instalado Node.js, descárgalo desde [nodejs.org](https://nodejs.org). El gestor de paquetes npm se incluye automáticamente. También necesitarás una API key de OpenAI, que puedes obtener en [platform.openai.com](https://platform.openai.com/api-keys).

> **Nota de este repo:** la API key no tiene por qué ser de OpenAI. Cualquier proveedor compatible con la API de OpenAI funciona (p. ej. DeepSeek) configurando `BASE_URL` en el `.env`. Ver [Guía del repo — Cómo probarlo](GUIA-DEL-REPO.md#2-cómo-probarlo).

## Configuración inicial

Una vez clonado el repositorio, instala las dependencias y configura tus credenciales:

| Paso | Comando | Descripción |
|---|---|---|
| 1 | `npm install` | Instalar dependencias |
| 2 | Crear el archivo `.env` | Ver configuración del entorno más abajo |
| 3 | `npm run build` | Comprobar la compilación de TypeScript |

### Configuración del entorno (`.env`)

Crea un archivo llamado `.env` en el directorio raíz del proyecto. Este archivo almacena tus credenciales de la API de forma segura. Hay un archivo de ejemplo llamado `.env.example` que puedes copiar:

```bash
cp .env.example .env
```

A continuación, actualiza estas dos líneas en tu `.env`:

| Variable | Valor |
|---|---|
| `OPENAI_API_KEY` | `tu-clave-de-api-aquí` |
| `OPENAI_MODEL` | `gpt-5-nano` |

Sustituye `tu-clave-de-api-aquí` por tu API key real. La configuración del modelo determina qué modelo de lenguaje utilizarán tus agentes.

> **En este repo** también puedes configurar `BASE_URL` (endpoint de la API) y `DEBUG=true` (logging detallado).

## Estructura del proyecto

El código está organizado en módulos que se corresponden con el avance del curso. Cada módulo se basa en conceptos de los anteriores:

| Carpeta | Contenido |
|---|---|
| `src/module1/` | Prompting básico y cuasi-agentes |
| `src/module2/` | Function calling y agent loops |
| `src/module3/` | Framework GAME y GAMEAgent |
| `src/module4/` | Self-prompting y patrones avanzados |
| `src/shared/` | Utilidades core (LLM, Tool, Agent, etc.) |
| `.env` | Tu configuración de la API |
| `package.json` | Dependencias del proyecto |
| `tsconfig.json` | Configuración de TypeScript |

La carpeta `shared/` contiene componentes reutilizables que sirven de base para todos los ejemplos. A medida que avances, te irás familiarizando con estos elementos básicos.

## Ejecutar tus primeros ejemplos

La forma más sencilla de ejecutar los ejemplos de TypeScript es usando `ts-node`, que ejecuta los archivos TypeScript directamente sin un paso de compilación independiente:

| Ejemplo | Comando |
|---|---|
| Prompting programático | `npx ts-node src/module1/ProgrammaticPrompting.ts` |
| Cuasi-agente | `npx ts-node src/module1/QuasiAgent.ts` |
| Function calling | `npx ts-node src/module2/FunctionCallingExample.ts` |
| Agente GAME | `npx ts-node src/module3/GAMEAgent.ts` |

> **Atajo en este repo:** cada ejemplo tiene su script npm — `npm run module1:prompting`, `module1:quasi`, `module2:function-calling`, `module3:game`, etc. (ver [Guía del repo](GUIA-DEL-REPO.md#2-cómo-probarlo)).

Si prefieres compilar primero y ejecutar el JavaScript resultante:

| Paso | Comando |
|---|---|
| 1 | `npm run build` |
| 2 | `node dist/module1/ProgrammaticPrompting.js` |

## Ejemplos por sección del curso

| Sección | Archivo clave |
|---|---|
| Introducción al prompting | `src/module1/ProgrammaticPrompting.ts` |
| Creación de cuasi-agentes | `src/module1/QuasiAgent.ts` |
| Tu primer agente real | `src/module1/AgentLoop.ts` |
| Function calling | `src/module2/FunctionCallingExample.ts` |
| El framework GAME | `src/module3/GAMEAgent.ts` |
| Patrones de tools | `src/shared/defineTool.ts`, `src/shared/ToolRegistry.ts` |
| Self-prompting | `src/module4/SelfPromptingAgent.ts` |

## Solución de problemas comunes

| Problema | Solución |
|---|---|
| Errores de "Cannot find module" | `npm install` |
| Problemas con la API key | Asegúrate de que `.env` existe y contiene una `OPENAI_API_KEY` válida |
| Errores de compilación de TypeScript | `npm run typecheck` (o `npx tsc --noEmit`) |

La mayoría de los problemas se deben a dependencias o configuraciones faltantes. En caso de duda, vuelve a ejecutar `npm install` y revisa bien tu archivo `.env`.

---

**Siguiente:** [02 — Messages & Programmatic Prompting](02-messages-and-prompting.md)
