---
title: "노이즈에서 이미지로, DDPM 뜯어보기"
date: 2026-09-17
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "확산 모델의 뿌리인 DDPM이 노이즈 예측이라는 단순한 아이디어로 GAN급 이미지 생성을 이뤄낸 과정을 살펴봅니다."
paper_title: "Denoising Diffusion Probabilistic Models"
paper_summary: "확산 모델의 뿌리인 DDPM이 노이즈 예측이라는 단순한 아이디어로 GAN급 이미지 생성을 이뤄낸 과정을 살펴봅니다."
paper_url: "https://arxiv.org/abs/2006.11239v2"
header:
  image: "/assets/images/posts/denoising-diffusion-probabilistic-models/figure-1.png"
  teaser: "/assets/images/posts/denoising-diffusion-probabilistic-models/figure-1.png"
---

## 이번 주 논문: 왜 DDPM인가

이번 주에는 요즘 로보틱스 논문들에서 당연하다는 듯 등장하는 Diffusion Policy, DiT 같은 방법론의 뿌리를 짚어보려고 합니다. VLA(Vision-Language-Action) 이후의 흐름을 따라가려면 결국 "diffusion이 뭔데 다들 이걸 쓰지"라는 질문을 한 번은 정면으로 통과해야 하는데, 그 답이 바로 오늘 다룰 Denoising Diffusion Probabilistic Models(DDPM, Ho et al. 2020)입니다. 스코어링 근거로도 관련성 1.00이 나온 논문이라 이번 주 첫 회차로 삼기에 적절하다고 판단했습니다. Imitation Learning이나 RL의 policy gradient 같은 고전 기법은 수업에서 배웠어도, 그 policy를 diffusion으로 표현한다는 아이디어는 생소할 수 있으니, 오늘은 로봇 얘기는 잠깐 접어두고 DDPM 자체를 기초부터 뜯어보겠습니다.

## 생성 모델이 푸는 문제

먼저 "생성 모델을 학습한다"는 게 뭘 의미하는지부터 짚고 가자. 우리에게 고양이 사진 수만 장이 있다고 하면, 그 사진들이 어떤 확률분포 $$p(x)$$에서 뽑힌 샘플이라고 가정할 수 있다. 생성 모델의 목표는 이 $$p(x)$$를 직접 알거나 최소한 그 분포에서 새로운 샘플을 뽑아낼 수 있는 방법을 배우는 것이다. GAN은 생성기와 판별기를 경쟁시켜 간접적으로 이 분포를 흉내 내고, VAE는 잠재 변수(latent variable)를 두고 변분 추론으로 근사하며, Flow 기반 모델은 가역 변환을 쌓아 확률을 정확히 계산할 수 있게 설계한다. 각각 나름의 트레이드오프가 있는데, GAN은 학습이 불안정하고 mode collapse가 잦고, VAE는 생성 결과가 흐릿한 경향이 있고, Flow는 아키텍처 제약이 크다.

Diffusion 모델은 이 셋과는 결이 다른 접근을 취한다. 아이디어 자체는 2015년 비평형 열역학에서 영감을 받아 나왔는데(Sohl-Dickstein et al.), 이론적으로는 우아했지만 GAN만큼 실제로 좋은 이미지를 뽑아내는지는 한동안 증명되지 않은 상태였다. DDPM 논문이 하는 일이 바로 이 증명, 정확히는 "이 방법을 제대로 파라미터화하면 GAN에 필적하는 품질이 나온다"는 것을 보여주는 것이다.

## 확산 모델의 두 방향: 망가뜨리기와 복원하기

Diffusion 모델의 구조는 생각보다 단순하다. 원본 이미지 $$x_0$$에서 시작해서, 아주 작은 가우시안 노이즈를 $$T$$번(보통 1000번 정도) 반복해서 더해가면 결국 $$x_T$$는 완전히 랜덤한 노이즈, 즉 표준 정규분포와 구분이 안 되는 상태가 된다. 이걸 순방향 과정(forward process)이라 부르고, 각 단계는 마르코프 체인(직전 상태에만 의존하는 확률 과정)으로 정의된다.

$$q(x_t \mid x_{t-1}) = \mathcal{N}(x_t; \sqrt{1-\beta_t}\, x_{t-1}, \beta_t \mathbf{I})$$

여기서 $$\beta_t$$는 매 단계 얼마나 노이즈를 섞을지 정하는 사전 정의된 스케줄이다. 재미있는 점은 이 순방향 과정이 닫힌 형태(closed form)로 정리되어서, $$t$$번을 하나씩 거치지 않고도 $$x_0$$로부터 임의의 시점 $$t$$의 $$x_t$$를 한 번에 계산할 수 있다는 것이다.

$$q(x_t \mid x_0) = \mathcal{N}(x_t; \sqrt{\bar{\alpha}_t}\, x_0, (1-\bar{\alpha}_t)\mathbf{I})$$

이건 훈련할 때 굉장히 편리하다. 학습 샘플을 하나 뽑을 때마다 $$t$$를 무작위로 고르고, 그 $$t$$에 해당하는 노이즈 낀 이미지를 즉석에서 만들어낼 수 있기 때문이다.

문제는 반대 방향, 즉 노이즈 $$x_T$$에서 출발해서 원본 $$x_0$$를 복원하는 역과정(reverse process)이다. 순방향은 노이즈를 더하는 것뿐이라 규칙이 명확하지만, 역방향은 "이 노이즈 낀 이미지에서 노이즈를 얼마나 어떻게 제거해야 하는지"를 알아야 하는데 이건 데이터 분포 전체를 알아야 풀리는 문제다. 그래서 신경망 $$p_\theta(x_{t-1} \mid x_t)$$를 두고 이걸 학습으로 근사한다. 순방향과 역방향이 마르코프 체인으로 나란히 대응되는 구조는 아래 그림에서 볼 수 있다.

![순방향 노이즈 추가 과정과 역방향 디노이징 과정을 나타낸 그래픽 모델 구조](/assets/images/posts/denoising-diffusion-probabilistic-models/figure-1.png)

## 왜 이론만으로는 안 됐는가

여기까지는 2015년 원조 논문에서 이미 제시된 그림이다. 그런데 이 역과정을 학습시키는 표준적인 방법은 변분 하한(Variational Lower Bound, ELBO)을 최대화하는 것인데, 이게 실전에서 잘 안 먹혔다. ELBO를 그대로 최적화하려면 역과정의 평균과 분산을 둘 다 신경망이 예측해야 하는데, 분산까지 같이 학습시키면 훈련이 불안정해지고, 손실 함수에 붙는 시점별 가중치 계수도 이론적으로 유도된 형태 그대로 쓰면 학습 신호가 특정 구간에 쏠려서 최종 샘플 품질이 떨어졌다. 즉 수식은 맞는데 "어떻게 파라미터화하고 어떤 손실을 쓸 것인가"라는 실용적인 선택지가 정리되지 않아서, 이론적 완성도에 비해 실제 생성 품질은 GAN에 명백히 못 미치는 상태였던 것이다.

## 고친 방법 1: 평균이 아니라 노이즈를 예측하게 하자

DDPM 저자들의 첫 번째 핵심 선택은 신경망이 무엇을 출력하게 할 것인가를 바꾼 것이다. 원래대로라면 신경망이 역과정의 평균 $$\mu_\theta(x_t, t)$$를 직접 예측해야 하는데, 저자들은 대신 순방향 과정에서 애초에 어떤 노이즈 $$\epsilon$$이 섞여 들어갔는지를 예측하도록 신경망 $$\epsilon_\theta(x_t, t)$$를 재구성했다. 수식으로 정리하면 평균은 다음과 같이 이 노이즈 예측값으로부터 역산된다.

$$\mu_\theta(x_t, t) = \frac{1}{\sqrt{\alpha_t}} \left( x_t - \frac{\beta_t}{\sqrt{1 - \bar{\alpha}_t}} \epsilon_\theta(x_t, t) \right)$$

왜 이게 더 잘 되는지는 스코어 매칭(score matching)이라는 개념과 연결해서 이해하면 자연스럽다. 데이터 분포의 로그 확률 그래디언트, 즉 "이 지점에서 어느 방향으로 가야 데이터가 더 그럴듯해지는가"를 학습하는 것과 노이즈를 예측하는 것이 수학적으로 같은 문제라는 걸 보인 것이다. 결과적으로 신경망은 평균이라는 다소 추상적인 양보다 노이즈라는 직관적이고 스케일이 일정한 양을 맞추는 문제를 풀게 되고, 이게 학습을 훨씬 안정적으로 만든다. 저자들은 여기에 더해 역과정의 분산 $$\Sigma_\theta(x_t, t)$$도 아예 학습 대상에서 빼고 $$\beta_t$$에서 유도된 상수로 고정해버렸다. 학습해야 할 대상을 줄인 것이 오히려 품질을 끌어올린 셈이다.

## 고친 방법 2: 손실 함수를 단순하게 깎아내기

두 번째 핵심은 손실 함수 자체를 손보는 것이다. 이론적으로 유도되는 ELBO 기반 손실에는 시점 $$t$$마다 다른 가중치 $$\frac{\beta_t^2}{2\sigma_t^2 \alpha_t (1-\bar{\alpha}_t)}$$가 붙는데, 저자들은 이 가중치를 그냥 없애버리고 단순한 MSE로 대체했다.

$$L_{\text{simple}}(\theta) = \mathbb{E}_{t, x_0, \epsilon}\left[ \lVert \epsilon - \epsilon_\theta(\sqrt{\bar{\alpha}_t}x_0 + \sqrt{1-\bar{\alpha}_t}\epsilon,\ t) \rVert^2 \right]$$

이게 왜 오히려 나은가 하면, 원래 가중치는 노이즈가 거의 없는 초반 시점(작은 $$t$$)의 손실을 과도하게 키우는 방향으로 작동한다. 그런데 작은 $$t$$에서의 디노이징은 사실 쉬운 문제이고, 이미지의 전체적인 구조와 형태를 결정하는 건 노이즈가 많이 낀 큰 $$t$$ 구간이다. 가중치를 없애면 자연스럽게 어려운 구간, 즉 모델이 정말 배워야 할 구간에 학습 역량이 더 실리게 되고 이게 최종 샘플의 지각 품질(perceptual quality)을 크게 끌어올린다. 이론적으로 엄밀한 하한을 최적화하는 것보다, 실용적으로 유효한 손실을 쓰는 게 더 나은 결과를 낸 흥미로운 사례다.

## 결과: 점진적으로 그림을 그려나가는 모델

이렇게 학습된 모델로 샘플링을 하면 $$x_T$$의 순수한 노이즈에서 출발해서 $$x_{T-1}, x_{T-2}, \dots$$ 순서로 조금씩 노이즈를 걷어내며 이미지를 완성해간다. 이 과정을 중간중간 들여다보면 재미있는 패턴이 보인다.

![역과정이 진행되며 각 시점의 잠재 변수로부터 예측된 복원 이미지가 큰 구조에서 세부 디테일 순으로 채워지는 과정](/assets/images/posts/denoising-diffusion-probabilistic-models/figure-2.png)

큰 $$t$$(노이즈가 많이 남은 초반)에서는 물체의 전체적인 윤곽과 배치 같은 거시적 구조가 먼저 잡히고, 작은 $$t$$로 갈수록 질감이나 미세한 디테일이 채워진다. 이는 오토리그레시브 모델이 픽셀을 순서대로 하나씩 그려나가는 것과 유사한 점진적 디코딩을 diffusion이 다른 방식으로 일반화한 것으로 볼 수 있다. 정량적으로는 CIFAR-10에서 FID 3.17을 기록하며 당시 GAN 기반 SOTA에 필적하는 결과를 냈고, 이는 diffusion 모델이 더 이상 이론적 흥미거리에 머무르지 않고 실용적인 생성 모델 후보가 되었음을 보여준 첫 사례였다.

## 그래도 남은 문제

물론 이 논문이 모든 걸 해결한 건 아니다. 가장 큰 한계는 샘플링 속도다. 이미지 하나를 만들려면 $$T$$번(논문에서는 1000번)의 순차적인 디노이징 스텝을 다 거쳐야 하는데, 이는 GAN이 한 번의 forward pass로 이미지를 뽑는 것과 비교하면 압도적으로 느리다. 또한 아키텍처로는 U-Net을 그대로 가져다 썼는데, 이는 국소적인 합성곱(convolution) 연산에 기반한 귀납적 편향을 갖고 있어서 모델을 키울 때의 확장성(scalability)에는 한계가 있다. 이 두 지점, 즉 샘플링 속도와 백본 아키텍처는 각각 DDIM류의 후속 연구와 DiT(Diffusion Transformer)가 나중에 파고드는 지점이 된다.

## 왜 지금 이 논문이 로보틱스와 만나는가

DDPM이 확립한 것은 결국 두 가지다. 노이즈를 예측하도록 학습시키자는 것, 그리고 그 손실을 단순한 MSE로 두자는 것. 이 두 가지 선택은 이미지 픽셀뿐 아니라 어떤 연속적인 신호에도 그대로 적용할 수 있는 일반적인 틀이다. 실제로 Diffusion Policy는 이 틀을 로봇의 action sequence에 그대로 적용한다. Behavior cloning으로 로봇을 가르칠 때 흔히 겪는 문제가, 같은 상황에서도 사람 시연이 여러 갈래로 갈리는 멀티모달 분포를 단일 회귀로는 제대로 못 배운다는 것인데, DDPM의 노이즈 제거 절차를 action에 적용하면 이 멀티모달성을 자연스럽게 표현할 수 있다. 오늘 다룬 논문 자체는 로봇과 무관하지만, VLA 이후 흐름을 따라가려는 우리 입장에서는 이 논문이 그 뒤에 나오는 방법론들의 공통 문법을 정의했다는 점이 중요하다.

## 용어 해설

- **변분 하한(ELBO, Evidence Lower Bound)**: 계산하기 어려운 확률분포의 로그우도를 직접 최적화하는 대신, 그보다 항상 작거나 같은 근사적인 하한을 대신 최적화하는 변분 추론의 핵심 개념이다.
- **스코어 매칭(Score Matching)**: 확률분포 자체가 아니라 그 로그밀도의 그래디언트(스코어)를 학습하는 방법으로, 정규화 상수를 몰라도 분포의 모양을 배울 수 있게 해준다.
- **마르코프 체인(Markov Chain)**: 현재 상태가 오직 바로 직전 상태에만 의존하고 그 이전 과거와는 독립적인 확률 과정을 말한다.
- **FID(Fréchet Inception Distance)**: 생성된 이미지 집합과 실제 이미지 집합의 특징 분포 사이 거리를 측정해 생성 모델의 품질을 정량적으로 비교하는 지표로, 값이 작을수록 좋다.

## 🤖 AI의 생각

개인적으로는 이 논문의 재미가 "이론적으로 완벽한 것"과 "실제로 잘 되는 것" 사이에 생각보다 큰 틈이 있고, 그 틈을 메운 게 거창한 새 이론이 아니라 파라미터화와 손실 함수라는 다소 소박한 엔지니어링 선택이었다는 데 있다고 생각합니다.

<details class="tc-faq">
<summary>노이즈 가중치를 없앤 \(L_{\text{simple}}\)이 왜 이론적 최적(ELBO)보다 더 나은 샘플을 만드는지, 그 이유를 지각 품질과 우도(likelihood) 사이의 트레이드오프로 더 엄밀하게 설명할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문도 이 부분은 실험적으로 관찰했을 뿐 완전한 이론적 설명은 후속 연구(예컨대 diffusion loss weighting에 대한 이후 분석들)로 넘어간 지점이라 더 파볼 가치가 있다

</div>
</details>

<details class="tc-faq">
<summary>샘플링에 필요한 수백~수천 스텝의 순차성은 로봇처럼 실시간 제어가 필요한 응용에서 특히 부담일 텐데, Diffusion Policy는 이 문제를 어떻게 우회하고 있을까</summary>
<div class="tc-faq__body" markdown="1">

보통 action horizon을 미리 chunk 단위로 뽑아두고 receding horizon으로 재사용하거나 DDIM류의 적은 스텝 샘플러를 쓰는 식으로 절충하는 것으로 알고 있다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 목표로 하는 embodiment-agnostic latent world model에서 상태 전이 자체를 diffusion으로 모델링한다면 어떤 이득과 비용이 생길까</summary>
<div class="tc-faq__body" markdown="1">

신체 구조에 따라 같은 task라도 다음 latent 상태가 여러 갈래로 갈리는 멀티모달성은 잘 표현하겠지만 순차 샘플링 비용이 world model rollout 속도에 그대로 부담으로 더해질 것 같다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2006.11239v2" target="_blank" rel="noopener">Denoising Diffusion Probabilistic Models</a> · arxiv</span></div>
</div>
</div>
