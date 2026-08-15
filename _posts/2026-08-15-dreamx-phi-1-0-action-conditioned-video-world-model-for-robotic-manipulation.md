---
title: "로봇 궤적을 어텐션에 새긴 비디오 월드 모델"
date: 2026-08-15
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "로봇 팔의 SE(3) 궤적을 트랜스포머 어텐션에 직접 새겨 조작 비디오를 생성하는 DreamX-Phi 1.0을 살펴본다."
paper_title: "DreamX-Phi 1.0: Action-Conditioned Video World Model for Robotic Manipulation"
paper_summary: "로봇 팔의 SE(3) 궤적을 트랜스포머 어텐션에 직접 새겨 조작 비디오를 생성하는 DreamX-Phi 1.0을 살펴본다."
paper_url: "https://huggingface.co/papers/2608.13489"
header:
  image: "/assets/images/posts/dreamx-phi-1-0-action-conditioned-video-world-model-for-robotic-manipulation/figure-1.png"
  teaser: "/assets/images/posts/dreamx-phi-1-0-action-conditioned-video-world-model-for-robotic-manipulation/figure-1.png"
---

이번 주 골라온 논문은 HuggingFace Daily Papers 기준 score 1.00으로 이번 주 가장 눈에 띈 로보틱스 비디오 생성 연구입니다. 로봇 조작을 위한 World Model이 최근 정책 학습의 시뮬레이터 대안으로 주목받는 흐름 속에서, "영상은 그럴듯한데 로봇이 시킨 대로 움직이지 않는다"는 오래된 문제를 정면으로 다룬다는 점이 눈에 들어와 이번 주 노트로 정리했습니다.

비디오 생성 모델을 로봇의 World Model로 쓰겠다는 아이디어 자체는 새롭지 않다. 초기 프레임과 언어 지시, 그리고 앞으로 취할 액션을 넣으면 미래 프레임을 예측해주는 모델이 있으면, 실제 로봇을 굴리지 않고도 정책을 검증하거나 학습 데이터를 증강할 수 있다. 문제는 이 예측 영상이 "말이 되는 그림"과 "지시한 대로 움직인 그림" 사이에서 자꾸 후자를 놓친다는 데 있다. 그리퍼가 열어야 할 타이밍에 안 열리거나, 왼팔에 명령을 줬는데 오른팔이 움직이거나, 접촉 순간 물체 형상이 뭉개지는 식이다.

왜 이런 일이 생기는지 파고들면 원인은 액션을 모델에 넣는 방식 자체에 있다. 기존 방법들은 로봇의 엔드이펙터 포즈나 그리퍼 상태를 저차원 토큰으로 임베딩하거나, AdaLN 같은 전역 변조(feature modulation)로 트랜스포머 블록에 흘려보낸다. 이 방식은 "지금 팔이 어디쯤 있다"는 정보는 전달하지만, 팔이 3D 공간에서 강체(rigid body)로 움직인다는 기하학적 구조 자체는 표현하지 못한다. 두 시점 사이의 움직임이 회전과 병진을 포함한 SE(3) 변환이라는 사실이 그냥 숫자 벡터 속에 묻혀버리는 셈이다. 또 하나의 원인은 손실 함수의 시야다. 영상 전체 픽셀에 걸쳐 평균 손실을 계산하면, 화면의 대부분을 차지하는 배경이 학습 신호를 지배하고 정작 중요한 조작 대상 물체나 접촉면은 손실 값에 거의 기여하지 못한다. 결과적으로 모델은 배경을 그리는 데는 능숙해지지만 파지 순간의 물체 형상 변화나 접촉 물리는 대충 넘어간다.

DreamX-Phi 1.0이 고친 방식은 크게 두 갈래다. 하나는 액션을 아예 어텐션 연산 안으로 밀어 넣는 것이다. 각 팔의 포즈를 기준 팔의 초기 좌표계로 정규화하고 최대 이동 거리로 스케일을 맞춘 뒤, 이 상대 변환 행렬을 쿼리·키·밸류에 직접 곱하는 PRoPE 방식을 쓴다. 이렇게 하면 두 토큰 사이의 상호작용이 절대 좌표가 아니라 상대 변환 $$D_i D_j^{-1}$$로 결합되도록 구조 자체가 강제된다. 액션을 "알려주는" 게 아니라 "어텐션의 문법"으로 만들어버린 셈이다. 그리퍼 개폐 같은 스칼라 신호는 별도로 어텐션 헤드 바이어스로 얹는다.

다른 하나는 손실 함수 쪽의 수정이다. Depth Anything 3으로 뽑은 깊이 맵을 잠재 공간에서 맞추는 보조 Depth 브랜치를 RGB 트랜스포머 후반부에 얹어 기하 인식을 보강하고, SAM3로 뽑은 물체 마스크 영역에는 Flow-matching 손실 가중치를 높여 배경에 묻히지 않게 한다. 여기에 더해 Frozen V-JEPA 교사 모델의 시공간 특징과 학생 모델의 Gram 행렬을 맞추는 관계적 손실을 추가해서, 특정 픽셀값을 그대로 베끼기보다 물체의 형상과 움직임 관계 자체를 유지하도록 유도한다. 마지막으로 Flow-matching 특유의 느린 다단계 생성을 DMD2 증류와 적대적 학습으로 눌러서 소수 스텝만으로 추론할 수 있게 만들었다.

![DreamX-Phi 1.0 전체 파이프라인: 초기 프레임, 언어 지시, 양팔 SE(3) 궤적으로부터 충실한 조작 비디오를 생성](/assets/images/posts/dreamx-phi-1-0-action-conditioned-video-world-model-for-robotic-manipulation/figure-1.png)

이 구조가 실제로 세 층위의 개선(기하 조건 주입, 물체 중심 손실 재가중, 후처리 증류)으로 나뉜다는 건 아래 구조도에서 더 뚜렷하게 드러난다.

![DreamX-Phi 1.0 세부 아키텍처: Arm-Grouped PRoPE, 3가지 보조 감독 신호, DMD 기반 Few-step 증류의 3단계 파이프라인](/assets/images/posts/dreamx-phi-1-0-action-conditioned-video-world-model-for-robotic-manipulation/figure-2.png)

정리하면 이 논문의 핵심 주장은 "액션 조건을 벡터로 주지 말고 기하 연산으로 주라"는 것과, "손실을 프레임 전체가 아니라 정말 중요한 영역에 집중시켜라"는 두 가지다. 둘 다 원리적으로는 단순하지만, PRoPE처럼 어텐션 수식 자체를 바꾸는 방식이나 V-JEPA의 Gram 행렬 정렬처럼 명시적 픽셀 대응이 아닌 관계 구조를 학습 신호로 쓰는 부분은 손실 설계의 결이 확실히 다르다. 다만 논문이 공개한 노트만으로는 정량 비교표(액션 충실도 지표, 실제 로봇 정책 학습 성능 등)가 충분히 드러나지 않아서, PRoPE와 물체 중심 손실 각각이 얼마나 기여했는지에 대한 ablation 수치는 원문을 더 들여다볼 필요가 있다.

## 용어 해설

- **SE(3) 그룹**: 3차원 공간에서 회전과 병진을 함께 표현하는 강체 운동의 수학적 집합으로, 로봇 팔의 포즈나 카메라 위치처럼 "회전하면서 동시에 이동하는" 변환을 하나의 4x4 행렬로 표현할 때 쓰인다.
- **Flow-matching**: Diffusion 모델처럼 노이즈에서 데이터로 가는 확률 경로를 학습하는 생성 모델 계열로, 노이즈 제거 대신 데이터 분포를 잇는 벡터장(flow)을 직접 회귀하도록 학습한다.
- **Gram 행렬**: 여러 벡터들 사이의 내적을 모아놓은 행렬로, 개별 값이 아니라 벡터들 간의 상대적 관계(유사도 구조)를 요약해서 표현할 때 쓰인다.
- **Distillation(증류)**: 크고 느린 교사 모델의 출력이나 행동을 모방하도록 작고 빠른 학생 모델을 학습시키는 기법으로, 성능 손실을 최소화하면서 추론 속도를 높이는 데 쓰인다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인적인 감상을 붙이자면, 액션 조건을 어텐션 연산의 대수적 구조로 바꿔 넣는 PRoPE 아이디어는 "조건 주입"을 표현력 문제가 아니라 대칭성(symmetry) 문제로 재정의했다는 점에서 인상적이다.

<details class="tc-faq">
<summary>PRoPE처럼 어텐션에 기하 구조를 강제하는 방식이 양팔 로봇을 넘어 다관절 휴머노이드나 이동형 로봇에도 그대로 확장될 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

팔이 늘어날수록 상대 변환의 조합 수가 커지므로, Arm-Grouped 방식의 계산 비용과 학습 안정성이 관절 수에 따라 어떻게 변하는지가 다음 검증 지점일 것이다.

</div>
</details>

<details class="tc-faq">
<summary>SAM3 마스크와 V-JEPA 관계 손실이 실제로 각각 얼마나 기여하는지 ablation 없이는 판단하기 어려운데, 어느 쪽이 물체 형상 유지에 더 결정적일까?</summary>
<div class="tc-faq__body" markdown="1">

마스크 가중치는 "어디를 볼지"를 정하고 V-JEPA 손실은 "무엇을 볼지"를 정하는 역할이라 상호 보완적일 가능성이 높지만, 문서만으로는 우열을 가리기 어렵다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 작업(task) 단위로 로봇 행동을 계획하는 프로젝트라면 이런 World Model을 정책 검증이 아니라 데이터 증강 파이프라인으로 쓸 여지가 있을까?</summary>
<div class="tc-faq__body" markdown="1">

액션 충실도가 보장된 롤아웃이라면 실제 로봇 없이도 실패 케이스나 희귀 조작 시나리오를 합성 데이터로 만들어 taskcraft의 task 분해 학습에 붙여볼 만하다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.13489" target="_blank" rel="noopener">DreamX-Phi 1.0: Action-Conditioned Video World Model for Robotic Manipulation</a> · hf-daily</span></div>
</div>
</div>
