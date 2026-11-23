# NagaAI provider: product acceptance for lobehub/lobehub#23010

Run on 2026-11-23, 07:50–08:18 UTC, on a local LobeHub dev instance (`pnpm dev` with the `docker-compose/dev` services, Postgres mode) built from PR head `d89d85f`. The NagaAI key came from `NAGAAI_API_KEY`, so it appears in no screenshot. The only change after the run is `e646eb3`, which deletes a stray scratch script and touches no code.

| # | Check | Model | Result | Time (UTC) | Screenshot |
|---|---|---|---|---|---|
| 1 | Settings > Provider > NagaAI: connectivity check, Fetch models | google/gemini-3.5-flash | Check passed. Fetch models: 347 models (344 chat, 3 image). The first check stalled with no upstream response and passed on retry | 07:53, 07:57 | [01](shots/01-provider-check.png), [02](shots/02-models.png) |
| 2 | Plain chat, streamed | anthropic/claude-sonnet-5.5 | OK, via `POST /v1/responses` | 07:59 | [03](shots/03-chat-claude.png) |
| 2 | Plain chat, streamed, with reasoning | deepseek/deepseek-v4.1-flash | OK, the reasoning summary renders as Deep Thinking. The topic title also came from this model | 08:10 | [08](shots/08-chat-deepseek.png) |
| 3 | Tool call with the rendered tool card (built-in Todos) | anthropic/claude-sonnet-5.5 | Tool card rendered with the right items | 08:00 | [04](shots/04-tool-claude.png) |
| 3 | Tool call (built-in Todos) | openai/gpt-6.1-sol | **Fails on the gateway.** The arguments reach the tool as `{}` and the agent repeats the call. NagaAI's stream for this model opens the item with `arguments: "{}"` in `response.output_item.added` and then streams the full arguments as deltas, so the client builds `{}{"adds":…}`. Claude and Gemini open the item with `""`. This is fixed on the NagaAI side, not in this PR | 08:06–08:08 | [07](shots/07-tool-gpt.png) |
| 4 | A turn after a tool call, `store: false`, no `include` | anthropic/claude-sonnet-5.5 | OK: the history carries `function_call` and `function_call_output` | 08:02 | [05](shots/05-tool-claude-done.png) |
| 4 | Web search tool, then a page read, then the answer: three steps | google/gemini-3.5-flash | OK. One step took about 60 s at the gateway, and the same payload sent again took 7 s | 08:03–08:05 | [06](shots/06-gemini-search-tool.png) |
| 5 | Image input | anthropic/claude-sonnet-5.5 | OK: "a red circle on the left and a blue square on the right" | 08:16 | [09](shots/09-image-claude.png) |
| 6 | Structured output (`generateObject` through the PR runtime) | claude-sonnet-5.5, gpt-6.1-sol, gemini-3.5-flash | All 200 with valid objects. Sent as `text.format` `json_schema` on `/v1/responses` | 08:11 | [log](structured-output-and-search.txt) |
| 6 | Chat with `response_format` `json_schema` | claude-sonnet-5.5, gpt-6.1-sol | 200, `{"city":"Paris"}`. Mapped to `text.format`, so `response_format` is not sent | 08:11 | [log](structured-output-and-search.txt) |
| 6 | `enabledSearch` | claude-sonnet-5.5, gpt-6.1-sol | 200, sent as the `web_search` tool. No search ran: NagaAI answered with `x-naga-dropped-tools: web_search` (documented fallback). In the UI the model cards do not set `search`, so LobeHub uses its own web-browsing tool (row 4, Gemini) | 08:11–08:12 | [log](structured-output-and-search.txt) |
| 7 | Wrong key | gemini-3.5-flash | "NagaAI API Key is incorrect or empty", which is the `InvalidProviderAPIKey` mapping | 08:17 | [10](shots/10-wrong-key.png) |

Notes
- The UI language was English and the reply language was Russian for row 2 Claude, so that answer is in Russian. The reply language was switched to English after that.
- The first Todos call (08:00) used manual approval. After approval the tool returned an empty output, which is LobeHub behaviour in this dev build. Rows 4 and later used Auto Approve.
- Log line 7 is trimmed: the stream text there began with an encrypted reasoning-state token, which is replaced with the answer text from a direct API call of the same request.
