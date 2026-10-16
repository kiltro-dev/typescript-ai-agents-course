# 05 — Quasi-Agents: Prompt Chaining in Practice

> A quasi-agent writes a TypeScript function through a chain of sequential prompts:
> code → documentation → tests. Not quite a real agent — it can't react and adapt —
> but it practices the two foundations: maintaining context across prompts and
> managing information flow between the user and the LLM.
>
> Source: course *AI Agents in Typescript/Javascript with Generative AI* — "Building a
> Quasi-Agent" (practice exercise) + "Possible Solution for Building a Quasi-Agent".
> Implementation: [`src/module1/QuasiAgent.ts`](../src/module1/QuasiAgent.ts).
>
> Run: `npm run module1:quasi`

---

For practice, we write a quasi-agent that can write TypeScript functions based on user requirements. It isn't quite a real agent — it can't react and adapt — but it can do something useful: ask the user what they want code for, write the function, add documentation, and finally include test cases using a testing framework.

This helps you understand how to **maintain context across multiple prompts** and **manage the information flow** between the user and the LLM. It also helps you understand the pain of trying to parse and handle the output of an LLM that is not always consistent.

## The exercise

Write a program that uses **sequential prompts** to generate any TypeScript function based on user input:

### First prompt — generate the function

- Ask the user what function they want to create
- Ask the LLM to write a basic TypeScript function based on the user's description
- Store the response for use in subsequent prompts
- Parse the response to separate the code from the LLM's commentary

### Second prompt — add documentation

- Pass the code generated from the first prompt
- Ask the LLM to add comprehensive documentation: function description, parameter descriptions, return value description, example usage, edge cases

### Third prompt — add tests

- Pass the documented code generated from the second prompt
- Ask the LLM to add test cases using Jest or Vitest covering: basic functionality, edge cases, error cases, various input scenarios

### Requirements

- Maintain conversation context between prompts
- Print each step of the development process
- Save the final version to a TypeScript file

> **Go further:** use the system message to force the LLM to always output code with a specific style or using particular libraries.

## Understanding the architecture

The quasi-agent works through a multi-turn conversation flow, e.g. for a code review task:

1. Initial code review with feedback
2. Follow-up clarification questions
3. Contextual review of improved code

The magic happens in how we maintain context between these steps, ensuring each interaction builds on the previous conversation history. The building blocks are already familiar from [02 — Messages](02-messages-and-prompting.md) (`Message` factory methods, ChatML) and [03 — The LLM Abstraction](03-llm-abstraction.md) (`LLM.generate()` converting messages, handling tools and errors). What is new is the `QuasiAgent` class that keeps the history together:

```typescript
class QuasiAgent {
  private readonly history: Message[] = [];
  private readonly llm: LLM;

  constructor(systemPrompt: string, llmConfig?: { model?: string }) {
    this.llm = new LLM(llmConfig);
    // Initialize with system message
    this.history.push(Message.system(systemPrompt));
  }

  async chat(userMessage: string): Promise<string> {
    this.history.push(Message.user(userMessage));
    const response = await this.llm.generate(this.history);
    this.history.push(Message.assistant(response));
    return response;
  }

  getHistory(): readonly Message[] { return this.history; }

  reset(): void {
    const systemMessage = this.history[0];
    this.history.length = 0;
    this.history.push(systemMessage);
  }

  get turnCount(): number {
    return this.history.filter(m => m.role === 'user').length;
  }
}
```

Note how the `QuasiAgent`:

- Maintains private conversation history
- Automatically adds messages to history in `chat()`
- Preserves the system prompt when resetting
- Provides readonly access to history for inspection

## Walkthrough: the Code Review Assistant

A practical multi-turn workflow (from the course solution — this particular demo lives in the material, not in the repo's file).

### Phase 1 — initialize with clear instructions

```typescript
const reviewer = new QuasiAgent(
  'You are a TypeScript code reviewer. When given code:\n' +
  '1. Identify potential issues\n' +
  '2. Suggest improvements\n' +
  '3. Remember previous feedback for consistency\n' +
  'Be constructive and specific.'
);
```

The system message establishes the LLM's role and behavior. It persists throughout the conversation and guides all responses.

### Phase 2 — first code review

```typescript
const firstCode = `
function getData(id) {
  return fetch('/api/data/' + id)
    .then(r => r.json());
}
`.trim();

const firstReview = await reviewer.chat(
  `Please review this TypeScript code:\n\n${firstCode}`
);
```

The reviewer analyzes the code and provides feedback. The response is automatically added to history.

### Phase 3 — follow-up questions

```typescript
const followUp = await reviewer.chat(
  'Can you show me how to implement your first suggestion?'
);
```

No need to repeat context: the agent remembers the previous review and can reference "your first suggestion" naturally.

### Phase 4 — contextual review of improvements

```typescript
const secondCode = `
async function getUserData(userId: string) {
  const response = await fetch(\`/api/users/\${userId}\`);
  return response.json();
}
`.trim();

const secondReview = await reviewer.chat(
  `Here's another function. Does it address the issues you mentioned before?\n\n${secondCode}`
);
```

The reviewer maintains context from the first review and can intelligently compare the new code against previously mentioned issues.

## Memory management through message history

Each step builds on previous responses:

- **System prompt persists** — guides all responses consistently
- **User messages accumulate** — each question builds on previous context
- **Assistant responses accumulate** — the LLM sees its own previous feedback

When reviewing the second code snippet, the LLM sees: it's a TypeScript code reviewer (system) · the first snippet and its review · the follow-up question and answer · the new code and question (current). This accumulated context enables sophisticated, contextual conversations that would be impossible with stateless prompting (see [04 — Memory](04-memory-and-conversation-history.md)).

## Key techniques

### 1. Memory manipulation (`chatWithMemoryManipulation`)

The most important technique is controlling what gets stored in memory:

```typescript
async chatWithMemoryManipulation(
  userMessage: string,
  transformResponse: (response: string) => string
): Promise<string> {
  this.history.push(Message.user(userMessage));
  const response = await this.llm.generate(this.history);

  // Store TRANSFORMED version, not the original
  const transformedResponse = transformResponse(response);
  this.history.push(Message.assistant(transformedResponse));

  return response;  // Caller sees original
}
```

**Why this matters:** by storing only the code block (not the LLM's explanatory text), we make the LLM think it always outputs in a consistent format. This guides its behavior in subsequent steps — e.g. in step 1 of the pipeline:

```typescript
const step1Response = await agent.chatWithMemoryManipulation(
  `Write a TypeScript function that ${description}. ` +
  'Output the function in a ```typescript code block.',

  // Memory manipulation: store ONLY the code block
  (response) => wrapCodeBlock(extractCodeBlock(response))
);
```

### 2. Prompt chaining

Each step builds on the previous:

1. Generate function → stores code
2. *"Add documentation to this function"* → LLM sees the code from step 1
3. *"Write tests for this function"* → LLM sees documented code from step 2

The chain works because memory maintains full context.

### 3. Code extraction utilities

`extractCodeBlock()` and `wrapCodeBlock()` handle the messy reality of parsing LLM output — markdown code fences (```` ```typescript ````, ```` ```ts ````, …). Real-world LLM usage requires parsing structured output from text; this pattern is fundamental.

### 4. Two methods, two purposes

| Method | Use when |
|---|---|
| `chat()` | Natural conversations where you want full context |
| `chatWithMemoryManipulation()` | Controlled workflows where you curate what the LLM "remembers" |

## In this repo

[`QuasiAgent.ts`](../src/module1/QuasiAgent.ts) implements everything above. `npm run module1:quasi` runs three demos:

| Demo | What it shows |
|---|---|
| `demonstrateMemoryImportance()` | Side-by-side "favorite color" comparison: without history the LLM cannot recall; with `QuasiAgent.chat()` it can |
| `demonstrateMemoryManipulation()` | Stores only the code block in history, then a follow-up ("add a parameter") continues from the cleaned-up version |
| `demonstrateFunctionBuilder()` | The exercise's `developFunction()` pipeline: generate → JSDoc → Jest tests, each step building on the previous context |

`developFunction()` returns the artifacts of the chain (`initialFunction`, `documentedFunction`, `testCases`, a suggested `filename`), all extracted via `extractCodeBlock()`.

## Why it's still not an agent

The `QuasiAgent` docstring is explicit. It does **not**:

- Parse responses for tool calls
- Execute actions in an environment
- Make autonomous decisions

It **does**:

- Maintain conversation context
- Chain multiple prompts together
- Demonstrate memory manipulation techniques

Those missing pieces — parsing actions, executing them, deciding next steps — are exactly what turns a quasi-agent into a real agent: the **agent loop**, coming next in the path.

## Key takeaways

- **Memory manipulation is powerful** — controlling what gets stored in conversation history guides the LLM's behavior and maintains consistent output patterns.
- **Prompt chaining enables complexity** — breaking a complex task into steps, each building on previous context, creates sophisticated workflows.
- **Code extraction is essential** — real-world LLM usage requires parsing structured output from text.
- **Two methods, two purposes** — `chat()` for full context, `chatWithMemoryManipulation()` for curated context.
- **Foundation for agents** — memory management, prompt chaining, and structured output parsing are the building blocks for full agents.

The `developFunction()` pipeline shows how a quasi-agent can accomplish complex, multi-step tasks by carefully controlling conversation flow. This is the bridge between simple prompting and full agentic behavior.

---

**Next:** [06 — From Text to Actions](06-from-text-to-actions.md)
