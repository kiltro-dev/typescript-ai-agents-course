# 04 — Memory & Conversation History

> Why LLMs have no memory, and how simulating continuity by replaying messages is the
> foundation of agent memory.
>
> Source: course *AI Agents in Typescript/Javascript with Generative AI* — "LLMs Do Not Have Memory".
> Related code: [`src/shared/ConversationMemory.ts`](../src/shared/ConversationMemory.ts),
> [`src/shared/Memory.ts`](../src/shared/Memory.ts).

---

When building an Agent, we need it to remember its actions and the result of those actions. For example, if it tries to create a calendar event for a meeting and the API call fails due to an incorrect parameter value it provided, we want it to remember that the API call failed and why — so it can correct the mistake and try again. If we have a complex task broken down into multiple steps, we need the Agent to remember the results of each step to continue the task from where it left off. **Memory is crucial for Agents.**

## LLMs do not have memory

When interacting with an LLM, the model does not inherently "remember" previous conversations or responses. Every time you call the model, it generates a response based **solely on the information provided in the `messages` parameter**. If previous context is not included in the messages, the model will not have any knowledge of it.

To simulate continuity in a conversation, you must explicitly pass all relevant prior messages (including system, user, and assistant roles) in the messages list for each request.

## Example 1: missing context

The second query does not include the previous interaction:

```typescript
const messages = [
  Message.system('You are an expert software engineer that prefers functional programming.'),
  Message.user('Write a function to swap the keys and values in an object.'),
];

const response = await llm.generate(messages);
console.log(response);

// Second query without including the previous response
const messages2 = [
  Message.user('Update the function to include documentation.'),
];

const response2 = await llm.generate(messages2);
console.log(response2);
```

Output — the first request generates the function:

```typescript
function swapKeysValues<T extends string | number>(obj: Record<T, T>): Record<T, string> {
  return Object.fromEntries(Object.entries(obj).map(([k, v]) => [v, k]));
}
```

But the second request fails to connect it to the first:

> I'm not sure what function you're referring to. Can you clarify?

In the second request the model doesn't "remember" the function it wrote in the first interaction. Since the information is not in the second prompt, the model cannot connect the two.

## Example 2: replaying history for continuity

To fix this, add new messages with the `assistant` role holding the content of prior responses. The model can then see what code it wrote previously and build on that:

```typescript
const messages = [
  Message.system('You are an expert software engineer that prefers functional programming.'),
  Message.user('Write a function to swap the keys and values in an object.'),
];

const response = await llm.generate(messages);
console.log(response);

// We are going to make this verbose so it is clear what
// is going on. In a real application, you would likely
// just append to the messages list.
const messages2 = [
  Message.system('You are an expert software engineer that prefers functional programming.'),
  Message.user('Write a function to swap the keys and values in an object.'),

  // Here is the assistant's response from the previous step
  // with the code. This gives it "memory" of the previous
  // interaction.
  Message.assistant(response),

  // Now, we can ask the assistant to update the function
  Message.user('Update the function to include documentation.'),
];

const response2 = await llm.generate(messages2);
console.log(response2);
```

Now the model maintains context and produces the documented version of *its own* previous function:

```typescript
/**
 * Swaps the keys and values in the given object.
 *
 * @param obj - The input object with unique values.
 * @returns A new object where the keys and values are swapped.
 * @throws Error if the object contains duplicate values.
 *
 * @example
 * swapKeysValues({ a: 1, b: 2, c: 3 })
 * // Returns: { 1: 'a', 2: 'b', 3: 'c' }
 */
function swapKeysValues<T extends string | number>(obj: Record<T, T>): Record<T, string> {
  const values = Object.values(obj);
  if (new Set(values).size !== values.length) {
    throw new Error('Duplicate values detected. Cannot swap keys and values.');
  }
  return Object.fromEntries(Object.entries(obj).map(([k, v]) => [v, k]));
}
```

## Key takeaways

- **No inherent memory** — the LLM has no knowledge of past interactions unless explicitly provided in the current prompt (via `messages`).
- **Provide full context** — to simulate continuity, include all relevant messages (both user and assistant responses) in the `messages` parameter.
- **Role of assistant messages** — adding previous responses as assistant messages keeps the conversation coherent. For an agent, this lets it remember what actions (e.g. API calls) it took in the past.
- **Memory management** — we control what the LLM remembers or forgets by managing which messages go into the conversation. Making the LLM forget things can be powerful in some circumstances, e.g. breaking a pattern of poor responses from an Agent.

## Why this matters

Understanding the stateless nature of LLMs is crucial for designing agents that rely on multi-turn conversations with their environment. Developers must explicitly manage and provide context to ensure the model generates accurate and relevant responses.

## In this repo

What the examples above do by hand (append `Message.assistant(response)` to the list), the repo's memory classes do systematically:

| Class | Role |
|---|---|
| [`ConversationMemory`](../src/shared/ConversationMemory.ts) | Typed conversation history (`user` / `assistant` / `environment` / `system`) that converts to LLM messages with `toMessages()` — the "memory" of the GAME framework |
| [`Memory`](../src/shared/Memory.ts) | Key-value store for state across loop iterations (e.g. the result of an API call) |

The `environment` item type is exactly the "remember that the API call failed and why" case from the intro: tool results are recorded as history the agent (and the model) can see on the next iteration.

---

**Next:** [05 — Quasi-Agents](05-quasi-agents.md)
