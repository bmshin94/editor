# Pascal Editor 분석 정리 📝

> 카리나랑 같이 이 레포 뜯어보면서 나눈 대화 정리본이야! ✨

## 📌 저장소 주소

| 구분 | 주소 |
|---|---|
| **내 저장소 (포크)** | https://github.com/bmshin94/editor |
| **원본 저장소** | https://github.com/pascalorg/editor |
| **공식 사이트 / 문서** | https://editor.pascal.app |
| **npm 패키지** | https://www.npmjs.com/org/pascal-app |
| **Discord 커뮤니티** | https://discord.gg/XRKsDcpqgS |
| **플러그인 예제 (Nature)** | https://github.com/pascalorg/plugin-trees |

- **작업 브랜치:** `claude/github-popular-folder-analysis-1bs6rn`
- **라이선스:** MIT (Copyright © 2026 Pascal Group Inc.)
- **버전:** `1.0.0-beta.1` (2026-07-30 릴리즈)

---

## 1. 이게 뭐하는 프로젝트야? 🏠

**한 줄 요약: 브라우저에서 돌아가는 3D 건물 설계 에디터**

심즈나 마인크래프트에서 집 짓는 것처럼, 바닥 깔고 → 벽 세우고 → 문·창문 달고 →
지붕 얹는 걸 **웹 브라우저에서** 하는 툴이야. 근데 게임이 아니라 진짜 건축 설계용!

**진짜 차별점은 AI 연동** 🤖 — MCP(Model Context Protocol) 서버가 내장돼 있어서
Claude 같은 AI 에이전트한테 *"3층집 지어줘, 남쪽에 큰 창문 달고"* 라고 말하면
실제로 3D 도면을 만들어줘. 사진을 3D로 바꿔주는 기능도 있어.

### 기술 스택

| 영역 | 사용 기술 |
|---|---|
| 3D 렌더링 | React Three Fiber + Three.js + **WebGPU** |
| 프레임워크 | Next.js 16 / React 19 |
| 상태관리 | Zustand (+ Zundo로 실행취소) |
| 모노레포 | Turborepo + Bun 1.3.14 |
| 언어 | TypeScript 6 |
| 린트/포맷 | Biome (+ Ultracite) |
| 저장소 | IndexedDB (브라우저) + SQLite (서버) |

**규모:** TS/TSX 파일 약 **2,009개** — 장난감 프로젝트 아님!

---

## 2. 폴더 구조 📁

### `apps/` — 실제로 실행되는 앱

| 폴더 | 설명 |
|---|---|
| `editor/` | **메인 에디터 앱** (Next.js). `bun dev` 하면 `localhost:3002`에 뜸 |
| `ifc-converter/` | IFC(건축 표준 파일) → Pascal JSON 변환기. ⚠️ **early alpha** |

### `packages/` — 재사용 가능한 npm 패키지 (진짜 알맹이)

| 패키지 | 역할 |
|---|---|
| **`core`** | 🧠 두뇌 — 노드 스키마, 씬 상태, 이벤트 버스, 공간 쿼리. **Three.js 절대 안 씀** (순수 로직) |
| **`viewer`** | 👀 눈 — 3D 렌더링, 카메라, 조명, 포스트프로세싱 |
| **`editor`** | ✋ 손 — 편집 툴, 패널, 선택 UI |
| **`nodes`** | 📦 부품창고 — 벽, 문, 창문, 계단, 지붕, 굴뚝, 태양광, 배관·덕트·HVAC까지 **47종** |
| **`mcp`** | 🤖 AI 연동 — MCP 서버, 씬 저장소 |
| **`cli`** | 클론 없이 설치·실행하는 `pascal` 명령어 |
| **`capture-protocol` / `capture-viewer`** | 실측 캡처(포인트클라우드, 디바이스 모션) 연동 |
| **`ui`** | 공통 UI 컴포넌트 |
| **`eslint-config` / `typescript-config`** | 공유 설정 |

### 그 외

| 폴더/파일 | 설명 |
|---|---|
| `wiki/architecture/` | 💎 **보물창고** — 아키텍처 문서 22개 |
| `.agents/skills/` | AI 에이전트용 스킬 (`review-architecture`, `open-pr`) |
| `.claude/` `.cursor/` `.codex/` | 위 폴더의 심볼릭 링크 (툴별 호환) |
| `AGENTS.md` | AI 에이전트 작업 규칙 (원래 `CLAUDE.md`가 이걸 가리키는 심링크였음) |
| `tooling/` | 빌드·릴리즈 도구 |

> 💡 **참고:** 원본에선 `CLAUDE.md`가 `AGENTS.md`의 심볼릭 링크였는데,
> 지금은 카리나 페르소나 파일로 교체돼 있어. 그래서 아키텍처 규칙이 자동으로
> 안 읽히니까, 코드 작업할 땐 `AGENTS.md`를 따로 챙겨 읽어야 해!

---

## 3. 작동 원리 ⚙️

### 노드 계층 구조

```
Site (부지)
└── Building (건물)
    └── Level (층)
        ├── Wall (벽) → Item (문, 창문)
        ├── Slab (바닥)
        ├── Ceiling (천장) → Item (조명)
        ├── Roof (지붕)
        ├── Zone (구역)
        ├── Scan (3D 참조)
        └── Guide (2D 참조)
```

- 트리가 아니라 **평평한 딕셔너리**(`Record<id, Node>`)로 저장 → 검색 빠름 ⚡
- 부모-자식 관계는 `parentId` / `children`로 표현

### Dirty Node 패턴 (성능의 핵심)

1. 값이 바뀌면 그 노드가 **`dirtyNodes`에 등록**됨
2. **System**들이 매 프레임(`useFrame`) dirty 목록만 골라서 지오메트리 재계산
3. 처리 끝나면 목록에서 제거

→ 벽 두께 하나 바꿔도 씬 전체를 다시 안 그림. **게임엔진 ECS 패턴을 React에 녹인 구조** 👏

### 상태 저장소 (Zustand 3개)

| 스토어 | 위치 | 역할 |
|---|---|---|
| `useScene` | `core` | 씬 데이터. IndexedDB 저장 + 50단계 실행취소 |
| `useViewer` | `viewer` | 선택 상태, 층 표시 모드(쌓기/분해/단독), 카메라 모드 |
| `useEditor` | `apps/editor` | 활성 툴, 패널 상태, 에디터 설정 |

### 레이어 규칙 (엄격함!)

- `core`는 Three.js / viewer / editor를 **import 금지**
- `viewer`는 `useEditor`, 에디터 툴, 모드를 **알면 안 됨**
- `apps/editor`가 편집 경험을 소유하고, `<Viewer>`에 props/children으로 주입

---

## 4. 설치 & 사용법 🚀

### 준비물

| 필요한 거 | 요구 버전 | 확인된 환경 |
|---|---|---|
| **Bun** | 1.3+ (권장 1.3.14) | `1.3.11` |
| **Node.js** | 20.9+ | `v22.22.2` |

> ⚠️ `package.json`의 `packageManager`는 `bun@1.3.14`야. 설치 중 문제 생기면 `bun upgrade`!

### 방법 A: 소스로 실행 (이 레포)

```bash
cd /home/user/editor
bun install     # 5~10분 소요
bun dev
```

👉 http://localhost:3002

| 명령어 | 설명 |
|---|---|
| `bun dev` | 개발 서버 (핫 리로드) |
| `bun kill` | 3002 포트 강제 종료 |
| `bun restart` | 캐시 삭제 후 재시작 (꼬였을 때 특효약) |
| `bun run build` | 전체 빌드 |
| `bun check:fix` | 린트/포맷 자동 수정 |
| `bun check-types` | 타입 검사 |
| `bun run test` | 테스트 실행 |

포트 변경: `cp .env.example .env` → `PORT` 수정

### 방법 B: CLI (클론 불필요, 제일 간단)

```bash
npx @pascal-app/cli editor
```

- 프로젝트는 `~/.pascal/data/pascal.db`에 저장 (런타임 업데이트해도 안 날아감)
- **Node.js 22.13+** 필요 / 공식 지원은 macOS, 리눅스는 검증 중

| 명령어 | 설명 |
|---|---|
| `pascal editor` | 설치·실행·브라우저 열기 |
| `pascal status` | 상태 확인 |
| `pascal projects` | 프로젝트 목록 |
| `pascal logs --follow` | 로그 실시간 |
| `pascal stop` | 종료 |
| `pascal doctor` | 🩺 진단 |
| `pascal update` | 런타임 업데이트 (실패 시 롤백) |

### 방법 C: 도커

```bash
docker compose up -d
```

👉 http://localhost:3000

> ⚠️ **컨테이너 포트는 3000에서 바꾸지 말 것!** `/scenes` 페이지가 빌드 타임에
> 박힌 주소로 자기 API를 호출해서, 포트를 바꾸면 500 에러가 남.
> 다른 주소로 호스팅하려면:
> ```bash
> MINT_PASCAL_HOST_ORIGIN=https://내주소.com docker compose up -d
> ```

데이터는 `pascal-data` 볼륨에 남아서 `docker compose down` 해도 안 날아감 💾

### AI(MCP) 연결

```bash
pascal mcp setup claude    # Claude Code
pascal mcp setup codex     # Codex
pascal mcp config          # JSON 설정 출력 (기타 클라이언트)
```

**보안:** `127.0.0.1`에만 바인딩 + 랜덤 토큰 인증 (토큰은 클라이언트 설정에 안 들어감)

**MCP 도구 35개 예시:**
`create-wall`(벽) · `cut-opening`(문·창 뚫기) · `place-item`(가구) · `check-collisions`(충돌) ·
`door-clearance`(문 열림 공간) · `measure`(치수) · `photo-to-scene`(사진→3D) ·
`export-glb` / `export-json` · `undo` / `redo` · `validate-scene` 등

**MCP 프롬프트 템플릿:** `from-brief`, `renovation-from-photos`, `iterate-on-feedback`, `scene-guidance`

### 에디터 사용 흐름

1. `localhost:3002` 접속
2. **바닥(Slab)** 깔기
3. **벽(Wall)** 클릭-드래그로 그리기
4. 벽 위에 **문/창문** 드래그 → 자동으로 구멍 뚫림
5. **가구/아이템** 배치 (충돌 자동 체크)
6. **지붕·계단·지형** 추가
7. `/scenes` 페이지에서 프로젝트 관리
8. **GLB / STL / OBJ** 내보내기

2D 평면도 ↔ 3D 뷰 전환 가능, 층 표시 모드 3가지(쌓기/분해/단독)

---

## 5. 💰 수익화 아이디어

MIT 라이선스라 **상업적 이용 100% 자유** (저작권 표시만 유지하면 됨)

### 현실성 높은 순 TOP 5

**1. 인테리어 견적 자동화 SaaS** 💸
- 고객이 평면도/사진 업로드 → AI가 3D 변환 (`photo-to-scene`)
- 벽 면적·바닥 평수 자동 계산 → 자재비·시공비 견적서 자동 생성
- 인테리어 업체 대상 월 구독 (5~30만원)
- 현재 인테리어 견적은 대부분 수기 → 시장성 있음

**2. 에셋 팩 / 플러그인 판매** 🪑
- `wiki/architecture/plugin-authoring.md`에 플러그인 규격이 이미 공개돼 있음
- 한국형 가구·자재 팩 (붙박이장, 한옥 요소, 아파트 새시 등)
- 본체 무료 + 에셋 유료 = 클래식한 모델
- 경쟁자 거의 없음

```ts
export const myPlugin: Plugin = {
  id: 'acme:furniture-pack',
  apiVersion: 1,
  nodes: [couchDefinition, armchairDefinition],
}
```

**3. 부동산 3D 매물 뷰어** 🏢
- `@pascal-app/viewer`만 npm으로 뽑아 부동산 사이트에 임베드
- 매물당 3D 투어 → 건당 과금 or 중개사 구독
- 분양/모델하우스 시장은 이미 예산이 도는 곳

**4. 가구 커머스 제휴 (수수료 모델)** 🛒
- 3D로 방 꾸미기 → 배치한 가구를 실제 상품과 연결
- "이 소파 사기" → 제휴 수수료 (오늘의집 스타일)
- 사용자는 무료, 수익은 커머스에서 → 스케일이 제일 큼

**5. IFC 변환 서비스** 📐
- `apps/ifc-converter`가 아직 early alpha (README에 명시)
- 건축사무소는 IFC를 일상적으로 다루는데 웹 뷰어가 비쌈
- 제대로 고쳐서 B2B 유료 변환/뷰잉 서비스로 → 난이도 높지만 단가도 높음

### 🌟 최우선 추천: "AI 리모델링 시뮬레이터"

```
📸 집 사진 업로드
  → 🤖 AI가 3D 공간으로 변환
  → 💬 "여기 벽 트고 주방 넓혀줘" 채팅으로 수정
  → 🏠 비포/애프터 3D 비교
  → 💰 예상 공사비 자동 산출
```

**이유:** 필요한 부품(`photo-to-scene`, MCP, 3D 뷰어, 측정 도구)이 전부 레포에 있음.
"AI + 인테리어"는 현재 가장 수요가 큰 조합. B2C/B2B 둘 다 가능.

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 프로젝트 1개, 워터마크 |
| Pro | 월 9,900원 | 무제한 + 고화질 렌더 + 내보내기 |
| Business | 월 99,000원 | 견적서 + 브랜딩 + 고객 공유 링크 |

### ⚠️ 시작 전 체크리스트

1. **상표는 별개** — MIT는 코드만 허용. "Pascal" 이름/로고는 사용 불가. 자체 브랜딩 필요
2. **서드파티 플러그인 주의** — `apps/editor/package.json`에 GitHub에서 직접 받아오는
   외부 의존성 3개 존재: `@mint/pascal-plugin`(mintdotgg), `@pascal-app/plugin-trees`,
   `@pascal-app/plugin-streetscape`. **라이선스 개별 확인 필요**, 상업용이면 제거 검토
3. **아직 1.0 베타** — 프로덕션 투입 전 안정화 기간 필요
4. **경쟁자** — 플래너5D, 룸플래너, 오늘의집 3D 등. 차별점은 **"AI 대화로 설계"**

---

## 6. 🐘 PHP로 만들 수 있어?

**결론: 반은 되고 반은 안 됨. 그래서 하이브리드가 정답!**

### ❌ PHP로 못 하는 부분: 3D 렌더링

3D는 브라우저 안에서 GPU를 직접 쓰는 영역(WebGPU/WebGL).
PHP는 서버 언어라 브라우저 화면을 직접 그릴 수 없음.

→ **화면 부분은 npm 패키지로 그대로 가져다 쓰는 게 정답.** 다시 만들 이유 없음.

### ✅ PHP로 충분히 되는 부분: 서버 전부

실제 API를 까보니 **전부 단순 CRUD**였음. 3D 데이터도 `graph`라는 **JSON 한 덩어리**로 통째 저장.

| 엔드포인트 | 하는 일 |
|---|---|
| `GET /api/scenes` | 도면 목록 |
| `POST /api/scenes` | 새 도면 저장 |
| `GET /api/scenes/{id}` | 도면 불러오기 |
| `PUT /api/scenes/{id}` | 도면 수정 |
| `PATCH /api/scenes/{id}` | 이름만 변경 |
| `DELETE /api/scenes/{id}` | 삭제 |
| `GET /api/scenes/{id}/events` | 실시간 동기화 (SSE) |
| `GET /api/health` | 헬스 체크 |

### 추천 아키텍처

```
┌─────────────────────────────┐
│   브라우저 (사용자 화면)      │
│   3D 뷰어 = 그대로 사용       │  ← JS (@pascal-app/viewer)
└──────────┬──────────────────┘
           │  JSON 주고받기
┌──────────▼──────────────────┐
│      🐘 PHP / Laravel        │  ← 직접 개발 영역
│  · 회원가입 / 로그인          │
│  · 도면 저장 / 불러오기       │
│  · 💳 결제 (아임포트/토스)     │
│  · 📄 견적서 PDF 생성         │
│  · 👥 팀 공유, 권한 관리       │
│  · 📊 관리자 페이지           │
└──────────┬──────────────────┘
           │
      MySQL / MariaDB
```

### Laravel 예시

```php
// routes/api.php
Route::middleware('auth:sanctum')->group(function () {
    Route::get   ('/scenes',      [SceneController::class, 'index']);
    Route::post  ('/scenes',      [SceneController::class, 'store']);
    Route::get   ('/scenes/{id}', [SceneController::class, 'show']);
    Route::put   ('/scenes/{id}', [SceneController::class, 'update']);
    Route::delete('/scenes/{id}', [SceneController::class, 'destroy']);
});
```

```php
// SceneController.php
public function store(Request $r)
{
    $data = $r->validate([
        'name'  => 'required|string|max:200',
        'graph' => 'required|array',        // 3D 데이터는 그냥 JSON
    ]);

    $scene = Scene::create([
        'user_id' => $r->user()->id,
        'name'    => $data['name'],
        'graph'   => $data['graph'],
        'version' => 1,
    ]);

    return response()->json($scene, 201);
}
```

```php
// 마이그레이션
Schema::create('scenes', function (Blueprint $t) {
    $t->uuid('id')->primary();
    $t->foreignId('user_id');
    $t->string('name', 200);
    $t->json('graph');              // 씬 전체가 여기 들어감
    $t->string('thumbnail_url')->nullable();
    $t->unsignedInteger('version')->default(1);
    $t->timestamps();
});
```

### ⚠️ 미리 알아둘 함정

**1. 버전 충돌 처리 (409)**
원본은 `expectedVersion`을 받아서 안 맞으면 409를 반환. 동시 편집 시 덮어쓰기 방지용.

```php
if ($r->expectedVersion !== null && $scene->version !== $r->expectedVersion) {
    return response()->json(['error' => 'version_conflict'], 409);
}
```

**2. 빈 도면 덮어쓰기 방어 (중요!)**
원본 주석: *"노드 0개짜리 그래프로 덮어쓰는 건 의도적 삭제보다 하이드레이션 경합이나
버그일 확률이 훨씬 높다"*. `force: true` 없으면 거부하고 409 `empty_graph_rejected` 반환.
**이건 반드시 따라 구현할 것** — 안 하면 고객 도면이 조용히 날아감.

**3. 실시간 동기화(SSE)는 PHP가 약함**
전통적인 PHP-FPM은 SSE 연결 하나가 워커를 계속 점유함.
- A) 실시간 빼고 주기적 저장으로 (단독 사용이면 충분)
- B) Laravel Reverb / Swoole / ReactPHP 사용
- C) 실시간만 Node로 분리

**4. MCP(AI 연동)는 TypeScript**
`@pascal-app/mcp`는 TS로 작성됨. PHP 재작성보다 Node로 따로 띄우는 게 효율적.

**5. 프론트 빌드에는 Node 필요**
배포 시 한 번만 빌드하고, 결과물(HTML/JS)을 PHP 서버에 올리면 됨.

### 최종 추천

> **"3D 화면은 있는 거 그대로, 서버는 PHP로 새로!"**

- 3D 엔진 재작성 = 수년 소요 → 하지 말 것
- PHP로 결제·회원·견적 같은 **실제 수익 로직**을 빠르게 구현
- 인테리어 견적 SaaS는 PHP 백엔드와 궁합이 좋음 (PDF 생성, 결제, 관리자 페이지)
- 일반 웹호스팅에 배포 가능

---

## 7. 다음 스텝 ✅

- [ ] `bun install && bun dev` 로 실제 실행해보기
- [ ] `wiki/architecture/README.md` 정독 (아키텍처 이해의 핵심)
- [ ] `packages/mcp/src` 구경 (MCP 서버 구현 참고용)
- [ ] `CLAUDE.md` ↔ `AGENTS.md` 관계 정리 방향 결정
- [ ] 수익화 아이템 하나 골라서 프로토타입
- [ ] 서드파티 플러그인 3개 라이선스 확인

---

*정리: 카리나 💖 · 브랜치 `claude/github-popular-folder-analysis-1bs6rn`*
