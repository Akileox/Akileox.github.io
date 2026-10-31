---
title: "실행 전에 미리 시뮬레이션하는 로봇 정책"
date: 2026-10-31
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "Robot-GST는 로봇이 행동하기 전에 3D Gaussian Splatting 장면에서 궤적을 먼저 검증해 실패를 걸러낸다."
paper_title: "Robot-GST: geometry-aware spatial-temporal robot policy representation and evaluation"
paper_summary: "Robot-GST는 로봇이 행동하기 전에 3D Gaussian Splatting 장면에서 궤적을 먼저 검증해 실패를 걸러낸다."
paper_url: "https://www.semanticscholar.org/paper/0da093ca751fa43c5b9b42cf9e4f6b5d4497a813"
header:
  image: "/assets/images/posts/robot-gst-geometry-aware-spatial-temporal-robot-policy-representation-and-evalua/figure-1.png"
  teaser: "/assets/images/posts/robot-gst-geometry-aware-spatial-temporal-robot-policy-representation-and-evalua/figure-1.png"
---

이번 주 논문은 semantic-scholar-rec 경로로 추천되어 스코어 0.85를 받은 글입니다. 로봇 조작 정책이 실패하는 이유를 "학습 데이터 부족"이 아니라 "실행 전 검증 부재"로 다시 정의했다는 점이 눈에 띄어서, 비전-언어 기반 매니퓰레이션과 Real-to-Sim을 함께 다루는 연구 흐름을 짚어보고 싶어 이번 주 노트로 골랐습니다.

## 문제 제기: 왜 로봇 정책은 장기 과업에서 자꾸 틀어질까

비전-언어 기반 매니퓰레이션 정책들은 대체로 "지금 이 이미지를 보고 다음 행동을 뭘 할지"만 예측한다. 그런데 바구니에 인형을 담거나 변형 가능한 천을 정리하는 것처럼 여러 단계를 거치는 장기 과업(long-horizon task)에서는 중간 단계의 작은 오차가 다음 단계로 그대로 전달되고 누적된다. 행동을 실행한 뒤 물체가 실제로 어떤 상태가 될지 예측하는 명시적 메커니즘이 없기 때문이다. ReKep, MOKA, SoFar 같은 키포인트-어포던스 기반 방법들은 이 문제를 어느 정도 완화하지만, 이들이 쓰는 기하 정보가 희소한 키포인트 수준이라 가림이나 변형, 복잡한 접촉이 끼면 금방 무너진다. 또한 물체의 최종 상태를 명시적으로 모델링하지 않아 충돌 회피 계획이나 접촉이 많은 변형체 조작에 한계가 있다. 결국 정책을 평가하려면 실제 로봇으로 직접 돌려봐야 하는데, 이건 비용이 크고 반복하기 어렵다.

## 왜 기존 방법으로는 안 됐는가

핵심은 두 가지가 한 번에 해결된 적이 없다는 점이다. 하나는 "물체의 최종 상태를 충분히 조밀한 기하로 표현하는 것", 다른 하나는 "그 표현 위에서 행동을 실행하기 전에 검증하는 것"이다. 키포인트 기반 방법은 전자가 약하고, 시뮬레이션 기반 평가가 없는 정책들은 후자가 아예 없다. 두 축이 따로 놀다 보니, 변형체나 접촉이 많은 조작에서는 계획 단계의 추론과 실행 단계의 물리가 어긋나는 상황을 사전에 잡아낼 방법이 없었다.

## 고친 방법: Real2Sim 환경에서 궤적을 먼저 돌려본다

Robot-GST가 택한 해법은 크게 네 단계로 이어진다.

먼저 멀티뷰 RGB-D 스캔으로부터 3D Gaussian Splatting(3DGS) 장면을 재구성한다. 각 Gaussian은 위치, 방향, 공분산, 색상, 불투명도뿐 아니라 SAM3D와 DINOv2, CLIP에서 가져온 의미 특징 벡터까지 들고 있어서, 언어로 "이 컵을 잡아" 같은 지시가 들어오면 해당 Gaussian 집합을 바로 분할해낼 수 있다. 동시에 SAM3D로 객체별 메시도 뽑아두는데, 이건 사실적 렌더링용 Gaussian과는 별도로 충돌 검사에 쓸 명시적 기하가 필요했기 때문이다.

이 장면 위에서 GPT-5 같은 VLM에게 과업을 주면 3D 키포인트와 거리·정렬·상대자세 제약을 생성하고, 과업을 여러 스테이지로 쪼개 각 스테이지마다 최적화 문제를 푼다. 물체 상태는 변환과 회전으로 표현하고, 파지 이후에는 엔드이펙터와 물체의 상대 변환이 고정된다고 가정한다.

여기서 핵심 아이디어가 나온다. 목표 배치 포즈 주변에 위치와 회전 섭동을 균등분포로 뿌려 여러 후보 해제 포즈를 만들고, 각 후보를 SDF(Signed Distance Function) 기반 충돌 검사로 거른 뒤 안정성 휴리스틱으로 순위를 매긴다. 재구성 과정에서 생기는 노이즈나 가림, 좁은 수용 공간 때문에 명목상 완벽한 포즈 하나만 믿기보다는 여러 후보를 만들어두는 전략이다.

마지막으로 이 후보 궤적들을 실제 로봇에 바로 쏘는 게 아니라, 재구성된 Gaussian 환경과 메시/TSDF/SDF 프록시를 결합한 시뮬레이터에서 먼저 굴려본다. 강체는 강체동역학으로, 변형체는 PhysTwin 기반 연체 모델로 전이를 예측하고, 모든 시점에서 역운동학(IK) 해가 존재하는지, 충돌 마진을 지키는지, 종료 상태가 목표 술어를 만족하는지를 하나의 지시함수로 판정한다. 이 판정을 통과한 궤적만 실제 로봇에 배포되고, 실패하면 장면을 리셋하고 재계획하는 루프를 돈다.

![Robot-GST 전체 파이프라인 개요](/assets/images/posts/robot-gst-geometry-aware-spatial-temporal-robot-policy-representation-and-evalua/figure-1.png)

전체 그림을 보면 관측에서 시작해 Gaussian 환경으로 재구성하고, 그 안에서 추론과 샘플링을 거쳐 성공 여부를 판단한 뒤에야 실제 환경으로 전이하는 폐루프 구조가 분명히 드러난다. 강체, 연체, 변형체 세 가지 과업 썸네일을 함께 제시한 것도 이 프레임워크가 특정 물체 카테고리에 국한되지 않는다는 주장을 뒷받침하기 위한 장치로 보인다.

![기하 인지 시공간 계획 및 정책 평가 과정](/assets/images/posts/robot-gst-geometry-aware-spatial-temporal-robot-policy-representation-and-evalua/figure-2.png)

슬로스 인형을 바구니에 담는 구체적 사례를 보여주는 두 번째 그림에서는, 손목 카메라 이미지가 SAM으로 분할되고 키포인트 어포던스가 추출된 뒤 GPT-5가 Grasp, Move, Release 세 단계로 과업을 쪼개는 과정과, 그렇게 만들어진 파지·배치 궤적이 PhysTwin 물리 제약과 함께 3DGS 시뮬레이터에서 검증되는 과정이 함께 나온다. 언어 지시에서 출발해 실제 궤적 생성과 검증까지 이어지는 흐름이 한 장에 압축되어 있다.

## 결과와 한계

논문은 Table II와 Fig. 7에서 시뮬레이션 기반 재계획 루프가 있을 때와 없을 때의 성공률 차이를 보여주며, 실패한 궤적을 사전에 걸러내고 재계획하는 것만으로 강체, 연체, 변형체 전 범주에서 신뢰성이 올라갔다고 보고한다. 다만 이 접근의 전제는 명확하다. 3DGS 재구성 품질이 떨어지면 충돌 검사나 상태 추정 자체가 흔들리고, 변형체 모델링에 쓰는 PhysTwin 같은 물리 근사도 실제 천이나 끈 같은 복잡한 변형을 완벽히 재현하지는 못한다. 또한 매 스테이지마다 후보를 여러 개 샘플링하고 물리 시뮬레이션을 돌리는 과정 자체가 추가 연산 비용이라, 실시간성이 중요한 과업에서는 이 "실행 전 검증" 루프가 병목이 될 수 있다. 결국 오차 누적 문제를 정책 학습 쪽이 아니라 추론 단계의 검증 문제로 옮겨놓은 것인데, 그만큼 검증 단계의 시뮬레이션 충실도에 전체 시스템의 신뢰성이 강하게 의존하게 된다.

## 용어 해설

- **3D Gaussian Splatting**: 장면을 수많은 3차원 가우시안 분포(위치, 모양, 색, 투명도를 가진 작은 타원체)의 집합으로 표현하고, 이를 2D 화면에 투영·합성해 실시간으로 사실적인 렌더링을 만드는 기법이다.
- **SDF (Signed Distance Function)**: 공간의 각 점에서 가장 가까운 물체 표면까지의 거리를 부호와 함께 나타내는 함수로, 물체 내부면 음수, 외부면 양수 값을 가져 충돌 검사나 형상 표현에 널리 쓰인다.
- **역운동학(IK, Inverse Kinematics)**: 로봇 팔의 끝단(엔드이펙터)이 원하는 위치와 자세에 도달하도록 각 관절의 각도를 거꾸로 계산하는 문제를 말한다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인적인 감상을 덧붙이면, 이 논문은 "더 좋은 정책을 학습하자"가 아니라 "학습된 정책의 출력을 실행 전에 검증하자"는 방향 전환이 인상적이다. 모델 성능 자체를 올리는 대신 안전망을 추론 단계에 붙인 설계라서, 비슷한 아이디어를 다른 도메인에도 적용해볼 여지가 있어 보인다.

더 생각해볼 것

<details class="tc-faq">
<summary>3DGS 재구성 품질이 떨어지는 가정(예: 반사가 심한 물체, 투명 용기)에서는 이 전체 파이프라인이 어떻게 무너질까?</summary>
<div class="tc-faq__body" markdown="1">

충돌 검사와 상태 추정이 모두 재구성된 기하에 의존하므로, 반사·투명 물체처럼 3DGS가 취약한 케이스에서는 실행 전 검증 자체가 틀린 판정을 내릴 위험이 커 보인다.

</div>
</details>

<details class="tc-faq">
<summary>매 스테이지마다 후보 샘플링과 물리 롤아웃을 반복하는 비용은 실제로 얼마나 될까, 실시간 조작에는 적용 가능할까?</summary>
<div class="tc-faq__body" markdown="1">

논문이 성공률 향상은 보여주지만 레이턴시 수치를 강조하지 않는 걸 보면, 지금 형태로는 느린 준비 단계를 감수할 수 있는 과업에 더 적합할 가능성이 높다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 LLM이 하위 과업을 분해하는 프로젝트에도 "실행 전 시뮬레이션 검증" 아이디어를 가져올 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

taskcraft가 생성한 서브태스크 계획을 실제로 실행하기 전에 가벼운 시뮬레이션이나 규칙 기반 검증으로 한 번 걸러내는 단계를 끼워 넣으면, 이 논문처럼 실패를 조기에 차단하는 효과를 기대해볼 만하다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/0da093ca751fa43c5b9b42cf9e4f6b5d4497a813" target="_blank" rel="noopener">Robot-GST: geometry-aware spatial-temporal robot policy representation and evaluation</a> · semantic-scholar-rec</span></div>
</div>
</div>
