# llama.cpp 설치·빌드·설치(push) 가이드 — S24U 기준

작성일: 2026-08-10 / 목적: Phase 0/1의 최우선 항목("llama.cpp 실물 설치·구동 0건") 해소. 공식 문서(`ggml-org/llama.cpp` `docs/backend/snapdragon/README.md`)를 S24U에 맞게 정리.

> **왜 S24U로 시작하나**: S26U가 지금 다른 사람이 쓰는 중이라 대신 진행. 공식 빌드는 HTP v73/v75/v79/v81을 한 패키지에 전부 포함하고 런타임에 기기가 자동으로 자기 버전(S24U=v75, S26U=v81)을 골라 쓰므로, **같은 패키지를 그대로 두 기기 모두에 push 가능** — S24U에서 검증된 절차는 나중에 S26U에도 그대로 재사용됨.

이 가이드는 컴퓨터(데스크탑 또는 노트북, S24U가 USB로 꽂혀있는 쪽)에서 그대로 따라하면 됩니다. 각 단계 실행 후 나오는 로그를 저한테 보여주시면 바로 다음 단계 도와드릴게요.

---

## 0. 사전 준비물 확인

- [ ] **Docker Desktop 설치 여부 확인** — Windows에서 `docker --version` 실행. 없으면 [docker.com](https://www.docker.com/products/docker-desktop/)에서 설치 후 실행해둘 것(빌드 중엔 Docker Desktop이 켜져 있어야 함).
- [x] **Android platform-tools(adb) — 데스크탑엔 이미 설치됨**(`C:\Program Files\platform-tools`, `PROGRESS_LOG.md` 2026-08-04 세션 참고)
- [ ] **S24U 개발자 옵션 + USB 디버깅 활성화** — 아직 미확인(`EXPERIMENT_SEQUENCE.md` Phase 0-1). 설정 > 휴대전화 정보 > 소프트웨어 정보 > 빌드번호 7번 연타 → 개발자 옵션 진입 → USB 디버깅 ON.
- [ ] S24U를 컴퓨터에 USB로 연결 후 `adb devices`로 인식되는지 확인(권한 팝업이 폰에 뜨면 "허용").

---

## 1. 소스 받고 툴체인 컨테이너로 빌드

PowerShell(또는 Git Bash)에서:

```powershell
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
docker run -it -u 0:0 --volume ${PWD}:/workspace --platform linux/amd64 ghcr.io/snapdragon-toolchain/arm64-android:v0.3
```

컨테이너 안으로 들어가면(프롬프트가 `[d]/>`처럼 바뀜):

```bash
cd /workspace
cp docs/backend/snapdragon/CMakeUserPresets.json .
cmake --preset arm64-android-snapdragon-release -B build-snapdragon -DLLAMA_BUILD_SERVER=OFF -DLLAMA_BUILD_TESTS=OFF -DLLAMA_BUILD_APP=OFF
cmake --build build-snapdragon
cmake --install build-snapdragon --prefix pkg-snapdragon/llama.cpp
```

> **(2026-08-10 실기 확인) 위 3개 플래그가 필요한 이유**: 기본 preset만 쓰면 빌드가 순서대로 3번 실패함(모두 우리한테 불필요한 대상이라 꺼도 안전) —
> 1. `LLAMA_BUILD_UI=OFF`/`LLAMA_BUILD_WEBUI=OFF`로는 안 꺼짐(웹 UI 임베드 툴이 NDK 호스트 컴파일에서 `inttypes.h` 못 찾아 실패) — 실제 원인은 `tools/ui`가 `LLAMA_BUILD_SERVER`에 종속돼 있는 것이라 **`LLAMA_BUILD_SERVER=OFF`**가 진짜 해결책.
> 2. `LLAMA_BUILD_SERVER=OFF`만 끄면 `tests/test-chat.cpp`가 서버 헤더(`mtmd.h`)를 못 찾아 실패 → **`LLAMA_BUILD_TESTS=OFF`**로 테스트 빌드 자체를 꺼서 해결(우리 테스트가 아니라 llama.cpp 자체 유닛테스트라 불필요).
> 3. 그다음 `app`(통합 바이너리 "llama") 설치 단계에서 실패 → **`LLAMA_BUILD_APP=OFF`**로 해결.
>
> 우리가 실제로 쓰는 `llama-bench`, `completion`(공식 가이드의 `run-completion.sh`)은 이 셋 중 무엇도 필요로 하지 않아 전혀 영향 없음.

빌드가 끝나면 `pkg-snapdragon/llama.cpp/lib/` 안에 `libggml-htp-v75.so`(S24U용) 포함 여러 `.so` 파일과 `bin/llama-cli`, `bin/llama-bench` 등이 생겨야 정상입니다. 여기까지 되면 컨테이너에서 `exit`로 나옵니다.

---

## 2. S24U에 설치(push)

컨테이너 밖, 일반 PowerShell에서(adb는 호스트 것을 씀 — 컨테이너 안엔 adb 없음):

```powershell
adb devices
# S24U가 device 상태로 보이는지 확인
adb push pkg-snapdragon\llama.cpp /data/local/tmp/
adb shell chmod -R 755 /data/local/tmp/llama.cpp/bin
```

> **(2026-08-13 실기 확인) `chmod` 필수**: Windows에서 `adb push`하면 실행권한(execute bit)이 안 챙겨져서 `bin/llama-completion` 등이 `Permission denied`로 실행 안 됨. push 직후 항상 위 `chmod` 한 줄을 실행할 것.
> **빌드 재사용 가능**: 1단계 빌드 결과물은 기기 무관 단일 패키지라, 이미 빌드해둔 게 있으면(예: 저장소 폴더에 `pkg-snapdragon\llama.cpp`가 이미 있으면) 다시 빌드할 필요 없이 이 push 단계부터 바로 시작해도 됨 — S24U든 S26U든 동일.

---

## 3. 테스트용 소형 모델 하나 받아서 push

(이건 파이프라인이 도는지 빠르게 확인하는 용도입니다 — **본 실험용 모델은 STUDY_DESIGN.md에 정한 대로 Qwen2.5-1.5B(4bit)이며, 이 단계에서 바꾸는 게 아닙니다.**)

```powershell
curl.exe -L -o Llama-3.2-1B-Instruct-Q4_0.gguf https://huggingface.co/bartowski/Llama-3.2-1B-Instruct-GGUF/resolve/main/Llama-3.2-1B-Instruct-Q4_0.gguf
adb shell mkdir -p /data/local/tmp/gguf
adb push Llama-3.2-1B-Instruct-Q4_0.gguf /data/local/tmp/gguf/
```

> **(2026-08-10 실기 확인) 주의**: 목적지 폴더(`/data/local/tmp/gguf`)를 미리 `mkdir`로 만들지 않고 push하면, adb가 "gguf"라는 이름의 **폴더가 아니라 파일**을 만들어버려서 나중에 `(Not a directory)` 에러가 남. 반드시 `mkdir -p`를 먼저 실행할 것.
> Windows PowerShell에서 `curl`은 `Invoke-WebRequest` 별칭이라 `-L`/`-o` 옵션을 못 알아들음 — 꼭 `curl.exe`로 명시할 것.

---

## 4. 3개 백엔드(CPU/GPU/NPU) 강제 지정 실행 — 이번 검증의 핵심

llama.cpp 저장소 루트에서, `scripts/snapdragon/adb/run-completion.sh`를 사용(컨테이너 밖에서 실행 — adb가 필요하므로):

```bash
# NPU(Hexagon HTP0)로 강제 실행
M=Llama-3.2-1B-Instruct-Q4_0.gguf D=HTP0 ./scripts/snapdragon/adb/run-completion.sh -p "what is the most popular cookie in the world?"

# GPU(OpenCL)로 강제 실행
M=Llama-3.2-1B-Instruct-Q4_0.gguf D=GPUOpenCL ./scripts/snapdragon/adb/run-completion.sh -p "what is the most popular cookie in the world?"

# CPU로 강제 실행
M=Llama-3.2-1B-Instruct-Q4_0.gguf D=CPU ./scripts/snapdragon/adb/run-completion.sh -p "what is the most popular cookie in the world?"
```

Windows PowerShell에는 저 `.sh` 스크립트가 바로 안 돌 수 있어 Git Bash나 WSL에서 실행하는 걸 권장합니다(adb는 Windows에 설치된 걸 그대로 씀).

### 4-A. PowerShell 창에서 그대로 실행하고 싶다면 (2026-08-12 추가)

Git Bash 창을 따로 열지 않고 PowerShell에서 계속 작업하고 싶으면 두 방법이 있습니다(둘 다 이 컴퓨터에서 확인됨 — Git Bash는 `C:\Program Files\Git\bin\bash.exe`에 설치돼 있음).

**방법 1 — 이 스크립트만 Git Bash로 위임(권장)**: PowerShell 프롬프트에서 그대로 실행하되, 뒤에서만 bash가 처리하게 함.

```powershell
& "C:\Program Files\Git\bin\bash.exe" -c "M=Llama-3.2-1B-Instruct-Q4_0.gguf D=HTP0 ./scripts/snapdragon/adb/run-completion.sh -p 'what is the most popular cookie in the world?'"
```

`D=GPUOpenCL`, `D=CPU`로 바꾸면 나머지 두 백엔드도 동일하게 실행됩니다. 업스트림 스크립트가 바뀌어도 그대로 반영되는 장점이 있어 이쪽을 기본으로 씁니다.

> **⚠️ (2026-08-13 실기 확인) 방법 1은 공백 있는 프롬프트에서 깨짐**: `-p "여러 단어 프롬프트"`가 PowerShell→Git Bash→스크립트 내부 `$@`→adb shell→기기 셸까지 여러 겹을 거치면서 따옴표가 깨져, 기기에서 `error: invalid argument: is`처럼 프롬프트가 단어 단위로 쪼개져 들어감. **프롬프트에 공백이 있으면 방법 1 대신 아래 방법 2(그것도 `-p` 말고 `-f` 파일 방식)를 쓸 것.**

**방법 2 — bash 없이 PowerShell 네이티브로(스크립트 내용을 직접 재현), 프롬프트는 파일로**: `run-completion.sh` 원문을 확인해보면 결국 `adb shell` 한 줄을 실행하는 게 전부라, 아래처럼 그대로 옮길 수 있습니다. 프롬프트를 커맨드라인에 직접 넣는 대신 **파일로 push하고 `-f`로 읽게 하면 따옴표 문제가 아예 안 생깁니다**(공식 지원 — `common/arg.cpp`의 `-f/--file` 옵션, 서버 모드 제외 전 도구 공통).

```powershell
"what is the most popular cookie in the world?" | Out-File -Encoding utf8 -NoNewline test_prompt.txt
adb push test_prompt.txt /data/local/tmp/gguf/

adb shell "cd /data/local/tmp/llama.cpp; ulimit -c unlimited; LD_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib ADSP_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib ./bin/llama-completion --no-mmap -m /data/local/tmp/gguf/Llama-3.2-1B-Instruct-Q4_0.gguf --poll 1000 -t 6 --cpu-mask 0xfc --cpu-strict 1 --ctx-size 8192 --ubatch-size 1024 -fa on -ngl 99 --device HTP0 -no-cnv -f /data/local/tmp/gguf/test_prompt.txt"
```

`--device`를 `CPU`로 바꾸면 CPU 백엔드로 돌아갑니다(단, 이건 `llama-completion` 한정 — 아래 `llama-bench` 참고). `-no-cnv`는 응답 한 번 생성하고 자동 종료시켜서 interactive 모드에 멈춰있지 않게 함.

**verbose로 NPU 트레이스 보려면 환경변수 + `-v` 둘 다 필요**: `GGML_HEXAGON_VERBOSE=1`을 `ADSP_LIBRARY_PATH=...` 뒤에 추가하고, **`llama-completion`의 CLI 인자에도 `-v`를 같이 줘야** ggml-hex 로그가 찍힘(둘 중 하나만 켜면 안 나옴 — 원본 스크립트도 `V` 설정 시 이 둘을 같이 켬). 이 로그는 양이 매우 많아서(레이어×연산마다 한 줄) 파일로 저장 권장:

```powershell
adb shell "... ADSP_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib GGML_HEXAGON_VERBOSE=1 ./bin/llama-completion ... --device HTP0 -v -no-cnv -f /data/local/tmp/gguf/test_prompt.txt" *> output_npu.log
```

(`*>`로 stdout+stderr 전부 파일로 — PowerShell의 `>`는 stdout만 잡아서 이 D-레벨 로그를 놓칠 수 있음)

**성공 판정 기준(Phase 1 "NPU 실제 점유 검증" 항목)** — 아래 두 가지 중 하나만 있어도 확인된 것:

1. 로그에 `ggml-hex: Hexagon Arch version v75`(S24U) / `v81`(S26U)와 `ggml-hex: allocating new session: HTP0`가 찍힘, 또는
2. `ggml-hex: HTP0 execute-op ...` 형태로 실제 텐서 연산(MUL_MAT, FLASH_ATTN_EXT 등)이 `-> HTP0`로 배정되는 트레이스가 찍힘(둘 다 있으면 더 확실).

이 줄들이 안 보이거나 CPU로 조용히 폴백된 흔적이 있으면 그게 우려했던 리스크이니 로그 전체를 공유해주세요.

**Snapdragon Profiler로 교차검증할 때 주의**: 짧은 단발성 completion(몇 초)은 실시간 그래프에 스파이크가 안 보일 수 있음(2026-08-13, S26U에서 실제 겪음) — `llama-bench`처럼 좀 더 지속되는 부하로 확인할 것.

### 4-B. NPU 순수 연산시간 vs 호스트 오버헤드 분리(컨텍스트 스위칭 오버헤드 측정, 2026-08-13 추가)

`GGML_HEXAGON_VERBOSE=1`의 `execute-op` 타임스탬프만으론 "연산이 큐에 들어간 시각"만 보여서 순수 컨텍스트 스위칭 비용을 못 뽑아냄(그 사이 실제 NPU 연산시간까지 섞임). 대신 **`GGML_HEXAGON_PROFILE=1`**(연산별 실제 하드웨어 `usec`/`cycles` 카운터, `docs/backend/snapdragon/README.md` 공식 문서화됨)을 쓰면 NPU가 실제로 연산하는 시간을 직접 잴 수 있음.

```powershell
adb shell "cd /data/local/tmp/llama.cpp; ulimit -c unlimited; LD_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib ADSP_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib GGML_HEXAGON_VERBOSE=1 GGML_HEXAGON_PROFILE=1 ./bin/llama-completion --no-mmap -m /data/local/tmp/gguf/Llama-3.2-1B-Instruct-Q4_0.gguf --poll 1000 -t 6 --cpu-mask 0xfc --cpu-strict 1 --ctx-size 8192 --ubatch-size 1024 -fa on -ngl 99 --device HTP0 -v -no-cnv -f /data/local/tmp/gguf/test_prompt.txt" *> profile.log
```

로그의 `profile-op OPBATCH|...|n-ops N|...|usec U`가 배치(토큰) 하나의 **총 NPU 연산시간**(U를 1000으로 나누면 ms). 분석 방법:

1. `n-ops`별로 그룹핑 — decode 배치는 전부 같은 `n-ops`(예: 196)를 가짐, 다른 값(보통 더 큼)은 prefill 배치라 제외할 것.
2. decode 배치들의 `usec` 평균 = NPU 순수 연산시간/토큰.
3. 같은 로그의 `common_perf_print`의 `eval time`(호스트 wall-clock, ms/token) − 위 값 = **호스트측 오버헤드(컨텍스트 스위칭+스케줄링+디스패치)**. ⚠️ 이건 직접 측정이 아니라 두 실측치의 **차분(간접 유도값)**임을 보고서에 명시할 것.
4. `GGML_HEXAGON_VERBOSE=1`도 같이 켜서 `execute-op` 중 `-> CPU`로 라우팅되는 게 있는지 확인 — 있으면 그 시간도 오버헤드에 섞여 있다는 뜻(2026-08-13 S26U 테스트에선 0건, 전부 HTP0로 확인됨).

(2026-08-13 S26U + Llama-3.2-1B 결과: NPU 순수 연산 ~12.7ms/토큰, 전체 decode 23.2~23.4ms/토큰 → 오버헤드 ~10.4~10.8ms/토큰, decode 지연의 45~46%. 독립 2회 재현 확인.)

`llama-bench`로 tok/s까지 표로 뽑으려면:

```powershell
adb shell "cd /data/local/tmp/llama.cpp; LD_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib ADSP_LIBRARY_PATH=/data/local/tmp/llama.cpp/lib ./bin/llama-bench -m /data/local/tmp/gguf/Llama-3.2-1B-Instruct-Q4_0.gguf -p 128 -n 256 -r 5 --device HTP0 -ngl 99"
```

> **⚠️ `llama-bench`는 `--device CPU`를 못 받음**(`llama-completion`과 달리 CPU를 "디바이스"로 인식 안 함 — `error: invalid device: CPU`). CPU 베이스라인을 재려면 `--device`를 빼고 **`-ngl 0`**(오프로드 레이어 0개)으로 대체할 것.

---

## 5. 막히면

- **(2026-08-10 실제 발생) `docker --version`이 계속 인식 안 됨**: Docker Desktop을 설치만 하고 한 번도 실행 안 하면 첫 실행 시 뜨는 "Subscription Service Agreement" 동의 화면에서 멈춰 엔진이 안 켜짐 → Docker Desktop 앱을 직접 열어서 Accept. 그래도 PowerShell에서 안 되면 **PATH가 갱신 안 된 기존 터미널 창이라 그런 것 — 완전히 새 터미널 창/탭을 열면 해결됨**(Docker Desktop 설치 전에 열어둔 창은 새로 안 열림).
- `docker run` 단계에서 이미지 다운로드가 안 되면: 네트워크/방화벽 문제 — Docker Desktop이 실제로 실행 중인지, 회사/학교 네트워크가 아닌지 확인.
- `adb devices`에 기기가 안 뜨면: USB 디버깅 허용 팝업을 놓쳤을 가능성 — 케이블 재연결.
- 빌드 자체가 실패하면: 에러 로그 마지막 20~30줄만 공유해주시면 바로 봐드릴게요.

완료되면 `PROGRESS_LOG.md`에 결과(성공한 백엔드, `v75` 로그 확인 여부, 걸린 시간 등) 기록하고 GitHub에 커밋·푸시하는 것도 잊지 마세요.
