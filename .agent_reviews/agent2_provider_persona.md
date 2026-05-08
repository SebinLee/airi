# Agent 2 보고서: AI Provider 설정 및 Persona/Memory/Personality 기능 분석

## 요약 결론

- 이 저장소의 Stage UI/앱 런타임은 **Grok-only가 아니다**. 채팅 LLM provider registry에는 OpenRouter, OpenAI, Anthropic/Claude, Google Gemini, Groq, xAI(Grok 계열), Ollama, DeepSeek, Mistral, Perplexity, Together, Fireworks, Cerebras, NVIDIA, LM Studio, OpenAI-compatible 등 다수 provider가 등록되어 있다.
- xAI/Grok 지원은 `packages/stage-ui/src/libs/providers/providers/xai/index.ts`에 있고, Groq는 별도 `packages/stage-ui/src/libs/providers/providers/groq/index.ts`이다. 이름이 비슷하지만 **xAI는 Grok 모델 API provider**, **Groq는 GroqCloud/OpenAI-compatible provider**로 분리되어 있다.
- 현재 provider 체계는 두 층이 공존한다.
  - 새 정의 기반 registry: `packages/stage-ui/src/libs/providers/providers/registry.ts`, `packages/stage-ui/src/libs/providers/providers/index.ts`, 각 provider 하위 `index.ts`.
  - 레거시/통합 Pinia runtime store: `packages/stage-ui/src/stores/providers.ts`.
- 실제 채팅 LLM 호출은 `packages/stage-ui/src/stores/chat.ts`가 메시지를 조립하고, `packages/stage-ui/src/stores/llm.ts`가 `@proj-airi/core-agent`의 `coreStreamFrom()`로 provider stream을 실행하는 구조다.
- persona/personality는 주로 **AIRI card**에 저장된다. 핵심 store는 `packages/stage-ui/src/stores/modules/airi-card.ts`이고, UI는 `packages/stage-pages/src/pages/settings/airi-card/...` 아래 dialog/page들이 담당한다.
- 메모리 기능은 일부 이름/노트북/컨텍스트 구조는 있으나, 사용자 설정 페이지의 장기/단기 메모리는 현재 WIP다. `packages/stage-pages/src/pages/settings/memory/index.vue`, `packages/stage-pages/src/pages/settings/modules/memory-long-term.vue`, `packages/stage-pages/src/pages/settings/modules/memory-short-term.vue`는 모두 `WIP`만 렌더링한다.
- broadcast/content 기반으로 **persona 자체를 자동 수정하는 경로는 확인되지 않았다**. 다만 `context:update`/BroadcastChannel로 들어온 외부 context가 prompt에 붙어서 응답을 바꾸는 경로는 존재한다. 또한 `spark:notify`의 `spark:command.guidance.persona`는 downstream agent에 전달되는 일회성 persona hint에 가깝고, AIRI card의 `personality`/`systemPrompt`를 갱신하지는 않는다.

## Provider 지원 범위: Grok-only 여부

### 다중 LLM 지원 근거

Provider import registry는 `packages/stage-ui/src/libs/providers/providers/index.ts`에 있다. 여기에는 다음이 포함된다.

- OpenRouter: `./openrouter-ai`
- OpenAI: `./openai`
- Anthropic: `./anthropic`
- Google Gemini: `./google-generative-ai`
- xAI: `./xai`
- Groq: `./groq`
- Ollama: `./ollama`
- DeepSeek, Cerebras, Together, Fireworks, Mistral, Moonshot, ModelScope, LM Studio, Cloudflare Workers AI, Azure OpenAI/Azure AI Foundry 등 다수

따라서 현재 구조는 특정 Grok provider만을 위한 코드가 아니라 **provider-agnostic registry + OpenAI-compatible 계열 adapter 중심 구조**다.

### 주요 provider 예시

- OpenAI provider: `packages/stage-ui/src/libs/providers/providers/openai/index.ts`
  - `id: 'openai'`, 기본 base URL `https://api.openai.com/v1`, `createOpenAI()` 사용.
- Anthropic provider: `packages/stage-ui/src/libs/providers/providers/anthropic/index.ts`
  - `id: 'anthropic'`, Claude 모델 목록을 `extraMethods.listModels`로 하드코딩 제공.
- Google Gemini provider: `packages/stage-ui/src/libs/providers/providers/google-generative-ai/index.ts`
  - `id: 'google-generative-ai'`, OpenAI-compatible endpoint `https://generativelanguage.googleapis.com/v1beta/openai/` 사용.
- xAI/Grok 계열 provider: `packages/stage-ui/src/libs/providers/providers/xai/index.ts`
  - `id: 'xai'`, 기본 base URL `https://api.x.ai/v1/`, `createXai()` 사용.
- Groq provider: `packages/stage-ui/src/libs/providers/providers/groq/index.ts`
  - `id: 'groq'`, 기본 base URL `https://api.groq.com/openai/v1/`, `createOpenAI()` 사용.
- Ollama provider: `packages/stage-ui/src/libs/providers/providers/ollama/index.ts`
  - `id: 'ollama'`, 기본 base URL `http://localhost:11434/v1/`, thinking mode 옵션 포함.

## Provider registry/settings/live runtime 위치

### Registry/definition 계층

- `packages/stage-ui/src/libs/providers/providers/registry.ts`
  - `providerRegistry` Map을 보유.
  - `defineProvider()`가 provider definition을 등록.
  - `listProviders()`가 `order`, `name` 기준으로 정렬해 반환.
  - `getDefinedProvider(id)`가 단일 definition 조회.
- `packages/stage-ui/src/libs/providers/providers/index.ts`
  - 각 provider 모듈을 import해서 side effect로 registry에 등록.
  - 새 provider 추가 시 이 파일에 import를 추가해야 registry에 실린다.
- `packages/stage-ui/src/libs/providers/types.ts`
  - `ProviderDefinition`, validator 타입, `ProviderValidationCheck`, provider extra methods/model/voice 타입 정의.
- `packages/stage-ui/src/libs/providers/validators/*`
  - OpenAI-compatible connectivity/model-list/chat-completions validation 실행 계층.

### Pinia settings/runtime store

- `packages/stage-ui/src/stores/providers.ts`
  - 사용자 provider credentials 저장 키: `settings/credentials/providers`.
  - 추가 표시 여부 저장 키: `settings/providers/added`.
  - `providerMetadata` 레거시 metadata와 새 registry definition을 병합.
  - `convertProviderDefinitionsToMetadata()`로 `defineProvider()` 기반 definition을 레거시 `ProviderMetadata`로 변환.
  - `validateProvider()`, `fetchModelsForProvider()`, `getProviderInstance()`, `disposeProviderInstance()` 제공.
- `packages/stage-ui/src/stores/providers/converters.ts`
  - `ProviderDefinition` → `ProviderMetadata` 변환.
  - zod schema default 추출, listModels/listVoices adapter, validator plan 실행 로직 연결.
- `packages/stage-ui/src/stores/provider-catalog.ts`
  - v2 provider catalog store.
  - `defs = listProviders()`, remote `/api/v1/providers` 및 local repo를 함께 사용.
  - provider config add/remove/commit 담당.
- `packages/stage-ui/src/database/repos/providers.repo.ts`
  - local-first provider catalog 저장소. storage key는 `local:providers`.

### 설정 UI

- Provider 목록/카테고리 UI: `packages/stage-pages/src/pages/settings/providers/index.vue`
  - chat/speech/transcription/artistry provider metadata를 store에서 읽어 표시.
- 구 provider chat 설정 화면: `packages/stage-pages/src/pages/settings/providers/chat/[providerId].vue`
  - `useProvidersStore().providers`의 `apiKey`, `baseUrl`을 직접 수정.
  - `useProviderValidation(providerId)`로 validation/force valid/model selection 연결.
- v2 provider edit 화면: `packages/stage-pages/src/pages/v2/settings/providers/edit/[providerId]/index.vue`
  - `getDefinedProvider()`, provider zod schema meta를 읽어 generic form을 구성.
  - `useProviderCatalogStore()`의 `commitProviderConfig()`로 저장.
- LLM module 선택 UI: `packages/stage-pages/src/pages/settings/modules/consciousness.vue`
  - `useConsciousnessStore()`의 `activeProvider`, `activeModel`을 설정.
- onboarding provider 선택/설정: `packages/stage-ui/src/components/scenarios/dialogs/onboarding/step-provider-selection.vue`, `packages/stage-ui/src/components/scenarios/dialogs/onboarding/step-provider-configuration.vue`.

### 실제 live chat/LLM 호출 경로

- `packages/stage-ui/src/stores/modules/consciousness.ts`
  - active chat provider/model 저장.
  - localStorage keys: `settings/consciousness/active-provider`, `settings/consciousness/active-model`, `settings/consciousness/active-custom-model`.
- `packages/stage-ui/src/stores/chat.ts`
  - 사용자 메시지 ingest/queue 처리.
  - session messages를 가져와 datetime prefix와 runtime context를 붙임.
  - `llmStore.stream(options.model, options.chatProvider, newMessages, ...)` 호출.
- `packages/stage-ui/src/stores/llm.ts`
  - `useLLM().stream()`이 `coreStreamFrom()` 호출.
  - tools compatibility/content-array compatibility fallback 보유.
  - built-in tools로 MCP/debug/spark command tool을 합쳐 provider stream에 전달.
- `packages/core-agent/src/runtime/llm-service.ts`, `packages/core-agent/src/contracts/llm-port.ts`, `packages/core-agent/src/messages/render-provider-chat.ts`
  - provider chat rendering/stream abstraction이 있는 core-agent 계층.
- Stage Web entry: `apps/stage-web/src/pages/index.vue`
  - active provider/model을 읽고 `providersStore.getProviderInstance()` 후 `chatStore.ingest()` 호출.
- Stage Tamagotchi/Electron sync entry: `apps/stage-tamagotchi/src/renderer/stores/chat-sync.ts`
  - active provider/model을 읽고 provider instance를 가져와 chat ingest.
- plugin/context bridge entry: `packages/stage-ui/src/stores/mods/api/context-bridge.ts`
  - `input:text` websocket event를 받으면 active provider/model로 `chatOrchestrator.ingest()` 실행.

### Server-side provider persistence/gateway

- Provider config DB schema: `apps/server/src/schemas/providers.ts`
  - `user_provider_configs`, `system_provider_configs`.
- Provider service: `apps/server/src/services/providers.ts`
  - user/system provider config CRUD.
- Provider HTTP routes: `apps/server/src/routes/providers/index.ts`, request schema `apps/server/src/routes/providers/schema.ts`.
- OpenAI-compatible hosted gateway route: `apps/server/src/routes/openai/v1/index.ts`
  - auth/billing/rate-limit 후 `env.GATEWAY_BASE_URL`의 `/chat/completions`, audio endpoints로 proxy.
  - `model: 'auto'`면 `env.DEFAULT_CHAT_MODEL`로 대체.

## Provider를 추가/수정할 위치

### 새 chat LLM provider 추가 시

1. 새 provider definition 파일 추가
   - 위치 예: `packages/stage-ui/src/libs/providers/providers/<new-provider>/index.ts`.
   - 기존 예시: `openai/index.ts`, `xai/index.ts`, `ollama/index.ts`, `anthropic/index.ts`.
2. `defineProvider()`로 등록
   - `id`, `name`, `description`, `tasks: ['chat']`, icon, `createProviderConfig()`, `createProvider()`, validators 설정.
3. `packages/stage-ui/src/libs/providers/providers/index.ts`에 `import './<new-provider>'` 추가.
4. 필요하면 i18n key 추가
   - 기존 provider들이 `settings.pages.providers.provider.<id>.title/description` 패턴을 사용.
5. validation/listModels 특수 처리가 필요하면
   - `packages/stage-ui/src/libs/providers/validators/*` 또는 provider definition의 `extraMethods.listModels`/custom validators 추가.
6. 구 provider settings 화면만으로 충분하지 않으면
   - generic v2 schema-driven 화면은 `packages/stage-pages/src/pages/v2/settings/providers/edit/[providerId]/index.vue`가 zod meta를 읽는다.
   - 구 custom 화면이 필요하면 `packages/stage-pages/src/pages/settings/providers/chat/<provider>.vue` 계열을 참고.

### speech/transcription provider 수정 시

- 아직 레거시 provider metadata가 `packages/stage-ui/src/stores/providers.ts`에 많이 남아 있다.
- OpenAI-compatible speech/transcription helper: `packages/stage-ui/src/stores/providers/openai-compatible-builder.ts`.
- 개별 speech/transcription settings pages: `packages/stage-pages/src/pages/settings/providers/speech/*`, `packages/stage-pages/src/pages/settings/providers/transcription/*`.

## Persona/personality/system prompt 구조

### AIRI card store

- 핵심 store: `packages/stage-ui/src/stores/modules/airi-card.ts`
  - localStorage keys:
    - `airi-cards`: card Map 저장.
    - `airi-card-active-id`: 활성 card id.
  - `AiriCard`는 기본 `Card`에 `extensions.airi`를 붙인 구조.
  - `AiriExtension.modules.consciousness`가 card별 LLM provider/model을 저장.
  - `AiriExtension.modules.speech`가 card별 TTS provider/model/voice를 저장.
  - `AiriExtension.modules.artistry`가 이미지/scene 관련 prompt와 autonomous 설정 저장.
- card import/normalization
  - `newAiriCard()`는 ccv3 card의 `data.personality`, `data.scenario`, `data.system_prompt`, `data.post_history_instructions`, `data.first_mes`, `data.mes_example` 등을 AIRI card 필드로 변환.
- default card
  - `initialize()`가 기본 card `ReLU`를 만든다.
  - 기본 description에는 `SystemPromptV2(t('base.prompt.prefix'), t('base.prompt.suffix')).content`가 들어간다.
- 최종 system prompt 조립
  - `systemPrompt` computed가 `card.systemPrompt`, `card.description`, `card.personality`, `card.extensions?.airi?.modules?.artistry?.widgetInstruction`를 `\n\n`로 결합한다.
  - 즉 현재 runtime의 기본 persona는 card의 `systemPrompt + description + personality + artistry widget instruction` 합성물이다.

### prompt 상수

- `packages/stage-ui/src/constants/prompts/system-v2.ts`
  - emotion 목록을 포함하는 system message factory.
- `packages/stage-ui/src/constants/prompts/character-defaults.ts`
  - artistry widget spawning instruction, image journal/scene control instruction 기본값.

### Character runtime store

- `packages/stage-ui/src/stores/character/index.ts`
  - active AIRI card의 `systemPrompt`와 `name`을 expose.
  - `emitTextOutput()`와 spark notify reaction streaming에서 LLM marker parser와 speech runtime intent를 사용해 TTS로 흘린다.
- `packages/stage-ui/src/stores/character/orchestrator/store.ts`
  - `spark:notify` event를 처리할 때 `getSystemPrompt: () => systemPrompt.value`를 core-agent spark notify handler에 전달.
  - active provider/model은 `useConsciousnessStore()`에서 가져오고, provider instance는 `useProvidersStore().getProviderInstance()`로 가져온다.
- `packages/core-agent/src/agents/spark-notify/handler.ts`
  - spark notify system message는 `deps.getSystemPrompt()` + `getSparkNotifyHandlingAgentInstruction(...)` + optional override instructions로 구성된다.

### AIRI card 설정 UI

- Card list/import/activation page: `packages/stage-pages/src/pages/settings/airi-card/index.vue`.
- Card create/edit dialog: `packages/stage-pages/src/pages/settings/airi-card/components/CardCreationDialog.vue`.
  - behavior tab에서 `personality`, `scenario`, greetings를 편집.
  - settings/system 쪽에서 `systemPrompt`, `postHistoryInstructions` 편집.
  - modules tab에서 card별 consciousness provider/model, speech provider/model/voice, display model, artistry prompt/options를 저장.
- Card detail dialog: `packages/stage-pages/src/pages/settings/airi-card/components/CardDetailDialog.vue`.
  - `personality`, `scenario`, `systemPrompt`, `postHistoryInstructions`를 character tab에서 표시.
  - 활성 card 전환은 `activeCardId` 변경.
- Profile popover: `packages/stage-ui/src/components/misc/profile-switcher-popover.vue`.
  - card/profile 전환 및 간단 생성 흐름.

### Server-side character/persona schema

- Server DB schema: `apps/server/src/schemas/characters.ts`
  - `character_i18n`에 `systemPrompt`, `personality`, `initialMemories` TODO 주석이 있다.
  - 실제 prompt 저장은 `character_prompts` table이 담당하며 type은 `'system' | 'personality' | 'greetings'`.
- Server character route schema: `apps/server/src/routes/characters/schema.ts`
  - `prompts` 배열 생성 schema 포함.
- Client character type: `packages/stage-ui/src/types/character.ts`
  - `CharacterPromptSchema`와 `PromptTypeSchema`가 `system/personality/greetings`를 반영.
- Client character local repo/store: `packages/stage-ui/src/database/repos/characters.repo.ts`, `packages/stage-ui/src/stores/characters.ts`.
  - server/local-first 캐시로 character relation을 다루지만, live AIRI card persona store(`airi-card.ts`)와는 별도 계층이다.

## Memory 기능 상태

### 구현된/부분 구현된 memory-like 구조

- Character notebook store: `packages/stage-ui/src/stores/character/notebook.ts`
  - `entries`: `note | diary | focus`.
  - `tasks`: scheduled task/reminder.
  - `getDueTasks()`가 due reminder를 반환.
- Character orchestrator task reminder: `packages/stage-ui/src/stores/character/orchestrator/store.ts`
  - notebook due task를 `spark:notify`로 변환해 character에 알림.
- Runtime context store: `packages/stage-ui/src/stores/chat/context-store.ts`
  - `context:update`나 input context update를 `activeContexts`와 `contextHistory`에 저장.
  - 전략은 `ContextUpdateStrategy.ReplaceSelf` / `AppendSelf`.
  - prompt 조립 시 context snapshot이 마지막 user message에 붙는다.
- Context prompt rendering: `packages/stage-ui/src/stores/chat/context-prompt.ts`
  - context snapshot을 `[Context]` bullet list로 변환.

### WIP인 memory settings

- `packages/stage-pages/src/pages/settings/memory/index.vue`: `WIP`만 렌더링.
- `packages/stage-pages/src/pages/settings/modules/memory-long-term.vue`: `WIP`만 렌더링.
- `packages/stage-pages/src/pages/settings/modules/memory-short-term.vue`: `WIP`만 렌더링.
- 별도 pgvector package는 존재한다: `packages/memory-pgvector/src/index.ts`, `packages/memory-pgvector/package.json`. 그러나 Stage UI의 memory settings에 직접 연결된 완성 UI는 확인되지 않았다.

## Broadcast/content 기반 persona update 여부

### 존재하는 broadcast/context 경로

- Context bridge: `packages/stage-ui/src/stores/mods/api/context-bridge.ts`
  - BroadcastChannel names:
    - `airi-context-update` via `CONTEXT_CHANNEL_NAME`.
    - `airi-chat-stream` via `CHAT_STREAM_CHANNEL_NAME`.
    - `airi-spark-notify-bridge` for spark notify reaction delegation.
  - server `context:update` event를 받으면 `chatContext.ingestContextMessage()` 후 broadcast로 다른 window/tab에 공유.
  - `input:text` event의 `contextUpdates`도 chat context store에 ingest.
  - `input:text` event는 active provider/model로 chat ingest를 실행.
- Chat context prompt injection: `packages/stage-ui/src/stores/chat.ts`
  - `chatContext.getContextsSnapshot()` → `formatContextPromptText()` → 최신 user message 끝에 context text part 추가.
  - 이는 persona 파일을 바꾸는 것이 아니라 **해당 turn의 prompt context를 바꾸는 방식**이다.
- Broadcast observation/devtools: `packages/stage-pages/src/pages/devtools/context-flow/index.vue`, `packages/stage-ui/src/stores/devtools/context-observability.ts`.

### spark:notify persona guidance

- `packages/core-agent/src/agents/spark-notify/schema.ts`
  - `sparkNotifyCommandGuidanceSchema`에 `persona` 배열이 있다.
  - 항목은 `{ traits, strength }` 형태.
- `packages/core-agent/src/agents/spark-notify/tools.ts`
  - 이 persona 배열을 `Record<string, strength>`로 정규화해 `spark:command` draft의 `guidance.persona`로 전달.
- `packages/stage-ui/src/stores/character/orchestrator/store.ts`
  - spark notify 처리 결과 command를 `modsServerChannelStore.send({ type: 'spark:command', ... })`로 전송.

판단: 이 `guidance.persona`는 command receiver가 참고할 수 있는 **일회성/하위 에이전트용 행동 힌트**다. `packages/stage-ui/src/stores/modules/airi-card.ts`의 card `personality`/`systemPrompt`를 수정하거나 localStorage `airi-cards`를 갱신하는 코드는 아니다.

### persona 자체 자동 갱신 경로 판단

- `useAiriCardStore().updateCard()`는 명시적 UI/카드 조작 경로에서 호출된다. 예: `packages/stage-pages/src/pages/settings/airi-card/components/CardCreationDialog.vue`, `packages/stage-pages/src/pages/settings/scene/index.vue`, image journal background 설정 등.
- `context:update`, BroadcastChannel, `input:text.contextUpdates`, `spark:notify` 검색 경로에서는 AIRI card의 `personality` 또는 `systemPrompt`를 content 기반으로 업데이트하는 로직을 확인하지 못했다.
- 따라서 현재 구현 기준으로는 **broadcast/content 기반 persona 영구 업데이트는 없음**으로 보는 것이 맞다. 대신 외부 context가 prompt에 주입되어 일시적으로 응답 스타일/내용에 영향을 줄 수 있다.

## 핵심 파일/디렉터리 색인

### Provider

- Registry core: `packages/stage-ui/src/libs/providers/providers/registry.ts`
- Provider import list: `packages/stage-ui/src/libs/providers/providers/index.ts`
- Provider type/validation contracts: `packages/stage-ui/src/libs/providers/types.ts`
- Provider definitions dir: `packages/stage-ui/src/libs/providers/providers/`
- Provider validators: `packages/stage-ui/src/libs/providers/validators/`
- Legacy/runtime provider store: `packages/stage-ui/src/stores/providers.ts`
- Definition → metadata bridge: `packages/stage-ui/src/stores/providers/converters.ts`
- OpenAI-compatible legacy builder: `packages/stage-ui/src/stores/providers/openai-compatible-builder.ts`
- Provider catalog store: `packages/stage-ui/src/stores/provider-catalog.ts`
- Local provider repo: `packages/stage-ui/src/database/repos/providers.repo.ts`
- Provider list/settings UI: `packages/stage-pages/src/pages/settings/providers/`
- v2 provider settings UI: `packages/stage-pages/src/pages/v2/settings/providers/`
- Consciousness provider/model selection: `packages/stage-pages/src/pages/settings/modules/consciousness.vue`
- Runtime chat orchestrator: `packages/stage-ui/src/stores/chat.ts`
- LLM stream store: `packages/stage-ui/src/stores/llm.ts`
- Server provider configs: `apps/server/src/schemas/providers.ts`, `apps/server/src/services/providers.ts`, `apps/server/src/routes/providers/index.ts`
- Server OpenAI-compatible gateway: `apps/server/src/routes/openai/v1/index.ts`

### Persona / memory / prompts

- AIRI card/persona store: `packages/stage-ui/src/stores/modules/airi-card.ts`
- Active character runtime store: `packages/stage-ui/src/stores/character/index.ts`
- Character notebook/task memory-like store: `packages/stage-ui/src/stores/character/notebook.ts`
- Character spark notify orchestrator: `packages/stage-ui/src/stores/character/orchestrator/store.ts`
- Spark notify core agent: `packages/core-agent/src/agents/spark-notify/handler.ts`, `packages/core-agent/src/agents/spark-notify/schema.ts`, `packages/core-agent/src/agents/spark-notify/tools.ts`
- Prompt constants: `packages/stage-ui/src/constants/prompts/system-v2.ts`, `packages/stage-ui/src/constants/prompts/character-defaults.ts`
- Chat context store/prompt: `packages/stage-ui/src/stores/chat/context-store.ts`, `packages/stage-ui/src/stores/chat/context-prompt.ts`
- Context bridge/broadcast: `packages/stage-ui/src/stores/mods/api/context-bridge.ts`, `packages/stage-ui/src/stores/chat/constants.ts`
- AIRI card settings UI: `packages/stage-pages/src/pages/settings/airi-card/`
- Memory settings WIP pages: `packages/stage-pages/src/pages/settings/memory/index.vue`, `packages/stage-pages/src/pages/settings/modules/memory-long-term.vue`, `packages/stage-pages/src/pages/settings/modules/memory-short-term.vue`
- Server character prompt schema: `apps/server/src/schemas/characters.ts`, `apps/server/src/routes/characters/schema.ts`
- Client character prompt type/repo/store: `packages/stage-ui/src/types/character.ts`, `packages/stage-ui/src/database/repos/characters.repo.ts`, `packages/stage-ui/src/stores/characters.ts`

## 검증 메모

- `omx explore`는 로컬에서 `cargo`가 없어 실패했다. 이후 `find`, `grep`, `sed`로 repository 파일을 직접 확인했다.
- 코드 변경은 하지 않았고, 요청된 보고서 파일 `.agent_reviews/agent2_provider_persona.md`만 작성했다.
