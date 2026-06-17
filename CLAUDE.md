# CLAUDE.md — PayFlow AI EPC

EPC-foundation variant of `payflow-ai`. Same Spring AI + React stack targeting
`api.sys.tpcf.tnz-field-epc.lvn.broadcom.net`.

Primary investigation goal: determine whether the EPC AI service (`chat-and-tools-model` plan)
supports the OpenAI `tools` field. If it does, re-activate `PaymentDataTools` for live tool calling.
If not, the same embedded-prompt approach as `payflow-ai` (dhaka) applies.

GitHub: https://github.com/zahooruk2022/payflow-ai-epc

---

## Commands

```bash
# Target EPC foundation
cf api https://api.sys.tpcf.tnz-field-epc.lvn.broadcom.net
cf login

# Check available AI service plans on EPC
cf marketplace -e ai-models

# Create service instances (one-time — adjust plans to what marketplace shows)
cf create-service ai-models <plan> payflow-ai-chat-tools
cf create-service p.rabbitmq <plan> payflow-rabbitmq
cf create-service p.redis    <plan> payflow-redis

# Build and deploy
./build.sh && cf push

# Inspect AI credentials (never store output)
cf create-service-key payflow-ai-chat-tools temp-key
cf service-key payflow-ai-chat-tools temp-key
cf delete-service-key payflow-ai-chat-tools temp-key -f
```

---

## Investigation checklist

- [ ] `cf marketplace -e ai-models` — what plans and models are available on EPC?
- [ ] Inspect service key — what model name is in the binding?
- [ ] Does basic chat (no tools) work on EPC?
- [ ] Does the AI endpoint accept the `tools` field without 400?
- [ ] If tools supported: re-add `defaultTools(paymentDataTools)` to `AiConfig` and rebuild

---

## Re-enabling tool calling

If the EPC foundation AI service supports tool calling, update `AiConfig.java`:

```java
// Add PaymentDataTools parameter and register with ChatClient
@Bean
public ChatClient chatClient(ChatClient.Builder builder, PaymentDataTools tools) {
    return builder
            .defaultSystem(SYSTEM_PROMPT)
            .defaultTools(tools)   // ← add this line
            .defaultAdvisors(MessageChatMemoryAdvisor.builder(
                    MessageWindowChatMemory.builder()
                            .chatMemoryRepository(new InMemoryChatMemoryRepository())
                            .build()).build())
            .build();
}
```

Then rebuild and push: `./build.sh && cf push`

---

## Key files

| File | Purpose |
|---|---|
| `backend/src/main/java/.../config/AiConfig.java` | ChatClient — system prompt, no tools (pending investigation) |
| `backend/src/main/java/.../config/GenAiVcapPostProcessor.java` | VCAP_SERVICES `ai-models` → Spring AI base-url + api-key |
| `backend/src/main/java/.../config/RedisSslFixPostProcessor.java` | java-cfenv-boot 3.1.x Redis SSL compat fix |
| `backend/src/main/java/.../tools/PaymentDataTools.java` | 6 @Tool methods — ready to re-activate |
| `backend/src/main/java/.../controller/ChatController.java` | POST /api/chat/stream (SSE) |
| `backend/src/main/resources/application.yml` | Model, H2 DB, services, retry disabled |
| `manifest.yml` | CF manifest — app name `payflow-ai-epc`, binds 3 services |

---

## Known issues on EPC (to investigate)

- Chat not working — root cause unknown; investigate basic connectivity, model name, and tools field support
- EPC GoRouter occasionally returns transient 404 on `cf push`. Workaround:
  `cf droplets payflow-ai-epc` → `cf set-droplet payflow-ai-epc <guid>` → `cf start payflow-ai-epc`
- Previous payflow-demo-cf deployment on EPC used RabbitMQ at
  `rmq-4bf2a6d7-619b-4011-bcdc-44ede175fd93.sys.tpcf.tnz-field-epc.lvn.broadcom.net`

---

## Difference from payflow-ai (dhaka)

| | payflow-ai (dhaka) | payflow-ai-epc (EPC) |
|---|---|---|
| Foundation | api.sys.dhaka.cf-app.com | api.sys.tpcf.tnz-field-epc.lvn.broadcom.net |
| AI plan | GTO-models | chat-and-tools-model (TBC) |
| Tool calling | ❌ (proxy limitation) | Under investigation |
| App name | payflow-ai | payflow-ai-epc |

---

## Spring AI / streaming notes (same as payflow-ai)

- **ChatClient** uses compact system prompt (~150 tokens) and `MessageChatMemoryAdvisor`.
- **SSE streaming**: `SseEmitter` + `Flux<String>.toIterable()`. Commits HTTP 200 before AI call. Keepalive every 15 s for CF GoRouter.
- **Retry disabled**: `spring.ai.retry.max-attempts: 1`.
- **GenAiVcapPostProcessor** maps VCAP_SERVICES AI credentials to Spring AI properties.
- **RedisSslFixPostProcessor** patches java-cfenv-boot 3.1.x SSL String→Boolean incompatibility.
