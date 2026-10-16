# LLM Usage Normalizer

Paste a raw usage object from Anthropic Messages, OpenAI Responses or Chat Completions, or Gemini generateContent. It names each field under that provider's own definition, rebuilds total input, cache read, cache write, uncached input, reasoning and non-reasoning output on one common basis, and flags the sums that double-count.

**Live demo:** https://0xelitesystem.github.io/llm-usage-normalizer/

## Use

1. Paste a full response or only its usage object. The page finds `usage`, `usageMetadata` or `response.usage` on its own.
2. Detection picks the provider: Gemini from its camelCase counts, Chat Completions from `prompt_tokens`, Anthropic from its cache fields or `type: "message"`, Responses from `input_tokens_details` or `total_tokens`. A bare `input_tokens` plus `output_tokens` fits both Anthropic and Responses, so the page asks you to choose. The provider menu overrides detection at any time.
3. Read the common basis table. Every row shows the provider field path and the arithmetic.
4. Read the findings. Each one names the double count, the trigger, and the numbers on both sides.
5. Optionally enter your own rates for a cost. No prices ship with the page.

Keys that start with `gen_ai.usage.` are read as OpenTelemetry-style exporter output, and the page shows both readings of the input number side by side. If an Anthropic-shaped input number came from an exporter, tick **This input number came from an exporter** to see the same comparison.

| Quantity | Anthropic Messages | OpenAI Responses / Chat | Gemini generateContent |
| --- | --- | --- | --- |
| total_input | `cache_read_input_tokens + cache_creation_input_tokens + input_tokens` | `input_tokens` / `prompt_tokens` | `promptTokenCount` (includes cached content) |
| cache_read | `cache_read_input_tokens` | `*_tokens_details.cached_tokens` | `cachedContentTokenCount` |
| cache_write | `cache_creation_input_tokens`, split by `cache_creation` | `*_tokens_details.cache_write_tokens` (absent counts as 0) | not reported by this API |
| uncached_input | `input_tokens` | input minus cached minus cache write | `promptTokenCount - cachedContentTokenCount` |
| total_output | `output_tokens` | `output_tokens` / `completion_tokens` | `candidatesTokenCount + thoughtsTokenCount` |
| reasoning | `output_tokens_details.thinking_tokens`, inside output | `reasoning_tokens`, inside output | `thoughtsTokenCount`, outside candidates |

Anthropic also gets an ITPM line, `input_tokens + cache_creation_input_tokens`, because the rate-limits doc says cache reads do not count toward input tokens per minute for most models. Output minus reasoning is labelled "non-reasoning output (may include formatting tokens)", never "visible text tokens", because OpenAI's token-counting guide says formatting tokens are not necessarily itemized separately in usage.

### The findings

- **D1.** An input number that already includes cache, with cache added again. Shown for `gen_ai.usage.*` keys and when the exporter box is ticked.
- **D2.** OpenAI: `cached_tokens` or `cache_write_tokens` added to an input that already contains them.
- **D3.** Reasoning added to output again (OpenAI and Anthropic both count it inside output).
- **D4.** Gemini: `candidatesTokenCount` read as all output (it leaves out thoughts), or `cachedContentTokenCount` added to a prompt count that already includes it.
- **D5.** Anthropic: `input_tokens` read as the whole prompt, which misses cache read and cache write.
- **SUM, TOTAL, IMPOSSIBLE.** `cache_creation` not adding up, a provider total that does not match its definition, or a cached count larger than the input it belongs to.

### Use cases

- **Your dashboard shows twice the tokens you expected.** Paste one raw response and one exported span. If the span's input already includes cache, D1 shows the inflated sum next to the right one.
- **You are writing a cost function across providers.** Use the common basis rows as the variable names, and the From column as the field path for each provider.
- **You are reviewing a gateway that maps one provider's usage onto another's.** Run the same response through both readings and check which sum the gateway emits.
- **You need a cost with your own negotiated rates.** Enter them in the cost panel. The page multiplies, it never guesses a price.

Streaming fragments (several usage objects) are out of scope for this version. To merge a stream first, see https://github.com/0xelitesystem/llm-stream-inspector.

## Why this exists

Each provider defines its own usage fields, and each definition is internally consistent. Anthropic's `input_tokens` excludes cache reads and writes. OpenAI's `input_tokens` includes `cached_tokens` and `cache_write_tokens`. Gemini's `promptTokenCount` includes cached content, and its thoughts sit outside `candidatesTokenCount`. Code that maps one convention onto another double-counts, and it has happened in the wild:

- langfuse/langfuse#12306 (closed, not planned): an emitter wrote `gen_ai.usage.input_tokens` already including cache, and Langfuse added cache read and cache write again. The worked example in the issue is 5 + 128955 + 1253 = 130213, reported as 260421. https://github.com/langfuse/langfuse/issues/12306
- pydantic/pydantic-ai#4364 (closed, not planned): the same root cause, from the emitter side. https://github.com/pydantic/pydantic-ai/issues/4364
- BerriAI/litellm#40006 (open): only the `custom_cost_per_token` path bills Anthropic cache reads twice; the issue says default model-map pricing is not affected ("measured at 1.000x"). https://github.com/BerriAI/litellm/issues/40006
- BerriAI/litellm pull request #25517 (merged 2026-04-10): a Bedrock Invoke streaming double count, `prompt_tokens` 65339 instead of 32671. https://github.com/BerriAI/litellm/pull/25517

Single file, no build step, no network call, MIT licensed. Nothing to install, and nothing watching what you paste.

## Privacy

Everything runs in your browser. There is no network call of any kind: no request for code, fonts, analytics or telemetry. Usage objects and full responses can carry request ids and model output, and nothing you paste is transmitted anywhere. A Content-Security-Policy in the page blocks every connection. The only thing stored is your light or dark theme choice, in your own browser's localStorage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/llm-usage-normalizer.git
cd llm-usage-normalizer
```

Open `index.html` in a browser. If you would rather serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and inline JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Sources

Re-read on 2026-10-16.

- Anthropic, prompt caching (the `input_tokens` definition, the total formula, the `cache_creation` sum): https://platform.claude.com/docs/en/build-with-claude/prompt-caching
- Anthropic, rate limits (cache reads and ITPM): https://platform.claude.com/docs/en/api/rate-limits
- Anthropic, extended thinking (`usage.output_tokens_details.thinking_tokens`): https://platform.claude.com/docs/en/build-with-claude/extended-thinking
- OpenAI, prompt caching guide (the ordinary input formula and the model table): https://developers.openai.com/api/docs/guides/prompt-caching
- OpenAI, Responses reference (`cache_write_tokens`): https://developers.openai.com/api/reference/resources/responses
- OpenAI, Chat Completions reference (`prompt_tokens_details.cache_write_tokens`): https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create
- OpenAI, token counting guide (output counts and formatting tokens): https://developers.openai.com/api/docs/guides/token-counting
- OpenAI, reasoning guide (the sample with `output_tokens` 1186, `reasoning_tokens` 1024, `total_tokens` 1261): https://developers.openai.com/api/docs/guides/reasoning
- Google, Gemini API generateContent reference, UsageMetadata (marked "Last updated 2026-10-13 UTC" when read): https://ai.google.dev/api/generate-content

The Gemini `totalTokenCount` definition changed during October 2026. On 2026-10-16 it reads "prompt + tool-use prompt + thoughts + response candidates", and the page checks the total against that sum.

The examples use your own numbers in the documented structures, except the first Anthropic example, whose three input counts come from langfuse/langfuse#12306, and the Responses reasoning example, whose counts come from OpenAI's reasoning guide.

## Independence

This is an independent tool. It is not affiliated with or endorsed by Anthropic, OpenAI, Google, Langfuse, Pydantic or LiteLLM. Vendor and project names appear only as subject matter, to identify the formats and reports described. Field names are quoted as facts; short definition sentences are quoted on the page with their links.

## Related

- https://github.com/0xelitesystem/prompt-cache-inspector
- https://github.com/0xelitesystem/llm-stream-inspector
- https://github.com/0xelitesystem/token-cost-estimator

## License

MIT. Copyright (c) 2026 0xelitesystem.
