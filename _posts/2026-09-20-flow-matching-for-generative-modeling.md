---
title: "디퓨전을 이긴 직선 경로, 플로우 매칭"
date: 2026-09-20
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "곡선을 그리며 되돌아가는 디퓨전 궤적을 최적 수송 직선으로 바꿔 훨씬 적은 스텝으로 생성하는 방법을 정리했다"
paper_title: "Flow Matching for Generative Modeling"
paper_summary: "곡선을 그리며 되돌아가는 디퓨전 궤적을 최적 수송 직선으로 바꿔 훨씬 적은 스텝으로 생성하는 방법을 정리했다"
paper_url: "https://arxiv.org/abs/2210.02747v2"
header:
  image: "/assets/images/posts/flow-matching-for-generative-modeling/figure-1.png"
  teaser: "/assets/images/posts/flow-matching-for-generative-modeling/figure-1.png"
---

이번 주 논문 노트는 score 1.00으로 가장 높은 우선순위에 오른 arXiv 논문, Lipman 등의 Flow Matching for Generative Modeling입니다. 최근 VLA 계열 로봇 정책들이 앞다투어 diffusion 대신 flow matching을 백본으로 갈아타는 흐름이 눈에 띄는데, 정작 그 출발점이 된 이 논문을 제대로 짚고 넘어가지 않으면 이후 흐름을 따라가기 어렵겠다 싶어 이번 주 첫 논문으로 골랐습니다.

## 생성 모델이 이미지를 만드는 두 가지 방식

생성 모델의 목표는 간단하다. 가우시안 노이즈처럼 단순한 분포에서 시작해서, 실제 데이터(사진, 행동 궤적 등)와 똑같이 생긴 복잡한 분포로 점을 옮기는 변환을 학습하는 것이다.

정규화 흐름(Normalizing Flow)은 이 변환을 여러 개의 가역적인 레이어로 쌓아서 만든다. 연속 정규화 흐름(Continuous Normalizing Flow, CNF)은 이 레이어를 이산적으로 쌓는 대신, 시간 $$t$$에 따라 점이 흘러가는 속도를 나타내는 벡터장 $$u_t(x)$$를 정의하고 이를 상미분방정식(ODE)으로 적분해서 $$t=0$$의 노이즈를 $$t=1$$의 데이터로 옮긴다. 문제는 이 벡터장을 최대우도로 학습하려면 순전파, 역전파 각각에서 ODE를 수치적으로 풀어야 한다는 점이다. 고해상도 이미지처럼 차원이 커지면 이 시뮬레이션 비용이 감당이 안 될 정도로 커진다.

디퓨전 모델은 이 문제를 다르게 우회했다. 데이터에 점진적으로 노이즈를 섞어가는 forward 과정을 미리 수식으로 고정해 두면, 임의의 시점 $$t$$에서의 분포가 닫힌 형태로 계산되기 때문에 시뮬레이션 없이(simulation-free) 잡음 제거 스코어 매칭(denoising score matching)만으로 학습할 수 있다. 그런데 이 forward 과정 자체가 확률미분방정식(SDE)으로 정의되어 있다 보니, 거꾸로 샘플링할 때의 궤적이 심하게 휘어지고 심지어 목적지를 지나쳤다가 되돌아오는 backtracking 현상까지 일어난다. 그래서 이미지 하나를 생성하려면 신경망을 수백 번씩 평가(NFE, Number of Function Evaluations)해야 했다.

## 핵심 아이디어: 벡터장을 시뮬레이션 없이 직접 회귀하기

이 논문의 발상은 단순하지만 강력하다. 디퓨전이 정의한 특정 SDE에 얽매이지 말고, 노이즈 분포와 데이터 분포를 잇는 임의의 확률 경로 $$p_t(x)$$를 먼저 정하고, 그 경로를 만드는 벡터장 $$u_t(x)$$를 신경망 $$v_t(x;\theta)$$로 직접 회귀하자는 것이다(Flow Matching, FM 목적함수).

문제는 전체 데이터에 대한 이 주변(marginal) 벡터장 $$u_t(x)$$는 적분이 불가능해서 계산할 수 없다는 점이다. 논문은 여기서 트릭을 쓴다. 데이터 포인트 하나 $$x_1$$을 고정하고 그 점에 대한 조건부 확률 경로 $$p_t(x \vert x_1)$$과 조건부 벡터장 $$u_t(x \vert x_1)$$을 정의하면, 이건 간단한 가우시안이라 닫힌 형태로 쉽게 계산된다. 그리고 이 조건부 것들을 데이터 분포에 대해 평균 내면 정확히 원래의 주변 확률 경로와 벡터장이 복원된다는 것을 증명했고(Theorem 1), 더 나아가 조건부 벡터장만으로 손실을 계산하는 조건부 플로우 매칭(Conditional Flow Matching, CFM)의 그래디언트가 원래 FM 목적함수의 그래디언트와 정확히 일치한다는 것도 증명했다(Theorem 2). 결과적으로 미니배치에서 데이터 $$x_1$$과 노이즈 $$x_0$$을 샘플링해서 그 사이 어딘가의 점 $$x$$를 만들고, 신경망이 그 점에서의 조건부 목표 속도를 예측하도록 MSE로 회귀하면 끝이다. 디퓨전의 denoising score matching과 구조적으로 매우 닮았지만, 예측 대상이 score(로그밀도의 그래디언트)가 아니라 velocity(속도 벡터장)라는 점이 다르다.

## 경로를 어떻게 그리느냐가 성능을 가른다

여기서 진짜 승부처가 나온다. 디퓨전이 쓰는 분산 보존(VP), 분산 발산(VE) 경로도 사실 가우시안 조건부 경로의 한 특수 사례로 재해석할 수 있다. 그런데 이 논문은 대신 최적 수송(Optimal Transport, OT) 기반의 변위 보간을 쓰자고 제안한다. 조건부 경로의 평균과 표준편차를 시간 $$t$$에 대해 선형으로 잇는 것이다.

$$\mu_t(x_1) = t x_1, \quad \sigma_t(x_1) = 1 - (1 - \sigma_{\min})t$$

이렇게 정의하면 노이즈 점 $$x_0$$과 데이터 점 $$x_1$$을 잇는 경로가 완전한 직선이 되고, 목표 속도 벡터장은 시간에 관계없이 항상 상수다.

$$v_t(\psi_t(x_0)) \approx x_1 - (1 - \sigma_{\min}) x_0$$

직선이라는 게 왜 중요한가 하면, 신경망이 회귀해야 할 타깃이 시간에 따라 방향을 바꾸지 않고 일정하기 때문에 학습이 훨씬 매끈해지고, 무엇보다 추론 단계에서 ODE solver가 궤적을 촘촘히 따라갈 필요 없이 큰 스텝으로 건너뛰어도 오차가 크게 쌓이지 않는다.

![디퓨전 경로와 최적 수송 경로의 궤적 비교](/assets/images/posts/flow-matching-for-generative-modeling/figure-1.png)

이 그림에서 디퓨전 경로는 목적지 근처에서 방향을 틀거나 지나쳤다가 되돌아오는 곡선을 그리는 반면, OT 경로는 시작점과 끝점을 최단 거리로 잇는 직선을 그린다. 같은 목적지에 도착하더라도 이동 거리와 궤적의 복잡도 자체가 다르다는 뜻이다.

## 결과: 스텝 수를 줄여도 무너지지 않는다

ImageNet $$32 \times 32$$ 실험에서 NFE(신경망 평가 횟수)를 줄여가며 FID(생성 품질 지표)를 비교한 결과가 이 차이를 숫자로 보여준다.

![NFE에 따른 FID 비교, FM-OT가 적은 스텝에서도 안정적임을 보여줌](/assets/images/posts/flow-matching-for-generative-modeling/figure-2.png)

Score matching이나 디퓨전 기반 flow matching은 NFE가 40 이하로 내려가면 FID가 급격히 치솟으며 품질이 무너진다. 반면 OT 기반 flow matching은 NFE 20 이하의 극단적으로 적은 스텝에서도 FID가 크게 흔들리지 않는다. 같은 오차 수준에 도달하는 데 필요한 NFE도 디퓨전 대비 약 60퍼센트 수준으로 줄었다. 이는 단순히 속도가 빨라졌다는 것을 넘어, 궤적이 곧다는 성질이 수치 적분의 안정성 자체를 바꿔놓는다는 것을 보여준다. 게다가 CNF 계열의 장점인 정확한 우도(likelihood) 계산도 그대로 유지되어, 이미지 생성 품질과 밀도 추정 성능 양쪽에서 경쟁력 있는 수치를 냈다.

## 한계

이 논문이 모든 걸 해결한 건 아니다. 여전히 연속시간 ODE를 푸는 틀이기 때문에 NFE를 1까지 줄이는 완전한 one-step 생성은 아니다. 이 지점은 이후 rectified flow, consistency model 계열 연구들이 이어받아 공략했다. 또한 조건부 경로를 가우시안으로 한정하고 노이즈와 데이터를 독립적으로(independent coupling) 짝지었기 때문에, 진짜 최적 수송이 주는 이론적 최단 경로와는 다소 차이가 있다. 이후 미니배치 단위로 실제 OT coupling을 찾아 짝짓는 후속 연구들이 이 간극을 메꾸려 했다. 마지막으로 이 논문은 철저히 이미지 생성에 초점을 맞추고 있어서, 로봇 행동처럼 시각 관측에 조건화된 저차원 궤적 생성에도 같은 이점이 그대로 이어지는지는 이 논문만으로는 알 수 없고, 이후 flow matching 기반 로봇 정책 연구들이 실증했다.

## 용어 해설

- **상미분방정식(ODE)**: 어떤 값의 시간에 대한 변화율(미분)을 정의하는 방정식으로, 초기값이 주어지면 그 값이 시간에 따라 어떻게 변해가는지를 적분해서 구할 수 있다.
- **벡터장(vector field)**: 공간의 각 점마다 방향과 크기를 가진 화살표(속도)를 대응시키는 함수로, 입자가 그 위치에서 어느 방향으로 얼마나 빠르게 움직여야 하는지를 알려준다.
- **최적 수송(Optimal Transport)**: 한 확률 분포의 질량을 다른 확률 분포로 옮길 때 드는 이동 비용을 최소화하는 방식을 찾는 수학 이론으로, 여기서는 노이즈 분포를 데이터 분포로 옮기는 가장 짧은 경로를 정의하는 데 쓰였다.
- **NFE(Number of Function Evaluations)**: 하나의 샘플을 생성하기 위해 신경망을 몇 번 호출했는지를 세는 지표로, 생성 모델의 추론 속도를 비교하는 대표적인 기준이다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인적인 인상을 남기자면, 이 논문은 생성 모델의 목적함수 문제를 경로 설계 문제로 환원했다는 점이 굉장히 깔끔하게 느껴진다. 디퓨전이라는 특정 SDE에 종속되지 않고 아무 확률 경로나 회귀 대상으로 삼을 수 있다는 발상이, 이후 rectified flow나 consistency model 같은 후속 연구들이 쏟아지게 만든 씨앗이었을 것 같다.

<details class="tc-faq">
<summary>조건부 경로를 가우시안이 아닌 다른 형태로 잡으면 학습이나 샘플링 품질이 어떻게 달라질까</summary>
<div class="tc-faq__body" markdown="1">

논문 스스로도 가우시안은 다루기 쉬워서 선택한 것일 뿐이라 밝히고 있어서, 이산 데이터나 매니폴드 데이터에 맞는 다른 조건부 경로를 설계하는 게 흥미로운 후속 질문이 될 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 Latent World Model에서 다음 latent state의 분포를 예측할 때 diffusion 대신 flow matching을 쓰면 실제로 이득이 있을까</summary>
<div class="tc-faq__body" markdown="1">

OT 경로가 주는 안정성과 적은 NFE는 embodiment 이식 파이프라인의 추론 속도를 실질적으로 낮출 잠재력이 있어

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2210.02747v2" target="_blank" rel="noopener">Flow Matching for Generative Modeling</a> · arxiv</span></div>
</div>
</div>
