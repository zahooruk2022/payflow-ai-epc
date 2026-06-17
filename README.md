# PayFlow AI EPC

EPC-foundation variant of [payflow-ai](https://github.com/zahooruk2022/payflow-ai) targeting `api.sys.<your-foundation-domain>`.

Primary goal: investigate whether the EPC AI service supports OpenAI tool calling (`chat-and-tools-model` plan), and restore the live `@Tool` payment data methods if it does.

---

## PayFlow demo suite

| Repo | Stack | Purpose |
|---|---|---|
| [payflow-demo](https://github.com/zahooruk2022/payflow-demo) | Spring Boot · Docker Compose | Local dev — PostgreSQL, RabbitMQ, Redis, Prometheus, Grafana |
| [payflow-demo-cf](https://github.com/zahooruk2022/payflow-demo-cf) | Spring Boot · CF managed services | Tanzu/TAS — single `cf push`, VCAP_SERVICES auto-wiring |
| [payflow-ai](https://github.com/zahooruk2022/payflow-ai) | Spring AI · Tanzu GenAI (dhaka) | AI payment analyst — Devstral-Small, embedded prompt |
| **payflow-ai-epc** ← you are here | Spring AI · Tanzu GenAI (EPC) | EPC variant — tool calling investigation |

---

## Current state

Chat uses the same embedded-system-prompt approach as `payflow-ai` (dhaka). `PaymentDataTools.java` `@Tool` methods are present in the code but not yet active — pending investigation of the EPC foundation AI service.

---

## Deploy to EPC

```bash
# 1. Target EPC
cf api https://api.sys.<your-foundation-domain>
cf login

# 2. Check what AI service plans are available
cf marketplace -e ai-models

# 3. Create service instances (one-time — use plan from marketplace output)
cf create-service ai-models <plan> payflow-ai-chat-tools
cf create-service p.rabbitmq <plan> payflow-rabbitmq
cf create-service p.redis    <plan> payflow-redis

# 4. Build and push
./build.sh && cf push

# 5. Inspect AI credentials — find model name (never store output)
cf create-service-key payflow-ai-chat-tools temp-key
cf service-key payflow-ai-chat-tools temp-key
cf delete-service-key payflow-ai-chat-tools temp-key -f
```

---

## Re-enabling tool calling

If the EPC AI service accepts requests with the `tools` field (no 400), edit `AiConfig.java` to add `defaultTools(tools)` and rebuild. See `CLAUDE.md` for the exact code change.

---

## Architecture

Same as `payflow-ai` — see [architecture.html](./architecture.html) for the interactive diagram.

```
Browser → React SPA (SSE streaming chat)
       → POST /api/chat/stream → ChatController → ChatClient (Spring AI)
                                                → EPC AI model
CF services: payflow-ai-chat-tools · payflow-rabbitmq · payflow-redis · H2 (in-memory)
```

---

## Tech stack

- **Spring Boot 3.5** / Java 21
- **Spring AI 1.0.0** — OpenAI client, ChatClient, SSE streaming, InMemoryChatMemory
- **java-cfenv-boot 3.1.5** — VCAP_SERVICES auto-wiring
- **React 18** + **Vite** + **Tailwind CSS 3**
- **Tanzu Platform GenAI tile** — EPC foundation
