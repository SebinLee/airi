# Agent 3 상호작용 경로 분석 보고서

## 0. 범위와 결론

이 보고서는 AIRI 저장소에서 **채팅 입력 수집**, **라이브/외부 플랫폼 연동**, **메시지 큐와 저장소**, **LLM 응답·TTS·액션 실행**, **후원/도네이션 지원 여부**를 코드 기준으로 추적한 결과다. 분석은 읽기 전용으로 수행했고, 산출물 파일(`.agent_reviews/agent3_interactions.md`)만 작성했다.

핵심 결론은 다음과 같다.

- Stage 계열 앱의 기본 채팅 경로는 `InteractiveArea`/음성 입력 → `chat-sync` 또는 `context-bridge` → `packages/stage-ui/src/stores/chat.ts` → LLM 스트리밍 → 세션 저장/훅/TTS/액션으로 이어진다.
- 외부 플랫폼은 **Discord**, **Telegram**, **Satori 호환 플랫폼**이 실제 메시지 수집·응답 루프를 가진다. **Bilibili 라이브 채팅 플러그인 패키지**는 존재하지만 구현 파일은 `console.warn('WIP')`뿐이라 현재 동작 경로로 보기는 어렵다.
- 서버형 채팅 동기화는 `/api/v1/chats` REST, `/ws/chat` WebSocket(Eventa RPC), DB `messages` 시퀀스, Redis Pub/Sub를 사용한다.
- 후원/도네이션/슈퍼챗/비트/기프트 처리 경로는 확인되지 않았다. Stripe/Flux 결제·구독·청구 경로는 있지만, 이는 사용량 과금/크레딧 구매용이며 실시간 도네이션 이벤트 처리와는 별개다.

## 1. Stage 앱의 로컬 채팅 입력 경로

### 1.1 텍스트·이미지 입력

Desktop renderer의 기본 입력 컴포넌트는 `apps/stage-tamagotchi/src/renderer/components/InteractiveArea.vue`다.

- 사용자가 텍스트 또는 첨부 이미지를 입력하면 `handleSend`가 값을 검증하고 입력 UI를 비운 뒤 `chatSyncStore.requestIngest({ text, attachments, toolset: 'artistry' })`를 호출한다 (`apps/stage-tamagotchi/src/renderer/components/InteractiveArea.vue:72-95`).
- 붙여넣기된 이미지 파일은 base64 `data:` URL 첨부로 변환되어 같은 ingest payload에 포함된다 (`apps/stage-tamagotchi/src/renderer/components/InteractiveArea.vue:164-180`).
- 채팅 표시 UI는 같은 컴포넌트에서 `ChatHistory`를 렌더링한다 (`apps/stage-tamagotchi/src/renderer/components/InteractiveArea.vue:217-224`).

`chat-sync`는 Electron 다중 창/권한 창 간 채팅 명령을 동기화한다.

- ingest 명령 payload는 `text`, `attachments`, `toolset`, `sessionId`, `providerId`, `modelId`를 가질 수 있다 (`apps/stage-tamagotchi/src/renderer/stores/chat-sync.ts:20-46`).
- 권한 창(authority)에서는 `executeIngest`가 활성 provider/model을 해석한 뒤 `chatOrchestrator.ingest`를 호출한다 (`apps/stage-tamagotchi/src/renderer/stores/chat-sync.ts:302-321`).
- follower 창에서는 BroadcastChannel 명령으로 authority 창에 ingest를 위임한다 (`apps/stage-tamagotchi/src/renderer/stores/chat-sync.ts:436-549`).

### 1.2 음성 입력과 ASR

Web과 Desktop 모두 음성 입력을 텍스트로 변환한 뒤 동일한 채팅 ingest 경로로 흘려보낸다.

- Web 홈 화면은 녹음 종료 시 `transcribeForRecording`으로 텍스트를 얻고 `chatStore.ingest(text, { model, chatProvider })`를 호출한다 (`apps/stage-web/src/pages/index.vue:77-88`).
- Web VAD는 음성 시작/종료 시 녹음 시작·중단을 트리거한다 (`apps/stage-web/src/pages/index.vue:99-115`).
- Desktop 홈 화면은 streaming sentence end에서 캡션을 갱신하고 `chatSyncStore.requestIngest({ text: finalText })`를 호출한다 (`apps/stage-tamagotchi/src/renderer/pages/index.vue:254-271`).
- Desktop non-streaming 녹음 종료 경로도 `transcribeForRecording` 후 `requestIngest({ text })`로 합류한다 (`apps/stage-tamagotchi/src/renderer/pages/index.vue:350-376`).
- 공통 hearing store는 Web Speech API 또는 provider streaming을 처리하고, 녹음 파일은 `recording.wav`로 만들어 `hearingStore.transcription`에 전달한다 (`packages/stage-ui/src/stores/modules/hearing.ts:538-855`).

## 2. 채팅 오케스트레이터, 큐, 세션 저장소

### 2.1 입력 큐와 LLM 호출 준비

핵심 오케스트레이터는 `packages/stage-ui/src/stores/chat.ts`다.

- `QueuedSend` 타입과 `pendingQueuedSends`, `sendQueue = createQueue<QueuedSend>`가 사용자 ingest를 직렬화한다 (`packages/stage-ui/src/stores/chat.ts:87-158`).
- `performSend`는 빈 입력을 버리고, `ChatStreamEventContext`를 만든 뒤, before-compose 훅을 실행한다 (`packages/stage-ui/src/stores/chat.ts:160-221`).
- 이미지 첨부는 OpenAI 호환 content part의 `image_url`로 변환된다 (`packages/stage-ui/src/stores/chat.ts:223-238`).
- 기본 입력 이벤트 envelope은 `input:text`로 구성된다 (`packages/stage-ui/src/stores/chat.ts:239-246`).
- 사용자 메시지는 세션 저장소에 먼저 append된다 (`packages/stage-ui/src/stores/chat.ts:251-256`).
- 도구 호출 결과 조각은 별도 `toolCallQueue = createQueue<ChatSlices>`로 assistant message slice에 반영된다 (`packages/stage-ui/src/stores/chat.ts:321-338`).

### 2.2 컨텍스트·세션 저장

채팅 세션 저장은 Pinia store와 로컬 DB repo가 담당한다.

- `useChatSessionStore`는 `activeSessionId`, `sessionMessages`, `sessionMetas`, `sessionGenerations`, `index`를 관리한다 (`packages/stage-ui/src/stores/chat/session-store.ts:13-21`).
- `persistQueue = Promise.resolve()`와 `enqueuePersist`로 세션 저장을 순차화한다 (`packages/stage-ui/src/stores/chat/session-store.ts:28-47`).
- `persistSession`은 메시지/메타/세대 정보를 repo에 저장한다 (`packages/stage-ui/src/stores/chat/session-store.ts:122-151`).
- `appendSessionMessage`, `ensureActiveSessionForCharacter`, `messages` computed getter/setter가 세션 조작의 주요 API다 (`packages/stage-ui/src/stores/chat/session-store.ts:169-321`).
- 실제 저장 키는 `local:chat/index/${userId}`와 `local:chat/sessions/${sessionId}`다 (`packages/stage-ui/src/database/repos/chat-sessions.repo.ts:6-23`).

컨텍스트 저장소는 모듈/플랫폼이 주입한 외부 컨텍스트를 병합한다.

- `useChatContextStore`는 `activeContexts`와 `contextHistory`를 관리한다 (`packages/stage-ui/src/stores/chat/context-store.ts:28-30`).
- `ingestContextMessage`는 `ReplaceSelf`/`AppendSelf` 전략으로 source별 컨텍스트를 갱신하고, 히스토리를 최대 400개로 제한한다 (`packages/stage-ui/src/stores/chat/context-store.ts:26-87`).

### 2.3 LLM 스트리밍 결과의 구조

- core type의 `ChatAssistantMessage`는 일반 문자열 대신 `slices`, `tool_results`, 선택적 `categorization.speech/reasoning`을 포함할 수 있다 (`packages/core-agent/src/types/chat.ts:23-34`).
- `ChatStreamEventContext`와 `ChatStreamEvent`는 compose/send/token/assistant/complete 생명주기 훅에서 공유되는 이벤트 형태다 (`packages/core-agent/src/types/chat.ts:52-69`).
- `createChatHooks`는 before/after compose, before send, literal/special token, stream end, assistant response/message, turn complete 훅을 제공한다 (`packages/core-agent/src/runtime/agent-hooks.ts:6-16`, `packages/core-agent/src/runtime/agent-hooks.ts:121-169`).

## 3. LLM 응답, 도구 호출, TTS, 액션

### 3.1 LLM 스트리밍과 도구

- `chat.ts`는 컨텍스트 프롬프트를 최신 user message에 병합하고 lifecycle record를 작성한 뒤 LLM을 호출한다 (`packages/stage-ui/src/stores/chat.ts:367-419`).
- 실제 스트리밍 호출은 `llmStore.stream(...)`이며, tool-call/tool-result/tool-error/text-delta/reasoning-delta 이벤트를 처리한다 (`packages/stage-ui/src/stores/chat.ts:441-509`).
- 스트림 종료 후 parser를 닫고 assistant message를 세션에 append하며, stream/end/assistant/complete 훅을 순서대로 emit한다 (`packages/stage-ui/src/stores/chat.ts:518-544`).
- public `ingest`와 `ingestOnFork`는 send queue에 작업을 넣는 API다 (`packages/stage-ui/src/stores/chat.ts:559-594`).
- `packages/core-agent/src/runtime/llm-service.ts`는 `@xsai/stream-text`의 `streamText`를 사용하고, builtin/custom tools를 병합하며, provider로부터 오는 error role/content array를 sanitize한다 (`packages/core-agent/src/runtime/llm-service.ts:35-70`, `packages/core-agent/src/runtime/llm-service.ts:164-260`).
- `packages/stage-ui/src/stores/llm.ts`는 MCP, debug, spark command, active tools를 builtin tool로 등록한다 (`packages/stage-ui/src/stores/llm.ts:36-79`).

### 3.2 speech categorization과 TTS 재생

- 스트리밍 텍스트는 parser/categorizer를 통과하고, speech-only text는 `filterToSpeech`로 분리되어 assistant building message와 token hook에 반영된다 (`packages/stage-ui/src/stores/chat.ts:271-297`).
- TTS provider/model/voice 상태와 `generateSpeech` 호출은 speech store가 관리한다 (`packages/stage-ui/src/stores/modules/speech.ts:25-43`, `packages/stage-ui/src/stores/modules/speech.ts:215-231`).
- `Stage.vue`는 `createSpeechPipeline`을 만들고 provider guard, OpenAI-compatible fallback model/voice, `generateSpeech`, audio decode, playback manager를 연결한다 (`packages/stage-ui/src/components/scenes/Stage.vue:226-327`).
- 재생 시작 시 캡션·present 상태가 갱신되고, Live2D lip sync loop가 mouth open 값을 구동한다 (`packages/stage-ui/src/components/scenes/Stage.vue:342-430`).
- emotion/special token은 별도 emotion queue와 `onSpecial` 핸들러로 처리된다 (`packages/stage-ui/src/components/scenes/Stage.vue:125-159`, `packages/stage-ui/src/components/scenes/Stage.vue:329-332`).

## 4. 모듈 서버 채널과 input/output 이벤트 브리지

AIRI 플러그인/모듈은 server channel 이벤트를 통해 Stage 채팅에 들어온다.

- protocol type에서 `InputSource`는 `stage-web`, `stage-tamagotchi`, `discord`를 포함한다 (`packages/plugin-protocol/src/types/events.ts:465-469`).
- `input:text`, `input:text:voice`, `input:voice`는 consumer-group `chat-ingestion`으로 선언되어 있다 (`packages/plugin-protocol/src/types/events.ts:1107-1133`).
- server channel store의 possible events에는 `context:update`, `input:text`, `input:text:voice`, `output:gen-ai:chat:*`, `spark:command`, `spark:notify` 등이 포함된다 (`packages/stage-ui/src/stores/mods/api/channel-server.ts:66-88`).
- `context-bridge`는 initialize 시 `input:text`와 `input:text:voice` consumer를 `chat-ingestion` 그룹으로 등록한다 (`packages/stage-ui/src/stores/mods/api/context-bridge.ts:43-44`, `packages/stage-ui/src/stores/mods/api/context-bridge.ts:181-190`).
- `input:text` 핸들러는 context update를 normalizing하고 `chatContext.ingestContextMessage`에 반영한 뒤, Web Locks로 다중 탭 중복 처리를 피하고 `chatOrchestrator.ingest`를 호출한다 (`packages/stage-ui/src/stores/mods/api/context-bridge.ts:334-456`).
- assistant message와 turn complete는 각각 `output:gen-ai:chat:message`, `output:gen-ai:chat:complete`로 server channel에 다시 송신된다 (`packages/stage-ui/src/stores/mods/api/context-bridge.ts:510-553`).
- server runtime은 `module:consumer:register`를 처리하고 consumer/consumer-group 이벤트를 선택된 peer에게 라우팅한다 (`packages/server-runtime/src/index.ts:666-767`).
- WebSocket core는 sticky/round-robin/first consumer selection을 지원한다 (`packages/server-runtime/src/server-ws/core/index.ts:353-401`).

## 5. 서버 채팅 동기화 경로

앱 서버는 별도의 영속 채팅 모델을 제공한다.

- 앱은 `/api/v1/chats` REST와 `/ws/chat` WebSocket route를 등록한다 (`apps/server/src/app.ts:120-130`, `apps/server/src/app.ts:206-216`).
- Eventa 공유 SDK는 `chat:send-messages`, `chat:pull-messages`, `chat:new-messages` RPC/event를 정의한다 (`packages/server-sdk-shared/src/index.ts:43-45`).
- WebSocket handler는 사용자별 connection map을 유지하고, Redis duplicate subscriber를 통해 다른 프로세스의 메시지를 로컬 장치에 broadcast한다 (`apps/server/src/routes/chat-ws/index.ts:21-99`).
- `sendMessages` RPC는 `pushMessages` 후 `pullMessages`로 wire messages를 가져오고, chat member별 local broadcast 및 Redis publish를 수행한다 (`apps/server/src/routes/chat-ws/index.ts:116-146`).
- DB schema에는 `chats`, `chatMembers`, `messages`가 있으며, `messages`는 `role`, `seq`, `content`, media/sticker/reply/forward 필드를 가진다 (`apps/server/src/schemas/chats.ts:40-91`).
- `pushMessages`는 membership을 확인하고 chat row lock 후 `seq`를 증가시키며 중복 id를 upsert한다 (`apps/server/src/services/chats.ts:224-305`).
- `pullMessages`는 membership 검증 후 `seq > afterSeq` 조건으로 정렬된 wire messages를 반환한다 (`apps/server/src/services/chats.ts:413-447`).

## 6. 라이브/외부 플랫폼 통합

### 6.1 Discord

Discord bot은 AIRI server channel에 `input:text`/`input:text:voice`를 보내고 `output:gen-ai:chat:message`를 다시 Discord로 전송한다.

- bot entry는 Discord token과 AIRI token/url로 `DiscordAdapter`를 생성한다 (`services/discord-bot/src/index.ts:11-20`).
- Discord client는 Guilds, VoiceStates, GuildMessages, MessageContent, DirectMessages intents를 사용한다 (`services/discord-bot/src/adapters/airi-adapter.ts:65-75`).
- AIRI server SDK client는 `input:text`, `input:text:voice`, `input:voice`, `module:configure`, `output:gen-ai:chat:message`를 possible event로 연다 (`services/discord-bot/src/adapters/airi-adapter.ts:77-89`).
- Discord message는 bot mention 또는 DM일 때만 처리되며, mention 제거 후 sessionId(`discord-guild-*` 또는 `discord-dm-*`)와 Discord metadata/context update를 포함한 `input:text`로 전송된다 (`services/discord-bot/src/adapters/airi-adapter.ts:210-285`).
- assistant output은 2000자 제한에 맞게 split되어 Discord channel로 전송된다 (`services/discord-bot/src/adapters/airi-adapter.ts:159-201`).
- 음성 command 쪽은 Opus를 WAV로 변환하고 OpenAI transcription 후 `input:text:voice` 및 `input:text`를 보낸다 (`services/discord-bot/src/bots/discord/commands/summon.ts:450-488`).

### 6.2 Telegram

Telegram bot은 Stage server channel과 직접 연결된 구조라기보다 독립 LLM 액션 루프다.

- `createBotContext`는 `messageQueue`, `unreadMessages`, `processedIds`, `chats`를 가진다 (`services/telegram-bot/src/bots/telegram/index.ts:351-361`).
- sticker/photo/text handler는 incoming message를 queue에 넣고 `onMessageArrival`을 호출한다 (`services/telegram-bot/src/bots/telegram/index.ts:491-531`).
- `onMessageArrival`은 queue를 순차 처리하고 DB에 joined chat/message를 기록하며, 채팅별 unread messages를 최대 100개까지 유지한다 (`services/telegram-bot/src/bots/telegram/index.ts:366-437`).
- 주기 loop는 60초마다 `handleLoopStep`를 돌며, 최근 context를 줄이고 action history도 최대 20개 수준으로 제한한다 (`services/telegram-bot/src/bots/telegram/index.ts:174-239`, `services/telegram-bot/src/bots/telegram/index.ts:333-349`).
- `imagineAnAction`은 system/personality/incoming/unread/action history를 prompt로 만들고 `generateText` 결과를 JSON action으로 파싱한다 (`services/telegram-bot/src/llm/actions.ts:18-180`).
- `dispatchAction`은 list/read/send/sleep/continue/break 계열 액션을 실행한다 (`services/telegram-bot/src/bots/telegram/index.ts:23-172`).
- `send_message` 액션은 LLM으로 메시지를 split한 뒤 typing delay와 함께 Telegram 메시지를 보내고 DB에 기록한다 (`services/telegram-bot/src/bots/telegram/agent/actions/send-message.ts:75-162`).
- DB에는 `chat_messages`, `chat_stickers`, `chat_photos`, `joined_chats`, `chat_completions_history`, memory tables가 있다 (`services/telegram-bot/src/db/schema.ts:3-107`).

### 6.3 Satori 호환 플랫폼

Satori bot도 독립 queue/DB/action loop를 가진다.

- `message-created` 이벤트는 de-dup 후 DB `eventQueue`와 in-memory `botContext.eventQueue`에 들어간다 (`services/satori-bot/src/core/loop/queue.ts:37-69`).
- scheduler는 채널별 처리 lock과 global queue lock을 사용하며, unread events가 있는 채널을 주기적으로 처리한다 (`services/satori-bot/src/core/loop/scheduler.ts:136-200`, `services/satori-bot/src/core/loop/scheduler.ts:211-342`).
- `handleLoopStep`는 최근 메시지를 DB에서 가져오고 `imagineAnAction` → `dispatchAction`을 반복하되 `MAX_LOOP_ITERATIONS`로 상한을 둔다 (`services/satori-bot/src/core/loop/scheduler.ts:26-115`).
- planner LLM client는 incoming events/actions/unread counts를 prompt에 포함하고 `generateText` 결과를 Valibot `ActionSchema`로 검증한다 (`services/satori-bot/src/core/planner/llm-client.ts:18-129`).
- `read_unread_messages`는 unread event를 읽은 뒤 memory/DB에서 삭제한다 (`services/satori-bot/src/capabilities/actions/read-messages.ts:7-67`).
- `send-message`는 새 unread event가 생겼으면 전송을 중단하는 안전장치를 가진 뒤 `client.sendMessage`와 `recordMessage`를 실행한다 (`services/satori-bot/src/capabilities/actions/send-message.ts:22-44`).
- DB schema는 `channels`, `messages`, `event_queue`, `unread_events`를 둔다 (`services/satori-bot/src/lib/schema.ts:3-35`).

### 6.4 Bilibili / 기타 라이브 플랫폼

- `plugins/airi-plugin-bilibili-laplace/package.json`은 “Bilibili live chat via LAPLACE Event Bridge into AIRI server channel”이라고 설명하고 `@laplace.live/event-bridge-sdk`와 `@proj-airi/server-sdk`에 의존한다 (`plugins/airi-plugin-bilibili-laplace/package.json`).
- 그러나 실제 `src/index.ts`는 `console.warn('WIP')`만 포함한다 (`plugins/airi-plugin-bilibili-laplace/src/index.ts:1`). 따라서 현재 저장소 기준으로는 Bilibili 라이브 채팅 수집/응답 구현은 미완성이다.
- keyword scan에서 Twitch/YouTube/SuperChat/Cheer/Bits 등의 실시간 플랫폼 후원 이벤트 처리 코드는 발견되지 않았다. `live2d` 관련 파일은 아바타 렌더링/립싱크 맥락이지 라이브 플랫폼 통합이 아니다.

## 7. 후원/도네이션 지원 여부

현재 코드 기준으로 **도네이션/후원 이벤트 지원은 부재**로 판단한다.

근거:

- `donation`, `donat`, `tip`, `gift`, `superchat`, `super_chat`, `cheer`, `bits`, `twitch`, `youtube` 키워드로 `apps/server/src`, `packages`, `services`, `plugins`, `integrations`를 검색했을 때 실시간 후원 수집·정규화·응답 경로가 확인되지 않았다.
- 존재하는 결제 경로는 Stripe/Flux 기반 과금이다. 예를 들어 app은 `/api/v1/stripe` route를 등록한다 (`apps/server/src/app.ts:216`).
- Stripe route는 `/checkout`, `/invoices`, billing portal, webhook을 제공하고 checkout completion에서 Flux를 credit한다 (`apps/server/src/routes/stripe/index.ts:119-227`, `apps/server/src/routes/stripe/index.ts:235-307`, `apps/server/src/routes/stripe/index.ts:375-378`).
- OpenAI-compatible proxy route도 요청 사용량에 따라 billing/Flux를 차감한다 (`apps/server/src/routes/openai/v1/index.ts:85-126`, `apps/server/src/routes/openai/v1/index.ts:175-308`).

따라서 Stripe billing은 “사용자 결제/크레딧/구독/청구” 기능이며, Twitch Bits·YouTube Super Chat·Bilibili 선물 같은 “라이브 도네이션 이벤트를 채팅 컨텍스트로 수집하고 AIRI가 반응하는 기능”은 현재 구현되어 있지 않다. Bilibili plugin package description은 향후 라이브 채팅 브리지 의도를 보여주지만 구현이 WIP라 donation support의 근거로 삼을 수 없다.

## 8. 주요 디렉터리와 파일 인덱스

| 영역 | 주요 경로 | 역할 |
| --- | --- | --- |
| Stage 채팅 오케스트레이터 | `packages/stage-ui/src/stores/chat.ts` | ingest queue, user/assistant message 구성, LLM stream 처리, hook emit |
| Stage 세션 저장 | `packages/stage-ui/src/stores/chat/session-store.ts`, `packages/stage-ui/src/database/repos/chat-sessions.repo.ts` | Pinia session state, local chat index/session persistence |
| Stage 컨텍스트 저장 | `packages/stage-ui/src/stores/chat/context-store.ts` | module/platform context update 병합과 history 관리 |
| Core chat runtime | `packages/core-agent/src/types/chat.ts`, `packages/core-agent/src/runtime/agent-hooks.ts`, `packages/core-agent/src/runtime/llm-service.ts` | chat event type, lifecycle hooks, xsai streaming abstraction |
| Desktop 입력 UI | `apps/stage-tamagotchi/src/renderer/components/InteractiveArea.vue` | text/image 입력, ChatHistory 표시 |
| Desktop sync | `apps/stage-tamagotchi/src/renderer/stores/chat-sync.ts` | authority/follower 창 동기화, BroadcastChannel command/snapshot |
| 음성 입력 | `packages/stage-ui/src/stores/modules/hearing.ts`, `apps/stage-web/src/pages/index.vue`, `apps/stage-tamagotchi/src/renderer/pages/index.vue` | VAD/recording/transcription/voice ingest |
| TTS/무대 출력 | `packages/stage-ui/src/stores/modules/speech.ts`, `packages/stage-ui/src/components/scenes/Stage.vue` | generateSpeech, playback, caption, lip sync, emotion token |
| 모듈 이벤트 브리지 | `packages/plugin-protocol/src/types/events.ts`, `packages/stage-ui/src/stores/mods/api/context-bridge.ts`, `packages/stage-ui/src/stores/mods/api/channel-server.ts` | input:text consumer registration, context update, output gen-ai chat events |
| 서버 런타임 라우팅 | `packages/server-runtime/src/index.ts`, `packages/server-runtime/src/server-ws/core/index.ts` | module consumer registration/routing, consumer selection |
| 서버 채팅 | `apps/server/src/routes/chat-ws/index.ts`, `apps/server/src/services/chats.ts`, `apps/server/src/schemas/chats.ts`, `packages/server-sdk-shared/src/index.ts` | Eventa chat RPC, PostgreSQL messages, Redis Pub/Sub |
| Discord | `services/discord-bot/src/adapters/airi-adapter.ts`, `services/discord-bot/src/bots/discord/commands/summon.ts` | Discord text/voice → AIRI input, AIRI output → Discord response |
| Telegram | `services/telegram-bot/src/bots/telegram/index.ts`, `services/telegram-bot/src/llm/actions.ts`, `services/telegram-bot/src/db/schema.ts` | Telegram queue/unread store, LLM action planner, send/read/list actions |
| Satori | `services/satori-bot/src/core/loop/queue.ts`, `services/satori-bot/src/core/loop/scheduler.ts`, `services/satori-bot/src/core/planner/llm-client.ts`, `services/satori-bot/src/lib/schema.ts` | Satori event queue, unread events, per-channel scheduler, action planner |
| Bilibili WIP | `plugins/airi-plugin-bilibili-laplace/package.json`, `plugins/airi-plugin-bilibili-laplace/src/index.ts` | live chat bridge package intent only; implementation WIP |
| 결제/청구 | `apps/server/src/routes/stripe/index.ts`, `apps/server/src/routes/openai/v1/index.ts`, `apps/server/src/services/billing/*` | Stripe checkout/invoice/subscription and Flux billing; donation support 아님 |

## 9. 검증 메모와 한계

- 검증 방식은 repository-local static analysis다. `rg`가 설치되어 있지 않아 `git grep`, `find`, `sed`로 확인했다.
- 실제 외부 플랫폼 runtime credential, WebSocket 서버, Redis/PostgreSQL 동작은 실행하지 않았다.
- “후원 기능 없음”은 현재 저장소 코드 검색 기준의 결론이다. 특히 Bilibili plugin은 package metadata만 live chat bridge를 암시하고 구현은 WIP이므로, 런타임 지원으로 계산하지 않았다.
