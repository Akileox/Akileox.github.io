---
title: "확산 모델이 GAN보다 느린 이유, 그리고 DDIM이 고친 방법"
date: 2026-09-06
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "Denoising diffusion probabilistic models (DDPMs) have achieved high quality image generation without adversarial training, yet they require simulating"
paper_title: "Denoising Diffusion Implicit Models"
paper_summary: "Denoising diffusion probabilistic models (DDPMs) have achieved high quality image generation without adversarial training, yet they require simulating"
paper_url: "https://arxiv.org/abs/2010.02502v4"
header:
  image: "/assets/images/posts/denoising-diffusion-implicit-models/figure-1.png"
  teaser: "/assets/images/posts/denoising-diffusion-implicit-models/figure-1.png"
---

# 확산 모델이 GAN보다 느린 이유, 그리고 DDIM이 고친 방법

이번 주 논문으로 Denoising Diffusion Implicit Models(DDIM)을 골랐습니다. score=1.00으로 이번 주 아카이브 후보 중 가장 높은 우선순위를 받았는데, 이유는 명확합니다. Diffusion 계열 모델을 다루는 최근 로봇 정책 연구(Diffusion Policy 등)나 생성 모델 기반 world model 논문들이 거의 예외 없이 "샘플링 가속" 파트에서 이 논문을 인용하기 때문입니다. VLA 이후 흐름을 처음 보는 독자 입장에서는 이 논문이 왜 그렇게 자주 인용되는지, DDPM과 뭐가 다른지부터 짚고 가는 게 이후 흐름을 따라가는 데 필요한 디딤돌이 될 거라 판단했습니다.

## Diffusion 모델은 왜 느린가

Diffusion 기반 생성 모델의 기본 아이디어는 간단하다. 깨끗한 이미지에 노이즈를 조금씩 섞어가며 완전한 노이즈로 만드는 과정(forward process)을 정의하고, 신경망이 그 반대 방향, 즉 노이즈를 조금씩 제거해서 원본을 복원하는 과정(reverse process)을 학습하게 만든다. 학습이 끝나면 순수한 노이즈에서 출발해서 이 reverse process를 수백에서 수천 번 반복하면 새로운 이미지가 만들어진다.

DDPM(Denoising Diffusion Probabilistic Models)은 이 방식으로 GAN에 필적하는 이미지 품질을 adversarial 학습 없이 달성해서 주목받았다. 문제는 속도였다. DDPM은 마르코프 체인(Markov chain) 구조를 쓰는데, 이는 $$x_t$$에서 $$x_{t-1}$$로 넘어갈 때 바로 이전 스텝의 정보만 보고 다음 스텝을 결정한다는 뜻이다. 이 체인을 T=1000 단계까지 순서대로 밟아야만 이미지 하나가 완성된다. 논문에 따르면 CIFAR10 이미지 5만 장을 생성하는 데 GPU로 약 20시간이 걸리는 반면, GAN은 1분도 안 걸린다. 256×256 이미지라면 거의 1000시간이다. 학습된 모델은 이미 좋은데, 그걸 써먹는 속도가 발목을 잡는 상황이었다.

## 왜 안 됐는지, 그리고 핵심 통찰

가장 단순한 해법은 "스텝 수를 줄이자"이지만, DDPM의 마르코프 체인을 그대로 두고 중간 스텝을 건너뛰면 각 스텝이 가정하는 노이즈 분포가 어긋나서 품질이 급격히 떨어진다. 그렇다고 스텝 수를 줄인 새 모델을 다시 학습시키는 건 비용이 너무 크다.

이 논문의 핵심 통찰은 DDPM의 학습 목적함수를 다시 들여다본 데서 나온다. DDPM의 손실함수 $$L_\gamma$$는 사실 forward process 전체의 결합분포 $$q(x_{1:T}\vert{}x_0)$$가 아니라, 각 시점의 주변분포(marginal distribution) $$q(x_t\vert{}x_0)$$에만 의존한다. 즉 forward process가 마르코프 체인이어야 할 필연적인 이유가 학습 목적함수 안에는 없다는 것이다. 그렇다면 같은 주변분포를 유지하면서도 forward process의 내부 구조를 자유롭게 바꿀 수 있고, 그렇게 바꾼 과정도 DDPM과 똑같은 손실함수로 학습된 모델을 그대로 재사용할 수 있다.

이 아이디어를 그림으로 보면 이해가 빠르다.

![DDPM의 마르코프 그래프 모델(좌)과 DDIM이 제안하는 비마르코프 그래프 모델(우) 비교](/assets/images/posts/denoising-diffusion-implicit-models/figure-1.png)

왼쪽 DDPM 그래프에서는 $$x_3 \to x_2 \to x_1 \to x_0$$가 인접한 스텝끼리만 연결된 순수 사슬이다. 오른쪽 제안 모델에서는 모든 잠재변수가 $$x_0$$에 직접 조건부로 연결되어 있고, $$x_2$$와 $$x_3$$ 사이에도 직접 연결이 생긴다. 이게 바로 비마르코프(non-Markovian) 과정이다. 이렇게 정의를 바꿔도 각 시점의 주변분포 $$q(x_t\vert{}x_0)$$는 DDPM과 똑같이 유지되도록 설계할 수 있다는 게 이 논문의 Lemma 1이다.

## 고친 방법: 결정론적 샘플링과 스텝 건너뛰기

비마르코프 forward process는 파라미터 $$\sigma$$로 확률성(stochasticity)의 크기를 조절한다. $$\sigma$$가 크면 DDPM과 비슷하게 매 스텝마다 무작위 노이즈가 섞이고, $$\sigma \to 0$$으로 보내면 노이즈 항이 사라지면서 forward process도 reverse process도 완전히 결정론적(deterministic)이 된다. 이 극단적인 경우가 바로 DDIM이다.

생성 과정의 업데이트 식은 다음과 같이 세 항으로 쪼개진다.

$$x_{t-1} = \sqrt{\alpha_{t-1}}\left(\frac{x_t-\sqrt{1-\alpha_t}\epsilon_\theta^{(t)}(x_t)}{\sqrt{\alpha_t}}\right) + \sqrt{1-\alpha_{t-1}-\sigma_t^2}\cdot\epsilon_\theta^{(t)}(x_t) + \sigma_t\epsilon_t$$

첫 번째 항은 모델이 지금 이 노이즈 낀 $$x_t$$를 보고 추정한 "깨끗한 원본 이미지"다. 두 번째 항은 다시 $$x_t$$ 방향으로 되돌리는 벡터고, 세 번째 항이 확률적 노이즈다. $$\sigma_t = 0$$으로 두면 세 번째 항이 완전히 사라지고, 남은 두 항만으로 다음 스텝을 결정론적으로 계산할 수 있다.

여기서 중요한 건 이 식이 노이즈 예측 모델 $$\epsilon_\theta$$ 하나만 있으면 계산 가능하다는 점이다. DDPM을 학습할 때 쓴 모델을 재학습 없이 그대로 가져다 쓸 수 있다. Theorem 1이 이를 정당화하는데, $$\sigma$$를 어떻게 고르든 동일한 대리 손실함수 $$L_1$$으로 학습한 모델이면 충분하다는 걸 증명한다.

그리고 여기서 한 걸음 더 나아간다. forward process를 원래의 T=1000 스텝 전부가 아니라, 그중 일부만 뽑은 부분 수열 $$\tau$$(길이 $$S < T$$)에 대해서만 정의하면, 생성 체인 자체를 짧게 만들 수 있다.

![원래 T스텝 forward process에서 부분 수열을 뽑아 가속 샘플링을 수행하는 과정](/assets/images/posts/denoising-diffusion-implicit-models/figure-2.png)

이 그림이 보여주는 건 학습은 원래 T=1000 스텝 그대로 해두고, 샘플링할 때만 예를 들어 10개나 100개 스텝만 골라 그 사이를 건너뛰며 생성한다는 것이다. 마르코프 체인이었다면 이렇게 건너뛰는 순간 분포가 어긋나 품질이 무너지지만, DDIM은 애초에 각 스텝이 $$x_0$$에 대한 결정론적 함수로 정의되어 있어서 스텝을 듬성듬성 골라도 품질 저하가 훨씬 덜하다.

## 결과: 얼마나 빨라졌고 무엇을 얻었나

논문의 Table 1 수치를 보면 CIFAR10에서 샘플링 스텝을 10개로 줄였을 때 DDIM의 FID는 13.36인 반면, 같은 스텝 수에서 DDPM 스타일 확률적 샘플링은 41.07, 분산을 다르게 잡은 버전은 367.43까지 치솟는다. 스텝이 적을수록 확률적 샘플링은 급격히 무너지고, 결정론적 DDIM은 상대적으로 안정적이라는 뜻이다. 즉 같은 학습된 모델, 같은 파라미터 수로 10~50배 적은 스텝만 밟고도 쓸 만한 품질을 낼 수 있다.

또 하나 흥미로운 결과는 DDIM의 결정론적 특성이 이미지의 "일관성(consistency)"을 만든다는 점이다. 같은 초기 노이즈 $$x_T$$에서 출발하면 샘플링 스텝 수를 10으로 하든 1000으로 하든 최종 이미지의 전체적인 구도, 포즈, 색감이 거의 그대로 유지된다. 이는 $$x_T$$가 단순한 노이즈가 아니라 이미지의 고수준 의미를 담은 일종의 latent 인코딩처럼 작동한다는 뜻이고, 이 덕분에 두 개의 $$x_T$$ 사이를 구면 선형보간(spherical linear interpolation)하면 얼굴 방향이나 헤어스타일이 자연스럽게 바뀌는 보간 이미지를 얻을 수 있다. 확률적인 DDPM에서는 매 스텝마다 무작위성이 섞이기 때문에 이런 깔끔한 보간이 불가능하다.

논문은 여기서 그치지 않고 DDIM의 업데이트 식이 상미분방정식(ODE)의 오일러 적분과 형태가 같다는 것도 보인다. 이는 이미지를 노이즈로 인코딩했다가 다시 복원하는 것도 가능하다는 뜻이며, 이후 latent diffusion이나 이미지 편집 계열 연구에서 "diffusion latent를 조작 가능한 표현으로 다룬다"는 흐름의 이론적 기반이 됐다.

## 한계

이 논문 자체가 명시하는 한계는 크지 않지만, 실용적으로 보면 스텝을 극단적으로 줄일 경우(예: 5스텝 이하) 품질 저하가 여전히 발생하고, 결정론적 샘플링이라 DDPM처럼 같은 조건에서 다양한 샘플을 뽑는 다양성(diversity)은 다소 희생된다. 또한 이 논문이 다루는 가속은 어디까지나 샘플링 알고리즘 차원의 개선이라, 노이즈 예측 네트워크 자체의 구조(U-Net이냐 Transformer냐)와는 독립적이다. 즉 "얼마나 빠르게 뽑을 것인가"의 문제는 풀었지만 "네트워크 자체를 더 잘 만드는 문제"는 별개로 남아 있다.

## 용어 해설

- **마르코프 체인(Markov chain)**: 다음 상태가 오직 현재 상태에만 의존하고 그 이전의 과거 상태들과는 무관하게 결정되는 확률 과정을 말한다.
- **주변분포(marginal distribution)**: 여러 변수의 결합분포에서 관심 없는 변수들을 적분(또는 합산)해 제거하고 남은, 특정 변수 하나에 대한 확률분포를 뜻한다.
- **상미분방정식(ODE)의 오일러 적분**: 미분방정식으로 주어진 변화율을 아주 작은 스텝 단위로 근사해 누적하면서 함수값을 수치적으로 구하는 가장 기본적인 방법이다.
- **구면 선형보간(slerp)**: 두 벡터 사이를 직선이 아니라 구 표면을 따라 일정한 각속도로 이동하며 보간하는 방법으로, 고차원 latent 공간에서 두 점 사이를 자연스럽게 잇는 데 자주 쓰인다.

## 🤖 AI의 생각


개인적인 감상을 덧붙이면, 이 논문은 새로운 모델을 제안했다기보다 "기존 모델을 다르게 읽는 법"을 제안한 논문에 가깝다는 인상이 강하다. 손실함수를 다시 들여다봐서 불필요한 가정(마르코프성)을 걷어낸 것만으로 실용성이 크게 뛰는 사례라, 문제 재정의의 힘을 보여주는 논문이라고 느꼈다.

<details class="tc-faq">
<summary>DDIM이 스텝 수를 극단적으로 줄일 때(예: 3~5스텝) 품질이 무너지는 지점은 어디부터이고, 그 한계를 극복하려는 후속 연구(예: 증류 기반 가속)들은 어떤 트레이드오프를 감수했을까</summary>
<div class="tc-faq__body" markdown="1">

논문 자체는 10스텝 근방까지만 자세히 보고하는데, 그 이하 구간의 급격한 품질 저하를 다루려면 progressive distillation 같은 후속 기법들이 어떤 추가 학습 비용을 치렀는지 비교해볼 만하다

</div>
</details>

<details class="tc-faq">
<summary>DDIM이 보여준 xT의 semantic latent 속성은 이미지 도메인 안에서의 일관성인데, 서로 다른 embodiment 사이에서도 비슷한 일관성을 갖는 latent를 설계하려면 무엇이 추가로 필요할까</summary>
<div class="tc-faq__body" markdown="1">

이미지 diffusion은 픽셀 공간이라는 단일 표현 공간 안에서의 일관성이라, taskcraft처럼 embodiment가 바뀌는 상황에서는 latent가 저수준 실행 디테일뿐 아니라 행동 공간 자체의 차이도 흡수해야 하므로 추가적인 정렬(alignment) 메커니즘이 필요해 보인다

</div>
</details>

<details class="tc-faq">
<summary>Diffusion Policy 계열 로봇 정책이 DDIM 가속을 가져다 쓸 때, 이미지 생성과 달리 행동 시퀀스는 물리적 실행 가능성이라는 제약이 있는데 결정론적 샘플링이 이 제약과 충돌하지는 않을까</summary>
<div class="tc-faq__body" markdown="1">

결정론적 샘플링은 다양성을 줄이는 대신 안정성을 높이므로, 안전 여유가 필요한 실제 로봇 제어에는 오히려 유리할 수 있지만 탐색적 행동이 필요한 상황에서는 확률적 샘플링과의 절충이 필요할 것 같다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2010.02502v4" target="_blank" rel="noopener">Denoising Diffusion Implicit Models</a> · arxiv</span></div>
</div>
</div>
