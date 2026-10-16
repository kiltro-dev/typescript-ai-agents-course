# 02 — Messages & Programmatic Prompting

> Core concepts for talking to an LLM from code: messages, roles, `LLM.generate()`, and the
> system message as the way to "program" the agent's behavior and output format.
>
> Source: course *AI Agents in Typescript/Javascript with Generative AI* — "Sending Prompts
> Programmatically & Managing Memory". Code: [`src/module1/ProgrammaticPrompting.ts`](../src/module1/ProgrammaticPrompting.ts).
>
> Run: `npm run module1:prompting`

---

To get started building agents, we need to understand how to send prompts to LLMs. Agents require two key capabilities:

1. **Programmatic prompting** — automating the prompt-response cycle that humans do manually in a conversation. This forms the foundation of the **Agent Loop**.
2. **Memory management** — controlling what information persists between iterations, like API calls and their results, to maintain context through the agent's decision-making process.

Programmatically sending prompts is how we move from a human typing prompts and acting on the LLM's response to an agent that does this automatically. The Agent Loop that we will build later is precisely this: programmatically sending prompts to the LLM and taking action based on its response.

We also need to understand how to manage what the LLM knows or remembers. This matters because we want to control what information the LLM has in each loop iteration. For example, if it just called an API, we want it to remember what API it asked to invoke and what the result of that action was.

## The building blocks: `Message` and roles

Everything you send to an LLM is a **list of messages**. Each message has a **role** (who said it) and **content** (what was said) — the standard ChatML format. The role allows the LLM to understand context and respond appropriately.

The `Message` class ([`src/shared/Message.ts`](../src/shared/Message.ts)) provides clean factory methods for the three valid roles:

```typescript
// Using static factory methods (recommended)
const system = Message.system("You are a helpful assistant.");
const user = Message.user("Hello!");
const assistant = Message.assistant("Hi there!");
```

| Role | Purpose |
|---|---|
| `system` | Instructions for the LLM that set its behavior — ground rules for the conversation (e.g. *"You will respond in JSON."*). |
| `user` | Input from the human user: prompts, questions, instructions. |
| `assistant` | Responses from the LLM. Used to provide context for a conversation that already started or to guide the model with sample responses — things "it" said in the past. |

## System messages "program" the agent

System messages are particularly important in a conversation and will be very important for AI agents. They set the ground rules and tell the model how to behave. **Models are designed to pay more attention to the system message than to the user messages.** Through system messages we can "program" the AI agent.

## Sending a prompt: `LLM.generate()`

The `LLM` class ([`src/shared/LLM.ts`](../src/shared/LLM.ts)) provides a clean abstraction for interacting with language models. It:

- Decouples your code from any specific LLM provider
- Makes testing easier with mockable interfaces
- Provides a single point for adding logging and monitoring
- Protects against API changes from providers

(See [03 — The LLM Abstraction](03-llm-abstraction.md) for the full picture.)

- **Input:** an array of `Message` objects representing the conversation.
- **Output:** a `Promise` that resolves to the model's response as a string.

The simplest possible interaction:

```typescript
const llm = new LLM();

const messages = [
  Message.system('You are a helpful assistant that gives concise answers.'),
  Message.user('What is TypeScript in one sentence?'),
];

const response = await llm.generate(messages);
console.log('Response:', response);
```

The flow is always the same: create messages (system + user) → call `llm.generate()` → receive a string.

## Controlling behavior with system prompts

### Personas: same question, different system messages

The system message is the most important part of the prompt. The same question gets three completely different responses based solely on it — this is how powerful system instructions are for "programming" the AI's behavior:

```typescript
const personas = [
  {
    name: 'Teacher',
    system: 'You are a patient teacher explaining to a beginner. Use simple analogies.',
  },
  {
    name: 'Expert',
    system: 'You are a computer science professor. Be precise and technical.',
  },
  {
    name: 'Comedian',
    system: 'You are a comedian. Explain technical concepts with humor.',
  },
];

const userQuestion = 'Explain recursion.';

for (const persona of personas) {
  const messages = [
    Message.system(persona.system),
    Message.user(userQuestion),
  ];

  const response = await llm.generate(messages);
  console.log(`${persona.name}'s explanation:\n${response}`);
}
```

### Code generation with constraints

The system message can constrain the output to make it predictable and useful for programmatic workflows — here: TypeScript only, clean and well-typed, no explanations:

```typescript
const codeMessages = [
  Message.system(
    'You are a TypeScript expert. Write clean, well-typed code. ' +
    'Only output the code, no explanations.'
  ),
  Message.user('Write a function that checks if a string is a palindrome.'),
];

const codeResponse = await llm.generate(codeMessages);
console.log('Generated code:\n');
console.log(codeResponse);
```

## Working with structured data

Messages can incorporate arbitrary information as long as it is in text form. LLMs can interpret just about any information we give them, even if it isn't easily human-readable. We will rely heavily on this ability — particularly with JSON — when we start building agents.

### JSON as input: send structured specs

Here we generate a function implementation from a typed spec:

```typescript
interface CodeSpec {
  name: string;
  description: string;
  params: Record<string, string>;
}

const codeSpec: CodeSpec = {
  name: 'swapKeysValues',
  description: 'Swaps the keys and values in a given object.',
  params: {
    obj: 'An object with unique values as a Record<string, string>.',
  },
};

const jsonMessages = [
  Message.system(
    'You are a TypeScript expert. Write clean, well-typed code. ' +
    'Always document your functions with JSDoc comments.'
  ),
  Message.user(`Please implement: ${JSON.stringify(codeSpec, null, 2)}`),
];

const response = await llm.generate(jsonMessages);
console.log('Generated implementation:\n');
console.log(response);
```

This is a simple example of sending JSON to the LLM, but the same mechanism lets us send, say, the JSON result of an API call — which is exactly what agents do in their loop.

### JSON as output: request structured responses

You can also ask the LLM to respond in JSON and parse the result programmatically:

```typescript
const jsonMessages = [
  Message.system(
    'You are a helpful assistant. Always respond in valid JSON format.'
  ),
  Message.user(
    'List 3 programming languages with their primary use case. ' +
    'Format: [{"language": "name", "useCase": "description"}]'
  ),
];

const jsonResponse = await llm.generate(jsonMessages);
console.log('JSON Response:', jsonResponse);

// Try to parse it
try {
  const parsed = JSON.parse(jsonResponse);
  console.log('Parsed successfully:', parsed);
} catch {
  console.log('Note: Response was not valid JSON');
}
```

The LLM will usually comply with the requested format, but it's good practice to wrap parsing in a `try-catch` as shown above.

## Running the examples

```bash
npm run module1:prompting
```

This demonstrates: simple completion with a concise assistant persona · code generation for a palindrome function · structured JSON input (CodeSpec) · structured JSON output · system prompt variations with different personas.

## Key takeaways

- **System messages are powerful** — they set the behavior and constraints for the LLM, and models attend to them more than to user messages.
- **The `Message` abstraction is clean** — use `Message.system()`, `Message.user()`, and `Message.assistant()` factory methods.
- **The LLM abstraction decouples providers** — your code doesn't need to know about OpenAI specifics.
- **JSON is your friend** — you can send *and* receive structured data for programmatic workflows.
- **TypeScript provides type safety** — interfaces and types help catch errors at compile time.

In the next sections we'll build on these fundamentals to create agents that can maintain conversation history, use tools, and execute complex workflows.

## Exercise

Try creating a prompt that only provides the response as a Base64 encoded string and refuses to answer in natural language. Can you get your LLM to only respond in Base64?

> Hint: make the system message very strict about the output format, and be specific in the user message that you want Base64 encoding.

---

**Next:** [03 — The LLM Abstraction](03-llm-abstraction.md)
