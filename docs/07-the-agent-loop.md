# 07 — The Agent Loop: Building a Simple AI Agent

> Now that we understand how to extract actions from an LLM ([06](06-from-text-to-actions.md)),
> we can build a simple AI agent: it lists files in a directory, reads their content, and answers
> questions about them. This chapter breaks down the agent loop step by step — how it receives
> input, decides on actions, executes them, and updates its memory.
>
> Source: course *AI Agents in Typescript/Javascript with Generative AI* — "Building a Simple AI
> Agent, Parts 1–3". Implementation: [`src/module1/AgentLoop.ts`](../src/module1/AgentLoop.ts).
>
> Run: `npm run module1:agent`

---

## The agent loop

The agent loop is the backbone of our AI agent: it performs tasks by combining response generation, action execution, and memory updates in an iterative process. It is what makes the agent dynamic and adaptive.

1. **Construct prompt** — combine the agent's memory, user input, and system rules into a single prompt. This ensures the LLM has all the context it needs to decide the next action, maintaining continuity across iterations.
2. **Generate response** — send the constructed prompt to the LLM and retrieve a response. This response guides the agent's next step, providing instructions in a structured format.
3. **Parse response** — extract the intended action and its parameters from the LLM's output. The response must adhere to a predefined structure (e.g. JSON) so it can be interpreted correctly.
4. **Execute action** — use the extracted action and its parameters to perform the requested task with the appropriate tool: listing files, reading content, printing a message…
5. **Convert result to string** — format the result of the executed action into a string, so the agent can store it in its memory and provide clear feedback to the user or itself.
6. **Continue loop?** — evaluate whether the loop should continue based on the current action and results. It may terminate if a `terminate` action is specified or the agent has completed the task.

The agent iterates through this loop, refining its behavior and adapting its actions until it reaches a stopping condition.

### How the steps come together in code

```typescript
// The Agent Loop
while (iterations < maxIterations) {

    // 1. Construct prompt: Combine agent rules with memory
    const prompt = [Message.system(agentRules), ...memory];

    // 2. Generate response from LLM
    console.log("Agent thinking...");
    const response = await llm.generate(prompt);
    console.log(`Agent response: ${response}`);

    // 3. Parse response to determine action
    const action = parseAction(response);

    let result = { result: "Action executed" };

    if (action.toolName === "listFiles") {
        result = { result: listFiles() };
    } else if (action.toolName === "readFile") {
        result = { result: readFile(action.args.fileName) };
    } else if (action.toolName === "error") {
        result = { error: action.args.message };
    } else if (action.toolName === "terminate") {
        console.log(action.args.message);
        break;
    } else {
        result = { error: "Unknown action: " + action.toolName };
    }

    console.log(`Action result: ${JSON.stringify(result)}`);

    // 5. Update memory with response and results
    memory.push(Message.assistant(response));
    memory.push(Message.user(JSON.stringify(result)));

    // 6. Check termination condition
    if (action.toolName === "terminate") {
        break;
    }

    iterations++;
}
```

## Step 1: constructing the agent prompt

The prompt is created by appending the agent's rules (system message) to the current memory of interactions. Part of the memory is a description of the task the agent should perform. This ensures the agent is always aware of its tools and constraints while remembering past actions.

```typescript
const prompt = [Message.system(agentRules), ...memory];
```

- **`agentRules`** — predefined system instructions, ensuring the agent behaves within its defined constraints and understands its tools.
- **`memory`** — a record of all past interactions: user input, the agent's responses, and the results of executed actions.

By constructing the prompt this way, the agent retains continuity across iterations and adapts its behavior based on previous actions and results. The memory tells it what just happened, what happened in the past, and informs its decision of the next action.

### Agent rules: defining the agent's behavior

Before the agent begins its loop, it must have a clear set of rules defining its behavior, capabilities, and constraints. These rules live in the system message and play a critical role in making the agent interact predictably and within its boundaries.

```typescript
const agentRules = `
You are an AI agent that can perform tasks by using available tools.

Available tools:
- listFiles(): string[]: List all files in the current directory.
- readFile(fileName: string): string: Read the content of a file.
- terminate(message: string): End the agent loop and print a summary to the user.

If a user asks about files, list them before reading.

Every response MUST have an action.
Respond in this format:

\`\`\`action
{
    "toolName": "insert toolName",
    "args": {...fill in any required arguments here...}
}
\`\`\`
`;
```

Three elements:

1. **Role of system messages** — the `system` role establishes the ground rules, so the LLM understands what it can do and how it should behave throughout the session.
2. **Tools description** — the rules explicitly list the tools, providing a structured interface for interacting with the environment.
3. **Output format** — a standardized format (```` ```action {...} ````) makes parsing and executing actions easier and less error-prone.

### The agent orchestrates, the code executes

Each "tool" in the system prompt corresponds to a function in the code. The agent chooses **what** function to execute and when, and decides the parameters provided to those functions.

The agent is **not creating** the functions; it is **orchestrating** their behavior. The logic for how each tool operates is predefined in code; the agent focuses on selecting the right tool for the job and providing the correct input.

Because agents can adapt as the loop progresses, they can dynamically decide which tool to use based on the current context and task. For example, if the user asks to read a specific file, the agent first uses `listFiles` to identify available files, then — based on the result — determines whether to proceed with `readFile` or respond with an error if the file does not exist. Each step is evaluated iteratively, informed by the current state of the environment.

This separation of **reasoning** (LLM) and **execution** (well-defined code) is what makes the agent loop so powerful: a modular, extensible framework that can handle increasingly complex tasks without rewriting the underlying tools.

It also eliminates much of the "glue code" traditionally required to tie functions together. Instead of hardcoding workflows, the agent dynamically decides the sequence of actions — effectively realizing a program on top of its components. The agent can combine its tools in ways that would typically require custom logic, addressing a broader range of use cases without additional development overhead.

**Example in practice.** If the user asks *"What files are here?"*, the agent rules guide the LLM to respond with:

```json
{"toolName": "listFiles", "args": {}}
```

### How `agentRules` integrate with the loop

The rules are combined with memory in step 1 to form the input for the LLM, guaranteeing the agent always has access to its instructions and tools at every iteration. If the user asks *"What files are in this directory?"*, the memory might look like:

```typescript
const memory = [
    Message.user("What files are in this directory?"),
    Message.assistant('```action\n{"toolName":"listFiles","args":{}}\n```'),
    Message.user('["file1.txt", "file2.txt"]')
];
```

Adding `agentRules` ensures the LLM understands what tools it can use to continue interacting.

## Step 2: generate response

After constructing the prompt, the agent sends it to the LLM to receive a response that defines the next action:

```typescript
const response = await llm.generate(prompt);
```

The `llm.generate` function sends the prompt to the LLM and retrieves its response. The response typically includes a structured action that the agent will parse and execute in the next steps. This is where the LLM decides what action the agent should take, based on the provided context and rules.

## Step 3: parse the response

Once the Agent has generated a response, we need to **interface the agent with the environment**. This involves figuring out how the Agent's response corresponds to an action in the environment. Once the correct action is determined, the interface can execute it and later provide the Agent feedback on the result.

The response is expected to follow a predefined structure — a JSON format encapsulated within a markdown code block. This structure ensures the action can be parsed and executed without ambiguity. In the code, this is accomplished by locating and extracting the content between the ` ```action ` markers:

```typescript
function parseAction(response: string): { toolName: string; args: Record<string, any> } {
    // Parse the LLM response into a structured action object
    try {
        const actionBlock = extractMarkdownBlock(response, "action");
        const responseJson = JSON.parse(actionBlock);
        if ("toolName" in responseJson && "args" in responseJson) {
            return responseJson;
        } else {
            return { toolName: "error", args: { message: "You must respond with a JSON tool invocation." } };
        }
    } catch {
        return { toolName: "error", args: { message: "Invalid JSON response. You must respond with a JSON tool invocation." } };
    }
}
```

This step is critical to making the response actionable. It produces a structured output such as:

```json
{
    "toolName": "listFiles",
    "args": {}
}
```

By breaking down the LLM's output into `toolName` and `args`, the agent can precisely determine the next action and its inputs.

**Graceful fallback.** If the response does not include a valid action block, the agent defaults to an error action (in the teaching code; some versions default to a termination action with the raw response as message). The error message appears to have come from the *"user"*, prompting the LLM to provide a valid JSON tool invocation. This fallback ensures the agent can **recover** if it starts outputting invalid responses outside the desired format.

## Step 4: execute the action

Once the response is parsed, the agent uses the extracted `toolName` and `args` to execute the corresponding function. Each predefined tool in the system instructions maps to a specific function in the code, enabling the agent to interact with its environment:

```typescript
if (action.toolName === "listFiles") {
    result = { result: listFiles() };
} else if (action.toolName === "readFile") {
    result = { result: readFile(action.args.fileName) };
} else if (action.toolName === "error") {
    result = { error: action.args.message };
} else if (action.toolName === "terminate") {
    console.log(action.args.message);
    break;
} else {
    result = { error: "Unknown action: " + action.toolName };
}
```

For example, if the action specifies `toolName: "listFiles"` with empty args, the `listFiles()` function is called and the agent returns the list of files in the directory. Similarly, a `readFile` action extracts the filename from the arguments and retrieves its content.

The execution step is where the agent performs **tangible work** — interacting with files, printing messages to the console. It bridges the decision-making process with concrete results that feed back into the agent's memory for subsequent iterations.

## Step 5: update the agent's memory

After executing an action, the agent updates its memory with the results. Memory is the agent's record of what has happened during the interaction: user requests, the actions performed, and their outcomes. By appending this information, the agent retains context and makes more informed decisions in future iterations.

In the code, memory is updated by extending it with both the LLM's response (the agent's **intention**) and the result of the executed action:

```typescript
memory.push(Message.assistant(response));
memory.push(Message.user(JSON.stringify(result)));
```

How this works:

- The **`assistant`** role captures the structured response generated by the LLM.
- The **`user`** role captures the feedback in the form of the action result, ensuring the LLM has a clear understanding of what happened after the action was performed. **The results of actions are always communicated back to the LLM with the "user" role.**

By keeping a running history of these exchanges, the agent maintains continuity, refining its behavior dynamically as the memory grows and tracking the status of its work.

## Step 6: decide whether to continue

The final step in each iteration is determining whether to continue or terminate. This decision is based on the action executed and the state of the task. If the parsed action specifies `terminate`, or if a predefined condition (e.g. maximum iterations) is met, the agent ends its loop:

```typescript
if (action.toolName === "terminate") {
    console.log(action.args.message);
    break;
}
```

If the action specifies termination, the loop exits and the agent provides a closing message defined in the `terminate` action's arguments. Otherwise, the agent loops back to process the next request or continue its task.

## Example: iterative adaptation

Imagine the agent is tasked with reading a file but encounters a missing filename in the initial request:

1. In the first iteration, it executes `listFiles` to retrieve the available files.
2. Based on the memory of this result, it refines its next action, prompting the user to select a specific file.
3. This iterative process continues until the task is completed or the agent determines that no further actions are required.

Each iteration, the agent can look back at its memory to decide if it has completed the overall task — **memory is a critical part of deciding if the agent should continue or terminate**. By deciding at each step, the agent balances dynamically adapting to new information with the need to eventually conclude its task. The agent can also be instructed on *when* to terminate, e.g. if more than two errors are encountered or a specific condition is met.

## In this repo

[`AgentLoop.ts`](../src/module1/AgentLoop.ts) implements this loop as `runTextParsingAgentLoop()` (and a minimal procedural twin, `runSimpleAgentLoop()`). Differences vs. the simplified teaching code above — same pattern, production-shaped:

| Teaching code | [`AgentLoop.ts`](../src/module1/AgentLoop.ts) |
|---|---|
| `memory` as `Message[]` | `ConversationMemory` (typed items) in the full version; plain array in `runSimpleAgentLoop()` |
| result pushed as `Message.user(JSON.stringify(result))` | `memory.addEnvironment(JSON.stringify(result))` — the `environment` item type of [04](04-memory-and-conversation-history.md) |
| `parseAction()` → `{toolName: "error"}` on bad output | `parseTextAction()` degrades to `terminate` with the raw response (both are graceful fallbacks against invalid LLM output) |
| inline `if/else` tool dispatch | `executeTextAction()` with structured `{success, result \| error}` results |
| `agentRules` | `AGENT_RULES` (adds `printMessage`, error handling rules, explicit examples) |
| `maxIterations` | `MAX_ITERATIONS = 10` |

The loop's docstring states the core pattern all agents follow:

> 1. Build prompt from rules + memory → 2. Get LLM response → 3. Parse action from text → 4. Execute action → 5. Update memory → 6. Check termination, repeat if needed

---

**Next:** more materials as they arrive (tools & function calling en profundidad, framework GAME…).
