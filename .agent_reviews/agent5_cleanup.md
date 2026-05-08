# Agent 5 정리/삭제 후보 분석 보고서

## 범위와 결론 요약

- 분석 범위: `/workspace/airi`의 기존 파일/디렉터리 중 **삭제해도 되는 생성물/캐시/빌드 산출물**, **조건부로 제거 가능한 레거시·데모·자산**, **삭제하면 안 되는 영역**을 분류했다.
- 수행 원칙: 코드 수정 없음. 본 보고서 파일 `.agent_reviews/agent5_cleanup.md` 작성만 수행했다.
- 핵심 결론:
  1. 현재 워킹트리에서 명확히 “생성물/캐시”로 보이는 실제 존재 항목은 주로 `.omx/cache`, `.omx/logs` 같은 OMX 런타임 산출물이다. 단, 현재 세션이 사용 중이므로 세션 중에는 삭제하지 않는 편이 안전하다.
  2. `.gitignore`는 `dist`, `out`, `coverage`, `.turbo`, `.wxt`, `.vite-inspect`, `*.tsbuildinfo`, VitePress cache, Capacitor/Android/iOS 빌드 결과 등 다수의 삭제 가능한 산출물 패턴을 이미 명시한다. 현재 검사 시 이들 대부분은 실제로 존재하지 않았다.
  3. `packages/drizzle-duckdb-wasm`와 `packages/duckdb-wasm`는 README만 있는 “이전됨” 안내 디렉터리라 가장 유력한 레거시 제거 후보지만, 루트/문서 README 링크를 먼저 정리해야 한다.
  4. `packages/scenarios-stage-tamagotchi-browser/artifacts/raw`는 이름상 산출물이지만 현재 소스에서 직접 import하므로 무조건 삭제 대상이 아니다. 재생성 파이프라인과 문서 빌드 의존성을 확인한 뒤에만 제거/외부화해야 한다.

## 확인한 증거 경로

- 루트 ignore 정책: `.gitignore`
  - Node/log/temp/coverage/build/Vite/cache/generated/data/plugin/media 산출물 패턴: `.gitignore:5-130`
- 워크스페이스/루트 스크립트:
  - 전체 앱/패키지 빌드 및 테스트 스크립트: `package.json:14-50`
  - `capture:tamagotchi`가 `packages/scenarios-stage-tamagotchi-browser/artifacts/raw`를 출력 디렉터리로 사용: `package.json:42`
  - workspace가 `packages/**`, `plugins/**`, `integrations/**`, `services/**`, `examples/**`, `docs/**`, `engines/**`, `apps/**`를 포함하되 `!**/dist/**`를 제외: `pnpm-workspace.yaml:3-12`
- 모바일/네이티브 생성물 ignore:
  - Android 빌드/Gradle/Capacitor 생성물: `apps/stage-pocket/android/.gitignore:3-101`
  - iOS build/Pods/output/public/DerivedData/Capacitor 생성물: `apps/stage-pocket/ios/.gitignore:1-16`
- VSCode 확장 생성물:
  - `src/generated/meta.ts` 재생성 스크립트: `integrations/vscode/vscode-airi/scripts/vscode-ext-gen.ts:9-23`
  - publish 시 루트 `LICENSE`를 확장 폴더로 복사: `integrations/vscode/vscode-airi/scripts/publish.ts:16-32`
  - ignore 대상: `.gitignore:84-86`
- 시나리오/스크린샷 산출물:
  - 브라우저 시나리오가 `artifacts/raw`를 입력으로 기대: `packages/scenarios-stage-tamagotchi-browser/README.md:14`, `:24-44`
  - Electron 시나리오가 raw 파일 27개를 `artifacts/raw`에 출력: `packages/scenarios-stage-tamagotchi-electron/README.md:17-36`, `:63-66`
  - 실제 tracked raw AVIF 27개 존재: `packages/scenarios-stage-tamagotchi-browser/artifacts/raw/*.avif`
- 레거시/이전 안내:
  - `packages/drizzle-duckdb-wasm/README.md:1-3`
  - `packages/duckdb-wasm/README.md:1-3`
- 데모/검증용 샘플:
  - Devtools sample plugin 목적과 사용법: `apps/stage-tamagotchi/src/main/services/airi/plugins/examples/devtools-sample-plugin/README.md:1-32`
  - MediaPipe workshop package 목적과 devtools 연결: `packages/model-driver-mediapipe/README.md:1-35`
- 현재 존재하는 런타임/작업 산출물:
  - `.omx/cache/codebase-map.json`
  - `.omx/logs/*.jsonl`
  - `.omx/state/*.json`
  - `.agent_reviews/*`는 이번 멀티 에이전트 리뷰의 산출물이므로 현재 요청 범위에서는 보존 대상이다.

## 1. 명확히 안전한 생성/캐시/빌드 산출물

아래는 `.gitignore` 또는 생성 스크립트가 명시하는 항목이다. **존재한다면 삭제해도 소스 손실 가능성이 낮다.** 단, 현재 세션 산출물은 세션 종료 뒤 삭제하는 것이 안전하다.

### 1.1 일반 Node/Vite/Turbo/테스트 산출물

- `node_modules/`
  - 근거: `.gitignore:5-6`
  - 재생성: `pnpm install`
- `coverage/`
  - 근거: `.gitignore:41-42`
  - 재생성: `pnpm test` 또는 `vitest --coverage`
- `dist/`, `out/`, `bundle/`, `*.output`
  - 근거: `.gitignore:44-57`
  - 재생성: 각 패키지/app의 `build` 스크립트
- `.turbo/`
  - 근거: `.gitignore:100-101`
  - 재생성: `turbo run ...`
- `.eslintcache`, `**/.cache/`, `**/*.tsbuildinfo`, `**/.vitepress/cache/`
  - 근거: `.gitignore:66-70`
  - 재생성: lint/typecheck/docs dev/build 실행 시 자동 생성
- `.vite-inspect`, `.vite-inspect*`
  - 근거: `.gitignore:58-60`
  - 재생성: vite-plugin-inspect 실행 시 자동 생성

현재 검사에서는 이 범주의 대표 디렉터리(`dist`, `out`, `coverage`, `.turbo`, `node_modules`)가 루트 워킹트리에 실질적으로 보이지 않았다. 즉, “삭제 후보”라기보다 앞으로 생기면 지워도 되는 산출물 목록에 가깝다.

### 1.2 모바일/Capacitor 네이티브 생성물

- Android:
  - `apps/stage-pocket/android/**/build/`, `.gradle/`, `bin/`, `gen/`, `out/`, `.externalNativeBuild/`, `.cxx/`
  - `apps/stage-pocket/android/app/src/main/assets/public`
  - `apps/stage-pocket/android/app/src/main/assets/capacitor.config.json`
  - `apps/stage-pocket/android/app/src/main/assets/capacitor.plugins.json`
  - `apps/stage-pocket/android/app/src/main/res/xml/config.xml`
  - 근거: `apps/stage-pocket/android/.gitignore:15-24`, `:60-62`, `:92-101`
- iOS:
  - `apps/stage-pocket/ios/App/build`, `Apps/Pods`가 아니라 정확히 `apps/stage-pocket/ios/App/Pods`, `App/output`, `App/App/public`, `DerivedData`, `xcuserdata`
  - `apps/stage-pocket/ios/App/App/capacitor.config.json`, `apps/stage-pocket/ios/App/App/config.xml`
  - 근거: `apps/stage-pocket/ios/.gitignore:1-16`

주의: `apps/stage-pocket/ios/App/App.xcodeproj/project.pbxproj`, Swift 파일, Android/Gradle 소스 설정 파일은 실제 네이티브 프로젝트 소스이므로 삭제 대상이 아니다.

### 1.3 VSCode 확장 생성물

- `integrations/vscode/vscode-airi/src/generated/`
  - 근거: `.gitignore:84-85`, 생성 스크립트 `integrations/vscode/vscode-airi/scripts/vscode-ext-gen.ts:15-22`
  - 현재 검사 시 `src/generated`는 존재하지 않았다.
- `integrations/vscode/vscode-airi/LICENSE`
  - 근거: `.gitignore:84-86`, publish 스크립트가 루트 `LICENSE`를 복사함 `integrations/vscode/vscode-airi/scripts/publish.ts:30-31`
  - caveat: 현재는 `git ls-files` 기준 tracked 파일이다. “안전 생성물” 성격은 강하지만, 실제 제거는 VSCode 패키징/배포 흐름에서 tracked LICENSE가 필요한지 확인하고 `.gitignore`와 Git 추적 상태를 맞춘 뒤 수행해야 한다.

### 1.4 런타임 로컬 상태/캐시

- `.omx/cache/codebase-map.json`, `.omx/logs/*.jsonl`, 일부 `.omx/state/*.json`
  - 근거: 현재 세션의 OMX 런타임 산출물로 실제 존재한다.
  - 안전성: 저장소 소스는 아니므로 장기적으로는 삭제 가능하다.
  - caveat: Ralph/OMX 세션이 활성 상태일 때는 진행 상태·로그·세션 추적에 사용될 수 있으므로 지금 삭제하면 안 된다. 세션 종료 후 정리 대상으로 분류한다.
- `.agent_reviews/`
  - 이번 리뷰 결과를 담는 산출물이다. 저장소 기능 소스는 아니지만 현재 멀티 에이전트 작업의 요구 산출물이므로 이 작업에서는 삭제 금지.

## 2. 조건부로 제거 가능한 레거시/데모/자산 후보

이 항목들은 “삭제해도 된다”가 아니라 **삭제 가능성이 있으나 소유자 확인, 문서 링크 정리, 빌드 영향 확인이 필요한 후보**다.

### 2.1 `packages/drizzle-duckdb-wasm/`와 `packages/duckdb-wasm/`

- 상태:
  - 각 디렉터리는 README 1개만 tracked로 보인다.
  - 두 README 모두 “package has been moved”라고 말한다: `packages/drizzle-duckdb-wasm/README.md:1-3`, `packages/duckdb-wasm/README.md:1-3`.
  - `pnpm-workspace.yaml`는 `packages/**`를 포함하지만, 두 디렉터리에는 `package.json`이 없어 현재 workspace package는 아니다.
- 제거 가능성: 높음.
- caveat:
  - 루트 README와 다국어 README가 이 경로를 링크한다. 예: `README.md`에서 두 패키지를 소개하고 Mermaid 그래프에 포함한다.
  - 제거 시 문서 링크를 외부 `https://github.com/proj-airi/duckdb-wasm`로 직접 바꾸거나, “moved” 안내를 docs 쪽으로 이관해야 한다.

### 2.2 `bucket/airi.json`

- 상태:
  - Scoop manifest로 보이며 `version`이 `0.9.0-alpha.18`이다.
  - 루트 `package.json` 버전은 `0.10.2`라 현재 앱 버전과 불일치한다.
  - README는 `scoop bucket add airi https://github.com/moeru-ai/airi` 형태의 설치 경로를 안내한다.
- 제거 가능성: 중간.
- caveat:
  - Scoop 배포 채널을 계속 유지한다면 삭제하면 안 된다.
  - 유지하지 않는다면 `bucket/airi.json` 삭제와 함께 README 설치 안내를 제거/갱신해야 한다.

### 2.3 `apps/component-calling/`

- 상태:
  - package 이름은 `@proj-airi/component-calling`, 설명은 “Realtime audio”이지만 실제 페이지는 component calling/weather widget 실험 UI를 포함한다.
  - 루트에 전용 `dev:*` 스크립트는 보이지 않고, `dev:apps`/`build:apps` 같은 glob 기반 앱 실행에는 포함된다.
  - 저장소 전체 검색상 직접 참조는 대부분 자기 자신이며, 별도 앱으로 동작한다.
- 제거 가능성: 중간.
- caveat:
  - component-calling 실험 또는 위젯 호출 UX 검증을 아직 사용한다면 보존해야 한다.
  - 제거하려면 workspace glob의 부작용, CI 빌드 대상, 관련 문서/README 링크 유무를 확인해야 한다.

### 2.4 `apps/stage-tamagotchi/src/main/services/airi/plugins/examples/devtools-sample-plugin/`

- 상태:
  - README가 “Plugin Host Inspector 검증용 sample plugin”이라고 명시한다: `.../README.md:1-4`.
  - 사용법은 devtools에서 registry root를 확인한 뒤 파일을 복사해 로드하는 수동 검증 흐름이다: `.../README.md:12-25`.
- 제거 가능성: 낮음~중간.
- caveat:
  - 플러그인 호스트 수동 QA/문서용 샘플로 가치가 있다.
  - 자동 테스트 fixture로는 보이지 않지만, 개발자 경험/검증 문서 역할을 하므로 “비필수 샘플”로만 분류한다.

### 2.5 `packages/model-driver-mediapipe/references/`

- 상태:
  - README가 upstream docs/snippets를 이 경로에 보관하라고 명시한다: `packages/model-driver-mediapipe/README.md:29-35`.
- 제거 가능성: 낮음.
- caveat:
  - 외부 문서 요약/스냅샷이라 런타임 필수 소스는 아닐 수 있지만, 패키지 개발 문맥과 API 근거로 유지 의도가 명시되어 있다.
  - 제거하려면 README의 “Docs” 섹션도 바꿔야 한다.

### 2.6 `packages/scenarios-stage-tamagotchi-browser/artifacts/raw/*.avif`

- 상태:
  - 이름상 generated artifact이며 Electron capture가 출력한다: `packages/scenarios-stage-tamagotchi-electron/README.md:17-36`.
  - 하지만 브라우저 시나리오 Vue 파일들이 `../../../../artifacts/raw/*.avif`를 직접 import한다.
  - README도 raw 27개가 존재해야 final export가 가능하다고 말한다: `packages/scenarios-stage-tamagotchi-browser/README.md:14`, `:41-44`, `:71-72`.
- 제거 가능성: 조건부.
- caveat:
  - 삭제만 하면 docs screenshot composition이 실패하거나 stale/missing input 문제가 생길 수 있다.
  - 안전하게 제거하려면 raw input을 CI에서 항상 생성하거나, 원본 raw를 외부 artifact storage로 옮기고 import 경로/빌드 흐름을 바꾸는 별도 작업이 필요하다.

### 2.7 대용량 docs 미디어 중복 자산

- 상태:
  - `docs/content/en`와 `docs/content/zh-Hans`에 동일한 이름/크기의 mp4/gif가 다수 존재한다.
  - `docs/content/public/assets`도 튜토리얼 mp4를 다수 포함한다.
- 제거 가능성: 낮음~중간.
- caveat:
  - 문서 콘텐츠 자산은 빌드 산출물이 아니라 사용자 문서의 원본일 가능성이 높다.
  - 중복 제거는 “삭제”보다 공용 `docs/content/public/assets`로 이동하고 각 locale 문서가 공용 경로를 참조하게 만드는 문서 구조 변경이 필요하다.

## 3. 삭제하면 안 되는 영역

아래는 이름만 보면 생성물/데모처럼 보이거나 크기가 커도 현재 repo 구조상 소스·문서·배포에 필요한 항목이다.

- `docs/.vitepress/`
  - `.gitignore`가 지우라고 하는 것은 `**/.vitepress/cache/`이지 `.vitepress` 전체가 아니다.
  - 실제 `docs/.vitepress`에는 VitePress config, theme, components, composables, data loader가 들어 있다.
- `packages/scenarios-stage-tamagotchi-browser/src/**`, `packages/scenarios-stage-tamagotchi-electron/src/**`, `packages/vishot-*`
  - 루트 `test:run`이 `test-vishot:run`을 포함하고, 시나리오 README가 docs screenshot pipeline의 소스임을 명시한다.
- `packages/scenarios-stage-tamagotchi-browser/artifacts/raw/*.avif`
  - 산출물이지만 현재 소스 import 대상이다. 삭제는 조건부 작업으로만 취급한다.
- `packages/stage-ui/src/assets/vrm/models/*/preview.png`, `packages/stage-ui/src/assets/live2d/models/*/preview.png`
  - `.gitignore`는 실제 모델 바이너리류를 막기 위한 패턴이지만, 현재 tracked preview 이미지는 UI/샘플 표시 자산일 수 있다. 무조건 삭제 금지.
- `packages/font-*`
  - 크기가 크지만 package 설명상 AIRI에서 사용하는 font package이며, workspace package다.
- `apps/stage-pocket/android/**`, `apps/stage-pocket/ios/**`의 소스/프로젝트 파일
  - nested `.gitignore`에 열거된 build/Pods/DerivedData/Capacitor generated 파일만 삭제 대상이다. Xcode project, Swift/Kotlin/Gradle 설정은 삭제 금지.
- `engines/stage-tamagotchi-godot/`
  - root `build:engines`, `typecheck:engines` 대상이고 Godot engine workspace로 명시되어 있다. `.godot/`, engine-local `/android/`만 ignore 대상이다.
- `patches/`
  - `pnpm-workspace.yaml`의 `patchedDependencies`가 실제 patch 파일들을 참조한다. 삭제하면 의존성 설치/패치 적용이 깨질 수 있다.
- `plugins/airi-plugin-game-chess/`
  - 현재 tracked 파일은 `package.json`뿐이라 불완전해 보이지만, 여러 테스트와 시나리오 문서가 plugin id `airi-plugin-game-chess`를 기준 fixture로 사용한다. 제거 판단은 별도 소유자 확인 필요.
- `services/*/data` 중 tracked migration/seed 성격 파일
  - 예: `services/satori-bot/data/.gitkeep`, `services/satori-bot/data/db.json`는 디렉터리 정책상 persistence와 섞여 있어 보인다. `.gitignore`는 DB persistence를 커버하지만 이미 tracked인 파일은 의미를 확인해야 한다.

## 4. 추천 정리 순서

1. 먼저 **실제 생성물만 삭제**한다.
   - `dist`, `out`, `coverage`, `.turbo`, `.wxt`, `.vite-inspect*`, `*.tsbuildinfo`, native build output, `node_modules` 등 `.gitignore`에 명시된 산출물.
   - 현재 검사에서는 대부분 존재하지 않았다.
2. 세션 종료 후 `.omx/cache`, `.omx/logs` 같은 로컬 OMX 산출물을 정리한다.
   - 현재 Ralph 세션 중이므로 즉시 삭제 금지.
3. 레거시 README-only 후보를 문서 링크와 함께 정리한다.
   - 1순위: `packages/drizzle-duckdb-wasm/`, `packages/duckdb-wasm/`
4. 배포 채널 유지 여부를 확인한 뒤 `bucket/airi.json`을 갱신하거나 제거한다.
5. 데모/실험 앱은 소유자 의사결정 후 제거한다.
   - 후보: `apps/component-calling/`
   - 확인: CI/build 대상, README/문서 링크, 기능 실험 지속 여부.
6. docs screenshot raw artifact는 삭제보다 파이프라인 개선 대상으로 다룬다.
   - `artifacts/raw`를 삭제하려면 import 구조 또는 CI artifact generation을 먼저 바꿔야 한다.

## 5. 신뢰도

- 높음: `.gitignore`에 명시된 build/cache/test/native generated 산출물, VSCode `src/generated`, README-only moved packages의 레거시 성격.
- 중간: `bucket/airi.json`, `apps/component-calling` 제거 가능성. 배포/실험 사용 여부가 repository evidence만으로 완전히 확정되지 않는다.
- 낮음: docs 대용량 미디어 제거. 중복처럼 보이는 파일은 많지만 문서 원본 자산일 수 있어 삭제 대신 구조 개선 검토가 필요하다.
