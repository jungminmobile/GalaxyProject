# 스마트폰 AI 관련 논문 목록

작성일: 2026-08-06 / EXPERIMENT_PLAN.md 13~22장 작업 중 웹 검색으로 찾은, 스마트폰 온디바이스 AI 관련 논문을 주제별로 정리. **대부분 원문 전체를 정독하지 않고 검색 결과 요약 기반으로 정리함 — 실제 인용·근거로 쓰기 전 원문 확인 필요.**

---

## 1. LLM 온디바이스 추론 성능 특성화(characterization)

1. **LLM Inference at the Edge: Mobile, NPU, and GPU Performance Efficiency Trade-offs Under Sustained Load** (arXiv 2603.23640, 2026) — Galaxy S24 Ultra 등 4개 플랫폼에서 지속 부하 시 처리량·온도·배터리 실측. 이 프로젝트 실험 프로토콜(10분 열안정화→20회 반복→CSV 기록)의 직접적 근거 논문. https://arxiv.org/html/2603.23640v2
2. **When NPUs Are Not Always Faster: A Stage-Level Analysis of Mobile LLM Inference** (arXiv 2605.27435, 2026) — Snapdragon 8 Gen 3에서 prefill=CPU 유리, decode=NPU만 제한적 이득. https://arxiv.org/pdf/2605.27435
3. **Is Your NPU Ready for LLMs? Dissecting the Hidden Efficiency Bottlenecks in Mobile LLM Inference** (arXiv 2607.05475, 2026) — 5개 프레임워크·3개 백엔드 비교, phase-aware(단계별) 스케줄링·백엔드 선택 제안. https://arxiv.org/html/2607.05475v1
4. **Fast On-device LLM Inference with NPUs** (ASPLOS 2025, arXiv 2407.05858) — 모바일 GPU가 NPU보다 다른 앱과 자원경합이 크다는 관찰. https://arxiv.org/pdf/2407.05858
5. **Quant.npu: Enabling Efficient Mobile NPU Inference for on-device LLMs via Fully Static Quantization** (arXiv 2605.20295, 2026) — 완전 정적 양자화로 NPU 추론 효율화. https://arxiv.org/pdf/2605.20295
6. **Scaling LLM Test-Time Compute with Mobile NPU on Smartphones** (EuroSys 2026) — 모바일 NPU에서 test-time compute 확장.
7. **Energy-Efficient On-Device RAG on a Mobile NPU: System Design and Benchmark on Snapdragon X Elite** (arXiv 2606.11257, 2026) — 모바일 NPU 기반 RAG 시스템·벤치마크. https://arxiv.org/pdf/2606.11257
8. **Efficient On-Device Diffusion LLM Inference with Mobile NPU** (arXiv 2606.13740, 2026) — 디퓨전 기반 LLM의 모바일 NPU 추론. https://arxiv.org/html/2606.13740
9. **EnerInfer: Energy-Aware On-Device LLM Inference** (arXiv 2606.23001, 2026) — 에너지 인지형 온디바이스 LLM 추론. https://arxiv.org/pdf/2606.23001
10. **Accelerating Mobile Language Model via Speculative Decoding and NPU-Coordinated Execution** (arXiv 2510.15312, 2025) — 스펙큘레이티브 디코딩+NPU 협조 실행. https://arxiv.org/pdf/2510.15312

---

## 2. NPU/GPU/CPU 자원경합·스케줄링·공정성 — RQ7과 직결

11. **Taming Memory Bandwidth Contention in Mobile LLM Inference** (OSDI 2026) — **가장 중요**: 모바일 OS가 NPU 트래픽을 관측·제어할 수단이 없어 OS 스케줄러 입장에서 NPU가 "보이지 않는" 존재라고 명시. RQ7-2(삼성 우대 여부) 메커니즘 가설을 다듬는 근거. https://www.usenix.org/system/files/osdi26-xin.pdf
12. **Characterizing Mobile SoC for Accelerating Heterogeneous LLM Inference** (HeteroInfer, arXiv 2501.14794, 2025) — GPU+NPU 이기종 병렬 실행. https://arxiv.org/abs/2501.14794
13. **Puzzle: Scheduling Multiple Deep Learning Models on Mobile Device with Heterogeneous Processors** (arXiv 2508.17764, 2025) — Galaxy S23 Ultra 실기로 CPU/GPU/NPU 걸친 다중 DNN 동시 스케줄링. https://arxiv.org/abs/2508.17764
14. **ShadowNPU: System and Algorithm Co-design for NPU-Centric On-Device LLM Inference** (MobiSys 2026, arXiv 2508.16703) — 희소 어텐션으로 CPU/GPU 폴백 없이 NPU에 상주해 다른 앱과의 자원경합 회피. 이 프로젝트 14장(적응형 디스패처)과 반대 철학의 대조군. https://arxiv.org/html/2508.16703v3
15. **V10: Hardware-Assisted NPU Multi-tenancy for Improved Utilization** (ISCA 2023) — NPU 멀티테넌시 공정성 지표 + 오퍼레이터 단위 선점. https://jianh.web.engr.illinois.edu/papers/v10-isca23.pdf
16. **XSched: Preemptive Scheduling for Diverse XPUs** — GPU/NPU/ASIC/FPGA 등에 선점형 스케줄링(우선순위·공정성) 부여. https://dl.acm.org/doi/10.5555/3767901.3767938
17. **iAware: Interaction Aware Task Scheduling for Reducing Resource Contention in Mobile Systems** (ACM TECS) — 유휴 구간을 활용한 상호작용 인지형 스케줄링. https://dl.acm.org/doi/10.1145/3609391
18. **ApproxDet: Content and Contention-Aware Approximate Object Detection for Mobiles** (arXiv 2010.10754) — 콘텐츠·경합 인지형 근사 객체탐지. https://arxiv.org/pdf/2010.10754
19. **Agent.xpu: Efficient Scheduling of Agentic LLM Workloads on Heterogeneous SoC** (arXiv 2506.24045, 2025) — 에이전틱 LLM 워크로드의 이기종 SoC 스케줄링. https://arxiv.org/html/2506.24045

---

## 3. 적응형 컴파일러 / 멀티버전 커널

20. **VELTAIR: Towards High-Performance Multi-tenant Deep Learning Services via Adaptive Compilation and Scheduling** (ASPLOS 2022, arXiv 2201.06212) — 레이어별 다중 버전 컴파일 + 적응형 스케줄러(CPU 서버 대상). 이 세션에서 이 프로젝트로의 이식 가능성을 검토한 원 논문(EXPERIMENT_PLAN.md 10-A-3 참고). https://arxiv.org/abs/2201.06212
21. **Miriam: Exploiting Elastic Kernels for Real-time Multi-DNN Inference on Edge GPU** (arXiv 2307.04339, 2023) — 엣지 GPU에서 다중 커널 버전 + 런타임 선택. https://arxiv.org/pdf/2307.04339

---

## 4. Vision-Language Model(VLM) 온디바이스

22. **MobileVLM: A Fast, Strong and Open Vision Language Assistant for Mobile Devices** (arXiv 2312.16886) — Snapdragon 888 CPU에서 21.5 tok/s 달성한 경량 VLM(1.4B/2.7B). https://arxiv.org/abs/2312.16886
23. **MobileVLM V2: Efficient Vision-Language Model** (arXiv 2402.03766) — MobileVLM 개선판.
24. **MagicVL-2B: Empowering Vision-Language Models on Mobile Devices with Lightweight Visual Encoders via Curriculum Learning** (arXiv 2508.01540, 2026-08) — 경량 비전 인코더 기반 최신 모바일 VLM. https://arxiv.org/pdf/2508.01540
25. **Efficient Deployment of Vision-Language Models on Mobile Devices: A Case Study on OnePlus 13R** (arXiv 2507.08505, 2026) — **이 프로젝트와 성격이 가장 비슷한 선행연구**: 실제 상용 스마트폰 한 대에 VLM을 배포한 사례 연구. https://arxiv.org/pdf/2507.08505
26. **ARIA: Optimizing Vision Foundation Model Inference on Heterogeneous Mobile Processors** (MobiSys 2025) — 이기종 모바일 프로세서 위 비전 파운데이션 모델 추론 최적화. 이 프로젝트 20-2(M축, 다중모델 동시실행)와 밀접.
27. **CoVSpec: Efficient Device-Edge Co-Inference for Vision-Language Models via Speculative Decoding** (arXiv 2605.02218, 2026) — 디바이스-엣지 협업 추론(스펙큘레이티브 디코딩)으로 VLM 가속. https://arxiv.org/pdf/2605.02218
28. **FastVLM: Efficient Vision Encoding for Vision Language Models** (Apple, CVPR 2025) — 고해상도 이미지 대응 하이브리드 비전 인코더. https://machinelearning.apple.com/research/fast-vision-language-models

---

## 5. 프레임워크 / 벤치마크 인프라

29. **ExecuTorch -- A Unified PyTorch Solution to Run AI Models On-Device** (arXiv 2605.08195, 2026) — Meta의 온디바이스 추론 런타임, Qualcomm QNN 백엔드 포함. 이 프로젝트 16·21장에서 GenieX 대안으로 검토. https://arxiv.org/pdf/2605.08195
30. **MLPerf Mobile Inference Benchmark** (arXiv 2012.02328) — 모바일 추론 표준 벤치마크. https://arxiv.org/pdf/2012.02328
31. **Neural Network Inference on Mobile SoCs** (arXiv 1908.11450) — 모바일 SoC 신경망 추론 개관. https://arxiv.org/pdf/1908.11450

---

## 6. 서베이 / 배경 자료

32. **Empowering Edge Intelligence: A Comprehensive Survey on On-Device AI Models** (arXiv 2503.06027, 2025) — 온디바이스 AI 모델 전반 서베이. 관련연구 절 초안에 활용 가능. https://arxiv.org/html/2503.06027v1

배경 통계(서베이·업계 자료 기반): 모바일 메모리 대역폭은 약 50~90GB/s로 데이터센터 GPU(2~3TB/s) 대비 30~50배 낮음 — BLG/Nx 실험에서 "메모리 대역폭 경합"이 왜 유력한 병목 후보인지 설명할 때 인용 가능.

---

## 7. 프라이버시 / 연합학습(연관도 낮음, 참고용)

33. **FLSys: Toward an Open Ecosystem for Federated Learning Mobile Apps** (arXiv 2111.09445) — 모바일 연합학습 생태계. 이 프로젝트는 연합학습을 다루지 않지만, "온디바이스=프라이버시"라는 삼성 마케팅 서사의 더 넓은 학계 맥락으로 인용 가능. https://arxiv.org/pdf/2111.09445

---

## 8. 큐레이션 목록(논문 아님, 추가 탐색용)

- **Awesome-On-Device-AI-Inference** (GitHub, iamseonghoon) — https://github.com/iamseonghoon/Awesome-On-Device-AI-Inference
- **Awesome-On-Device-AI-Systems** (GitHub, jeho-lee) — https://github.com/jeho-lee/Awesome-On-Device-AI-Systems

---

## 참고

이 목록은 EXPERIMENT_PLAN.md 16장·22장에서 이미 다룬 내용을 포함해 한 문서로 통합한 것. 새로운 실험 아이디어나 리스크 판단의 근거로 쓸 때는 EXPERIMENT_PLAN.md 해당 장(10-A-3, 16, 22 등)을 함께 참고할 것.
