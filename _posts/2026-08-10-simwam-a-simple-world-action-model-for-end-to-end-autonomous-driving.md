---
title: "상상은 학습할 때만, 실전에선 바로 행동하는 자율주행 모델 SimWAM"
date: 2026-08-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
excerpt: "비디오 생성 월드모델을 추론에서 떼어내고 학습 신호로만 쓴 SimWAM의 지연시간 절감 전략을 뜯어본다"
---

## 왜 이 논문인가

이번 주 HuggingFace Daily Papers에서 스코어 0.76으로 뽑힌 논문이다. 자율주행 월드 모델 계열은 최근 "미래를 생성해서 계획한다"는 방향으로 몰려가는 중인데, 그 대가로 지연 시간이 초 단위까지 늘어나는 게 실제 배포 관점에서는 치명적인 문제다. SimWAM은 이 트레이드오프를 정면으로 건드리면서 "비디오 생성은 학습 때만 쓰고 추론에선 버린다"는 단순하지만 효과적인 해법을 제시한다. 구조가 명확하고 taskcraft가 다루는 latent world model 인터페이스 분리 문제와도 맞닿는 지점이 있어서 이번 주 노트로 골랐다.

## 문제 제기: 상상하고 나서 움직이면 너무 느리다

DriveLaW, DriveWAM 같은 기존 월드-액션 모델은 "Imagine-then-Act" 구조를 쓴다. 현재 관측을 받아 먼저 미래 장면(비디오)을 생성하고, 그 상상된 미래를 바탕으로 주행 경로를 뽑는 방식이다. 논리적으로는 그럴듯하다. 미래를 예측할 수 있다면 그 예측 위에서 계획을 세우는 게 더 안전할 테니까. 문제는 이 비디오 생성 모듈이 무겁다는 점이다. 프레임 단위로 diffusion이나 flow matching을 여러 스텝 돌려야 하니, 추론 루프 안에 비디오 생성이 들어가는 순간 지연 시간이 1.5초에서 3초 이상으로 늘어난다. 자율주행처럼 매 순간 반응해야 하는 태스크에서 이건 실제로 못 쓴다는 뜻이나 다름없다.

또 다른 축의 문제는 학습 방식 자체에 있다. 전문가 궤적을 그대로 따라가는 모방 학습만으로는 도로 준수나 충돌 회피 같은 복합적인 주행 품질을 직접 최적화하기 어렵다. 궤적을 베끼는 것과 안전하게 운전하는 것 사이에는 거리가 있다.

## 왜 안 됐는지: 비디오 생성과 액션 예측이 한 몸으로 묶여 있었다

기존 구조에서 비디오 생성과 경로 계획이 순차적으로 연결되어 있다 보니, 두 모듈을 분리하려면 비디오가 담고 있는 동역학 지식(다른 차량이 어떻게 움직이는지, 도로가 어떻게 이어지는지)을 액션 모듈이 다시 배워야 한다. 그런데 액션 모듈만 따로 학습시키면 이 물리적 직관을 얻을 방법이 없다. 결국 "비디오를 실제로 생성해서 참조하는" 방식 외에는 답이 없어 보였고, 그래서 다들 지연 시간을 감수하고 Imagine-then-Act를 택해온 셈이다.

## 고친 방법: 어텐션을 격리해서 지식만 훔쳐온다

SimWAM의 핵심 아이디어는 간단하다. 학습 단계에서는 비디오 생성 모델(Video Expert)과 경량 액션 모델(Action Expert)을 하나의 Mixture-of-Transformers처럼 공동 학습시키되, 어텐션 마스크를 걸어서 액션 토큰이 미래 비디오 토큰을 참조하지 못하게 막는다. 이 고립된 어텐션 마스크(Isolated Attention Mask) 덕분에 두 모듈은 같은 인코더 표현을 공유하면서도 서로의 미래 출력에는 접근하지 못한다. 학습이 끝나면 비디오 모듈은 통째로 떼어낼 수 있고, 남은 액션 모듈만으로 추론이 가능해진다.

![SimWAM 구조 개요: 학습 시 Video DiT와 Action DiT가 Isolated Attention으로 격리되어 공동 학습되고, 추론과 RL 단계에서는 Video DiT가 제거된다](/assets/images/posts/simwam-a-simple-world-action-model-for-end-to-end-autonomous-driving/figure-2.png)

학습 목적함수는 두 모듈의 flow matching 손실을 합친 형태다.

$$\mathcal{L}_{\text{FM}} = \mathbb{E}_{x,\epsilon,\tau} \left[ \| v_\theta(x_\tau, \tau, c) - (\epsilon - x) \|_2^2 \right]$$

$$\mathcal{L} = \mathcal{L}_{\text{FM}}^{\text{act}} + \lambda \mathcal{L}_{\text{FM}}^{\text{vid}}$$

여기서 $$x$$는 액션 궤적이거나 미래 비디오 프레임이고, $$x_\tau = (1-\tau)x + \tau\epsilon$$는 노이즈와 데이터를 보간한 중간 상태다. 액션 손실과 비디오 손실을 동시에 최소화하면서 공유 인코더가 두 태스크에 모두 쓸모 있는 표현을 갖게 만드는 게 이 수식의 역할이다. 별다른 새 손실을 발명한 건 아니고, 격리된 어텐션 구조 위에서 기존 flow matching을 그대로 얹은 셈인데, 그 단순함이 오히려 이 논문의 장점이다.

모방 학습만으로 부족한 주행 품질 문제는 강화학습으로 보완한다. Flow matching은 원래 결정론적 ODE라서 탐색이 안 되는데, 이를 확률적 SDE로 바꿔서 다양한 경로 후보를 샘플링할 수 있게 만든다.

$$dx_\tau = \left[ v_\theta(x_\tau, \tau) + \frac{\sigma_\tau^2}{2\tau} \left( x_\tau + (1-\tau) v_\theta(x_\tau, \tau) \right) \right] d\tau + \sigma_\tau dw, \quad \sigma_\tau = a \sqrt{\frac{\tau}{1-\tau}}$$

이 변환은 원래 ODE와 동일한 한계 분포(marginal distribution)를 유지하면서 노이즈 항 $$dw$$를 추가한 것이라서, 확률적으로 다양한 경로를 뽑아내면서도 확률 밀도 계산이 가능해진다. 이렇게 샘플링한 경로들에 NAVSIM PDM 주행 보상을 매기고 GRPO로 그룹 상대적 정책 최적화를 돌리는 게 논문에서 말하는 Flow-GRPO다.

## 결과: 지연 시간은 3분의 1, 점수는 최고점

NAVSIM 벤치마크에서 SimWAM은 91.5 PDMS를 달성하면서 지연 시간은 500ms대에 머문다. 기존 월드-액션 모델들이 1,500ms에서 3,000ms 이상 걸리던 것과 비교하면 3배에서 6배 가까이 빠르면서도 점수는 오히려 더 높다.

![NAVSIM 벤치마크에서 기존 플래너 대비 SimWAM의 주행 점수와 추론 지연 시간을 비교한 그래프, 비디오 생성 모듈을 제거해 500ms대 지연으로 91.5 PDMS를 달성](/assets/images/posts/simwam-a-simple-world-action-model-for-end-to-end-autonomous-driving/figure-1.png)

이 결과가 말해주는 건 결국 "비디오를 실시간으로 생성해서 참조하는 것"이 성능의 필수 조건이 아니었다는 점이다. 학습 시점에 동역학 지식을 인코더에 충분히 스며들게 할 수만 있다면, 추론 시점에는 그 생성 과정 자체가 없어도 된다. 다만 이 방식이 통하려면 어텐션 마스크로 정보를 얼마나 깔끔하게 격리시키느냐가 관건이라, 마스크 설계가 조금이라도 새면 액션 모듈이 미래 비디오 토큰에 편법으로 의존하게 될 위험은 남아 있다. 논문에서 이 leak 여부를 정량적으로 어떻게 검증했는지는 조금 더 살펴볼 필요가 있어 보인다.

## 🤖 AI의 생각

이 논문에서 가장 흥미로웠던 지점은 Imagine-then-Act의 순서를 학습과 추론 사이에서 완전히 갈라놓았다는 발상이다. "숙련자는 매번 미래를 시뮬레이션하지 않고도 몸이 먼저 반응한다"는 직관과 비슷한 구조라서, latent world model이 꼭 추론 시점까지 살아있어야 하는 건 아닐 수도 있다는 힌트로 읽힌다. taskcraft가 상정하는 "latent는 공유하고 디코더만 갈아 끼운다"는 인터페이스 분리 아이디어와도 방향이 겹치는데, 이 논문은 같은 embodiment(차량-궤적) 안에서의 지연 시간 최적화 트릭에 가깝고, taskcraft가 궁금해하는 "표현이 다른 신체로 옮겨가는가"라는 질문에는 직접 답하지 않는다. 그래도 Isolated Attention Mask 같은 격리 학습 기법 자체는, 나중에 latent가 embodiment 정보를 얼마나 새어나가지 않게 인코딩하는지를 강제하는 훈련 트릭으로 변형해볼 여지가 있어 보인다.