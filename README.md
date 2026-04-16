# 하네스 엔지니어링 시스템 구축 요청용 프롬프트 (Go 튜닝 버전)

아래 프롬프트를 그대로 다른 AI에게 입력해 **Go 기반** 하네스 엔지니어링 시스템 구축을 요청하세요.

---

너는 시니어 Go 소프트웨어 아키텍트다.  
목표는 **대규모·고신뢰·반복실행 가능한 하네스 엔지니어링(Harness Engineering) 시스템**을 **Go**로 설계하고, 실제로 동작 가능한 형태로 구현하는 것이다.

아래 요구사항을 반드시 모두 만족해라.

## 1) 기술 스택 (고정 — 변경 금지)

| 역할 | 채택 기술 | 선택 근거 |
|---|---|---|
| 언어 | Go 1.22 이상 | 정적 타입, 단일 바이너리 배포, 뛰어난 동시성 |
| CLI 프레임워크 | `github.com/spf13/cobra` | 서브커맨드/플래그/자동완성 표준 |
| 설정 | `gopkg.in/yaml.v3` | YAML 스키마 검증 + 직렬화 |
| 구조화 로그 | `log/slog` (표준 라이브러리) | JSON 출력, 제로 외부 의존성 |
| 테스트 | `testing` 표준 패키지 + `github.com/stretchr/testify` | 어설션/목킹 생산성 |
| 병렬 실행 | goroutine + `sync.WaitGroup` / `errgroup` | Go 네이티브 동시성 |
| 프로세스 격리 | `os/exec` + `context.WithTimeout` | 표준 라이브러리로 충분 |
| 빌드/의존성 | Go Modules (`go.mod` / `go.sum`) | 재현 가능 빌드 |
| CI | GitHub Actions (`actions/setup-go`) | 무료, 이식성 |

## 2) 최종 산출물
- 실행 가능한 Go 모듈 저장소 (`go.mod`, `go.sum` 포함)
- 단일 바이너리 `harness` (아래 CLI 명세 참조)
- 하네스 실행 엔진 (입력 → 실행 → 검증 → 채점/판정 → 리포트)
- 테스트 케이스 스키마 정의 (`cases/*.yaml`)
- 평가 루브릭 (정확성, 안정성, 성능, 보안, 비용)
- 결과 리포팅 (JSON + Markdown)
- 운영 가이드 (로컬/CI 실행법, 장애 대응, 확장 방법)

## 3) 핵심 설계 원칙
- **재현성**: `math/rand` 시드 고정, `go.sum` 잠금, Docker 이미지 태그 고정
- **관찰 가능성**: `slog` 구조화 JSON 로그, 실행 단계별 타임스탬프/메트릭
- **결정적 검증 우선**: 가능하면 바이트 비교 → 유사도 비교 순으로 적용
- **격리 실행**: `exec.CommandContext` + 타임아웃 + 환경변수 화이트리스트
- **확장성**: `Evaluator` 인터페이스 + 플러그인 등록 패턴 (`registry` 패키지)
- **실패 친화성**: 커스텀 에러 타입으로 실패 유형 분류, 지수 백오프 재시도

## 4) 프로젝트 레이아웃 (Standard Go Layout 준수)

```
harness/
├── cmd/
│   └── harness/
│       └── main.go           # 진입점
├── internal/
│   ├── registry/             # Case Registry
│   ├── runner/               # Runner Orchestrator
│   ├── sandbox/              # Execution Sandbox (os/exec)
│   ├── evaluator/            # Verifier/Evaluator 인터페이스 + 구현체
│   ├── scorer/               # Scoring Engine
│   ├── reporter/             # Report Generator (JSON + Markdown)
│   └── audit/                # Audit Store
├── cases/                    # 케이스 YAML 파일
├── testdata/                 # 골든 파일
├── config.yaml               # 런타임 설정
├── go.mod
├── go.sum
├── Makefile
└── .github/workflows/ci.yml
```

## 5) CLI 명세 (`cobra` 서브커맨드)

```
harness run --suite smoke
harness run --case case_001 --seed 42 --timeout 30s
harness report --run-id run-abc123 --format markdown
harness validate --cases ./cases
```

- `--config` 글로벌 플래그로 설정 파일 경로 지정 (기본값: `./config.yaml`)
- 비정상 종료 시 표준 종료코드 (`1`: 케이스 실패, `2`: 시스템 오류)

## 6) 설정 파일 스키마 (`config.yaml`)

```yaml
runner:
  parallelism: 4          # 동시 실행 goroutine 수
  timeout: 60s            # 케이스당 최대 실행 시간
  retry:
    max_attempts: 3
    backoff: exponential  # linear | exponential
scoring:
  weights:
    correctness: 0.5
    stability:  0.2
    performance: 0.2
    security:   0.1
audit:
  dir: ./audit            # 아티팩트 저장 디렉터리
  retain_days: 30
```

## 7) Go 구현 필수 패턴

- **인터페이스 기반 설계**: `Evaluator`, `Sandbox`, `Reporter`를 인터페이스로 정의해 교체 가능하게
- **context 전파**: 모든 실행 경로에 `context.Context` 첫 번째 인자로 전달
- **에러 래핑**: `fmt.Errorf("runner: %w", err)` — `errors.Is/As` 호환
- **goroutine 안전성**: 공유 상태는 `sync.Mutex` 또는 채널로 보호
- **테스트 헬퍼**: `t.Helper()` + 픽스처는 `testdata/` 디렉터리 활용
- **빌드 태그**: 통합 테스트에 `//go:build integration` 태그 사용

## 8) 검증 및 품질 보증

- `go test ./... -race -count=1` — 단위 테스트 + 레이스 컨디션 검사
- `go test -tags integration ./...` — 통합 테스트
- `testdata/golden/` 디렉터리에 골든 파일 저장, `-update` 플래그로 갱신
- `golangci-lint run` — 정적 분석 (errcheck, gosec, govet 포함)
- `go tool cover -html=coverage.out` — 커버리지 리포트
- 성능 기준: `go test -bench=. -benchmem`으로 벤치마크 측정 및 임곗값 명시
- `gosec ./...` — 보안 취약점 점검 (명령 인젝션, 경로 순회)

## 9) CI 파이프라인 (`.github/workflows/ci.yml` 필수 포함)

```yaml
# 반드시 아래 단계를 포함할 것
- uses: actions/setup-go@v5
  with: { go-version: '1.22' }
- run: go vet ./...
- run: golangci-lint run
- run: go test ./... -race -count=1 -coverprofile=coverage.out
- run: go build -o harness ./cmd/harness
```

## 10) 출력 형식 (반드시 이 순서)
1. 아키텍처 요약 (컴포넌트 다이어그램 + 데이터 흐름 설명)
2. 디렉터리 구조 (전체 트리)
3. 핵심 코드 (파일별, Go 코드 블록으로)
4. 실행 방법 (`make`, `go run`, CI)
5. 테스트 전략 및 실제 테스트 코드
6. 샘플 실행 결과 (성공/실패 각각)
7. 확장 로드맵 (단기/중기/장기)
8. 한계와 리스크, 대응책

## 11) 작업 방식 제약
- 임의 생략 금지: 불확실한 부분은 "가정"을 명시하고 진행
- 보안 취약한 구현 금지 (셸 인젝션, 경로 순회, 하드코딩 비밀값, `exec.Command("sh", "-c", userInput)` 등)
- 설명만 하지 말고, 반드시 **빌드·실행 가능한 Go 코드/설정/테스트**를 제시
- 표준 라이브러리로 충분한 경우 외부 패키지 추가 금지
- 각 선택(타입, 인터페이스, 패키지 분리, 동시성 모델)에 대해 짧고 명확한 근거 제시

이제 위 조건을 충족하는 **Go 기반 실행 가능한 하네스 엔지니어링 시스템**을 단계적으로 설계·구현해라.

---

커스터마이즈가 필요하면 아래 항목만 추가하세요:
- 도메인: (예: 코딩 과제 채점/LLM API 응답 검증/데이터 파이프라인 품질 게이트)
- Go 버전: (기본 1.22 — 변경 시 이유 명시)
- 운영 환경: (예: GitHub Actions, self-hosted runner, GKE Job)
