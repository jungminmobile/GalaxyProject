# 실험 계획서 — Galaxy S26 Ultra 중심 온디바이스 LLM 성능·스케줄·전력/온도 벤치마킹

작성일: 2026-08-02 / 최종 갱신: 2026-08-04(11장 — 실행가능성 비판적 재검증 반영) / 근거 문서: `SNAPDRAGON_FEASIBILITY.md`(1~10장, 이 계획의 모든 변인·도구 선택은 그 문서에서 교차검증된 항목만 사용함)
주 대상 기기: **Galaxy S26 Ultra**(Snapdragon 8 Elite Gen 5, SM8850, HTP v81) / 보조 비교군: Galaxy S22·Z Fold4·Z Flip4(전부 2022년형 v69)

> **먼저 11장부터 읽을 것**: "이 계획이 실제로 실행 가능한가"를 다시 비판적으로 검증한 결과입니다. 특히 11-2(GenieX가 S22/Fold4/Flip4를 공식적으로 지원 안 함이 재확인됨)와 11-3(S26 Ultra 실기가 없어도 되는 공식 경로 발견)은 아래 본문 설계에 직접 영향을 줍니다.

---

## 0. 연구 질문

**메인 질문**: 동일한 온디바이스 AI 모델(LLM)을 실행할 때, 스케줄링 방식·성능 프로파일·전력/온도 조건을 바꾸면 추론 성능(지연시간·처리량)이 어떻게 달라지는가?

**하위 질문**
1. compute unit(NPU / Adreno GPU / CPU) 선택이 처리량·지연시간에 미치는 영향은?
2. 같은 NPU 안에서도 "고정 배치"(HTP0 pinned) vs "자동 분배"(hybrid, CPU+NPU 텐서 단위 스케줄링)가 성능에 미치는 영향은?
3. 반복 추론(지속 부하)에 따라 자연 발생하는 열 스로틀링이 처리량을 얼마나, 언제부터 떨어뜨리는가?
4. (S26 Ultra 한정, 심화) HTP Performance Profile(burst/balanced/low_power 등)과 Graph Priority를 바꾸면 위 3번 곡선이 어떻게 달라지는가?
5. (선택, 기기 간 비교) 완전히 같은 세대의 칩(2022년형 v69)이라도 폼팩터(바형 S22 / 북형폴더블 Fold4 / 클램쉘 Flip4)에 따라 스로틀링 양상이 다른가?

---

## 1. 난이도 3단계(Tier) 구조 — 왜 이렇게 나눴는가

`SNAPDRAGON_FEASIBILITY.md` 10장에서 확인했듯, GenieX의 공식 Android Kotlin API(`ModelConfig`)는 `nCtx`, `nThreads`, `nBatch`, `compute_unit` 등은 노출하지만 **HTP Performance Profile이나 Graph Priority는 노출하지 않음**(raw QNN/QAIRT C++ 레벨 기능). 그래서 "얼마나 깊이 들어갈 것인가"에 따라 3단계로 나눔 — 시간이 부족하면 Tier 1만으로도 질문 1~3에 답이 되고, 여유가 있으면 Tier 3까지 확장.

| Tier | 난이도 | 필요 기술 | 답할 수 있는 질문 |
|---|---|---|---|
| **1** | 낮음 | Kotlin(GenieX SDK)만 | 질문 1, 2, 3 |
| **2** | 중간 | Tier1 + Android 표준 API(ADPF) + Snapdragon Profiler(데스크톱, USB 연결) | 질문 3(정밀 계측), NPU/GPU 실시간 사용률·클럭 관측 |
| **3** | 높음 | Tier1~2 + Android NDK/C++로 raw QNN(QAIRT) SDK 직접 호출 | 질문 4 |

---

## 2. 독립변인(통제 변인) 목록

### Tier 1 — GenieX Kotlin API로 즉시 통제 가능 (공식 API 문서 `geniex.aihub.qualcomm.com/en/run/android/api-reference` 확인)

| 변인 | 설정 방법 | 수준(levels) |
|---|---|---|
| compute unit | `LlmCreateInput.compute_unit` | `"npu"`(기본) / `"gpu"` / `"cpu"` |
| 디스패치 모드(llama_cpp 경로만) | `compute_unit="npu"` → 내부적으로 hybrid(기본) vs `device_id="HTP0"` 고정 중 선택(SDK가 자동 결정하나, GenieX CLI의 `--device npu`/`--device hybrid`처럼 코드에서도 분기 가능) | hybrid / pinned-HTP0 |
| 실행 런타임 | `runtime_id` | `"llama_cpp"`(GGUF, 유연) / `"qairt"`(AI Hub 사전컴파일, NPU 전용·고정 context) |
| 컨텍스트 길이(llama_cpp만) | `ModelConfig.nCtx` | 예: 512 / 2048 / 4096 (qairt 경로는 고정이라 불가 — 9장·10-4 참고) |
| 배치 크기 | `ModelConfig.nBatch` / `nUBatch` | 예: 512 / 2048 |
| CPU 스레드 수 | `ModelConfig.nThreads` / `nThreadsBatch` | 예: 2 / 4 / 8 |
| 양자화 정밀도 | `ModelPullInput.precision` | `"Q4_0"` / `"Q4_K_M"` / `"Q8_0"` |
| 모델 크기 | 모델 선택 | 예: Qwen3-0.6B vs Qwen3-4B |

### Tier 2 — Android 표준 API + Snapdragon Profiler(데스크톱, USB)

| 변인 | 설정 방법 | 근거 |
|---|---|---|
| 안드로이드 전원 모드 | 설정 > 배터리 > 전원모드(고성능/최적화/절전) | Samsung 공식 지원 문서 |
| 화면 on/off | 코드에서 화면 강제 off 유지 | 8-4 논문 방법론과 동일 |
| NPU/CDSP 클럭 강제 set/limit | `profilerUtilityApp --dsp --clocks set/limit --cengClock <값>` | 9-3 (S26 Ultra는 지원 칩셋 목록에 포함) |
| 반복 횟수(지속 부하) | 실험 프로토콜 자체(아래 5장) | 준독립변인이자 열 상태의 대리 변수 |

### Tier 3 — raw QNN(QAIRT) C++/NDK 필요

| 변인 | API | 근거 |
|---|---|---|
| HTP Graph Priority | `QnnGraph`/`QnnContext` 생성 시 priority 설정(협조적 선점) | QAIRT 공식 문서 |
| HTP Performance Profile | QNN HTP Performance Infrastructure(DCVS 투표) | burst / balanced / low_power / high_performance / sustained_high_performance / default |
| VTCM 크기 | `QNN_HTP_GRAPH_CONFIG_OPTION_VTCM_SIZE_IN_MB` | 그래프별 예약 메모리 |
| HVX 스레드 수 | `QNN_HTP_GRAPH_CONFIG_OPTION_NUM_HVX_THREADS` | 기본 4 |
| Concurrent Resource Sharing | `QNN_HTP_CONTEXT_CONFIG_OPTION_REGISTER_CONCURRENT_RESOURCE_SHARING` | **S26 Ultra(V81)+Android 전용** |

---

## 3. 종속변인(측정 항목)

| 항목 | 측정 방법 | 획득 난이도 |
|---|---|---|
| 처리량(tok/s), TTFT, prefill/decode 시간 | GenieX `LlmStreamResult.Completed`가 반환하는 `ProfilingData`(정확한 필드명은 실기 확인 필요, 통상 prefill/decode/tok-s 계열로 추정) 또는 직접 타임스탬프 | Tier 1(가장 쉬움) |
| op(레이어)별 시간 분해, HVX 병렬화 효율 | QAIRT SDK 내장 Optrace/Hextimate(QAIRT 2.39+) | Tier 2 |
| 배터리 온도/전류/전압 | Android `BatteryManager` | Tier 1 |
| 열 상태(정성)/열 여유(정량) | Android ADPF `getThermalHeadroom()`, `addThermalStatusListener` | Tier 1 |
| NPU/GPU 실시간 사용률(%), 클럭(MHz), DDR 대역폭 | Snapdragon Profiler(150+ 카운터, 22개 카테고리) | Tier 2 |

---

## 4. 모델·데이터 선정

- **주 모델**: Qwen 계열 소형 LLM, 4bit 양자화(Q4_0) — GGUF는 `unsloth/Qwen3-0.6B-GGUF` 등, NPU 전용 비교용은 `ai-hub-models/Qwen3-4B-Instruct-2507`(pull 시 `chipset="SM8850"` 필수 지정).
- **프롬프트**: 고정된 하나의 긴 서술형 프롬프트(예: 8-4에서 소개한 arXiv 논문이 쓴 258토큰짜리 "여러 관점에서 설명하는 에세이" 프롬프트를 그대로 재사용 — 그러면 그 논문의 실측치와 직접 비교도 가능).
- **디코딩 설정**: greedy(temperature=0, top-k=1)로 고정해 출력 길이를 결정적으로 만듦(반복 간 노이즈 제거).
- **비교용 대안 모델**: 원래 프로젝트가 CV(YOLOv8)였던 점을 고려해, 여유가 있으면 Qualcomm AI Hub의 YOLOv8-Detection도 QNN(HTP)으로 실행해 "LLM은 GPU/NPU 어디가 유리한지"와 "CV 모델은 NPU가 확실히 유리한지"를 대조하는 것도 가능(9장에서 이미 YOLOv8은 HTP v69/v81 전부 공식 지원 확인됨).

---

## 5. 실험 프로토콜 (arXiv 2603.23640 방법론을 그대로 채택 — 이미 S24 Ultra에서 검증된 절차)

1. 기기를 22±2°C 상온에서 **10분간 열 안정화**.
2. 모델을 메모리에 로드하고 **워밍업 1회 실행 후 결과 폐기**.
3. 60초간 온도 변화가 2°C 미만(ΔT<2°C)인지 확인 후 본 실험 시작.
4. 화면은 꺼진 상태 유지(배터리 측정 정확도를 위해, 8-4 논문과 동일).
5. 하나의 조건(독립변인 조합)마다 **20회 반복, 반복 간 1초 대기**.
6. 매 반복마다 CSV 한 줄 기록: `timestamp, condition_id, decode_tokens, decode_time_ms, prefill_time_ms, throughput_tok_s, ttft_ms, battery_temp_c, battery_current_ma, battery_voltage_mv, thermal_status, (Tier2)npu_util_pct, (Tier2)gpu_freq_mhz`.
7. 조건이 바뀔 때마다 **기기를 30분 이상 식히거나 재부팅**해서 이전 조건의 잔열이 다음 조건에 영향을 주지 않게 함.
8. 각 조건은 **3세트 반복**(재현성 확인 — Hailo 프로젝트에서 쓰던 관례와 동일).

---

## 6. 실험 매트릭스(예시)

### 6-1. 최소 실행 버전(Tier 1만, 우선 이것부터)

| 조건 ID | runtime | compute_unit | 디스패치 | 비고 |
|---|---|---|---|---|
| A | llama_cpp | npu | hybrid(기본) | 가장 빠를 것으로 예상되는 기본값 |
| B | llama_cpp | npu | pinned HTP0 | 결정적이지만 느릴 것으로 예상 |
| C | llama_cpp | gpu | — | Adreno GPU 경로 |
| D | llama_cpp | cpu | — | 순수 CPU 베이스라인 |
| E | qairt | npu(전용) | — | AI Hub 사전컴파일 NPU 경로(고정 context) |

→ 5조건 × 20회 × 3세트 = **300회 실행**. 조건당 예상 소요시간을 고려해 하루 안에 끝내기 어려우면 조건 수를 A/B/D 3개로 줄여도 질문 1·2는 답변 가능.

### 6-2. 확장 버전(Tier 3까지, S26 Ultra 심화)

6-1의 조건 A(가장 성능 좋은 조합) 위에 **HTP Performance Profile 4수준**(burst/balanced/low_power/sustained_high_performance) × **Graph Priority 2~3수준**(낮음/기본/높음)을 추가로 곱해 조건을 확장. 이건 raw QNN NDK 코드가 필요하므로 시간 여유에 따라 선택.

### 6-3. 기기 간 비교(선택, S22/Fold4/Flip4)

6-1의 조건 A/D(가장 단순한 hybrid, cpu)만 골라 세 기기에서 동일하게 실행 — Snapdragon Profiler/NPU 클럭 계측은 빼고 Tier 1 측정치(tok/s, 온도, 배터리)만으로 "동일 세대 칩, 폼팩터별 스로틀링 차이"를 비교.

---

## 7. 분석 계획

- 조건별 처리량 평균·표준편차·변동계수(CV) 산출(8-4 논문의 보고 방식과 동일 포맷으로 표 작성 — 직접 비교 가능해짐).
- 반복 회차(1~20회)별 처리량 추이 그래프 — 몇 회차부터 "플래토"(안정화)에 들어가는지 시각적으로 표시(아이폰형 급락 vs 안드로이드형 완만한 감소, 8-4 논문의 두 패턴과 비교).
- 조건 간 차이가 통계적으로 유의미한지 반복측정 ANOVA 또는 조건쌍별 t-test.
- 온도(x축) vs 처리량(y축) 산점도로 스로틀링 상관관계 확인.

---

## 8. 준비물 체크리스트

- [ ] 무료 Qualcomm 계정(QPM 가입) — QAIRT SDK, Snapdragon Profiler 설치용
- [ ] Android Studio(+NDK, Tier 3용)
- [ ] GenieX Android SDK 의존성 추가 — `implementation("com.qualcomm.qti:geniex-android:0.3.16")`(Maven Central 실존 확인됨, 11-1 참고. `google()`+`mavenCentral()` 표준 저장소만 있으면 됨)
- [x] **S26 Ultra 실기 확보 — 확보 완료(실험 목적으로 지급받음, 11-3 갱신 참고). QDC 원격 경로는 불필요해짐**
- [ ] Snapdragon Profiler 데스크톱 설치(Windows/Mac/Linux, USB로 기기 연결 — 실기 보유로 물리적 USB 연결 가능, QDC 호환성 이슈 소멸)
- [ ] 개발자 옵션 → USB 디버깅 활성화(루팅 불필요, 9-5 참고)
- [ ] 온도 재현 가능한 실험 공간(에어컨 등으로 22±2°C 유지)
- [ ] 실험 로그용 CSV 스키마 사전 확정(6장 참고)

---

## 9. 위험요소·한계 (정직하게 명시)

- GenieX `ProfilingData`의 정확한 필드명은 공개 문서에서 100% 확인 못함 — 실기에서 로그 찍어 확인 필요.
- Tier 3(raw QNN NDK)은 개발 난이도가 높고 시간이 오래 걸릴 수 있음 — 학기 일정에 맞춰 Tier 1~2만으로 우선 결과를 내고 여유 되면 확장하는 걸 권장.
- `qairt` 경로는 context 길이가 컴파일 시점 고정이라, 이 변수를 스윕하려면 조건별로 미리 여러 바이너리가 필요(런타임 변경 불가).
- 배터리 온도(BatteryManager)는 칩 자체 온도의 근사치이지 정확한 다이(die) 온도가 아님.
- Snapdragon Profiler의 S26 Ultra 지원 여부는 "SM8850이 지원 칩셋 목록에 있다"는 정황 증거 기반 — 실기 연결 시 최종 확인 필요(9-2/9-7 참고).
- **(11장 반영) GenieX는 S22/Fold4/Flip4를 공식적으로 지원하지 않음이 재확인됨** — 6-3(기기 간 비교)을 GenieX 기반으로 그대로 실행할 수 없음. 대안 경로 재설계 필요(11-2, 11-7 참고).
- **(11장 반영) Tier 3를 raw QNN NDK/adb로 구현할 경우, 비루팅 리테일 기기에서는 CLI 실행이 GUI 대비 평균 9.7%(최대 58%) 느리게 측정되는 공식 문서화된 편차가 있음** — 이 편차를 통제하거나 한계로 명시해야 함(11-6 참고).

---

## 10. 참고

세부 근거·출처는 전부 `SNAPDRAGON_FEASIBILITY.md`(특히 8~10장)에 정리되어 있음. 이 문서는 그 조사 결과를 실행 가능한 절차로 압축한 것이므로, 특정 항목의 "왜?"가 궁금하면 그 문서를 참고할 것.

---

## 11. 실행 가능성 재검증 (2차, 비판적 재확인 — 2026-08-04)

**요청 배경**: "위 실험을 실제로 실행할 수 있는지 다시 한번 확인해줘." 앞선 조사에서 확신하지 못했던 5가지 전제를 공식 문서(Qualcomm GitHub 저장소, Maven Central, Qualcomm AI Hub, GenieX 공식 문서)로 재검증했다. 결과: 확정 리스크 1개(신규), 강력한 해결책 1개(신규), 3개는 기존 판단 유지/보강.

### 11-1. GenieX Android Maven 아티팩트 실존 여부 → **확인됨(실존). 이전 우려는 정정.**

- `com.qualcomm.qti:geniex-android`는 Maven Central에 실제로 등록되어 있음 — Sonatype Central에서 groupId/artifactId/POM 실물 확인.
- 공식 설치 문서 예시는 `0.3.1`이지만 Maven Central 실측 최신판은 **`0.3.16`**(패치 약 2주 전) — 활발히 갱신되는 중.
- 저장소 설정은 표준 `google()` + `mavenCentral()`만 있으면 되고, 별도 Qualcomm 전용 repository URL은 불필요.
- (정정) GenieX 저장소 내부 기여자용 문서(`bindings/android/README.md`)에는 "Maven Central 배포 절차 TODO"라고 적혀 있어 지난 조사에서 "혹시 미배포 아닐까"라는 의심을 남겨뒀었는데, 이건 오래된 내부 개발 메모였고 공식 사용자 문서·Maven Central 실물 모두 이미 배포 완료를 보여줌.

### 11-2. GenieX Android 최소 기기 요구사항 → **신규 확정 장벽(중요, 기존 판단보다 강하게 재확인)**

공식 설치 문서(Prerequisites)에 다음이 "권장"이 아니라 **명문 요구사항**으로 박혀 있음:

> "A phone running **Snapdragon 8 Elite (SM8750)** or **Snapdragon 8 Elite Gen 5 (SM8850)**"

- Galaxy S22(SM8450)·Z Fold4/Z Flip4(SM8475)는 이 요구사항을 만족하지 못함 — GenieX Android SDK 공식 지원 대상이 아님이 재확인됨.
- Galaxy S26 Ultra(SM8850)만 공식 지원 범위.
- llama_cpp의 CPU 전용 경로는 이론상 범용 ARM64 코드라 S22 등에서 완전히 안 될 거라 단정할 순 없지만, Qualcomm이 테스트·보증하는 범위 밖의 "비공식 시도" 영역으로 봐야 함.

### 11-3. S26 Ultra 실기 미보유 문제 → **해결됨(실기 지급 확인). QDC 관련 불확실성은 전부 소멸**

**갱신(2026-08-04, 사용자 확인)**: 사용자가 이 실험을 위해 Galaxy S26 Ultra 실기를 지급받았음을 확인함. 즉 아래에 정리했던 QDC(원격 클라우드) 경로는 "실기가 없을 경우의 대안"으로서만 의미가 있고, 실기가 있는 지금은 **불필요**. 이에 따라 이전까지 남아있던 두 개의 마지막 불확실성 — ① QDC 개인 계정 승인 여부, ② Tier 2/3(Snapdragon Profiler 등)이 QDC 원격 세션에서 되는지 — 은 **둘 다 무관해짐**:

- Snapdragon Profiler는 실기를 데스크톱에 직접 USB로 연결하면 되므로 원격 세션 호환성 문제 자체가 사라짐.
- raw QNN NDK(Tier 3)도 실기에 직접 USB 디버깅으로 접근 가능.
- 남는 유일한 전제는 9-2에서 정리한 "Snapdragon Profiler가 SM8850을 실제로 지원하는지"인데, 이건 실기가 있으니 USB로 연결해 직접 켜보면 그 자리에서 바로 확인됨(더 이상 정황 증거에 의존할 필요 없음) — **가장 먼저 해볼 실기 검증 항목으로 격상**.

아래는 실기가 없었다면 유효했을 원래 조사 내용(참고용으로 보존):

이번 재검증에서 가장 중요한 발견. GenieX 공식 FAQ에 **기기 없이 원격으로 실제 8 Elite/8 Elite Gen 5 기기에 접속해 GenieX 데모 앱을 테스트하는 공식 절차**가 명시되어 있음(`qdc.qualcomm.com`):

1. `qualcomm/ai-hub-apps`의 GenieX 데모 앱을 Android Studio에서 소스 빌드해 `.apk` 생성.
2. QDC에 Qualcomm 계정으로 로그인 → "Snapdragon 8 Elite" 또는 "8 Elite Gen 5" 기기의 Interactive Session 선택.
3. 세션 설정에서 Wi-Fi 켬 + 화면 꺼짐 방지 켬 → 세션 시작 **전에** APK 업로드.
4. 세션 시작 후 미러링된 화면에서 APK 설치·실행(이 플로우는 SSH 불필요).

즉 **S26 Ultra를 직접 사지 않아도 Tier 1(GenieX 앱 실행, tok/s·TTFT·배터리·열상태 측정)은 원격으로 실행 가능**하다는 뜻 — 지난 조사에서 확인하지 못했던, 실행 가능성을 크게 높이는 요소.

다만 아래는 **미확인/제약으로 정직하게 남김**:
- 가입 절차 안내에 "회사 이메일(company email address) 입력 → 확인 후 검토"라고 되어 있어, 개인/학생 이메일로도 승인되는지는 공식 문서에 명시돼 있지 않음(불확실 — 조기에 직접 가입 시도해봐야 함).
- 무료 사용량은 커뮤니티(비공식) 정보로 기기당 약 5000분이라는 언급이 있으나 공식 페이지로 재확인은 못함 — 참고용으로만 취급.
- 이 플로우는 화면 미러링 + 파일 업로드 위주로 확인됨. Tier 2/3에서 계획한 **Snapdragon Profiler(데스크톱-USB 물리 연결 전제)**가 QDC 원격 세션에서도 되는지는 문서상 확인 못함 — Tier 1은 QDC로 확실히 가능하나, Tier 2/3은 **추가 확인 필요**.
- 원격 기기는 데이터센터에 있으므로 "손에 쥐고 주머니에 넣었을 때" 같은 물리적 환경 변인은 통제 불가(다만 이 실험 자체가 애초에 상온 22±2°C로 통제하는 실내 실험이라, 데이터센터 환경도 오히려 온도 통제 조건 자체는 안정적으로 만족시킬 가능성 있음).

### 11-4. `ai-hub-models/Qwen3-4B-Instruct-2507` 번들 실존 여부 → **확인됨(실존), 기기명 카탈로그만 S26 Ultra 미등재**

- Qualcomm AI Hub 모델 페이지 직접 확인 + GenieX 공식 README의 실행 예시(`geniex pull ai-hub-models/Qwen3-4B-Instruct-2507`)에 그대로 등장 — 실존, pull 가능.
- **지원 칩셋**: Snapdragon 8 Elite Mobile, **Snapdragon 8 Elite Gen 5 Mobile**(=SM8850=S26 Ultra), Snapdragon X/X2 Elite, QCS9075 — S26 Ultra 세대 칩은 확실히 포함.
- **지원 기기 목록**: Galaxy S25/S25+/S25 Ultra는 기기명으로 명시돼 있으나 Galaxy S26 Ultra는 아직 등재 안 됨(S26 시리즈가 2026년 3월 출시라 카탈로그 갱신이 못 따라간 것으로 추정. 칩셋 자체는 지원 목록에 있어 기능적으로 문제 없을 가능성이 높지만 100% 확정은 아님).

### 11-5. 리테일(비루팅) 기기에서 서드파티 앱의 Hexagon DSP 접근 제약 → **기존 판단 유지**

QNN/TFLite Hexagon 델리게이트는 다수 상용 앱이 이미 쓰는 표준 경로이며, 서드파티 앱이 루팅 없이 HTP에 접근하는 데 별도 OEM 서명이 필요하다는 근거는 이번 재검색에서도 나오지 않음. 기존 결론(루팅 불필요) 유지.

### 11-6. (신규 발견) CLI vs GUI 스케줄링 우선순위 차이 — 계획에 직접 영향

Qualcomm AI Hub 공식 FAQ에 실측치가 명시돼 있음: `adb shell`로 실행하는 CLI 도구(`qnn-net-run` 등)는 GUI 앱보다 커널 스케줄링 우선순위가 낮아 **평균 9.7%(중앙값 5.3%, 최대 58%) 느리게 측정됨**(65개 기기 실측). 해결하려면 `nice -n -10`이 필요한데 이는 root 권한 필요.

→ 이 계획은 처음부터 Tier 1을 "GenieX Android 앱(GUI)"으로 설계했기 때문에 이 문제를 자동으로 피해감 — 기존 설계가 옳았다는 재확인. 다만 Tier 3에서 raw QNN을 NDK/adb로 직접 건드릴 경우, 비루팅 리테일 기기에서는 이 9.7~58% 편차가 결과에 섞여 들어갈 수 있어 반드시 GUI 앱 내부 측정으로 우회하거나 한계로 명시할 것.

### 11-7. 종합 결론 (2026-08-04 최종 갱신 — S26 Ultra 실기 확보 반영)

**실험은 실행 가능하며, 실기 확보로 사실상 남아있던 최대 리스크(하드웨어 접근)까지 해소됐다.** 계획에 반영할 사항:

1. S22/Fold4/Flip4에서 GenieX 공식 지원은 확정적으로 불가 — 6-3(기기 간 비교)은 GenieX가 아닌 별도 경로(예: TFLite+QNN delegate 같은 더 범용적인 스택)로 재설계하거나 "참고용 비공식 시도"로 격하할 것. (S26 Ultra 실기 확보와는 무관하게 여전히 유효한 제약)
2. ~~S26 Ultra 실기가 없다면 QDC가 유력한 대안~~ → **실기를 지급받아 해소됨.** QDC 관련 두 불확실성(개인 계정 승인, Tier 2/3 호환성)은 더 이상 이 실험의 리스크가 아님.
3. Maven 아티팩트·모델 번들 실존은 확인 완료 — 더 이상 리스크 아님.
4. Tier 3를 raw NDK/adb로 할 경우 CLI 스케줄링 편차(최대 58%)를 한계로 명시.
5. **(신규) 실기를 받았으면 가장 먼저 할 일**: Snapdragon Profiler를 데스크톱에 설치하고 USB로 S26 Ultra를 연결해 실제로 기기가 인식되고 카운터가 잡히는지 확인할 것. 지금까지의 "SM8850 지원" 판단은 전부 공식 문서 기반 정황 증거였고, 실기로 직접 검증되는 순간 이 계획의 마지막 불확실성이 사라짐.

---

## 12. 최종 종합 확인 — "실험을 막거나 불가능하게 하는 요소가 남아있는가?" (2026-08-04)

**결론부터: 지금까지 조사한 범위 안에서 실험 자체를 원천적으로 막는(=불가능하게 만드는) 요소는 없음.** 다만 "막는 것"과 "번거롭게/불확실하게 만드는 것"은 다른 문제라서, 남아있는 항목을 전부 등급을 나눠 솔직하게 정리한다.

### 12-1. 진짜로 실험을 막는 요소 — 없음

QDC 계정 승인 불확실성은 실기 확보로 해소됐고, 그 외에 "이래서 아예 불가능하다"고 판단할 근거는 이번까지의 조사에서 나오지 않았음.

### 12-2. 아직 실기로 직접 확인 안 된 것 (막을 가능성은 낮지만 0%는 아님)

- Snapdragon Profiler ↔ SM8850 실제 USB 연결/카운터 수집 — 공식 지원 칩셋 목록엔 있지만 실측 테스트는 아직 안 함(11-7에서 "가장 먼저 해볼 것"으로 이미 표시해둠).
- GenieX `ProfilingData`가 반환하는 정확한 필드 구성 — 공개 문서로 100% 확정 못함, 실행 후 로그로 직접 확인 필요.

### 12-3. 확실히 되지만 추가 작업량이 필요한 것 (막는 게 아니라 시간이 드는 것)

- Tier 3(Graph Priority, Performance Profile, VTCM, HVX threads)는 GenieX 공식 Kotlin API가 노출하지 않음 → raw QNN/QAIRT NDK(C++)를 직접 작성해야 하는, 앱 개발과 별개의 추가 엔지니어링 작업. "안 되는 것"이 아니라 "일정이 더 드는 것" — 학기 일정상 Tier 1~2 먼저 끝내고 여유 되면 확장 권장.
- `qairt` 경로는 context 길이가 컴파일 시점 고정 → 이 변수를 스윕하려면 조건별로 여러 바이너리를 미리 준비해야 함.

### 12-4. 소프트웨어 성숙도 리스크

GenieX는 공식적으로 "Developer Preview"(정식 출시 이전) 배지가 붙어 있고, 최근 2주 사이에도 버전이 올라감(문서 예시 0.3.1 → Maven Central 실측 0.3.16). 프리뷰 소프트웨어라 실험 도중 API나 동작이 바뀔 수 있음 — **실험 시작 시점에 사용 버전을 고정(pin)해서 끝까지 그 버전으로 진행할 것을 권장**.

### 12-5. 물리적·실무적 리스크 (이번에 새로 짚는 부분)

- **지급받은 기기라는 점**: 학교/연구실에서 이 실험 목적으로 대여·지급된 기기라면 MDM(기기 관리) 프로파일이 걸려 있어 "출처를 알 수 없는 앱" 설치나 개발자 옵션이 막혀 있을 수 있음. 이건 문서 조사로는 알 수 없는, 실기에만 있는 정보라 **설정 > 생체인식 및 보안 / 설정 > 기기 관리 앱**에 잠금 프로파일이 있는지 직접 확인이 필요함 — 문서 조사로 대신 확인해줄 수 없는 유일한 항목.
- 반복 벤치마킹(조건당 20회×3세트, 조건 사이 30분 냉각)은 실측상 조건 수에 따라 수 시간~하루 단위로 소요될 수 있음 — "불가능"이 아니라 일정을 넉넉히 잡아야 하는 문제.
- 장시간 고부하 반복 실험은 배터리 마모·발열 스트레스를 누적시킴 — 반납해야 하는 기기라면 이 점을 고려해 조건 수/반복 수를 조절할 것.

### 12-6. 결론

막는 요소는 없음. 남은 건 전부 "실기로 그 자리에서 몇 분 안에 확인 가능한 것"(Profiler 연결, 필드명) 아니면 "시간·개발량이 드는 것"(Tier 3 NDK)이지 "안 되는 것"은 아니다. 문서 조사로 대신 확인할 수 없고 사용자가 직접 봐야 하는 항목은 딱 하나 — 지급받은 기기에 MDM 제약이 걸려 있는지 여부.
