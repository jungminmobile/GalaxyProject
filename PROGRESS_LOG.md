# 진행 로그 — 기기 바꿔서 접속하면 여기부터 읽을 것

> 노트북/데스크탑을 번갈아 쓰기 때문에, 작업 마칠 때마다 이 파일에 그날 한 일을 정리하고 깃허브에 푸시합니다. 다른 기기로 넘어가면 이 파일의 최신 섹션부터 읽고 이어서 진행하면 됩니다.

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

### 다음 세션 최우선 순위 (`CONTRIBUTION_FEASIBILITY.md` 3장과 동일)

1. Document Scan/Interpreter 실제 온디바이스 여부 실기 확인(비행기모드 테스트) — Track B(RQ7 핵심) 생존 여부가 걸림.
2. llama.cpp Hexagon/OpenCL 백엔드 실기 빌드·구동(`ghcr.io/snapdragon-toolchain/arm64-android` 툴체인, GPU는 반드시 OpenCL로) — 지금까지 전부 "문서상 가능"만 확인, 실물 설치 0건.
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
