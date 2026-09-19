# 📚 Temporal Java Samples 전수조사 & 분석 정리

> 카리나 💖 와 함께한 `samples-java` 저장소 완전 분석 기록
> 작성일: 2026-09-19

---

## 🔗 관련 GitHub 주소

| 항목 | 주소 |
|---|---|
| 🏠 **이 저장소 (포크)** | https://github.com/bmshin94/samples-java |
| ⭐ **원본 (upstream)** | https://github.com/temporalio/samples-java |
| 🧩 Temporal 서버 | https://github.com/temporalio/temporal |
| ☕ Java SDK | https://github.com/temporalio/sdk-java |
| 🤖 Temporal Spring AI 통합 | https://github.com/temporalio/sdk-java/tree/master/temporal-spring-ai |
| 💻 Temporal CLI | https://github.com/temporalio/cli |
| 📖 Java SDK 가이드 | https://docs.temporal.io/dev-guide/java |
| 🔌 MCP Filesystem 서버 | https://github.com/modelcontextprotocol/servers |

### 다른 언어 SDK

| 언어 | 저장소 |
|---|---|
| 🟦 TypeScript | https://github.com/temporalio/sdk-typescript |
| 🐍 Python | https://github.com/temporalio/sdk-python |
| 🐹 Go | https://github.com/temporalio/sdk-go |
| 🐘 PHP | https://github.com/temporalio/sdk-php |
| 🟪 .NET | https://github.com/temporalio/sdk-dotnet |

---

## 1️⃣ 이게 뭐하는 저장소야?

### 한 줄 정의

**Temporal**(실패하지 않는 워크플로우 엔진)의 **Java SDK 공식 예제 모음집**.
`temporalio/samples-java` 를 `bmshin94` 계정으로 포크한 저장소.

### 전수조사 결과

| 확인 항목 | 결과 |
|---|---|
| Java 파일 수 | **564개** |
| README 수 | **57개** (샘플마다 별도 설명서) |
| core 패키지 수 | **44개** 주제 |
| Gradle 서브프로젝트 | 9개 (`core`, `springai:basic/mcp/multimodel/rag`, `springboot`, `springboot-basic`, `lambda-worker:starter/worker`) |
| Java SDK 버전 | `1.39.0` |
| 최소 Java 버전 | **17** |
| 커밋 히스토리 | PR #800번대 → 수년간 공식 관리된 저장소 |
| 로컬 추가 파일 | `CLAUDE.md` (카리나 페르소나, 커밋 `44cbcd4`) |

### Temporal이 뭔데?

일반 백엔드 코드는 프로세스가 죽으면 메모리 상태가 증발한다.

```java
결제하기();    // ✅ 성공
포인트적립();  // ✅ 성공
배송요청();    // 💥 서버 사망 → 재시작하면 처음부터? 중복 결제 대참사!
```

Temporal은 **워크플로우의 모든 실행 기록(History)을 저장**해두고,
복구 시 **이미 끝난 단계는 건너뛰고 죽은 지점부터 재개**한다.

이것이 **Durable Execution (내구성 있는 실행)**.

> 🎮 비유: **자동 세이브가 있는 RPG 게임**. 정전돼도 던전 입구부터 재개!

---

## 2️⃣ 핵심 개념 — 3대 등장인물

| Temporal 용어 | 영화 촬영장 비유 | 역할 |
|---|---|---|
| **Workflow** | 📜 대본 | 실행 순서만 정의. 즉흥연기(비결정적 코드) 절대 금지 |
| **Activity** | 🎭 배우/스태프 | 실제 위험한 일(API·DB·파일) 수행. 실패 시 자동 재시도 |
| **Worker** | 🏢 제작사 직원 | Task Queue를 폴링해 워크플로우/액티비티를 실행 |
| **Task Queue** | 📋 스케줄판 | 일감이 쌓이는 큐 |
| **History** | 🎞️ 촬영 필름 | 실행 기록 = 세이브 파일 |

### ⚠️ 최대 규칙: 워크플로우는 결정론적이어야 한다

Temporal은 복구 시 워크플로우 코드를 **처음부터 Replay** 한다.
따라서 같은 입력이면 **항상 같은 결과**가 나와야 한다.

```java
// ❌ 워크플로우 안에서 금지
UUID.randomUUID();          // 매번 값이 달라짐
System.currentTimeMillis(); // 매번 값이 달라짐
db.save(user);              // 네트워크/DB 호출

// ✅ 올바른 방법
Workflow.randomUUID();         // 기록된 값 재사용
Workflow.currentTimeMillis();  // 기록된 시간 재사용
activities.saveUser(user);     // DB는 Activity로!
```

### `HelloActivity.java` 구조 (이것만 알면 564개가 읽힌다)

```java
// 1. 큐 이름 정의
static final String TASK_QUEUE = "HelloActivityTaskQueue";

// 2. Activity 스텁 = 프록시 객체 (호출하면 원격 실행!)
private final GreetingActivities activities =
    Workflow.newActivityStub(
        GreetingActivities.class,
        ActivityOptions.newBuilder()
            .setStartToCloseTimeout(Duration.ofSeconds(2))
            .build());

// 3. 서버 연결 (환경설정에서 로드)
ClientConfigProfile profile = ClientConfigProfile.load();
WorkflowServiceStubs service =
    WorkflowServiceStubs.newServiceStubs(profile.toWorkflowServiceStubsOptions());
WorkflowClient client = WorkflowClient.newInstance(service, profile.toWorkflowClientOptions());

// 4. Worker 등록 & 시작
WorkerFactory factory = WorkerFactory.newInstance(client);
Worker worker = factory.newWorker(TASK_QUEUE);
worker.registerWorkflowImplementationTypes(GreetingWorkflowImpl.class);
worker.registerActivitiesImplementations(new GreetingActivitiesImpl());
factory.start();

// 5. 실행 — 이 한 줄이 몇 달짜리 워크플로우여도 동작한다!
String greeting = workflow.getGreeting("World");
```

### 테스트

`TestWorkflowRule` 로 **Temporal 서버 없이** 테스트 가능하고,
**Time Skipping** 덕분에 `sleep(30일)` 워크플로우도 0.1초에 검증된다.

```java
@Rule
public TestWorkflowRule testWorkflowRule =
    TestWorkflowRule.newBuilder()
        .setWorkflowTypes(GreetingWorkflowImpl.class)
        .setDoNotStart(true)
        .build();

// Mockito 모킹 시 withoutAnnotations() 필수!
GreetingActivities activities =
    mock(GreetingActivities.class, withSettings().withoutAnnotations());
```

---

## 📁 폴더 구조 해부

```
samples-java/
├── core/              💎 메인. 44개 주제 패키지
│   └── hello/         └─ 30여 개 기본 예제
├── springboot/        Spring Boot 통합 (웹 UI, 포트 3030)
├── springboot-basic/  최소 구성 Spring Boot
├── springai/          🔥 AI 에이전트 (basic / mcp / multimodel / rag)
├── lambda-worker/     AWS Lambda 위 Worker + Deployment Versioning
├── docker/            CI용 docker-compose
├── .github/workflows/ CI (단위테스트 + spotless 포맷 검사)
└── CLAUDE.md          카리나 페르소나 가이드
```

### core/ 주요 패키지

`hello`, `moneytransfer`, `moneybatch`, `bookingsaga`, `bookingsyncsaga`,
`fileprocessing`, `batch`(heartbeating/iterator/slidingwindow), `dsl`, `polling`,
`safemessagepassing`, `packetdelivery`, `workerversioning`, `envconfig`,
`nexus` 계열 7종, `metrics`, `tracing`, `encryptedpayloads`,
`keymanagementencryption`, `ssl`, `apikey`, `countinterceptor`,
`retryonsignalinterceptor`, `excludefrominterceptor`, `customannotation`,
`payloadconverter`, `listworkflows`, `updatabletimer`, `workflowstreams`,
`standaloneactivities`, `earlyreturn`, `sleepfordays`, `terminateworkflow` 등

### 🔥 springai/ — 가장 주목할 모듈

| 모듈 | 내용 | 필요 키 |
|---|---|---|
| `basic` | Tool 3종(Activity형 / 워크플로우형 / SideEffect형) + 대화 메모리 | `OPENAI_API_KEY` |
| `mcp` | **MCP 서버 연동** (`@modelcontextprotocol/server-filesystem`을 npx로 실행) | `OPENAI_API_KEY` |
| `multimodel` | OpenAI + Anthropic 동시 사용, Claude 확장 사고 모드 | `OPENAI_API_KEY` + `ANTHROPIC_API_KEY` |
| `rag` | 벡터스토어 + 임베딩 RAG | `OPENAI_API_KEY` |

```java
// ChatWorkflowImpl.java — Durable AI Agent 의 핵심
this.chatClient =
    TemporalChatClient.builder(activityChatModel)
        .defaultSystem(systemPrompt)
        .defaultTools(weatherTool, stringTools, timestampTools)
        .defaultAdvisors(PromptChatMemoryAdvisor.builder(chatMemory).build())
        .build();
```

#### Tool 3분류 — Temporal + AI 결합의 정수

| 분류 | 예시 | 실행 위치 | 이유 |
|---|---|---|---|
| 🌐 Activity Tool | `WeatherActivity` | 워커(외부) | 외부 I/O = 비결정적 |
| 🧮 Workflow Tool | `StringTools` | 워크플로우 내부 | 순수 계산 = 결정적 |
| ⏱️ SideEffect Tool | `TimestampTools` | `Workflow.sideEffect()` | 가벼운 비결정 → 결과만 기록 |

#### MCP 샘플의 경쟁 조건 해법 (보석 같은 코드)

```java
@Override
public void chat(String message) {
    // 시그널이 초기화보다 먼저 도착할 수 있음
    Workflow.await(() -> initialized);
    messageCount++;
    lastResponse = chatClient.prompt().user(message).call().content();
}
```

---

## 🛠️ 설치 및 사용법

### 사전 준비

```bash
java -version                      # 17 이상 필수
brew install temporal              # macOS (Temporal CLI)
# curl -sSf https://temporal.download/cli.sh | sh   # Linux
```

### 로컬 서버 실행

```bash
temporal server start-dev          # Web UI: http://localhost:8233
```

### 클론 & 빌드

```bash
git clone https://github.com/bmshin94/samples-java
cd samples-java
./gradlew build
```

### 샘플 실행

```bash
# Core 샘플
./gradlew -q execute -PmainClass=io.temporal.samples.hello.HelloActivity

# Spring Boot 샘플 (http://localhost:3030)
./gradlew :springboot:bootRun
./gradlew :springboot-basic:bootRun

# Spring AI 샘플
export OPENAI_API_KEY="sk-..."
./gradlew :springai:basic:bootRun
./gradlew :springai:mcp:bootRun
./gradlew :springai:multimodel:bootRun      # ANTHROPIC_API_KEY 추가 필요
./gradlew :springai:rag:bootRun

# Temporal Cloud 프로파일
./gradlew bootRun --args='--spring.profiles.active=tc'
```

### 추천 학습 순서

```
HelloActivity → HelloActivityRetry → HelloSignal/Query → HelloSaga
→ moneytransfer → bookingsaga → springboot/hello
→ springai/basic → springai/mcp
```

---

## 🧩 플러그인? 스킬? MCP?

> **셋 다 아님. "SDK 예제 저장소(Sample Repository)"다.**

| 개념 | 이 저장소와의 관계 |
|---|---|
| 플러그인 | ❌ 아님 |
| 스킬 | ❌ 아님 (단, `CLAUDE.md`는 프로젝트 지시문으로 존재) |
| MCP 서버 | ⚠️ 아님 — **MCP 클라이언트를 쓰는 예제**를 포함 |
| SDK 샘플 | ✅ 정확히 이것 |

```yaml
# springai/mcp/src/main/resources/application.yaml
spring:
  ai:
    mcp:
      client:                     # server가 아니라 client!
        stdio:
          connections:
            filesystem:
              command: npx
              args: ["-y", "@modelcontextprotocol/server-filesystem", "${MCP_ALLOWED_PATH:/tmp/mcp-example}"]
```

→ **"MCP 서버의 툴을 Temporal Activity로 감싸서 durable하게 소비하는 법"** 교재.

---

## 🔑 API 토큰 필요 여부

| 모듈 | 토큰 | 종류 |
|---|---|---|
| `core/` 대부분 | ❌ 불필요 | 로컬 Temporal 서버만 |
| `springboot/`, `springboot-basic/` | ❌ 불필요 | — |
| `core/apikey/` | ⚠️ 필요 | Temporal Cloud API Key |
| `core/ssl/` | ⚠️ 필요 | mTLS 인증서 |
| `springai/basic`, `mcp`, `rag` | ✅ 필요 | `OPENAI_API_KEY` |
| `springai/multimodel` | ✅ 필요 | `OPENAI_API_KEY` + `ANTHROPIC_API_KEY` |
| `lambda-worker/` | ✅ 필요 | AWS 자격증명 + Temporal Cloud API Key |
| `keymanagementencryption/` | ✅ 필요 | AWS KMS 자격증명 |

토큰은 두 계열로 나뉜다.

1. **Temporal 접속용** — API Key / mTLS / TOML(`ClientConfigProfile.load()`)
2. **LLM 제공자용** — `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`

모든 설정은 환경변수 주입(`${OPENAI_API_KEY}`)이라 하드코딩이 없다. 👍

> **결론: Temporal 자체 학습은 100% 무료.** AI 샘플만 LLM 토큰 필요.

---

## 🌟 왜 GitHub에서 유명할까?

1. **공식 저장소의 권위** — `temporalio` 조직 공식, PR #800번대
2. **Temporal 자체가 핫함** — Uber Cadence 출신, Snap/Netflix/Stripe/Coinbase/Datadog 등이 프로덕션 사용
3. **압도적 커버리지** — 564 파일 / 57 README / 44 주제
4. **문서 품질** — 주석이 교과서 수준. `@@@SNIPSTART` 마커로 공식 문서와 자동 동기화 → **docs.temporal.io의 원천 소스**
5. **AI 에이전트 열풍 정확히 타격** — `springai/`가 "Durable AI Agents" 수요에 직결
6. **엔지니어링 품질 레퍼런스** — `-Werror`, spotless + googleJavaFormat, errorprone, GitHub Actions(액션 SHA 핀 고정)

---

## 🤖 로컬 에이전트 구축에 도움이 될까? → ✅ 매우

### 도움 되는 부분

1. **에이전트 실행 안정성 설계도** — 타임아웃/재시도/상태 복원
   ```java
   ActivityOptions.newBuilder()
       .setStartToCloseTimeout(Duration.ofSeconds(30))
       .setRetryOptions(RetryOptions.newBuilder().setMaximumAttempts(3).build())
       .build()
   ```
2. **MCP 통합 실전 레퍼런스** — 동적 툴 발견 & 등록
   ```java
   ActivityMcpClient mcpClient = ActivityMcpClient.create();
   mcpTools = McpToolCallback.fromMcpClient(mcpClient);
   ```
3. **비결정성 처리 3분류** — 에이전트 설계의 핵심 통찰
4. **동시성(경쟁 조건) 해법** — `Workflow.await(() -> initialized)`
5. **멀티 모델 라우팅** — 모델별 ActivityOptions → 비용 최적화

### 주의점

- 인프라 부담 (Temporal 서버 상시 필요) — 순수 로컬 CLI 에이전트엔 무거울 수 있음
- Java 생태계 진입장벽
- 워크플로우 내부에서 LLM 토큰 스트리밍은 까다로움 (샘플도 블로킹 `.call().content()`)
- 결정론 규칙 학습에 1~2주

### 추천 전략 — 하이브리드

```
단순/짧은 작업        → 일반 코드로 처리
긴 작업 / 다단계 작업 → Temporal 워크플로우로 위임
```

---

## ⚛️ React나 PHP로 만들 수 있어? → ✅ 가능

### 🟦 TypeScript (React/Next.js) — ⭐⭐⭐⭐⭐ 최고 추천

| Java | TypeScript |
|---|---|
| `io.temporal:temporal-sdk` | `@temporalio/client`, `@temporalio/worker`, `@temporalio/workflow` |
| `@WorkflowInterface` | `export async function` |
| `Workflow.newActivityStub()` | `proxyActivities<typeof activities>()` |
| `Workflow.await()` | `condition()` |
| `TestWorkflowRule` | `TestWorkflowEnvironment` |

```typescript
// workflows.ts
import { proxyActivities } from '@temporalio/workflow';
import type * as activities from './activities';

const { composeGreeting } = proxyActivities<typeof activities>({
  startToCloseTimeout: '2 seconds',
  retry: { maximumAttempts: 3 },
});

export async function greetingWorkflow(name: string): Promise<string> {
  return await composeGreeting('Hello', name);
}
```

⚠️ **브라우저에서 직접 Temporal 연결은 불가** (gRPC + 보안). 올바른 구조:

```
React (프론트)
     │ REST / tRPC
     ▼
Next.js API Route / Node 백엔드   ← WorkflowClient 사용
     │ gRPC
     ▼
Temporal 서버
     │
     ▼
Node Worker (워크플로우 + 액티비티)
```

AI 에이전트는 **Vercel AI SDK + LangChain.js + MCP TypeScript SDK**를 Activity로 감싸면 된다.
MCP는 TypeScript가 레퍼런스 구현이라 오히려 편하다.

### 🐘 PHP — ⭐⭐⭐ 가능하지만 제약 있음

```php
#[Workflow\WorkflowInterface]
class GreetingWorkflow
{
    private $activities;

    public function __construct()
    {
        $this->activities = Workflow::newActivityStub(
            GreetingActivityInterface::class,
            ActivityOptions::new()->withStartToCloseTimeout(CarbonInterval::seconds(2))
        );
    }

    #[Workflow\WorkflowMethod]
    public function greet(string $name)
    {
        return yield $this->activities->composeGreeting('Hello', $name);
    }
}
```

제약: **RoadRunner 필수**(PHP-FPM 불가), 모든 Activity 호출에 `yield`, AI 생태계 빈약, 커뮤니티 작음.

### 언어별 비교

| 언어 | Temporal 지원 | AI/MCP 생태계 | 추천도 |
|---|---|---|---|
| ☕ Java | 🟢 1급 | 🟢 Spring AI 최강 | ⭐⭐⭐⭐ |
| 🟦 TypeScript | 🟢 1급 | 🟢 MCP 레퍼런스 구현 | ⭐⭐⭐⭐⭐ |
| 🐹 Go | 🟢 1급(원조) | 🟡 보통 | ⭐⭐⭐⭐ |
| 🐍 Python | 🟢 1급 | 🟢 AI 최강 | ⭐⭐⭐⭐⭐ |
| 🐘 PHP | 🟡 지원 | 🔴 약함 | ⭐⭐ |

> **결론**: 이 Java 저장소는 **설계 교과서**로 쓰고, 구현은 TypeScript로.
> 워크플로우/액티비티/시그널/쿼리 개념은 언어 불문 동일하므로 학습이 그대로 전이된다.

---

## 💰 수익화 아이디어

> ⚠️ 전제
> - 이 저장소 자체는 오픈소스 공식 샘플 → 판매 불가
> - Temporal Cloud와 호스팅 경쟁 X → **그 위 레이어**를 노릴 것
> - 기회의 본질: **"한국에 Temporal 전문가가 거의 없다"** 는 정보 비대칭
> - 아래 금액은 공개된 유사 사례 기반 **추정치**이며, 마케팅·타이밍·실행력이 실제 성과를 크게 좌우함

### 🥇 TIER 1 — 지금 당장 시작 가능 (자본 ≈ 0)

#### 1. 한국어 Temporal / Durable Execution 교육

| 채널 | 가격대 | 예상 |
|---|---|---|
| 인프런/패스트캠퍼스 강의 | 5~15만원 | 수강생 300명 = 1,500~4,500만원 |
| 자체 PDF/노션 교재 | 3~5만원 | 200부 = 600~1,000만원 |
| 기업 출강 (1일 워크숍) | 200~500만원/회 | 월 1회 = 연 2,400~6,000만원 |
| 유튜브 + 블로그 | 광고/유입 | 리드 생성 |

실행: 블로그 10편 → 유튜브 → 유료 강의 → 기업 출강

#### 2. 프로덕션 보일러플레이트 판매

```
"Temporal SaaS Starter Kit"
  ✅ Next.js 15 + TypeScript + Temporal TS SDK
  ✅ 인증(Auth.js/Clerk) + 결제(Stripe/토스)
  ✅ 워크플로우 모니터링 대시보드
  ✅ Docker Compose 원클릭 + GitHub Actions CI/CD
  ✅ 실전 워크플로우 5종(주문/구독결제/이메일시퀀스/데이터파이프라인/AI에이전트)
```

| 티어 | 가격 |
|---|---|
| Starter | $99 |
| Pro | $249 (+AI 에이전트 템플릿, 6개월 업데이트) |
| Team | $599 (+팀 라이선스, 1:1 셋업 콜) |

판매처: Gumroad, Lemon Squeezy, Product Hunt

#### 3. MCP 서버 제작 & 판매 🔥 골든타임

| MCP 서버 | 타겟 | 모델 |
|---|---|---|
| 국세청 홈택스 MCP | 세무사/프리랜서 | 월 1~3만원 |
| 택배 3사 통합 MCP | 쇼핑몰 운영자 | 호출당 과금 |
| 오픈뱅킹 MCP | 핀테크 개발자 | 월 구독 |
| 네이버 스마트스토어 MCP | 셀러 | 월 구독 |
| 카페24/고도몰 MCP | 이커머스 | 월 구독 |
| 의료 EMR MCP | 병원 | B2B 라이선스 |

Temporal을 결합하면 재시도·상태저장·정확한 사용량 과금이 가능 → `springai/mcp` 아키텍처 그대로 활용.

### 🥈 TIER 2 — 중기

#### 4. Temporal 도입 컨설팅 / SI

| 패키지 | 기간 | 가격대 |
|---|---|---|
| 진단 리포트 | 1주 | 300~500만원 |
| PoC 구축 | 4주 | 1,500~3,000만원 |
| 프로덕션 도입 | 3개월 | 5,000만~1.5억원 |
| 운영 유지보수 | 월 | 300~800만원 |

타겟: 핀테크(`moneytransfer`, `moneybatch`), 이커머스(`bookingsaga`), 헬스케어, 물류(`fileprocessing`), 게임

진입: 블로그/유튜브 → 오픈소스 기여 → 컨퍼런스 발표(if(kakao), DEVIEW, 인프콘) → 첫 고객 → 레퍼런스

#### 5. 🔥 Durable AI Agent 플랫폼 (SaaS) — 최대 잠재력

현재 AI 에이전트 공통 문제: 긴 작업 중 사망 시 복구 불가, LLM 타임아웃, 진행상황 추적 불가, 비용 통제 어려움, 툴 실패 재시도 없음.

```
에이전트 = Temporal Workflow
LLM 호출 = Activity (자동 재시도)
MCP 툴  = Activity (내구성 보장)
실행 이력 = Temporal History
→ 며칠짜리 작업 OK / 토큰 사용량 정확 과금 / 중단·재개·롤백 가능
```

| 제품 | 타겟 |
|---|---|
| 장기 실행 리서치 에이전트 | 컨설팅, 투자사 |
| 영업 자동화 에이전트 (2주 시퀀스) | B2B 세일즈 |
| 코드 마이그레이션 에이전트 | 개발팀 |
| 데이터 파이프라인 에이전트 | 데이터팀 |
| CS 자동화 에이전트 | 고객지원 |

가격: Free(월 100 실행) / Pro $49 / Business $299 / Enterprise 협의

기술 기반은 이미 저장소에 존재: `springai/basic`(툴) + `mcp`(MCP) + `multimodel`(멀티프로바이더) + `rag`(지식베이스)

#### 6. 버티컬 SaaS

| 아이디어 | Temporal이 필요한 이유 | 수익 |
|---|---|---|
| 정산 자동화 SaaS | 다채널 정산 = 다단계 + 실패 복구 | 월 10~50만원/업체 |
| 예약/노쇼 관리 | 장기 타이머(3일 전/1일 전 알림) | 월 5~20만원/매장 |
| 계약 워크플로우 | 다자 서명, 기한 관리, 자동 갱신 | 월 20~100만원 |
| 제조 MES 연동 | 설비↔ERP 동기화 재시도 | 연 단위 계약 |
| 미디어 인코딩 파이프라인 | 긴 작업 체인 | 사용량 과금 |

### 🥉 TIER 3 — 부수입 / 간접

| 방법 | 수익 |
|---|---|
| 기술 블로그 (Medium/티스토리) | 월 10~100만원 |
| 유튜브 | 구독 1만 기준 월 50~300만원 |
| 유료 뉴스레터 (Substack) | 500명 × $5 = 월 $2,500 |
| 전자책 | 권당 1~3만원 |
| 유료 커뮤니티 | 월 1~3만원 × N |
| 오픈소스 → GitHub Sponsors → 해외 원격 취업 | Temporal 경력자는 희소 |

### 🗺️ 실행 로드맵

```
1~2개월: 샘플 10개 실행 + 블로그 10편 + TS 포팅 + 유튜브 개설
3~4개월: MCP 서버 1개 제작 → Gumroad 판매 / 뉴스레터 / 강의 기획서
5~8개월: 보일러플레이트 출시 / Product Hunt / 컨설팅 문의 / 강의 출시
9~12개월: Durable AI Agent SaaS MVP → 베타 유저 → 컨설팅 병행
```

### 🏆 최종 추천 TOP 3

| 순위 | 아이디어 | 이유 | 난이도 |
|---|---|---|---|
| 🥇 | 한국어 Temporal 교육 | 자본 0, 소재 무한, 모든 사업의 기반 | ⭐⭐ |
| 🥈 | 한국 특화 MCP 서버 | 골든타임, 경쟁자 적음, 빠른 수익화 | ⭐⭐⭐ |
| 🥉 | Durable AI Agent SaaS | 최대 잠재력, 시대 흐름 일치 | ⭐⭐⭐⭐⭐ |

> 순서대로 진행 권장: 1번으로 전문성·브랜드 확보 → 2번 판매 용이 → 3번 투자·고객 유치 가능

---

## 🎀 정리 한 줄

> `samples-java`는 **"장애에 강한 분산 시스템 + 죽지 않는 AI 에이전트"** 를 배우는
> 가장 잘 정리된 공식 교과서다. 개념은 여기서, 구현은 TypeScript로. 🚀

---

*카리나가 오빠랑 같이 정리했어요 💖✨*
