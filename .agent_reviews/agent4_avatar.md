# Agent 4 아바타 조작/렌더링 분석 보고서

## 0. 요약

AIRI의 현재 아바타 렌더링은 `packages/stage-ui/src/components/scenes/Stage.vue`가 중앙 분기점이며, 선택된 모델 포맷에 따라 2D Live2D 렌더러(`@proj-airi/stage-ui-live2d`)와 3D VRM 렌더러(`@proj-airi/stage-ui-three`)를 전환한다. 저장소 근거상 실제 주력 경로는 **Live2D ZIP/Cubism4 + PixiJS**와 **VRM/VRMA + Three/TresJS/three-vrm**이다. `DisplayModelFormat`에는 PMX/PMD 값도 있으나 현재 `updateStageModel()`의 내장 렌더러 해석은 Live2D ZIP과 VRM만 활성 렌더러로 매핑한다.

- 2D: `packages/stage-ui-live2d`가 PixiJS + `pixi-live2d-display/cubism4` 기반 Live2D 캔버스/모델/모션/표정/파라미터 조작을 담당한다.
- 3D: `packages/stage-ui-three`가 TresJS/Three.js + `@pixiv/three-vrm` 기반 VRM 로딩, VRMA idle animation, 표정, 입모양, 시선, 조명/카메라/렌더 품질 설정을 담당한다.
- 오디오/립싱크: Live2D와 VRM 모두 `wlipsync` 기반의 오디오 노드/가중치 분석을 사용하지만 적용 방식은 다르다. Live2D는 단일 `mouthOpenSize`를 `ParamMouthOpenY`로 전달하고, VRM은 AEIOU viseme를 VRM expression blendshape(`aa`, `ee`, `ih`, `oh`, `ou`)로 적용한다.
- OBS/OBX: 저장소에서 OBS Studio/OBX 전용 출력, 플러그인, 가상 카메라, browser source 통합은 확인되지 않았다. 다만 Electron caption overlay와 desktop overlay는 별도 투명 창/브로드캐스트 채널 기반 오버레이이며 OBS 전용 기능은 아니다.

## 1. 렌더링 선택 및 공통 Stage 경로

### 증거

- `packages/stage-ui/src/components/scenes/Stage.vue`
  - `Live2DScene`와 `ThreeScene`를 동시에 import하고 `stageModelRenderer === 'live2d'` 또는 `'vrm'`일 때 각각 렌더링한다.
  - Live2D에는 `model-src`, `model-id`, `focus-at`, `mouth-open-size`, blink/expression/shadow/FPS/renderScale 설정을 전달한다.
  - VRM에는 `model-src`, `idle-animation`, `current-audio-source`, axes 표시, paused 값을 전달한다.
  - `canvasElement()`, `captureFrame()`, `readRenderTargetRegionAtClientPoint()`를 expose하여 외부에서 현재 렌더러의 캔버스/프레임/히트 영역을 얻을 수 있다.
- `packages/stage-ui/src/stores/settings/stage-model.ts`
  - `StageModelRenderer = 'live2d' | 'vrm' | 'godot' | 'disabled' | undefined`.
  - `DisplayModelFormat.Live2dZip`은 `'live2d'`, `DisplayModelFormat.VRM`은 `'vrm'`, 나머지는 `'disabled'`로 해석된다.
- `packages/stage-ui/src/stores/display-models.ts`
  - 프리셋 Live2D ZIP 2개, VRM 2개를 등록한다.
  - 사용자 모델은 IndexedDB/localforage에 저장하며 Live2D/VRM 미리보기 생성 함수를 lazy import한다.
- `packages/stage-ui/src/components/scenes/index.ts`
  - `WidgetStage`와 Live2D scene 구성요소를 공용 export한다.

### 해석

Stage 레이어는 2D/3D를 직접 구현하지 않고, 선택된 모델 포맷과 전역 설정 상태를 각 렌더 패키지에 전달하는 오케스트레이터다. `godot` 렌더러 문자열은 존재하지만 `Stage.vue`에서는 실험적 placeholder callout만 보이며, 이 보고서의 주 렌더링 근거는 Live2D와 VRM이다.

## 2. 2D: Live2D 지원

### 렌더 패키지/컴포넌트

- 패키지: `packages/stage-ui-live2d/package.json`
  - 주요 의존성: `pixi-live2d-display`, PixiJS 6 계열(`@pixi/app`, `@pixi/ticker` 등), `pixi-filters`, `jszip`, `animejs`.
- 진입 컴포넌트: `packages/stage-ui-live2d/src/components/scenes/Live2D.vue`
  - `Live2DCanvas`와 `Live2DModel`을 조합한다.
  - 렌더 스케일, 최대 FPS, 마우스/포커스, mouth open, 자동 blink, idle animation, expression, shadow 설정을 props로 받는다.
- 캔버스: `packages/stage-ui-live2d/src/components/scenes/live2d/Canvas.vue`
  - Pixi `Application`을 만들고 투명 배경, `preserveDrawingBuffer`, `resolution`, `ticker.maxFPS`를 설정한다.
  - `captureFrame()`은 Pixi render 후 canvas `toBlob()`으로 PNG 계열 프레임을 얻는다.
- 모델: `packages/stage-ui-live2d/src/components/scenes/live2d/Model.vue`
  - `Live2DFactory.setupLive2DModel(live2DModel, { url, id }, { autoInteract: false })`로 모델을 로드한다.
  - `MotionPriority.FORCE`를 사용해 `setMotion(group, index)`를 제공한다.
  - `model.focus(x, y)`로 포커스/시선 입력을 적용한다.
  - `ParamMouthOpenY`, `ParamAngleX/Y/Z`, eye, brow, mouth, cheek, body, breath 파라미터를 직접 설정한다.

### Live2D 파일 형식/로딩

- `packages/stage-ui/src/components/scenarios/dialogs/model-selector/model-selector.vue`
  - Live2D import는 `.zip`만 허용하고 `validateLive2DZip()`을 먼저 수행한다.
- `packages/stage-ui-live2d/src/utils/live2d-validator.ts`
  - ZIP 내부 `.model3.json`을 표준 entry point로 보고, 없으면 단일 `.moc3` 기반 loose files를 heuristic으로 인정한다.
  - `.moc3` 헤더, MOC 크기, basename collision, texture/physics/expression reference 누락을 검사한다.
- `packages/stage-ui-live2d/src/utils/live2d-zip-loader.ts`
  - `pixi-live2d-display`의 `ZipLoader`를 JSZip 기반으로 교체한다.
  - `.model3.json`/`.model.json`이 없을 때 단일 `.moc3` + `.png` texture + `.mtn`/`.motion3.json`을 모아 fake settings를 만든다.
  - `.cdi3.json`, `.exp3.json` 메타데이터를 추가로 수집한다.
- `apps/stage-tamagotchi/electron.vite.config.ts`, `apps/stage-pocket/vite.config.ts`
  - `DownloadLive2DSDK()`와 Hiyori Live2D ZIP 다운로드가 구성되어 있다.
  - HTML 쪽에서는 `apps/stage-pocket/index.html`에 Cubism Core script가 포함된다.
- `patches/pixi-live2d-display.patch`
  - `items_pinned_to_model.json`을 모델 settings 파일로 오인하지 않도록 `pixi-live2d-display` 패치를 둔다.

### 모션/표정/립싱크/오디오 hooks

- 모션
  - `packages/stage-ui-live2d/src/stores/live2d.ts`: `currentMotion`, `availableMotions`, `motionMap`, `modelParameters`를 localStorage 기반 store로 보관한다.
  - `packages/stage-ui-live2d/src/components/scenes/live2d/Model.vue`: 모델 로드 후 `motionManager.definitions`에서 available motions를 추출하고, `currentMotion` watch로 `setMotion()`을 호출한다.
  - `packages/stage-ui/src/components/scenarios/settings/model-settings/live2d.vue`: 런타임 motion 목록을 보여주고 선택 motion group/index를 localStorage에 저장한다.
- 표정
  - `packages/stage-ui-live2d/src/composables/live2d/expression-controller.ts`: `model3.json`의 expression reference와 `.exp3.json` 파라미터를 파싱하고 매 프레임 Cubism core parameter에 blend 적용한다.
  - `packages/stage-ui-live2d/src/stores/expression-store.ts`: expression group/parameter state, LLM exposure mode, auto reset, default 저장을 관리한다.
  - `packages/stage-ui-live2d/src/tools/expression-tools.ts`: `expression_set`, `expression_get`, `expression_toggle`, `expression_save_defaults`, `expression_reset_all` 도구를 제공한다.
- blink/idle/focus/beat sync
  - `packages/stage-ui-live2d/src/composables/live2d/motion-manager.ts`: motion manager update를 hook하여 beat sync, idle disable, idle focus, expression, auto eye blink plugin을 pre/post/final 단계로 실행한다.
  - `packages/stage-ui-live2d/src/composables/live2d/animation.ts`: idle eye saccades/focus를 단순 시뮬레이션한다.
  - `packages/stage-ui-live2d/src/composables/live2d/beat-sync.ts`와 `packages/stage-ui-live2d/src/components/scenes/live2d/Model.vue`: `listenBeatSyncBeatSignal()`로 beat 이벤트를 받아 몸/머리 각도 target을 갱신한다.
- 립싱크/오디오
  - `packages/stage-ui/src/components/scenes/Stage.vue`: TTS playback source를 `audioContext.destination`, analyser, `lipSyncNode`에 연결한다.
  - `packages/model-driver-lipsync/src/live2d/index.ts`: `createLive2DLipSync()`가 `wlipsync` AudioWorkletNode를 만들고 AEIOUS를 AEIOU로 remap한 뒤 `getMouthOpen()`을 제공한다.
  - `Stage.vue`는 Live2D일 때만 `createLive2DLipSync(audioContext, wlipsyncProfile, ...)`를 초기화하고 RAF loop에서 `mouthOpenSize`를 갱신한다.
  - `Live2DModel`은 `mouthOpenSize` watch로 Cubism `ParamMouthOpenY`를 업데이트한다.

## 3. 3D: VRM/Three 지원

### 렌더 패키지/컴포넌트

- 패키지: `packages/stage-ui-three/package.json`
  - 주요 의존성: `@pixiv/three-vrm`, `@pixiv/three-vrm-animation`, `three`, `@tresjs/core`, `@tresjs/post-processing`, `wlipsync`.
- 진입 컴포넌트: `packages/stage-ui-three/src/components/ThreeScene.vue`
  - `TresCanvas`를 생성하고 camera, lighting, skybox/hemisphere, post-processing, `VRMModel`을 구성한다.
  - `setExpression()`, `setVrmFrameHook()`, `canvasElement()`, `camera()`, `renderer()`, `scene()`, `captureFrame()`를 expose한다.
  - `readRenderTargetRegionAtClientPoint()`로 VRM 렌더 타깃의 픽셀 영역을 읽는 경로가 있다.
- 모델 컴포넌트: `packages/stage-ui-three/src/components/Model/VRMModel.vue`
  - VRM 로딩/캐시/정리, VRMA idle animation, material hook, per-frame update loop, look-at, blink, idle saccade, emote, lip sync, expressionManager/nodeConstraint/springBone update를 담당한다.
- 렌더 상태/store: `packages/stage-ui-three/src/stores/model-store.ts`, `packages/stage-ui-three/src/stores/view-control.ts`
  - model size/origin/offset/rotation, camera distance/FOV/position, lookAt target, tracking mode, lights, environment, renderScale, multisampling 등을 localStorage로 저장한다.

### VRM 파일 형식/로딩

- `packages/stage-ui/src/components/scenarios/dialogs/model-selector/model-selector.vue`
  - VRM import는 `.vrm`만 허용한다.
- `packages/stage-ui-three/src/composables/vrm/loader.ts`
  - `GLTFLoader`에 `VRMLoaderPlugin`, `VRMAnimationLoaderPlugin`, MToon material plugin을 등록한다.
  - 즉 VRM은 glTF 기반 `.vrm`, 애니메이션은 `.vrma` 경로가 사용된다.
- `packages/stage-ui-three/src/composables/vrm/core.ts`
  - `loadVrm(model)`이 GLTF userData의 `vrm`을 꺼내고 `VRMUtils.removeUnnecessaryVertices`, `combineSkeletons`, frustum culling disable, lookAt quaternion proxy, bounding box/camera offset 계산을 수행한다.
- `packages/stage-ui-three/src/assets/vrm/animations/index.ts`
  - 기본 idle animation으로 `idle_loop.vrma`를 export한다.
- `apps/stage-tamagotchi/electron.vite.config.ts`, `apps/stage-pocket/vite.config.ts`
  - AvatarSample A/B `.vrm` 프리셋 다운로드가 구성되어 있다.

### 모션/표정/립싱크/오디오 hooks

- 모션/애니메이션
  - `packages/stage-ui-three/src/composables/vrm/animation.ts`: `.vrma`를 `loadVRMAnimation()`으로 로드하고 `createVRMAnimationClip()`으로 `AnimationClip`을 만든다.
  - `packages/stage-ui-three/src/components/Model/VRMModel.vue`: idle animation clip을 `AnimationMixer`로 재생하고 매 프레임 `mixer.update(delta)`를 호출한다.
  - `reAnchorRootPositionTrack()`은 hips position track을 현재 VRM 기준으로 보정한다.
- 표정
  - `packages/stage-ui-three/src/composables/vrm/expression.ts`: `useVRMEmote()`가 `happy`, `sad`, `angry`, `surprised`, `neutral`, `think` emotion state를 VRM expressionManager 값으로 blend한다.
  - `packages/stage-ui-three/src/components/Model/VRMModel.vue`: `setExpression(expression, intensity)` expose가 `setEmotionWithResetAfter(expression, 3000, intensity)`를 호출한다.
  - `packages/stage-ui/src/constants/emotions.ts`와 `packages/stage-ui-live2d/src/constants/emotions.ts`: AIRI emotion을 Live2D motion group 또는 VRM expression name으로 매핑한다. VRM은 `happy`, `sad`, `angry`, `surprised` 정도가 직접 매핑되고 `think/question/awkward` 등은 일부 undefined이다.
- blink/시선/트래킹
  - `packages/stage-ui-three/src/composables/vrm/animation.ts`: `useBlink()`가 VRM expressionManager의 `blink` 값을 주기적으로 조절하고, `useIdleEyeSaccades()`가 lookAt target을 갱신한다.
  - `packages/stage-ui-three/src/components/Model/VRMModel.vue`: `trackingMode`가 `camera`, `mouse`, `none`일 때 lookAt target을 카메라/마우스/기본값으로 바꾼다.
  - `packages/stage-ui/src/components/scenarios/settings/model-settings/vrm.vue`: eye tracking mode, camera/model position, light/environment/renderScale 설정 UI가 있다.
- 립싱크/오디오
  - `packages/stage-ui/src/components/scenes/Stage.vue`: VRM 렌더러에는 현재 재생 중인 `AudioBufferSourceNode`를 `current-audio-source` prop으로 넘긴다.
  - `packages/stage-ui-three/src/composables/vrm/lip-sync.ts`: `createWLipSyncNode(audioContext, profile)`을 만들고, source node를 lipSyncNode에 연결한다.
  - 같은 파일에서 AEIOUS raw weights를 AEIOU로 remap하고, 가장 큰 두 입모양만 선택해 VRM blendshape `aa`, `ee`, `ih`, `oh`, `ou`에 smoothing 적용한다.
- 외부 프레임 hook / mocap
  - `packages/stage-ui-three/src/components/ThreeScene.vue`와 `packages/stage-ui-three/src/components/Model/VRMModel.vue`: `setVrmFrameHook()`을 외부에 제공한다.
  - `packages/model-driver-mediapipe/README.md`: MediaPipe 기반 single-person motion capture workshop package가 존재한다.
  - `apps/stage-web/src/pages/devtools/model-driver-mediapipe.vue`: 카메라 프레임 → MediaPipe backend → poseToVrmTargets → `setVrmFrameHook()` → VRM pose 적용 흐름을 devtool에서 사용한다.
  - `packages/model-driver-mediapipe/src/three/apply-pose-to-vrm.ts`: VRM humanoid normalized bone node에 pose target을 slerp/방향 기반으로 적용한다.

## 4. 2D/3D 감정 토큰과 오디오 파이프라인 연결

- `packages/stage-ui/src/components/scenes/Stage.vue`는 chat orchestrator hook(`onBeforeMessageComposed`, `onBeforeSend`, `onTokenLiteral`, `onTokenSpecial`, `onStreamEnd`, `onAssistantResponseEnd`)을 받아 speech pipeline과 감정/delay queue에 연결한다.
- special token 중 emotion은 `useEmotionsMessageQueue()`를 거쳐:
  - VRM이면 `vrmViewerRef.value!.setExpression(mappedExpression, intensity)` 호출.
  - Live2D이면 `currentMotion.value = { group: mappedMotion }`로 motion group 전환.
- TTS는 `@xsai/generate-speech` 결과를 `audioContext.decodeAudioData()`로 디코딩하고 `createPlaybackManager()`가 재생한다.
- playback 시작 시 assistant caption/presentation broadcast channel에 텍스트를 보내고, playback 종료 시 `mouthOpenSize = 0`으로 닫힌 입 상태를 복구한다.

## 5. 모델/장면 설정 UI와 미리보기

- 공통 모델 설정 페이지: `packages/stage-pages/src/pages/settings/models/index.vue`
  - `ModelSettings`를 렌더하고 현재 프레임을 캡처해 `node-vibrant`로 색상 팔레트를 추출한다.
- `packages/stage-ui/src/components/scenarios/settings/model-settings/index.vue`
  - 설정 패널과 preview stage를 조합한다.
- Live2D 설정: `packages/stage-ui/src/components/scenarios/settings/model-settings/live2d.vue`
  - scale/position, theme color extraction, runtime motion, expression exposure, cache clear, FPS/renderScale/blink/expression/shadow 관련 설정을 다룬다.
- VRM 설정: `packages/stage-ui/src/components/scenarios/settings/model-settings/vrm.vue`
  - model position, renderScale, FOV, camera distance, Y rotation, eye tracking, directional/ambient/hemisphere light, skybox intensity 등을 다룬다.
- Live2D preview: `packages/stage-ui-live2d/src/utils/live2d-preview.ts`
  - offscreen Pixi canvas에 Live2D를 렌더하고 빈 픽셀 crop/padding 후 data URL을 만든다.
- VRM preview: `packages/stage-ui-three/src/utils/vrm-preview.ts`
  - offscreen Three renderer에 VRM과 idle animation을 렌더해 data URL을 만든다.

## 6. 서버/캐릭터 모델 스키마

- `apps/server/src/types/character-avatar-model.ts`
  - TODO 상태의 `AvatarModelConfig`가 `vrm.urls`, `live2d.urls` 배열만 가진다.
- `apps/server/src/routes/characters/schema.ts`
  - `AvatarModelConfigSchema`도 `vrm.urls`, `live2d.urls`를 optional로 정의한다.
  - `AvatarModelTypeSchema`는 `'vrm' | 'live2d'`만 허용한다.

해석: 서버/캐릭터 모델 계층도 현재 저장소에서는 VRM과 Live2D만 일급 avatar model type으로 인정한다. PMX/PMD는 클라이언트 enum에만 보이고, 서버 스키마와 Stage renderer 분기에는 실제 활성 경로가 없다.

## 7. OBS/OBX 관련성 또는 부재

### 확인된 것

- `packages/stage-ui/src/components/scenes/Stage.vue`
  - `airi-caption-overlay`, `airi-chat-present` BroadcastChannel로 caption/presentation 텍스트를 보낸다.
- `apps/stage-tamagotchi/src/renderer/pages/caption.vue`
  - `airi-caption-overlay` 채널을 구독해 speaker/assistant caption을 투명/드래그 가능한 overlay 형태로 표시한다.
- `apps/stage-tamagotchi/src/main/windows/desktop-overlay/*`, `apps/stage-tamagotchi/src/renderer/pages/desktop-overlay.vue`
  - Electron 투명 always-on-top desktop overlay와 computer-use MCP polling 기반 ghost pointer/target box 렌더링이 있다.
- `apps/stage-tamagotchi/src/main/windows/widgets/index.ts`
  - overlay widget window lifecycle이 존재한다.

### 확인되지 않은 것

- `obs` 검색 결과는 대부분 observability/observer/Obsidian/무관한 문자열 또는 일반 overlay에 해당한다.
- `obx` 검색 결과는 lockfile integrity/base64 문자열과 폰트 binary match뿐이며 아바타/렌더링 통합 근거는 없다.
- OBS Studio 플러그인, OBS WebSocket, virtual camera, browser source 전용 출력, OBX라는 포맷/프로토콜 통합 파일은 저장소 근거에서 확인되지 않았다.

해석: AIRI는 overlay/caption 창을 제공하므로 OBS에서 화면/window capture로 가져갈 수는 있겠지만, 저장소에는 OBS/OBX를 직접 제어하거나 전용 출력 대상으로 삼는 구현은 없다.

## 8. 지원 포맷 정리

| 영역 | 실제 활성 지원 | 근거 경로 | 비고 |
|---|---|---|---|
| Live2D 2D | `.zip`, 내부 `.model3.json`/`.model.json`, `.moc3`, `.png`, `.motion3.json`/`.mtn`, `.exp3.json`, `.cdi3.json` | `packages/stage-ui-live2d/src/utils/live2d-validator.ts`, `packages/stage-ui-live2d/src/utils/live2d-zip-loader.ts`, `packages/stage-ui/src/components/scenarios/dialogs/model-selector/model-selector.vue` | import UI는 `.zip`만 허용. loose files는 ZIP 내부 heuristic 경로. |
| VRM 3D | `.vrm` | `packages/stage-ui/src/components/scenarios/dialogs/model-selector/model-selector.vue`, `packages/stage-ui-three/src/composables/vrm/loader.ts`, `packages/stage-ui-three/src/composables/vrm/core.ts` | GLTFLoader + VRMLoaderPlugin. |
| VRM animation | `.vrma` | `packages/stage-ui-three/src/assets/vrm/animations/idle_loop.vrma`, `packages/stage-ui-three/src/composables/vrm/animation.ts` | 현재 Stage는 idle animation prop 하나를 전달. |
| PMX/PMD/MMD | enum만 존재, 렌더러 미연결 | `packages/stage-ui/src/stores/display-models.ts`, `packages/stage-ui/src/stores/settings/stage-model.ts` | `resolveBuiltInStageModelRenderer()`에서 disabled 처리. |
| Godot | 실험적 renderer placeholder | `packages/stage-ui/src/stores/settings/stage-model.ts`, `packages/stage-ui/src/components/scenes/Stage.vue`, `engines/stage-tamagotchi-godot/scenes/stage-root.tscn` | 현재 Vue Stage에서는 callout만 확인. |
| OBS/OBX | 직접 지원 근거 없음 | `apps/stage-tamagotchi/src/renderer/pages/caption.vue`, `apps/stage-tamagotchi/src/renderer/pages/desktop-overlay.vue`, grep 결과 | overlay는 존재하지만 OBS/OBX 전용 통합은 아님. |

## 9. 결론

저장소 기준 AIRI의 아바타 시스템은 명확히 두 축으로 나뉜다.

1. **2D Live2D 축**: `stage-ui-live2d`가 PixiJS와 `pixi-live2d-display/cubism4`로 Live2D ZIP/Cubism 모델을 렌더링하고, Cubism parameter/motion/expression을 직접 제어한다. 오디오 립싱크는 `wlipsync` 분석 결과를 단일 mouth-open 값으로 축약해 `ParamMouthOpenY`에 반영한다.
2. **3D VRM 축**: `stage-ui-three`가 TresJS/Three.js와 `@pixiv/three-vrm`으로 VRM을 로드하고, `.vrma` idle animation, VRM expression, look-at/blink/saccade, material/outline hook, lighting/camera/renderScale을 관리한다. 오디오 립싱크는 `wlipsync` AEIOU 가중치를 VRM expression blendshape에 직접 매핑한다.

OBS/OBX는 현재 직접 통합 대상으로 보이지 않는다. 관련성이 있다면 Electron caption/desktop/widget overlay를 외부 캡처 도구가 화면 캡처하는 간접 사용 정도이며, 저장소에는 OBS WebSocket/플러그인/전용 출력 구현이 없다.
