# NPU 스케줄러 실험, 갤럭시(스냅드래곤) 이식 가능성 조사 — 결론 및 근거

작성일: 2026-08-02(갱신) / 대상 기기 확정: **Galaxy S22, Galaxy Z Fold4, Galaxy Z Flip4** (실제 보유 기기로 확인됨 — 애초 검토했던 S24 Ultra/S26 Ultra는 보유 기기가 아니므로 이하 결론은 이 3개 기기 기준으로 재정리)
전제: `PROJECT_SUMMARY.md`(Raspberry Pi + Hailo-8/8L, HailoRT Model Scheduler로 YOLOv8 Det/Seg/Pose 3모델 동시추론, priority/threshold/timeout/batch_size 스윕) 실험을 동일하게 재현할 수 있는지 확인

**중요**: 3개 기기 전부 **Snapdragon 8 Gen 1 / 8+ Gen 1 세대(2022년, Hexagon HTP v69)**로 확인됨 — 즉 "여러 세대를 비교"하는 실험은 안 되고, **같은 칩 세대를 세 가지 폼팩터(바형 S22 / 북형 폴더블 Fold4 / 클램쉘 Flip4)에서 비교**하는 실험이 됨. 아래 9장에서 상세.

---

## 0. 결론 먼저 (요약)

**"똑같은 실험"은 안 되고, "같은 질문을 다른 방식으로 재현하는 실험"은 된다.**

- Hailo의 **Model Scheduler**(ROUND_ROBIN + priority/threshold/timeout/batch_size)에 1:1 대응하는 기능이 Qualcomm 쪽엔 **없음**. 스냅드래곤(QNN/Hexagon)은 **priority 기반 협조적 선점(cooperative preemption)** 방식이라 threshold·timeout 같은 "기아 방지용 타이머" 개념 자체가 공식 문서에 없음. → 4개 파라미터를 그대로 스윕하는 실험 설계는 불가능, 파라미터를 새로 정의해야 함.
- 3모델(Det/Seg/Pose) 동시추론 자체, NPU 가속, 지연시간 측정은 **가능**. 다만 "NPU 사용률(%)"을 Hailo의 `/proc` 급으로 손쉽게 뽑는 표준 인터페이스는 없고, Snapdragon Profiler(디버거블 앱 + USB) 또는 QNN 자체 프로파일러(그래프 단위 latency)로 대체해야 함.
- YOLOv8 후처리(NMS)는 Hailo와 마찬가지로 **Det/Seg/Pose 전부 NPU가 아니라 host(CPU 또는 앱 코드)에서 처리**됨 — 오히려 Hailo보다 더 확실하게 전부 host 처리로 확인됨.
- 4개 후보 기기 모두 Snapdragon 탑재(한국 모델 기준)이고 QNN 공식 지원 대상이라 **환경 자체는 준비 가능**. 다만 기기 세대(Hexagon 아키텍처 버전)에 따라 쓸 수 있는 기능 폭이 다름 — 최신 기기(Galaxy S26 Ultra급, HTP v81)에서만 되는 기능이 있음.
- 부트로더/루트 이슈: 한국 통신사 모델은 미국 모델과 달리 공식 OEM 언락이 가능하지만, 루팅하면 Knox가 영구적으로 트립되어 삼성페이 등 일부 기능이 못 씀. **다만 이 실험 자체는 루팅 없이(일반 개발자 모드 + USB 디버깅만으로) 대부분 가능**.

아래는 각 항목의 근거.

---

## 1. 대상 기기별 AP/NPU 세대 확인

| 기기 | AP(한국/전세계 기준) | SoC 모델넘버 | Hexagon NPU 아키텍처 | 근거 |
|---|---|---|---|---|
| Galaxy S22 (한국 모델) | Snapdragon 8 Gen 1 (원래 국내는 엑시노스 예정이었으나 엑시노스 2200 수율 문제로 스냅드래곤으로 변경) | SM8450 | **HTP v69** | ZDNet Korea, 디일렉 보도 / Ultralytics QNN 문서의 HTP 타깃 표 |
| Galaxy Z Fold4 | Snapdragon 8+ Gen 1 (전 세계 공통, 이 세대는 지역 분기 없이 전부 스냅드래곤) | SM8475 | **HTP v69**(S22와 동일 세대) | AndroidHeadlines, SamMobile |
| Galaxy Z Flip4 | Snapdragon 8+ Gen 1 (전 세계 공통) | SM8475 | **HTP v69** | AndroidHeadlines, SamMobile |

**정리**: 세 기기 모두 **2022년형 Snapdragon 8 Gen 1 / 8+ Gen 1 세대(HTP v69)**로 동일 세대임. QNN(Qualcomm AI Engine Direct / QAIRT) HTP 백엔드의 공식 지원 대상 아키텍처 목록(v68/v69/v73/v75/v79/v81)에 v69가 포함되어 있어 **"모델을 NPU로 돌리는 환경 자체는 갖춰져 있다"**는 1차 결론은 유효함. 다만 이 세대는 아래 9장에서 확인하듯 Qualcomm의 최신 프로파일링/LLM 전용 도구 일부의 공식 지원 대상에서는 빠짐.

---

## 2. 스케줄러 기능 비교 — 이게 이번 조사의 핵심

### 2-1. Hailo Model Scheduler가 하는 일 (복습)

- 물리적으로 하나뿐인 Hailo 칩 위에서, 독립적으로 컴파일된 여러 네트워크그룹(HEF)을 ROUND_ROBIN으로 시분할.
- 모델별로 `priority`(스케줄 우선순위), `threshold`(스케줄 자격을 얻기 위한 최소 큐 누적 프레임 수), `timeout`(threshold 미달이어도 강제로 자격을 주는 대기시간) — 즉 **기아(starvation) 방지용 타이머가 내장**된 fair-queueing 스케줄러.
- `batch_size`는 "네트워크그룹 입출력 큐 크기"로 동작 — 스케줄러 파라미터이자 버퍼링 파라미터를 겸함.

### 2-2. Qualcomm QNN/Hexagon(HTP)이 실제로 제공하는 것 (공식 문서 확인)

Qualcomm AI Runtime(QAIRT, 구 QNN) SDK 공식 문서(`docs.qualcomm.com`, HTP Backend 문서)에서 확인된 것:

- **Graph Priority**: 그래프(모델) 단위로 우선순위를 설정할 수 있음. 높은 우선순위 그래프가 실행을 요청하면 HAP Compute Resource Manager가 낮은 우선순위 그래프에게 **협조적으로 자원(VTCM 등)을 양보(yield)하도록 요청**하는 방식(Yielding and Pre-Emption). 이는 강제 인터럽트가 아니라 협조적 선점(cooperative preemption).
- **Parallel Graph Execution / Concurrent Resource Sharing**: 여러 그래프를 동시에 로드해 리소스(스필-필 버퍼, VTCM)를 공유하며 실행하는 기능이 존재. 단, 공식 문서에 **"Android에서 V81 Hexagon 아키텍처에서만 지원"**이라고 명시됨. 즉 이 기능은 네 후보 기기 중 **Galaxy S26 Ultra(및 Z Fold8이면 그 기종)에서만 정식 지원**되고, S22(v69)·S24 Ultra(v75)에서는 이 특정 concurrent-sharing 최적화는 대상이 아님. 같은 그룹 내 그래프들은 **우선순위가 전부 동일해야 함**(다른 우선순위 그래프는 다른 그룹으로 분리해야 함)도 문서에 명시.
- **batch(배치)**: QNN에서 배치는 "한 그래프 호출에 여러 입력을 한꺼번에 넣는 텐서 배치 차원"을 의미 — Hailo처럼 "큐 깊이/버퍼링"과 결합된 개념이 아님. 그래프 준비(prepare) 시점의 배치 차원의 정수배로만 실행 가능(variable batch 지원은 제한적).
- **threshold / timeout 개념 자체가 없음**: 검색·문서 확인 결과 "낮은 우선순위 그래프가 일정 시간 이상 기다리면 강제로 스케줄된다"는 anti-starvation 타이머 파라미터를 QNN 공식 문서에서 찾지 못함. 즉 우선순위 차이가 크면 낮은 우선순위 모델이 Hailo보다 더 오래(이론상 계속) 밀릴 수 있는 구조로 보임 — 이는 Hailo 프로젝트에서 이미 발견한 "priority 차이 15 이상이면 starvation" 현상과 유사한 문제가 QNN에서는 **완화 장치 없이** 더 극단적으로 나타날 가능성을 시사(직접 실측 전까지는 추정).

### 2-3. 결론

| 항목 | Hailo Model Scheduler | Qualcomm QNN/HTP |
|---|---|---|
| 스케줄링 방식 | Round-robin 시분할 + fair queueing | Priority 기반 협조적 선점 |
| priority | O (0~31) | O (그래프 단위) |
| threshold | O | **없음** |
| timeout(기아 방지) | O | **없음(확인 안 됨)** |
| batch_size = 큐 깊이 | O | **없음(배치는 텐서 차원일 뿐)** |
| 다중 모델 동시 실행 자체 | O | O (단, 최신 칩 V81+Android 조합에서만 리소스 공유 최적화 정식 지원) |

→ **"같은 4개 파라미터(priority/threshold/timeout/batch_size)를 스윕하는 실험"은 스냅드래곤에서 그대로 재현 불가능.** 대신 "priority 차이에 따른 지연시간/기아 현상"만 골라서 재현하는 축소판 실험은 가능함.

---

## 3. YOLOv8 후처리(NMS)가 NPU에서 도는가 — Hailo와 비교

Hailo 프로젝트 결론(문서 6번 항목): Detection은 HEF에 baked-in NMS(칩상 처리), Segmentation/Pose는 host CPU 처리.

Qualcomm 쪽 확인 결과(Ultralytics 공식 QNN 통합 문서 + GitHub PR #18484 확인):

- QNN(HTP) 익스포트는 `torch.topk` 등 일부 연산자를 지원하지 않아서, **NMS가 내장된 end-to-end 그래프(NMSModel)를 만들 수 없고 "traditional output"으로 폴백**됨. 즉 **Detect/Segment/Pose/OBB 전부**, NPU 그래프는 raw tensor(박스 좌표+점수 등)만 출력하고, **NMS·박스 디코딩은 host(앱 코드, CPU)에서 수행**.
- Qualcomm AI Hub의 YOLOv8-Detection 벤치마크 표에서도 "Primary Compute Unit: NPU"로 표기된 것은 **NMS 이전 raw head까지의 그래프**를 의미하는 것으로 해석해야 함(NMS는 별도 host 단계).

즉 Hailo는 "Detection만은 칩에서 NMS까지 끝냄"이었는데, Qualcomm/Snapdragon 경로(QNN)는 **Detection까지 포함해 전부 host 후처리** — Hailo보다 더 단순하지만 더 제약적인 그림. (단, ONNX Runtime의 다른 실행 프로바이더(TensorRT 등)는 NMS 내장이 되는 경우가 있으나, 이는 NPU가 아닌 별도 경로라 이번 질문과 무관.)

---

## 4. NPU 사용률/지연시간을 어떻게 잴 수 있나 — 실측 방법론 확인

Raspberry Pi + Hailo에서는 `/proc`류 인터페이스 + HailoRT 공식 API(`get_queue_size_accumulators()` 등)로 손쉽게 NPU 사용률·큐 상태를 뽑았음. 안드로이드/스냅드래곤에서는 계층이 다름:

- **지연시간(latency)**: QNN SDK 자체 프로파일러(HTP Optrace/Hextimate Profiling, QNN HTP Analysis Summary)로 그래프·op 단위 실행시간을 뽑을 수 있음 — 앱에 별도 루트 권한 불필요. Hailo 프로젝트의 CSV(전처리/추론/후처리 시간 기록) 방식과 가장 가까운 대체재.
- **NPU 사용률(%)**: Qualcomm의 데스크톱 도구 **Snapdragon Profiler**(Windows/Mac/Linux에서 USB로 기기 연결, CPU/GPU/DSP/메모리/전력/열 실시간 그래프)가 공식 지원 도구. 삼성 자체 Galaxy GameDev 문서에서도 이 도구를 공식 추천 도구로 안내함 — **일반 소매(retail) 갤럭시 기기에서 사용 가능**하다는 뜻으로, 루팅을 전제로 하지 않음. 다만 일부 기능(Simpleperf, Vulkan Snapshot 등)은 "앱이 디버거블 빌드일 것"을 요구(루팅 기기는 예외) — 즉 **자체 제작한 추론 앱을 debug 빌드로 만들면 루팅 없이 대부분의 프로파일링 가능**.
- Hexagon SDK의 `sysmon` 툴도 있으나 이는 보통 칩벤더/OEM 파트너 대상 SDK 성격이 강해 일반 개발자가 접근하기엔 Snapdragon Profiler보다 문턱이 높음(확인 필요 사항으로 남김 — 단정하지 않음).

### 부트로더/루트 관련 주의

- 미국/캐나다 판매 스냅드래곤 갤럭시(예: SM-S928U)는 2023년 이후 **펌웨어 레벨에서 부트로더 언락 자체가 막혀** 있고, 비공식 유료 서비스(UNSAMLOCK 등)로만 우회 가능.
- **한국 통신사/자급제 모델은 공식적으로 OEM 언락이 가능한 경우가 많음**(과거부터 국내는 스냅드래곤 모델도 개발자 옵션의 OEM 잠금해제가 동작). 다만 통신사(SKT/KT/LGU+) 약정 모델은 정책상 막혀 있을 수 있어 보유 기기가 자급제인지 통신사 모델인지 확인이 필요함.
- 언락 시 **Knox가 영구적으로 trip**되어 삼성페이/시크릿폴더 등 일부 기능이 무력화됨 — 이건 되돌릴 수 없음.
- 하지만 **이번 실험(추론 앱 실행 + QNN/Snapdragon Profiler 프로파일링)은 루팅이 필수가 아님** — 루팅은 "더 깊은 시스템 카운터"가 필요할 때만 고려 대상.

---

## 5. 유사 선행 연구로 교차검증

- **"Puzzle: Scheduling Multiple Deep Learning Models on Mobile Device with Heterogeneous Processors"** (arXiv 2508.17764, 2025) — **Samsung Galaxy S23 Ultra(Snapdragon 8 Gen 2, Hexagon NPU)를 실제 기기로 사용**해 CPU/GPU/NPU에 걸친 다중 DNN 동시 스케줄링(모델을 서브그래프로 쪼개 이기종 프로세서에 매핑)을 연구. → **일반 소매 갤럭시 기기에서 다중 모델 동시추론 스케줄링 실험 자체가 학계에서 실제로 수행되고 있다는 강력한 방증.** 다만 이 연구는 Qualcomm의 네이티브 스케줄러 파라미터를 튜닝하는 게 아니라 자체 스케줄러(소프트웨어 레이어)를 얹는 접근이라, 우리 프로젝트처럼 "NPU 벤더가 제공하는 스케줄러 파라미터"를 스윕하는 방식과는 실험 성격이 다름.
- 같은 분야 관련 논문들("Optimizing Multi-DNN Inference on Mobile Devices through Heterogeneous Processor Co-Execution", "A Co-Scheduling Framework for DNN Models on Mobile...")도 다수 존재 — 이 주제(모바일 이기종 프로세서 위 다중 DNN 스케줄링)가 활발한 연구 분야임을 재확인.

---

## 6. 종합 결론 및 권장 방향

**가능한 것**
- 4개 후보 기기 모두 스냅드래곤 + QNN 공식 지원 대상 → YOLOv8 Det/Seg/Pose를 QNN(HTP)으로 변환해 온디바이스 NPU 추론 자체는 확실히 가능(Qualcomm AI Hub·Ultralytics 실측치로 검증됨).
- 3모델을 동시에(멀티스레드/멀티그래프로) 띄워 서로 다른 `priority`를 주고, 지연시간·처리량 변화를 관찰하는 실험은 가능. 학계에서도 동일 계열(갤럭시+Hexagon) 기기로 이미 하고 있음.
- QNN 자체 프로파일러 또는 Snapdragon Profiler로 지연시간/DSP 활용도 측정 가능(루팅 불필요).

**안 되는 것 / 제약**
- Hailo의 threshold·timeout·"batch_size=큐 깊이" 개념은 QNN에 대응 기능이 없어 **동일한 4파라미터 풀 팩토리얼 실험은 재현 불가**.
- Concurrent Resource Sharing(여러 그래프의 리소스 공유 최적화)은 **HTP v81(Galaxy S26 Ultra급) + Android 조합에서만** 공식 지원 — S22·S24 Ultra에서는 이 특정 최적화 없이 각 그래프가 더 독립적으로(자원 경쟁이 더 거칠게) 동작할 가능성.
- YOLOv8 세 모델 모두 NMS/후처리가 host(CPU) 처리라, Hailo 실험에서 발견한 "후처리가 무거우면 측정 latency가 밀린다"는 현상은 여기서도 재현될 개연성이 높지만, Hailo처럼 Detection만 예외적으로 칩 NMS를 쓰는 비교군은 만들 수 없음.

**권장 실험 재설계 방향(제안)**
1. 파라미터를 Hailo와 동일하게 맞추려 하지 말고, QNN이 실제로 제공하는 것(Graph Priority, HVX 스레드 수, VTCM 크기)으로 축을 다시 정의.
2. threshold/timeout 대신 "priority 차이 × 동시 실행 모델 수"에 따른 지연시간/기아 현상 관찰로 연구 질문을 좁히기.
3. 측정은 QNN 자체 프로파일러(코드 계측, Hailo의 CSV 방식과 가장 유사) + Snapdragon Profiler(DSP 활용도, 디버거블 빌드로 충분) 조합으로 설계.
4. 정확히 어떤 Z Fold 모델인지 확인 후 위 1장 표에서 세대(v69/v73/v75/v79/v81)를 확정 — 세대에 따라 2-2절의 concurrent-sharing 기능 사용 가능 여부가 갈림.

---

## 7. 확인이 더 필요한 부분(단정하지 않은 것들)

- QNN Graph Priority가 정확히 몇 단계(레벨)이고 몇 개까지 동시 등록 가능한지의 세부 API 값은 문서 원문 전체를 못 받아와(페이지 용량 문제) 검색엔진 요약으로만 교차확인함 — 실제 구현 전 `QNN_SDK_ROOT/include/QNN/HTP/QnnHtpGraph.h`에서 재확인 권장.
- Hexagon SDK `sysmon` 툴의 일반 개발자 접근 가능 여부(서명된 이미지 필요한지 등)는 확정 못함.
- 보유 중인 Z Fold의 정확한 모델명(SM-F9xx) 미확인 — 1장 표 적용을 위해 필요.
- 국내 통신사 모델 OEM 언락 가능 여부는 통신사/개통 형태(자급제 vs 약정)에 따라 달라 일반화하지 않음.

---

## 8. 방향 전환 — "모델은 자유(LLM 등), 대신 스케줄·전력·온도 파라미터로 벤치마킹" (추가 조사)

사용자가 모델을 YOLOv8에 고정하지 않고 LLM 등으로 바꾸고, 대신 **스케줄/전력/온도 같은 "성능 제한" 파라미터를 바꿔가며 환경별 벤치마킹**하고 싶다고 방향을 조정함에 따라 추가로 조사한 내용.

### 8-1. 결론 먼저

**이 방향은 원래 계획(Hailo 스케줄러 파라미터 재현)보다 오히려 스냅드래곤에 훨씬 잘 맞고, 실제로 학계에서 거의 동일한 실험이 이미 수행되어 방법론까지 공개돼 있음.** 특히 갤럭시 S24 Ultra로 정확히 이 실험(LLM 온디바이스 추론 + 전력/온도 관측)을 수행한 2026년 논문을 찾아 원문을 확인함 — 이게 가장 강력한 실현 가능성 증거.

### 8-2. 모델: 온디바이스 LLM을 스냅드래곤에서 돌리는 경로

| 경로 | 실행 프로세서 | 비고 |
|---|---|---|
| **Qualcomm GenieX**(구 Genie, `github.com/qualcomm/GenieX`) | NPU(Hexagon, QAIRT 경로) 또는 NPU/GPU/CPU(llama.cpp/GGUF 경로) | 공식 Android 지원 기기가 **"Snapdragon 8 Elite · 8 Elite Gen 5"로 명시** — 즉 NPU 전용(QAIRT) 경로는 최신 2세대 칩(Z Fold7급, S26 Ultra/Z Fold8급)이 공식 대상. S22·S24 Ultra는 GenieX 공식 지원 목록에 없음(단, llama.cpp 백엔드는 더 넓은 Hexagon 세대를 커버할 가능성 있으나 공식 확인 못함) |
| **MLC-LLM**(Apache TVM 기반) | Adreno GPU(OpenCL) | 실제로 아래 8-5 논문에서 **S24 Ultra에 이 경로로 LLM을 돌려 성공**함 — NPU가 아니라 GPU를 씀 |
| **llama.cpp** | CPU/GPU/Vulkan(일부 Hexagon 실험적 지원) | 범용성 가장 높음, 가장 많은 레퍼런스/커뮤니티 존재 |

→ **정리**: S22·S24 Ultra는 "NPU로 LLM 스케줄 파라미터를 조정"하는 실험보다는 **GPU(Adreno) 경로로 LLM을 돌리고 GPU DVFS/온도를 관측하는 실험**이 현재 검증된 방법. S26 Ultra(및 Z Fold8이면 그 기종)에서만 Qualcomm 공식 NPU 전용 LLM 스택(GenieX qairt 백엔드)이 대상 기기로 명시됨.

### 8-3. "스케줄/성능 제한" 파라미터로 실제 존재하는 것

- **Qualcomm HTP(NPU) Performance Profile**: QNN HTP Performance Infrastructure API가 `default / burst / balanced / low_power / high_performance` 같은 성능 모드를 노출하며, 내부적으로 DCVS(Dynamic Clock & Voltage Scaling) 전압-클럭 코너를 투표(voting)하는 방식으로 동작(Hexagon SDK의 HAP_Power DCVS v3와 연결). GenieX 관련 자료에서도 "HTP 백엔드 설정에서 burst(최대 처리량) vs sustained_high_performance(열 안정성)"를 고를 수 있다고 확인됨 — **이게 원래 하려던 "스케줄 파라미터 스윕"의 가장 근접한 대응물**.
- **Android ADPF(Android Dynamic Performance Framework)**: 벤더 무관 공식 API. `Thermal API`(`getThermalHeadroom()`, 0.0=쓰로틀링 없음 ~ 1.0=심각)로 열 상태를 앱이 관측 가능. `PerformanceHintManager`로 CPU 클럭/코어 배정에 힌트를 줄 수 있음. 둘 다 **루팅 불필요, 4개 기기 전부(안드로이드 12+)에서 동작**.
- **시스템 레벨 전원 모드**(설정 > 배터리 > 전력 모드: 고성능/최적화/절전/최대 절전): 절전 모드는 CPU 최고 속도를 제한하는 것으로 알려짐(예: 중간 절전 시 약 70%로 제한) — 코드 없이 OS 설정만으로 만들 수 있는 "환경" 축.
- **자연 발생 열 스로틀링**: 별도 장치 없이 반복 추론(sustained load)만으로 기기가 스스로 클럭을 낮추는 현상을 유도해 관측하는 방식 — 아래 논문이 정확히 이 방법으로 실험함. 리뷰어들이 쓰는 "CPU Throttling Test" 앱도 동일한 원리(지속 부하 → GIPS/클럭 추이 기록)로, 루팅 없이 커뮤니티 표준처럼 쓰임.

### 8-4. 직접적 선행연구 확인 — Galaxy S24 Ultra로 사실상 동일한 실험을 수행한 논문

**"LLM Inference at the Edge: Mobile, NPU, and GPU Performance Efficiency Trade-offs Under Sustained Load"** (arXiv 2603.23640, 2026) — 원문 전체를 직접 읽고 확인함.

- 4개 플랫폼(RTX 4050 노트북, Raspberry Pi5+Hailo-10H, iPhone 16 Pro, **Samsung Galaxy S24 Ultra**)에서 Qwen2.5-1.5B(4bit 양자화)를 **20회 연속 추론**시키며 처리량(tok/s)·지연시간·온도·전력·배터리 소모를 기록.
- **S24 Ultra 실험 방법(그대로 재현 가능한 수준으로 상세히 공개됨)**:
  - 프레임워크: MLC-LLM 0.1.0 (Apache TVM 0.14, OpenCL 3.0) — **Adreno 750 GPU**에서 실행(NPU 아님)
  - 온도: 화면 22°C±2°C에서 10분 안정화 후 시작, 매 반복마다 최대 GPU/CPU 온도 기록
  - 전력: **Android `BatteryManager` API**(루팅 불필요)로 전류·전압을 반복마다 샘플링 → 전력·에너지/토큰 산출(단, 절대 정확도는 하드웨어 계측 대비 낮다고 명시)
  - **GPU 클럭(주파수, MHz)**: 반복마다 관측 기록(안드로이드 전용 메트릭으로 명시) — 이게 "스케줄/DVFS" 축의 실측치
  - 실측 결과: GPU가 1,000MHz(4~5회차, 부스트)에서 720~770MHz 플랫토(8회차 이후)로 스스로 다운클럭 → **처리량이 피크 대비 15% 감소, 10.83±0.75 tok/s로 안정화**. GPU 온도는 68.5°C까지 오른 뒤 63~65°C에서 안정. "OS 강제 주파수 하한(frequency floor)" 이벤트도 관측된 적 있다고 언급(설정을 안 맞추면 더 일찍 발생).
  - 저자들은 이 안정적인 20회 완주를 위해 **화면 끄기 + context 2048 토큰 + prefill chunk 128 토큰**이라는 구체적 튜닝이 필요했다고 명시 — 기본 설정으로는 더 일찍 열 제한에 걸림.
- 비교 대상 아이폰16 프로는 반대로 3번째 반복만에 -38% 급락 후 "Hot" 상태로 굳어지는 **완전히 다른 양상**(iOS는 서서히 조절하는 거버너가 아니라 3단계 thermal state 방식)을 보여, "환경(기기)에 따라 벤치마크가 어떻게 달라지는가"라는 질문에 정확히 부합하는 교차비교 사례.

관련 후속 크로스체크 논문(제목만 확인, 본문 전체는 미독):
- **"When NPUs Are Not Always Faster: A Stage-Level Analysis of Mobile LLM Inference"**(arXiv 2605.27435) — Snapdragon 8 Gen 3에서 Prefill 단계는 CPU가 NPU보다 빠르고 Decode 단계에서만 NPU가 제한적 이득을 준다고 보고 — "NPU만 쓰면 무조건 좋다"는 가정이 틀릴 수 있음을 시사, 실험 설계 시 CPU/GPU/NPU 백엔드를 축으로 넣을 근거가 됨.
- **"Characterizing Mobile SoC for Accelerating Heterogeneous LLM Inference"**(arXiv 2501.14794) — 모바일 SoC의 GPU+NPU 이기종 병렬 실행을 다룸(HeteroInfer), 대형 모바일 LLM 추론 연구가 활발함을 재확인.

### 8-5. 기기별 재정리 (LLM+전력/온도 벤치마킹 관점)

| 기기 | 확인된 실행 경로 | 스케줄/전력 파라미터 |
|---|---|---|
| Galaxy S22 (v69) | GPU(Adreno)/CPU 경로가 현실적(GenieX NPU 전용 경로 공식 대상 아님) | ADPF Thermal API, 시스템 전원모드, 자연 스로틀링 관측 |
| Galaxy Z Fold4 (v69) | 동일 — GPU/CPU 경로 | 동일 + **폼팩터 비교축**(북형 폴더블, 배터리/방열 설계가 S22와 다름) |
| Galaxy Z Flip4 (v69) | 동일 — GPU/CPU 경로 | 동일 + **폼팩터 비교축**(클램쉘, 배터리 용량이 셋 중 가장 작아 열 여유가 가장 적을 가능성) |

참고로 위 8-4 논문의 S24 Ultra(SM8650, v75) 실측(MLC-LLM+Adreno GPU, GPU 클럭 다운클럭→15% 성능저하)은 **칩 세대가 다르므로 절대치는 그대로 적용 안 되지만, "GPU 경로로 LLM을 돌리고 GPU 클럭/온도를 관측한다"는 방법론 자체는 v69 세 기기에도 그대로 적용 가능**함(같은 MLC-LLM/llama.cpp 스택이 v69도 지원).

### 8-6. 권장 실험 설계(제안, S22/Fold4/Flip4 기준으로 수정)

1. 모델: Qwen2.5-1.5B급 소형 LLM(위 논문과 동일 모델로 하면 결과를 직접 비교할 수 있어 유리) — GGUF 4bit. 세 기기 다 v69로 동급이므로 동일 모델·동일 포맷으로 통일 가능.
2. 실행 스택: 세 기기 전부 MLC-LLM 또는 llama.cpp(GPU/CPU 경로) — GenieX의 NPU 전용(qairt) 경로는 8 Elite 이상 공식 대상이라 이 세 기기엔 해당 없음. YOLOv8류는 QNN HTP(v69 공식 지원)로 NPU 실행 가능.
3. **독립변수(환경) 후보를 "칩 세대 차이"가 아니라 "폼팩터 차이"로 재설계**: ① 기기(S22 바형 / Fold4 북형폴더블 / Flip4 클램쉘 — 배터리 용량·방열 설계가 다름), ② 시스템 전원 모드(고성능/절전), ③ 화면 on/off, ④ 반복 횟수(sustained load로 자연 스로틀링 유도). HTP 성능 프로파일(burst/balanced/low_power)은 v69에서 API 자체는 있을 가능성이 높으나(9장 참고, 실기 확인 필요), Snapdragon Profiler 기반 정밀 계측은 셋 다 공식 지원 밖이라 자체 계측(QNN 자체 프로파일러 + ADPF)에 의존해야 함.
4. 종속변수(측정치): 처리량(tok/s), 첫 토큰 지연(TTFT), GPU/CPU/배터리 온도, BatteryManager 기반 전력, ADPF 열 상태(thermal headroom/status).
5. 프로토콜: 위 논문의 "10분 온도 안정화 → 워밍업 1회 폐기 → ΔT<2°C 확인 → 20회 반복, 1초 간격 → CSV 기록" 절차를 그대로 채택 가능(재현성 검증된 절차).

---

## 9. 실행 가능성 상세 조사(2차) — "정말 손에 넣고 돌릴 수 있는가"

기능 존재 여부를 넘어, 실제로 계정·비용·설치·서명 장벽이 있는지까지 파고든 결과.

### 9-1. 칩 모델넘버 확정 (실제 보유 기기 3개 → 정확한 SoC 모델)

| 기기 | 정확한 SoC 모델 | 세대 |
|---|---|---|
| Galaxy S22(한국) | SM8450 | Snapdragon 8 Gen 1 (2022) |
| Galaxy Z Fold4 | SM8475 | Snapdragon 8+ Gen 1 (2022, 전 지역 공통) |
| Galaxy Z Flip4 | SM8475 | Snapdragon 8+ Gen 1 (2022, 전 지역 공통) |

(참고로 비교 기준점으로 조사했던 S24 Ultra=SM8650-AC/Gen3, S26 Ultra=SM8850/Elite Gen5는 사용자 보유 기기가 아님 — 아래는 실제 보유 기기 3개 기준으로 재작성)

### 9-2. Snapdragon Profiler(Qualcomm Profiler) 실제 지원 칩셋 — 중요한 새 발견 (보유 기기엔 불리한 소식)

Qualcomm 자료(사내/파트너 배포용으로 보이는 문서에서 확인 — 명시적으로 재배포 금지 표기가 있어 이 문서의 URL은 참고자료에 올리지 않음. 독립적으로 재확인 권장)에 따르면, **Qualcomm Profiler(Snapdragon Profiler)가 Android에서 공식 지원하는 칩셋은 SM8850 / SM8750 / SM8650 / SXR2230P뿐**입니다.

| 기기 | SoC | NPU/DSP 딥 프로파일링(Snapdragon Profiler) |
|---|---|---|
| Galaxy S22 (SM8450) | Gen 1 | **미지원** — 리스트에 없음 |
| Galaxy Z Fold4 (SM8475) | 8+ Gen 1 | **미지원** — 리스트에 없음 |
| Galaxy Z Flip4 (SM8475) | 8+ Gen 1 | **미지원** — 리스트에 없음 |

**세 기기 모두 공식 지원 목록 밖입니다.** 즉 9-3의 NPU 클럭 set/limit 툴과 Snapdragon Profiler의 GUI 기반 실시간 NPU 사용률 그래프는 **셋 다 공식적으로는 못 쓸 가능성이 높음**(도구 자체가 설치는 되어도 해당 칩에서 캡션 없이 동작 안 하거나 데이터가 안 나올 수 있음 — 실기 확인 전까지 단정은 어려움, QPM에서 기기 연결 시도해보는 게 가장 확실).

### 9-3. 결정적으로 유용한 발견 — NPU(Q6/CDSP) 클럭을 직접 세팅/제한하는 공식 툴이 있음

같은 자료에서 확인된 `profilerUtilityApp`(온디바이스 CLI, Qualcomm Profiler에 포함)의 **Clocks 서비스**가 정확히 사용자가 원하는 "성능 제한 파라미터"에 해당합니다:

- `profilerUtilityApp --dsp --clocks set --cengClock 700` → Q6(NPU/DSP) 클럭을 원하는 값으로 **직접 설정**
- `--clocks limit` → 최대 클럭을 **제한**(스로틀 흉내)
- `--clocks remove` → 기본값 복원
- `qprof -sm profiler --q6 npu0 --duration 5 ...` → NPU(npu0) 단위로 부하/사용률 프로파일링

→ 이게 있으면 "온도가 자연스럽게 오를 때까지 기다리는" 간접적 방법 대신, **NPU 클럭 자체를 스크립트로 여러 단계(예: 낮음/중간/최대)로 강제 설정해놓고 동일 워크로드를 돌려 처리량 변화를 재는**, 훨씬 통제된 실험이 가능해집니다. **다만 이 툴은 Qualcomm Profiler 패키지에 포함되어 있어 9-2와 동일한 지원 칩셋 제약(SM8850/8750/8650)을 받을 가능성이 높고, S22/Fold4/Flip4(SM8450/SM8475)에서 그대로 동작한다는 보장은 없습니다** — 이 부분이 실기로 가장 먼저 확인해봐야 할 항목입니다(설치 자체는 무료 계정으로 되니, 연결 시도해보고 안 되면 대안으로 넘어가는 방식 추천).

### 9-4. QAIRT/Hexagon SDK 다운로드 — 계정은 필요하지만 무료

- Qualcomm AI Runtime(QAIRT) SDK는 **Qualcomm Package Manager(QPM)**를 통해 배포되며, **무료 Qualcomm ID(계정) 가입 후** "Qualcomm AI Runtime - Community Edition"을 설치하면 됨. 비용 없음, 학생 개인 사용에 제약 없어 보임(단, 소프트웨어 이용약관 동의는 필요).
- Hexagon SDK도 동일하게 QPM 계정 + `qpm-cli --license-activate` 로 라이선스 활성화가 필요(역시 무료, 재배포만 금지).
- **Snapdragon Profiler도 동일 QPM 계정으로 설치**(`qpm-cli --license-activate Qualcomm_Profiler`).
→ 즉 "회사/기관 소속이어야 한다" 같은 장벽은 없고, **개인 Qualcomm 계정 하나면 다 됨**.

### 9-5. Knox/사이드로딩 — 오해 정리

- 직접 만든 NDK 기반 벤치마크 앱을 갤럭시에 설치해 QNN API를 호출하는 것은 **루팅이 아님**. 개발자 옵션에서 USB 디버깅만 켜면 `adb install`로 얼마든지 가능하고 Knox 워런티 비트(e-fuse)에 영향 없음.
- Knox e-fuse가 트립되는 경우는 **부트로더 언락 + 커스텀 롬/루트 플래시**를 할 때뿐. 일반 개인 기기(회사 MDM에 등록되지 않은 경우)는 USB 디버깅이 기본적으로 막혀있지 않음 — 막혀 있다면 그건 기업용 Knox Workspace/MDM이 깔린 경우.
- 따라서 **9-3의 NPU 클럭 제어, Snapdragon Profiler 프로파일링, QNN 앱 실행까지 전부 "일반 개발자 모드"만으로 가능**하고, 루팅·Knox 트립은 필요 없음.

### 9-6. 최종 실행 가능성 표 (실제 보유 기기 3개 기준으로 확정)

| 항목 | Galaxy S22 (SM8450) | Z Fold4 (SM8475) | Z Flip4 (SM8475) |
|---|---|---|---|
| QNN(NPU)으로 YOLOv8류 모델 실행 | **O**(HTP v69 공식 지원) | **O** | **O** |
| LLM 온디바이스 실행(MLC-LLM/llama.cpp, GPU·CPU 경로) | **O** | **O** | **O** |
| LLM NPU 전용 경로(GenieX qairt) | 공식 대상 아님(8 Elite 이상만) | 공식 대상 아님 | 공식 대상 아님 |
| HTP Graph Priority(우선순위 기반 선점) API | 이론상 O(버전 게이팅 확인 안 됨, v81 전용 기능만 제외하면 될 걸로 추정) | 상동 | 상동 |
| Snapdragon Profiler로 NPU 딥 프로파일링 | **X**(미지원 칩셋) | **X** | **X** |
| NPU 클럭 직접 set/limit 툴 | **불확실/가능성 낮음**(위와 같은 이유) | 상동 | 상동 |
| QNN 자체 그래프 프로파일러(Optrace/Hextimate, latency 계측) | O로 추정(그래프 레벨 기능이라 칩 목록 제약과 무관할 가능성) | 상동 | 상동 |
| Android ADPF(Thermal API 등, 온도 관측) | O(안드로이드 12+ 공통 기능) | O | O |
| 루팅 필요 여부 | 불필요 | 불필요 | 불필요 |
| 계정/비용 장벽 | 무료 Qualcomm 계정만 | 〃 | 〃 |

**결론**: 세 기기 모두 **"모델을 NPU/GPU로 돌리고, 그 위에서 latency·온도·배터리를 자체 계측"하는 수준의 실험은 확실히 가능**합니다. 다만 **Qualcomm의 최신 GUI 프로파일링 툴(Snapdragon Profiler)과 그에 딸린 NPU 클럭 강제 제어 기능은 셋 다 공식 지원 밖**이라, "NPU 사용률을 그래프로 실시간 확인" 같은 손쉬운 방법은 못 쓰고 **자체 앱 코드에 타이머를 심어 CSV로 기록하는 방식**(Hailo 프로젝트에서 이미 하던 방식과 사실상 동일)에 의존해야 합니다. 즉 실험은 가능하지만 계측 편의성은 최신 기기보다 떨어집니다.

**중요한 재해석**: 세 기기가 전부 동일 세대(HTP v69)라는 게 단점만은 아닙니다. 오히려 **"칩은 고정하고 폼팩터(바형/북형폴더블/클램쉘)만 바꿔서 열 스로틀링·지속 성능 차이를 보는" 실험으로는 이상적인 조합**입니다 — Fold4·Flip4는 배터리 용량과 힌지 구조 때문에 방열 여유가 S22와 다를 가능성이 높고, 8-4의 논문이 보여준 "동일 SoC라도 기기(발열설계)에 따라 스로틀링 곡선이 다르다"는 발견을 정확히 재현/확장하는 셈이 됩니다.

### 9-6-1. 오해 방지 — "측정 가능한 게 온도뿐인가?" 아님, 계층을 구분해야 함

9장 전체가 "Snapdragon Profiler가 안 된다"는 쪽에 무게가 실려 자칫 "성능 측정 자체가 안 된다"로 오해될 수 있어 명확히 구분함. **측정 항목을 3계층으로 나누면:**

| 계층 | 항목 | 세 기기(S22/Fold4/Flip4)에서 가능 여부 | 필요 도구 |
|---|---|---|---|
| ① 앱 레벨 스톱워치(가장 쉬움, 사실상 항상 됨) | 추론 1회 latency(ms), 처리량(FPS/tok/s), TTFT, 반복별 CSV 기록 | **확실히 O** — 별도 Qualcomm 도구 불필요, `System.nanoTime()` 수준으로 충분 | 자체 앱 코드만 |
| ② QNN SDK 자체 그래프 프로파일러 | Optrace/Hextimate — op(레이어) 단위 시간 분해, HVX 병렬화 효율 | **O로 추정**(Snapdragon Profiler와 별개 기능이라 9-2의 칩 제약과 무관할 가능성 높음, QAIRT 2.39+ 필요) | QAIRT SDK(무료 계정) |
| ③ Snapdragon Profiler 하드웨어 카운터 | NPU/GPU 실시간 사용률(%), GPU/NPU 클럭(MHz) 실시간 그래프·강제 고정 | **미지원 가능성 높음**(9-2 참고) | Snapdragon Profiler(칩 제약 있음) |

즉 **"모델이 얼마나 빨리 도는가"(latency·throughput 같은 성능 지표)는 ①번 계층이라 세 기기 전부 확실히 측정 가능**합니다. 8-4에서 소개한 arXiv 논문의 처리량(tok/s)·지연시간(TTFT)·온도·배터리 소모 데이터도 전부 ①(스톱워치)+Android 공개 API(BatteryManager) 조합으로 얻은 것이지 Snapdragon Profiler를 쓴 게 아닙니다. 안 되는 건 ③번, 즉 "그 성능이 나오는 동안 칩 내부 사용률·클럭이 어떻게 움직였는지"를 세밀하게 보는 것뿐입니다.

### 9-7. 아직 남은 불확실성

- Snapdragon Profiler 지원 칩셋 리스트의 출처가 사내 문서로 보여 재배포하지 않았음 — **실제 설치 후 QPM/기기 연결을 시도해봐야 SM8450/SM8475에서 정말 안 되는지 100% 확인됨**(현재는 강한 정황 증거 수준).
- HTP Graph Priority API가 v69에서도 동일하게 노출되는지(문서상 버전 게이팅이 명시된 건 "Concurrent Resource Sharing"=v81 전용 하나뿐이었고, Graph Priority 자체의 최소 버전 요구사항은 확인 못함) — 실기/SDK 헤더 확인 필요.
- `--enableRoot` 같은 옵션이 CLI에 존재하는 것으로 보아, 기본 프로파일링은 루팅 불필요해도 일부 심화 기능(스레드 레벨 상세 통계 등)은 루팅 시 더 많은 정보를 줄 가능성 있음 — 실기 테스트 전까지 단정 불가.
- Fold4/Flip4의 정확한 배터리 용량·발열 설계 차이가 실제로 스로틀링 곡선에 유의미한 차이를 만드는지는 가설이며, 검증된 선행연구를 찾지는 못함(합리적 추정).

---

## 10. Galaxy S26 Ultra 기준 — 측정/통제 변인 상세 (다회 교차검증)

S26 Ultra(Snapdragon 8 Elite Gen 5, SM8850-AC/AD, HTP v81)는 9장에서 확인한 여러 제약(Snapdragon Profiler 미지원, GenieX NPU 전용 경로 미지원 등)이 전부 풀리는 기기임. 여기서는 "측정할 수 있는 것(종속변인)"과 "통제할 수 있는 것(독립변인)"을 나눠 정리함. 각 항목은 공식문서(docs.qualcomm.com QAIRT 문서, Qualcomm 제품 브리프)와 GenieX 실제 소스코드/문서(qualcomm/GenieX 저장소의 notes/run.md, 트러블슈팅 문서) 두 계열의 출처를 교차 대조함.

### 10-1. 측정 가능(종속변인) — 계층별 정리

| 계층 | 구체 항목 | 도구 | 확인 상태 |
|---|---|---|---|
| ① 앱 레벨 | 추론 latency(ms), 처리량(FPS/tok/s), TTFT, 반복별 CSV | 자체 코드(타임스탬프) | 확실(모든 계층 공통) |
| ② QNN SDK 자체 프로파일러 | op(레이어)별 시간 분해, HVX 병렬화 효율, roofline 분석 | QAIRT SDK 내장 Optrace/Hextimate(QAIRT 2.39+ 필요) | 공식문서 확인, v81 제약 없음 |
| ③ Snapdragon Profiler(SM8850 공식 지원) | NPU/GPU/CPU 실시간 사용률(%), DDR 대역폭, 150개+ 하드웨어 카운터(22개 카테고리), Perfetto trace, NSP roofline | Qualcomm Profiler GUI/CLI(무료 계정) | 공식 지원 칩셋 목록에 SM8850 포함 확인 |
| ③-부가 | Q6(NPU) 클럭(MHz) 실시간 조회 | `profilerUtilityApp --dsp --clocks`(getstate) | 같은 패키지, SM8850 지원 |
| ④ Android 공식 API | 열 상태(Thermal headroom 0.0~1.0), 배터리 온도/전류/전압 | ADPF Thermal API, BatteryManager | 안드로이드 12+ 공통, 벤더 무관 |

**셋 다 지원되는 카테고리 22개의 정확한 명칭 목록은 공개 문서에서 전부 확인하지 못함**(CPU/GPU/DSP/메모리/전력/열/네트워크로 크게 묶인다는 것만 확인) — 실기에서 Snapdragon Profiler GUI의 `--capabilities` 명령으로 직접 뽑아보는 게 가장 정확함.

### 10-2. 통제 가능(독립변인) — 항목별 근거

| 변인 | 설정 방법 | 근거/출처 | 확실성 |
|---|---|---|---|
| **HTP Graph Priority**(우선순위) | QnnGraph/QnnContext 설정 시 priority 지정, 협조적 선점(cooperative preemption)으로 낮은 우선순위 그래프가 VTCM 등 자원 양보 | QAIRT 공식 문서(HTP Backend – Yielding and Pre-Emption) | 확실(다만 정확한 단계 수·enum 값은 SDK 헤더 확인 필요) |
| **HTP Performance Profile** | burst / balanced / low_power / high_performance / sustained_high_performance / default 중 선택 — 내부적으로 DCVS(클럭-전압 코너) 투표 | QNN HTP Performance Infrastructure 공식 문서 + GenieX 관련 자료(HTP 백엔드 설정에서 확인) | 확실 |
| **Concurrent Resource Sharing**(여러 그래프 자원 공유 최적화) | 동일 우선순위 그룹으로 묶어 spill-fill/VTCM 백업 버퍼 공유 | QAIRT 공식 문서 — **"Android에서 V81 아키텍처만 지원"이라고 명시** | 확실, **네 후보 기기 중 S26 Ultra만 해당** |
| **VTCM 크기**(그래프별 예약 메모리) | `QNN_HTP_GRAPH_CONFIG_OPTION_VTCM_SIZE_IN_MB` | QAIRT 공식 문서 | 확실 |
| **HVX 스레드 수**(그래프별) | `QNN_HTP_GRAPH_CONFIG_OPTION_NUM_HVX_THREADS`(기본 4) | QAIRT 공식 문서 | 확실 |
| **배치(batch) 크기** | 그래프 준비 시점 텐서 배치 차원의 정수배로 실행(가변 배치는 제한적) | QAIRT 공식 문서 | 확실(단, Hailo의 "큐 깊이" 개념과는 다름) |
| **정밀도/양자화** | INT2/INT4/INT8/INT16/FP8/FP16 및 혼합 정밀도 | Qualcomm 8 Elite Gen5 제품 브리프(공식) | 확실 — INT2는 이 세대에 새로 추가됨 |
| **실행 배치 모드(llama_cpp 경로)**: `npu`(HTP0 고정) vs `hybrid`(텐서 단위 CPU+NPU 자동분배) | `--device npu` / `--device hybrid` 플래그 | **GenieX 공식 소스 문서(notes/run.md)에서 실측치까지 직접 확인** — hybrid가 prefill 더 빠르지만 비결정적, npu 고정은 느리지만 결정적 | 매우 확실(직접 원문 확인) |
| **RPC Control Latency**(세션별 CPU 저전력 모드 웨이크업 지연 투표) | QNN HTP Performance Infrastructure API | QAIRT 공식 문서 | 확실 |
| **NPU/CDSP 클럭 직접 set/limit** | `profilerUtilityApp --dsp --clocks set/limit/remove --cengClock` | Qualcomm Profiler 패키지(SM8850 지원 목록에 포함) | 확실(9-3에서 이미 확인, S26U는 지원 목록 안에 있음) |
| **안드로이드 시스템 레벨**(전원모드, 화면 on/off, Game Booster 프로필) | OS 설정 | Samsung 공식 지원 문서 | 확실, 기기 무관 |
| **자연 열 스로틀링 유도** | 반복 추론으로 온도 상승 유도 | 8-4 논문에서 실증 | 확실 |

### 10-3. 교차검증 중 정정한 내용 — "듀얼 NPU 코어" 주장은 근거 부족으로 기각

이전 리서치 단계에서 "Snapdragon 8 Elite에 NPU 코어가 2개 있어 선택 가능하다"는 정보를 얻었으나, 이번에 GenieX 공식 저장소 문서(`notes/run.md`)를 직접 읽어 대조한 결과:

- **일반 모바일용 Hexagon NPU(HTP)는 표준 AI 워크로드에 대해 "HTP0" 하나의 연산 장치로 노출됨**(`qairt` 런타임은 "QAIRT exposes only one device"라고 명시).
- Qualcomm 공식 8 Elite Gen5 제품 브리프에도 메인 Hexagon NPU는 "12 scalar + 8 vector + 1 accelerator" 구성(하나의 NPU 안의 연산 유닛 구성)이라고만 나오고, 별도의 "두 번째 범용 NPU 코어"는 없음.
- "듀얼 NPU"라고 언급된 부분은 **Qualcomm Sensing Hub의 별도 "Micro NPU"(음성/센서용 초저전력 상시감지 전용 블록)** 를 가리키는 것으로 보이며, 이는 YOLOv8/LLM 같은 일반 추론 워크로드를 올릴 수 있는 대상이 아님.
- `HTP0/HTP1/HTP2/HTP3` 같은 다중 장치 ID가 SDK에 존재하는 건 맞지만, 이는 자동차용(SA8775P 등) 등 여러 NSP 코어를 가진 다른 칩군을 위한 범용 추상화이고, S26 Ultra(모바일 8 Elite Gen5)에 실제로 여러 개 물리적으로 있다는 근거는 못 찾음.

→ **정정**: S26 Ultra에서 "여러 NPU 코어에 모델을 분산 배치"하는 것은 Hailo의 멀티칩 스케줄링과 대응되지 않음. 대신 10-2의 "priority + performance profile + concurrent resource sharing" 조합, 그리고 llama_cpp의 "npu vs hybrid 배치 모드"가 실질적인 통제 변인임.

### 10-4. QAIRT(NPU 전용) 경로의 제약 — 실험 설계 시 주의

GenieX 공식 문서에서 확인된 중요한 제약: **`qairt`(Qualcomm AI Hub에서 받은 사전컴파일 바이너리) 경로는 context length(nCtx)와 GPU 레이어 수가 컴파일 시점에 고정(baked-in)** 되어 있어, 실행 중에 바꾸려 하면 `PARAM_NOT_SUPPORTED` 에러가 남. 즉 "context 길이를 실험 변인으로 스윕"하려면 조건별로 **미리 여러 개의 바이너리를 컴파일**해둬야 함(런타임 플래그로 못 바꿈). 반면 `llama_cpp`(GGUF) 경로는 이런 제약이 없어 더 유연함.

---

## 참고 자료(출처)

- Qualcomm QAIRT(QNN) HTP Backend 공식 문서: https://docs.qualcomm.com/doc/80-63442-10/topic/htp_backend.html
- Qualcomm QAIRT HTP Yielding and Pre-Emption: https://docs.qualcomm.com/bundle/publicresource/topics/80-63442-50/htp_yielding.html
- Ultralytics 공식 QNN 익스포트 가이드(HTP 아키텍처 버전-기기 매핑표 포함): https://docs.ultralytics.com/integrations/qnn
- Ultralytics NMS 관련 PR(QNN은 topk 미지원으로 traditional output 폴백): https://github.com/ultralytics/ultralytics/pull/18484
- Qualcomm AI Hub YOLOv8-Detection 벤치마크: https://huggingface.co/qualcomm/YOLOv8-Detection
- Google AI Edge LiteRT — Qualcomm NPU 지원 기기 목록(SM8450/8550/8650/8750 등): https://ai.google.dev/edge/litert/android/npu/qualcomm
- Samsung Galaxy GameDev — Qualcomm Snapdragon Profiler 안내: https://developer.samsung.com/galaxy-gamedev/resources/tool-guides/qualcomm.html
- Puzzle: Scheduling Multiple DL Models on Mobile Device with Heterogeneous Processors (arXiv 2508.17764, Galaxy S23 Ultra 실측): https://arxiv.org/abs/2508.17764
- Galaxy S26 Ultra Snapdragon 8 Elite Gen 5 확인: https://www.thelec.net/news/articleView.html?idxno=10051 , https://www.forbes.com/sites/ewanspence/2026/02/18/samsung-galaxy-s26-ultra-snapdragon-specs-launch/
- Galaxy Z Fold8 칩셋(Snapdragon 8 Elite Gen 5) 확인: https://samsung.gadgethacks.com/news/samsung-galaxy-z-fold-8-snapdragon-8-elite-gen-5-matches-s26-ultra/
- Galaxy S24 Ultra Snapdragon 8 Gen 3 for Galaxy 확인: https://www.sammobile.com/news/galaxy-s24-ultra-snapdragon-8-gen-3-chip-official/
- 갤럭시 S22 한국 모델 스냅드래곤 탑재 확인(엑시노스 2200 수율 문제): https://zdnet.co.kr/view/?no=20220210105317 , https://www.digitaltoday.co.kr/news/articleView.html?idxno=431657
- 미국 스냅드래곤 갤럭시 부트로더 언락 제한/한국 모델 차이: https://www.thecustomdroid.com/samsung-galaxy-bootloader-unlock-guide/
- LLM Inference at the Edge: Mobile, NPU, and GPU Performance Efficiency Trade-offs Under Sustained Load(arXiv 2603.23640, Galaxy S24 Ultra 실측 원문 확인): https://arxiv.org/html/2603.23640v2
- When NPUs Are Not Always Faster: A Stage-Level Analysis of Mobile LLM Inference(arXiv 2605.27435): https://arxiv.org/pdf/2605.27435
- Characterizing Mobile SoC for Accelerating Heterogeneous LLM Inference(arXiv 2501.14794): https://arxiv.org/abs/2501.14794
- Qualcomm GenieX(Genie 커뮤니티판) 공식 지원 플랫폼 목록: https://github.com/qualcomm/geniex
- Qualcomm QNN HTP Performance Infrastructure(DCVS, 성능 프로파일): https://docs.qualcomm.com/doc/80-63442-10/topic/htp_backend.html
- Android Dynamic Performance Framework — Thermal API: https://developer.android.com/games/optimize/adpf/thermal
- Android Performance Hint API: https://source.android.com/docs/core/perf/performance-hint-api
- Samsung Galaxy 전력 모드(절전 시 CPU 속도 제한) 안내: https://www.samsung.com/us/support/answer/ANS00079037/
- QAIRT SDK 설치(QPM, 계정 필요) 공식 안내: https://docs.qualcomm.com/bundle/publicresource/topics/80-63442-10/general_overview.html
- Galaxy S24 Ultra SM8650-AC / 일반판 SM8650-AB 모델넘버 확인: https://www.androidauthority.com/snapdragon-8-gen-3-for-galaxy-3404325/
- Galaxy S26 Ultra SM8850 모델넘버 확인: https://www.androidauthority.com/galaxy-s26-ultra-fcc-certificate-snapdragon-8-elite-gen-5-3623566/
- Samsung Knox e-fuse/부트로더 언락과 일반 USB 디버깅의 차이: https://www.thecustomdroid.com/samsung-galaxy-bootloader-unlock-guide/
- GenieX 공식 실행 모드 문서(npu vs hybrid 디스패치, QAIRT 고정 컨텍스트 제약 등 1차 확인): https://github.com/qualcomm/GenieX/blob/main/notes/run.md
- GenieX 트러블슈팅 문서(QAIRT NPU-only 제약 등): https://deepwiki.com/qualcomm/GenieX/8.2-troubleshooting-and-faq
- Qualcomm Snapdragon 8 Elite Gen 5 공식 제품 브리프(Hexagon NPU 구성, Sensing Hub Dual Micro NPU 등): https://www.qualcomm.com/content/dam/qcomm-martech/dm-assets/documents/Snapdragon-8-Elite-Gen-5-product-brief.pdf
- Snapdragon Profiler 150개 카운터/22개 카테고리 언급(Qualcomm 공식): https://developer.samsung.com/galaxy-gamedev/resources/tool-guides/qualcomm.html
