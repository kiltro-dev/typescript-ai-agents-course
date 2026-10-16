# 03 — The LLM Abstraction

> Why [`src/shared/LLM.ts`](../src/shared/LLM.ts) exists as a layer between application code and
> the provider SDK, and what it buys you.
>
> Source: course *AI Agents in Typescript/Javascript with Generative AI* — "Sending Prompts
> Programmatically & Managing Memory" (section *Understanding the LLM Abstraction*).

---

Rather than using the OpenAI SDK directly throughout your code, the `LLM` class is a crucial architectural component. It is the only place that knows about the provider; everything else in the app talks to `LLM.generate()`.

## Benefits

| Benefit | What it means |
|---|---|
| **Provider independence** | Switch between LLM providers (OpenAI, Anthropic, Google, …) without changing application code |
| **API evolution protection** | Insulates your codebase from changes in provider APIs |
| **Testing and mocking** | Unit testing becomes significantly easier — mock `LLM`, not an HTTP API |
| **Cost control** | Centralized point for fallback strategies and rate limiting |
| **Observability** | Single location for logging, metrics, and monitoring |

This is also why the rest of the codebase deals in the repo's own `Message` / `Prompt` types instead of provider payloads: the abstraction boundary is `LLM.generate(prompt) → string`.

## Configuration

The model is configured via environment variables (or constructor options), so switching models or providers does not touch application code:

| Source | Variable / option | Default |
|---|---|---|
| env / `LLMConfig.apiKey` | `OPENAI_API_KEY` | — (required) |
| env / `LLMConfig.model` | `OPENAI_MODEL` | `gpt-5-nano` |
| env / `LLMConfig.baseURL` | `BASE_URL` | `https://api.openai.com/v1` |
| env / `LLMConfig.debug` | `DEBUG` | `false` |

> **Note on this repo:** `BASE_URL` was added so the same code works against any OpenAI-compatible
> endpoint (e.g. DeepSeek with your own key). If unset, the client falls back to OpenAI's default
> URL with no extra configuration.

## How `generate()` works internally

1. Accepts either a `Message[]` (simple case) or a `Prompt` (when tools are needed).
2. If the prompt has tools and the model chooses to call one, the result is a JSON string like
   `{"tool": "readFile", "args": {"fileName": "test.txt"}}` — the uniform action format the rest
   of the system consumes.
3. Otherwise it returns the plain text content of the response.

This contract (`string` in, `string` out, tool calls as JSON) is what lets modules 1–4 plug in
different *agent languages* on top of the same LLM layer — see the [Guía del repo](GUIA-DEL-REPO.md#32-tres-formas-de-hablar-con-el-llm-agent-languages).

---

**Next:** [04 — Memory & Conversation History](04-memory-and-conversation-history.md)
