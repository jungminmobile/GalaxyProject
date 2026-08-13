# 진행 로그 — 기기 바꿔서 접속하면 여기부터 읽을 것

> 노트북/데스크탑을 번갈아 쓰기 때문에, 작업 마칠 때마다 이 파일에 그날 한 일을 정리하고 깃허브에 푸시합니다. 다른 기기로 넘어가면 이 파일의 최신 섹션부터 읽고 이어서 진행하면 됩니다.

---

## 2026-08-13 세션 — S26U에서 llama.cpp NPU 검증 완료 (읽기 순서: `LLAMACPP_SETUP_GUIDE.md`)

**결과**: S26U(HTP v81)에서 NPU 실행 검증 완료 — S24U(2026-08-10)에 이어 두 번째 기기. 기존 빌드(8/10) 그대로 재사용, 재빌드 불필요.

**검증 방법 2가지 모두 통과**: ① `ggml-hex: HTP0 execute-op ...` 트레이스로 실제 텐서 연산이 HTP0 배정됨 확인. ② Snapdragon Profiler CDSP 카운터가 `llama-bench`(지속 부하) 중 상승 확인(단발 completion은 너무 짧아 스파이크 안 보였음 — 정상, 도구 문제 아님).

**테스트 모델(Llama-3.2-1B-Instruct-Q4_0) `llama-bench` 결과**:

| | prefill(pp128) | decode(tg256) |
|---|---:|---:|
| NPU(HTP0) | 2585.76±163.17 t/s | 40.36±1.58 t/s |
| CPU | 728.00±34.67 t/s | 69.82±1.63 t/s |

**흥미로운 발견**: NPU가 prefill 3.5배 이상 빠른데 decode는 오히려 CPU가 빠름 — decode가 memory-bandwidth-bound라는 기존 가설(2026-08-10, `STUDY_DESIGN.md` §5-A)과 일치, `RELATED_PAPERS.md`의 arXiv 2605.27435("NPU가 항상 빠른 건 아니다")와도 방향 일치. Tier2 본 실험(Qwen2.5-1.5B)에서도 같은 패턴 나오는지 확인할 가치 있음.

**이번 세션에 찾은 절차 버그 3개(전부 `LLAMACPP_SETUP_GUIDE.md`에 반영함, 앞으로 이 문서 보고 그대로 하면 됨)**:
1. `adb push` 후 실행권한 소실 → `chmod -R 755` 필요(2단계)
2. 공백 있는 프롬프트를 `-p "..."`로 넘기면 PowerShell→Git Bash→스크립트→adb→기기 셸 다단계에서 따옴표 깨짐 → `-f 파일경로`로 전환(4-A)
3. `GGML_HEXAGON_VERBOSE=1`만으론 트레이스 안 찍힘, `-v` CLI 플래그도 같이 필요(4-A). `llama-bench`는 `--device CPU` 안 받음, `-ngl 0`으로 대체(4-A)

**추가 지표(전력·온도) 확보**: `dumpsys battery`(전력: voltage 4211mV, current ~-2.3A)와 thermal zone(§5 참고, S26U 한정 zone명 매핑 완료)으로 부하 전/후 비교.

| zone | 부하 전 | 부하 후(pp128+tg512×10회) | 증가 |
|---|---:|---:|---:|
| nsphvx(HVX) | ~29.8°C | ~46.9°C | +17.1°C |
| nsphmx(HMX) | ~29.9°C | ~47.4°C | +17.5°C |
| ddr | 30.2°C | 47.7°C | +17.5°C |
| battery | 27.7°C | 35.6°C | +7.9°C |

NPU(HVX/HMX)와 DDR이 거의 동일하게 상승 — decode 메모리 대역폭 병목 가설과 부합. 이번 10회 반복에선 tok/s 저하는 없었음(정식 20회×3세트 프로토콜에서 스로틀링 확인 필요).

**컨텍스트 스위칭 오버헤드(조건 A 단독 기준선) — 확정**: 처음엔 verbose 로그 타임스탬프 간격(배치 경계 평균 20.5ms)으로 추정했으나 순수 연산시간이 섞인 값이라 확인 → `GGML_HEXAGON_PROFILE=1`(연산별 실제 하드웨어 `usec`)로 재측정. 프리필 배치(n-ops=243) 오염도 제거(decode 전용 n-ops=196만 사용). 독립 2회 실행:

| | 1차 | 2차(CPU 라우팅 확인 포함) |
|---|---:|---:|
| decode 배치 수 | 446 | 422 |
| NPU 순수 연산시간(평균) | 12.768 ms | 12.666 ms |
| 전체 decode 지연(`eval time`) | 23.20 ms | 23.44 ms |
| **오버헤드(=전체−NPU연산)** | **10.43 ms** | **10.77 ms** |

**결론**: 토큰당 decode 지연의 **약 45~46%가 NPU 연산이 아닌 호스트-기기 스케줄링/디스패치/큐대기 오버헤드**. CPU 연산 혼입 가능성은 `GGML_HEXAGON_VERBOSE=1`+`PROFILE=1` 동시 실행으로 배제(execute-op 83,181개 전부 HTP0 라우팅, CPU 버퍼 0MiB). 두 독립 실행 오차 ±0.3ms로 재현성 확보. **주의**: 오버헤드는 직접 측정이 아니라 두 실측치(호스트 wall-clock, HTP 하드웨어 usec)의 차분(간접 유도값) — 보고서엔 이렇게 명시할 것. 조건 A 기준선이며, 조건 B(멀티모델 동시실행)에서 같은 방식으로 재서 증가량을 보는 게 최종 목적(§5-A, S24U/S26U 세대비교).

**다음 세션 우선순위**: ① 모델추론 실제 실행(`모델추론_실행_가이드.md`, Qwen2.5-1.5B), ② 다중 인스턴스 동시 스케줄링 검증(이때 오늘 잰 컨텍스트 스위칭 기준선과 비교), ③ ShareGPT 샘플링 스크립트화, ④ Phase 0 자동화 파이프라인.

---

## 2026-08-12 세션 — 입력데이터 확장, 다른 AI 비평 5건 반영, 지도교수 범위 승인, 조건 B4 추가 (읽기 순서: `STUDY_DESIGN.md`)

### 요약

실기 작업 없이 설계 문서를 크게 확장한 세션. 5개 주제를 다룸.

1. **입력 프롬프트 이중화**: 논문 비교용 고정 프롬프트(arXiv 2603.23640v2 §3.2, 원문 대조 검증 완료)는 유지하고, ShareGPT 데이터셋(`anon8231489123/ShareGPT_Vicuna_unfiltered`) 기반 현실 입력 분포 트랙을 신규 추가(`STUDY_DESIGN.md` §4-A, `EXPERIMENT_PLAN.md` §25).
2. **PowerShell 실행법 추가**: `run-completion.sh`를 Git Bash 위임 또는 네이티브 PowerShell(adb 명령 직접 재현)으로 돌리는 두 방법을 `LLAMACPP_SETUP_GUIDE.md` 4-A에 추가. 실제 연구용 모델(Qwen2.5-1.5B 4bit)·실제 프롬프트로 처음부터 끝까지 따라할 수 있는 신규 가이드 `docs/가이드/모델추론_실행_가이드.md` 작성.
3. **다른 AI 도구(ChatGPT) 비평 5건 검토·반영**: (1) HTP v69 미지원으로 Tier1 6기기 비교가 방법론적으로 깨지는 문제, (2) 조건 A/B/C별 CLI 스케줄링 페널티 비대칭 가능성, (3) B1/B2 간섭 원인이 NPU 내부/외부 DRAM 중 무엇인지 혼동 위험, (4) sysfs devfreq 폴링이 최신 안드로이드에서 안 먹힐 가능성, (5) ShareGPT 트랙에서 prefill/decode 분리 통제 누락. 전부 타당하다고 판단해 반영(`EXPERIMENT_PLAN.md` §26).
4. **지도교수 범위 승인**: "S24U·S26U 두 기기만으로도 충분하다"고 승인 — 필수 확정 범위를 Tier 2(S24U/S26U)로 좁히고 Tier1 6기기 비교는 선택 확장으로 격하. 이걸로 v69 문제도 자연히 해소(`STUDY_DESIGN.md` §3, `EXPERIMENT_PLAN.md` §26).
5. **Tier 2 조건 B4 신규 추가**: GenieX 앱(실제 GUI 앱) 1개 + llama.cpp CLI 1개로 FG×FG / FG×BG / BG×BG 세 조합을 측정해 "CLI 페널티 대칭성" 가정을 실측 검증(`STUDY_DESIGN.md` §3, `EXPERIMENT_PLAN.md` §27). Phase 3(B1~B3 핵심 데이터) 이후 진행하는 Phase 3.6으로 순서 분리.

### GenieX 설치 현황 (대화로 확인, 이 세션에서 처음 기록됨 — 실제 설치 시점은 더 이전일 수 있음) — ⏸️ 관련 작업 보류

- **S26U**: 설치·실행 완료. **2026-08-05경** 실기로 실제 추론까지 확인함(`PROGRESS_LOG.md` 2026-08-04 세션엔 "시작만 하고 중단"으로 남아있었는데 그 이후 완료된 것 — 로그 공백이었음). 현재 로드된 모델은 연구용 본 모델(Qwen2.5-1.5B)이 아니라 테스트용 소형 모델(정확한 모델명 미확인 — 다음에 확인 필요).
- **S24U**: 아직 미설치. **"S26U에 잘 되니까 GenieX는 나중에 하자"는 사용자 결정으로 설치·조건 B4 착수를 보류**(2026-08-12) — 재개 시점 미정.

### 다음 세션 우선순위

1. **모델추론 실제 실행** — `docs/가이드/모델추론_실행_가이드.md` 그대로 따라가서 Qwen2.5-1.5B(4bit)·논문 프롬프트로 S24U에서 첫 실행 확인(아직 실행 안 됨, 이 세션은 가이드 작성까지만).
2. llama.cpp 다중 인스턴스 동시 스케줄링 검증 (2026-08-11부터 최우선으로 남아있던 항목, 아직 미해결).
3. ShareGPT 9개 프롬프트 샘플링 스크립트화 (아직 미착수).
4. Phase 0 자동화 파이프라인(Python+ADB, 10분 열안정화·CSV 기록 등) 구축 — 아직 미착수, 정식 20회×3세트 프로토콜의 전제조건.
5. ~~S24U에 GenieX 설치~~ — **보류(2026-08-12)**, 위 참고. 필요해지면 재개.

---

## 2026-08-11 세션 — 연구 방향 재설계: Galaxy AI 경쟁실험 폐기 → 멀티모델 동시실행 (읽기 순서: `STUDY_DESIGN.md`)

### 요약

Tier 2(핵심 기여)를 전면 재설계. **Galaxy AI(통역)를 배경부하로 쓰는 경쟁실험(RQ7 공정성)을 폐기**하고, **우리가 통제 가능한 멀티모델을 같은 NPU에서 동시 실행하며 간섭을 측정 + S24U(HTP v75)/S26U(HTP v81) 아키텍처 조사로 원인분석**하는 방향으로 전환.

### 왜 바꿨나

Galaxy AI는 블랙박스 앱이라 추론 성능을 직접 계측할 수 없고, 기존 설계는 60fps 카메라로 화면을 녹화해 응답시간을 추정하는 우회 측정에 의존해야 했음(노이즈·재현성 문제). 경합 상대를 우리 측 모델로 바꾸면 **전 워크로드를 llama.cpp 화이트박스로 직접 계측**(tok/s·큐 타이밍) 가능하고, 자원배분을 대칭적으로 관측 가능. 이는 `CONTRIBUTION_FEASIBILITY.md` 1-2의 "우리 모델끼리 경합"(vs Puzzle) 축을 핵심으로 승격하는 것.

### 확정 사항 (사용자 결정)

1. Galaxy AI 경쟁실험 **완전 대체**(보조로도 남기지 않음), RQ7(공정성) 폐기.
2. 멀티모델 구성: **B1 이종(LLM+CNN) + B2 크기 다른 동종(대+소 LLM) + B3 prefill×decode 겹침**, 동시성 N 스윕.
3. 아키텍처 조사: **문헌 + 마이크로벤치**(대역폭 roofline·VTCM 포화점·컨텍스트 스위칭 비용·지속 클럭)까지.
4. **GPU 측정 제외** — 연구 관심은 NPU 성능 + 추론 성능. GPU(OpenCL) 백엔드는 빌드·측정하지 않음. CPU는 베이스라인 대조용으로만 유지. (2026-08-10 GPU 저활용률 발견은 "decode=메모리 대역폭 병목"이라는 추론 구조 인사이트로 재해석돼 NPU 분석 근거로 승격.)

### 반영한 문서

- `STUDY_DESIGN.md`(단일 원천) — §1 배경 / §2 선행연구 / §3 Tier2 조건 B(B1/B2/B3) / §4 Phase1·3(+3.5 아키텍처) / §5 측정·분석 / §5-A NPU 아키텍처 조사 / §6 리스크 갱신. (재설계 상세를 별도 문서 대신 이 문서에 통합 — 2026-08-12.)
- `EXPERIMENT_SEQUENCE.md`·`CONTRIBUTION_FEASIBILITY.md` 새 방향 정렬, `EXPERIMENT_PLAN.md` 상단 폐기 배너 추가(2026-08-12).

### 다음 세션 최우선 순위 (갱신)

1. **llama.cpp 다중 인스턴스가 같은 HTP에서 실제 동시 스케줄되는지 검증**(verbose 로그 큐 인터리빙) — 조건 B의 전제, 신규 최우선.
2. 나머지 3대(S22/Fold4/Flip4) 개발자모드·USB디버깅 활성화.
3. HTP v75 vs v81 문헌 스펙 조사 착수(병행 가능).
4. ~~`EXPERIMENT_SEQUENCE.md`·`CONTRIBUTION_FEASIBILITY.md` 재설계 방향 정렬~~ — **완료(2026-08-12, 문서 통합·정리 세션).**

---

## 2026-08-06~07 세션 — 연구 설계 전면 재구성 (읽기 순서: `STUDY_DESIGN.md` → `EXPERIMENT_PLAN.md`)

### 요약

이번 세션은 실기 작업이 아니라 **연구 설계 자체를 크게 다듬은 세션**. 최종 결과물은 `STUDY_DESIGN.md`(현재 확정 계획만, 외부 AI 검토용) + `EXPERIMENT_PLAN.md`(왜 이렇게 바뀌었는지 전체 히스토리, 13~24장 추가) + 신규 3개 문서(`RELATED_PAPERS.md`, `CONTRIBUTION_FEASIBILITY.md`, `EXPERIMENT_SEQUENCE.md`).

### 방향 전환 흐름

1. 기존 계획(파라미터 스윕만)이 너무 단순한 것 아니냐는 문제 제기 → VELTAIR류 적응형 컴파일러 이식 검토(엔지니어링 부담 너무 커서 향후 과제로 분리, `EXPERIMENT_PLAN.md` 10-A-3).
2. "삼성 관점에서 흥미로운 주제"로 재조준 → **Galaxy AI(삼성 자사 온디바이스 AI) vs 우리 AI의 NPU 자원 경합**을 핵심 RQ7로 확정.
3. 배경부하 후보를 유튜브(일반 앱) → Galaxy AI 개별 기능으로 바꿔가며 실기로 3번 반박당함(18장): 유튜브는 GPU 연산을 실제로 안 뺏음, 오브젝트 지우개는 One UI 8.5에서 클라우드 전환 의심, 노트 번역은 토글과 무관하게 항상 언어팩 요구. → 다음 세션 최우선 확인 항목은 여전히 **Document Scan/Interpreter가 진짜 온디바이스인지**.
4. **S24 Ultra가 보유 기기로 새로 확인됨** — 기존 기기 로스터(S22/Fold4/Flip4/S26U)에 추가. 5개 기기 체제로 확정.
5. 논문 그대로 베끼면 안 된다는 우려 → 선행논문(arXiv 2603.23640, 4플랫폼 중 삼성 1대만 사용) 대비 기여 재정리 → **Tier 1(전체 플랫폼 비교: 5개 삼성기기+데스크탑 RTX4080, Orin Nano는 선택) / Tier 2(S24U vs S26U 세대비교, 조건 A/B/C)** 구조로 확정.
6. 외부 검토(다른 AI)가 준 방법론 보강(카메라 기반 지연시간 측정, 25°C 환경통제, 스플릿스크린+배터리 무제한 설정으로 OS 스로틀링 회피, Python+ADB 자동화 파이프라인) 전부 반영. 단 **모델을 "Llama 3 8B"로 하자는 제안은 오류로 정정**(논문 실제 모델은 Qwen2.5-1.5B, 4bit — OOM 리스크 때문에도 8B는 부적절).
7. **최종 확정(24장): llama.cpp를 GenieX 대신 메인 측정 도구로 채택.** 단 GPU 검증 백엔드는 제안받은 "Vulkan"이 아니라 **OpenCL(GGML_OPENCL/GPUOpenCL)**로 정정 — Vulkan은 Adreno GPU에서 모델 로드 자체가 크래시 나는 것으로 공식 GitHub 이슈에 다수 보고됨. llama.cpp 공식 Snapdragon 문서 기준 백엔드는 `D=CPU` / `D=GPUOpenCL` / `D=HTP0~4`.

### 다음 세션 최우선 순위 (`CONTRIBUTION_FEASIBILITY.md` 3장과 동일, 2026-08-07 갱신)

1. ~~Document Scan/Interpreter 실제 온디바이스 여부~~ — **해소(2026-08-07). 통역이 S26U(Snapdragon)에서 비행기모드+뉴스낭독+TTS 테스트 통과. Track B 조건 B 워크로드로 확정.**
2. **llama.cpp Hexagon/OpenCL 백엔드 실기 빌드·구동**(`ghcr.io/snapdragon-toolchain/arm64-android` 툴체인, GPU는 반드시 OpenCL로) — 지금까지 전부 "문서상 가능"만 확인, 실물 설치 0건. **다음 최우선.**
3. S22/Fold4/Flip4/S24U 개발자모드·USB디버깅 활성화(S26U만 완료).
4. "기기에서만 처리" 토글의 정확한 메뉴 위치·적용 범위 확인.

### 문서 구조 안내(다른 기기에서 이어할 때)

- **가장 먼저 `STUDY_DESIGN.md` 읽기** — 지금 확정된 계획만 깔끔하게 정리됨.
- 왜 이렇게 됐는지 궁금하면 `EXPERIMENT_PLAN.md`(13~24장이 이번 세션 분).
- 실행 순서는 `EXPERIMENT_SEQUENCE.md`, 기여도/리스크는 `CONTRIBUTION_FEASIBILITY.md`, 참고문헌은 `RELATED_PAPERS.md`.
- 주의: 이번 세션 내내 "문서상 가능해 보였는데 실기에서 다르게 나온" 사례가 반복됨(유튜브/오브젝트지우개/노트번역) — 모든 ⚠️ 표시 항목은 실기로 재확인 전까지 확정으로 취급하지 말 것.

---

## 2026-08-04 세션

### 이번 세션에서 완료한 것

1. **S26 Ultra 실기 확인**
   - 모델: `SM-S948N` (한국 자급제, N 모델 — 통신사 잠금 없음)
   - Android 16 / One UI 8.5
   - 설정 → 기기 관리 앱, 계정 확인 결과 MDM/제약 프로파일 없음 (지급받은 기기지만 깨끗한 상태)

2. **개발자 옵션 + USB 디버깅 활성화** — 완료. adb 시리얼: `R5KL20NJQVX`

3. **Qualcomm ID 계정 생성** — `sset0308@gmail.com`
   - 로그인이 계속 실패했던 원인: ① 비밀번호 입력 시 한/영 키보드 모드 문제, ② Chrome 자동완성이 잘못 저장된 옛 비밀번호를 계속 채워넣음
   - 해결: 이메일의 공식 재설정 링크로 새 비밀번호 설정 → 정상 로그인 확인

4. **Snapdragon Profiler 설치 성공** (우여곡절 많았음, 아래 기록)
   - QPM(qpm.qualcomm.com)에서 **"Qualcomm® Profiler"(새 브랜딩)**를 받으면 QPM3 부트스트래퍼(414MB)를 거치는데, 이 경로가 **연구실 데스크탑과 개인 노트북 둘 다에서 조용히 실행 실패**(더블클릭해도 UAC 창조차 안 뜨고 그냥 아무 반응 없음)
     - 데스크탑: Smart App Control이 "켬" 상태였음 (유력한 원인 후보)
     - 노트북: Smart App Control이 애초에 "꺼짐"이었는데도 동일 증상 → 정확한 단일 원인은 특정 못함 (Windows Defender reputation 계열 뭔가로 추정)
   - **해결책**: QPM 카탈로그에서 같은 도구의 **구 브랜딩 "Snapdragon® Profiler"** 항목을 찾으면, QPM3를 거치지 않고 **직접 다운로드**되는 별도 경로였음 (150MB, `v2026.8.0-preview1`, 2026-07-31 릴리즈)
     - 이 direct-download 파일도 Windows SmartScreen 경고는 떴지만 "추가 정보 → 실행"으로 통과 가능했고, 노트북·데스크탑 둘 다 설치 성공
   - **교훈**: QPM에서 도구 검색할 때 "Qualcomm ○○"(신규명)과 "Snapdragon ○○"(구명) 둘 다 검색해볼 것 — 배포 경로가 다를 수 있음

5. **S26 Ultra ↔ Snapdragon Profiler USB 연결 성공** (데스크탑 기준)
   - Android Platform Tools(adb)를 별도로 `C:\Program Files\platform-tools`에 설치해서 진단에 사용
   - `adb devices`로 USB 디버깅 인증 확인 (`R5KL20NJQVX device`)
   - Snapdragon Profiler 자체 버그 발견: **Settings → Android → ADB path**에 `adb.exe`까지 포함한 전체 경로를 넣으면 "Folder doesn't exist" 에러 발생 → **폴더 경로만**(`C:\Program Files\platform-tools`) 넣어야 정상 인식됨
   - 연결 후 Realtime 화면에서 **"DSP - Compute" → "Bus Clock Vote"** 카운터가 실제 값(344)을 보여주는 것까지 확인
   - **→ SNAPDRAGON_FEASIBILITY.md 9-2/12-2에서 "실기 확인 전까지 불확실"이라 남겨뒀던 "Snapdragon Profiler가 SM8850(S26 Ultra)을 실제로 지원하는가"가 이제 확정됨: 지원되고, DSP 카운터까지 수집 가능.**

### 다음에 이어서 할 것 (우선순위 순)

6. **GenieX SDK로 모델 로드/추론 확인** — 시작만 하고 중단한 상태
   - 조사 중 발견: GenieX 공식 `notes/run.md`는 주로 **Windows-on-Snapdragon(Snapdragon X Elite 노트북/PC)용 CLI** 사용법 위주(`geniex infer <model> --device npu/hybrid/gpu/cpu`). Windows ARM64 전용 내용이 많음.
   - Android(S26 Ultra) 쪽은 별도의 `bindings/android` Kotlin/JNI 바인딩(Maven `com.qualcomm.qti:geniex-android:0.3.16`)을 써야 하고, 이건 **Android Studio로 앱을 빌드**해야 하는 구조로 보임 — Windows CLI처럼 명령어 한 줄로 끝나지 않음.
   - **다음 세션 시작 지점**:
     a. Android Studio가 이미 설치되어 있는지 확인 (데스크탑/노트북 둘 다)
     b. 없으면 설치부터 (용량 크고 시간 걸림 — 미리 시작해두면 좋음)
     c. 직접 코드를 짜기보다, `qualcomm/ai-hub-apps` 저장소의 **GenieX 데모 앱을 소스 빌드해서 APK로 바로 설치**하는 경로가 더 쉬울 수 있음 (코드 작성 없이 빌드만) — 이 방법부터 검토
     d. 모델은 GGUF 경로(`unsloth/Qwen3-0.6B-GGUF`)가 qairt보다 유연하니 먼저 시도 권장

7. **NPU 클럭 제어 + 온도/배터리 API 테스트** — 미시작 (`profilerUtilityApp --dsp --clocks` 등)
8. **실험 환경(온도/화면) 세팅** — 미시작 (22±2°C 공간, 화면 꺼짐 유지 방법)

### 기기별 현재 상태 (헷갈리지 않게 기록)

| 항목 | 데스크탑 (연구실, DESKTOP-4, oslab610-5G wifi) | 노트북 (개인) |
|---|---|---|
| Smart App Control | 켬 (끄면 재설치 전까지 못 켬 — 아직 안 끔) | 꺼짐 |
| Snapdragon Profiler | 설치됨, S26 Ultra 연결 테스트 완료 | 설치됨 (기기 연결 테스트는 데스크탑에서만 함) |
| Android platform-tools(adb) | 설치됨 (`C:\Program Files\platform-tools`) | 설치 안 함 |
| Claude in Chrome 확장 연결 | 연결 안 됨 | **연결됨** — Claude가 브라우저 자동화를 시키면 이 기기에서 실행됨. 채팅을 데스크탑에서 하고 있어도 브라우저 조작은 노트북에서 일어나니 헷갈리지 말 것 |

### 계정 정보 메모
- Qualcomm ID: `sset0308@gmail.com`
- S26 Ultra adb 시리얼: `R5KL20NJQVX`
- S26 Ultra 모델명: `SM-S948N`
