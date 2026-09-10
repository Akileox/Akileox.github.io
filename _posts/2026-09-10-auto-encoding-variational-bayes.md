---
title: "잠재변수를 미분 가능하게 만든 트릭"
date: 2026-09-10
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "VAE의 핵심인 재파라미터화 트릭이 왜 필요했고 무엇을 바꿨는지 기초부터 짚어본다."
paper_title: "Auto-Encoding Variational Bayes"
paper_summary: "VAE의 핵심인 재파라미터화 트릭이 왜 필요했고 무엇을 바꿨는지 기초부터 짚어본다."
paper_url: "https://arxiv.org/abs/1312.6114v11"
header:
  image: "/assets/images/posts/auto-encoding-variational-bayes/figure-1.png"
  teaser: "/assets/images/posts/auto-encoding-variational-bayes/figure-1.png"
---

이번 주에 다룰 논문은 Kingma와 Welling의 Auto-Encoding Variational Bayes입니다. arxiv 소스에서 score 1.00으로 선정되었는데, 단순히 인용수가 많아서가 아니라 이 논문이 제안한 재파라미터화 트릭이 VAE는 물론 diffusion 계열 모델, 그리고 제가 요즘 들여다보고 있는 taskcraft의 latent world model 설계까지 관통하는 기본기이기 때문입니다. VLA 이후의 최신 흐름을 이해하려면 결국 "확률적인 잠재변수를 어떻게 미분 가능하게 다루는가"라는 이 논문의 질문으로 돌아가야 한다고 판단해 이번 주 첫 논문으로 골랐습니다.

## 문제는 어디서 시작되는가

이미지 하나를 생성하는 모델을 상상해보자. 어떤 저차원의 "숨은 요인" $$z$$가 있고, 이 $$z$$로부터 실제 이미지 $$x$$가 만들어진다고 가정하는 것이 방향성 확률 그래프 모델이다. 우리는 $$p_\theta(z)$$와 $$p_\theta(x\vert z)$$를 신경망으로 정의하고, 데이터 $$x$$가 주어졌을 때 이 모델의 파라미터 $$\theta$$를 학습하고 싶다.

그런데 여기서 바로 막힌다. 학습을 하려면 주변우도 $$p_\theta(x) = \int p_\theta(z)p_\theta(x\vert z)dz$$를 계산해야 하는데, $$z$$가 연속이고 $$p_\theta(x\vert z)$$가 신경망이면 이 적분은 닫힌 형태로 풀리지 않는다. 사후분포 $$p_\theta(z\vert x)$$도 마찬가지로 계산 불가능하다. 즉 "이 이미지를 만든 숨은 요인은 무엇이었을까"라는 질문에 답할 방법이 없다.

## 기존 방법들이 왜 안 됐는가

이런 상황에서 전통적으로 쓰던 방법은 두 갈래다. 하나는 EM 알고리즘이나 MCMC 기반의 몬테카를로 EM으로, 사후분포에서 샘플을 반복적으로 뽑아 기댓값을 근사한다. 문제는 데이터포인트 하나마다 이런 반복 샘플링을 새로 돌려야 해서, 데이터셋이 크면 감당이 안 된다는 점이다.

다른 하나는 평균장(mean-field) 변분추론인데, 이 방법은 근사 사후분포 $$q(z)$$의 형태를 아주 단순하게(보통 인수분해 가능한 형태로) 가정하고 그 파라미터를 데이터포인트별로 직접 최적화한다. 이것도 신경망 기반 모델처럼 사후분포가 복잡한 경우에는 잘 맞지 않고, 역시 데이터포인트마다 별도의 최적화가 필요해 확장성이 떨어진다.

그럼 그냥 몬테카를로로 그래디언트를 추정하면 안 되나 싶겠지만, 확률적 노드 $$z \sim q_\phi(z\vert x)$$를 직접 샘플링해서 $$\phi$$에 대한 그래디언트를 구하는 score function estimator 방식은 분산이 너무 커서 실제로는 거의 학습이 안 된다. 이게 이 논문이 진짜로 풀고 싶었던 문제다. 그래디언트 추정량의 분산을 낮추면서도 대규모 데이터셋에 확장 가능한 학습 방법이 필요했다.

## 어떻게 고쳤는가: 인코더를 두고, 샘플링을 바깥으로 빼낸다

첫 번째 아이디어는 신경망으로 된 인식모델(recognition model) $$q_\phi(z\vert x)$$를 도입하는 것이다. 이게 지금 우리가 "인코더"라고 부르는 것이다. 이 $$q_\phi(z\vert x)$$는 실제 사후분포 $$p_\theta(z\vert x)$$를 근사하는 역할을 하는데, 계산 불가능한 진짜 사후분포 대신 다루기 쉬운 가우시안 형태로 근사한다.

이 인코더를 도입하면 로그우도를 다음과 같이 분해할 수 있다.

$$\log p_\theta(x^{(i)}) = D_{KL}(q_\phi(z\vert x^{(i)})\,\Vert\,p_\theta(z\vert x^{(i)})) + \mathcal{L}(\theta,\phi;x^{(i)})$$

KL 발산은 항상 0 이상이므로 $$\mathcal{L}$$은 로그우도의 하한이 된다. 이게 그 유명한 변분 하한(ELBO)이다. $$\mathcal{L}$$을 풀어보면 두 개의 직관적인 항으로 나뉜다.

$$\mathcal{L}(\theta,\phi;x^{(i)}) = -D_{KL}(q_\phi(z\vert x^{(i)})\Vert p_\theta(z)) + \mathbb{E}_{q_\phi(z\vert x^{(i)})}\left[\log p_\theta(x^{(i)}\vert z)\right]$$

앞 항은 인코더가 내놓는 분포를 사전분포 쪽으로 잡아당기는 정규화 항, 뒤 항은 인코더가 뽑은 $$z$$로부터 원본 $$x$$를 얼마나 잘 재구성하는지를 나타내는 항이다. 이 둘을 동시에 최대화하면 되는데, 문제는 뒤 항 안에 $$z \sim q_\phi(z\vert x)$$라는 샘플링이 들어 있다는 점이다. 샘플링 연산은 그 자체로 미분이 안 되기 때문에, $$\phi$$에 대해 역전파를 하려면 다른 방법이 필요하다.

여기서 재파라미터화 트릭이 등장한다. 아이디어는 단순하다. $$z$$를 직접 확률변수로 취급해 샘플링하는 대신, $$z$$를 결정론적 함수와 독립적인 노이즈의 조합으로 다시 쓰는 것이다.

$$\tilde z = g_\phi(\epsilon, x), \quad \epsilon \sim p(\epsilon)$$

가우시안 인코더라면 이건 아주 구체적인 형태를 띤다. $$\mu$$와 $$\sigma$$를 인코더 신경망이 출력하고, $$\epsilon \sim \mathcal{N}(0, I)$$을 따로 뽑은 다음, $$z = \mu + \sigma \odot \epsilon$$으로 조합하는 것이다. 이렇게 하면 확률성은 전부 $$\epsilon$$이라는, $$\phi$$와 무관한 변수 쪽으로 빠지고, $$z$$에서 $$\mu, \sigma$$로 가는 경로는 완전히 결정론적인 미분 가능 함수가 된다. 결과적으로 역전파가 $$\epsilon$$을 그냥 상수처럼 취급하면서 $$\mu, \sigma$$를 거쳐 인코더 파라미터까지 흘러갈 수 있게 된다.

![Auto-Encoding Variational Bayes의 그래픽 모델 구조, 실선은 생성모델(디코더), 점선은 변분 근사 사후분포(인코더)를 나타낸다](/assets/images/posts/auto-encoding-variational-bayes/figure-1.png)

Figure 1이 바로 이 구조를 보여준다. 실선은 사전분포에서 $$z$$를 뽑고 그로부터 $$x$$를 생성하는 디코더 경로, 점선은 관측된 $$x$$로부터 $$z$$에 대한 근사 사후분포를 만드는 인코더 경로다. 이 두 경로가 동시에, 같은 손실함수로 학습된다는 것이 오토인코더라는 이름이 붙은 이유다.

가우시안 사전분포와 가우시안 인코더를 가정하면 KL 항은 아예 닫힌 형태로 계산되고, 최종 목적함수는 다음과 같이 정리된다.

$$\mathcal{L}(\theta,\phi;x^{(i)}) \simeq \frac{1}{2}\sum_{j=1}^J\left(1+\log((\sigma_j^{(i)})^2)-(\mu_j^{(i)})^2-(\sigma_j^{(i)})^2\right) + \frac{1}{L}\sum_{l=1}^L \log p_\theta(x^{(i)}\vert z^{(i,l)})$$

앞부분은 인코더 출력만으로 해석적으로 계산되는 정규화 항이고, 뒷부분은 재파라미터화된 $$z^{(i,l)} = \mu^{(i)} + \sigma^{(i)} \odot \epsilon^{(l)}$$을 몇 개 뽑아 평균낸 재구성 오차 추정치다. 이 전체를 미니배치 단위로 확률적 경사상승법(SGD)으로 최적화하는 것이 Auto-Encoding VB, 줄여서 AEVB 알고리즘이다.

구조를 표로 정리하면 이렇다.

| 구성요소 | 역할 | 대응하는 신경망 |
|---|---|---|
| $$p_\theta(z)$$ | 잠재변수의 사전분포 | 보통 표준 가우시안, 학습 안 함 |
| $$q_\phi(z\vert x)$$ | 근사 사후분포 | 인코더 |
| $$p_\theta(x\vert z)$$ | 데이터 생성 분포 | 디코더 |

## 결과와 한계

논문은 MNIST와 Frey Face 데이터셋에서 AEVB를 wake-sleep 알고리즘, 몬테카를로 EM과 비교한다. 잠재변수 차원을 늘려가며 학습 곡선을 비교했을 때 AEVB가 더 빠르게 수렴하고 더 높은 하한에 도달했으며, 잠재변수 수를 늘려도 과적합이 나타나지 않았다. 이는 KL 정규화 항이 잠재공간을 계속 압축된 상태로 유지시키는 효과를 낸다는 뜻이다. 저차원 잠재공간에서 MCMC로 추정한 실제 주변우도와 비교했을 때도 AEVB가 MCEM과 wake-sleep보다 더 우수했다.

다만 이 논문 자체의 한계도 분명하다. 재구성 대상이 정적인 이미지라서, 시간에 따라 변화하는 상태나 행동의 결과 같은 동적인 신호를 다루려면 이 틀만으로는 부족하다. 또한 KL 정규화가 만들어내는 압축은 어디까지나 재구성에 필요한 정보만 남기도록 강제하는 것이지, 그 압축된 표현이 우리가 원하는 의미(예를 들어 신체 구조와 무관한 태스크 표현)를 자동으로 갖는다는 보장은 없다. 이 지점은 나중에 diffusion 계열 모델이나 latent dynamics model이 각자의 방식으로 다시 마주치는 문제이기도 하다.

## 용어 해설

- **score function estimator**: 확률변수의 기댓값에 대한 그래디언트를 구할 때, $$\nabla \log q(z)$$를 이용해 샘플링 기반으로 근사하는 방법. 재파라미터화가 불가능한 이산 변수 등에 쓰이지만 분산이 크다는 단점이 있다.
- **평균장(mean-field) 근사**: 복잡한 결합분포를 여러 개의 독립적인 인수(factor)의 곱으로 단순화해서 다루는 변분추론 기법. 각 인수의 파라미터를 데이터포인트별로 최적화한다.
- **조상 샘플링(ancestral sampling)**: 그래픽 모델에서 부모 노드부터 자식 노드 순서로 차례차례 샘플을 뽑아 전체 변수의 결합 샘플을 생성하는 방법. 반복적인 수렴 과정 없이 한 번에 샘플을 얻을 수 있다.

## 🤖 AI의 생각


이 논문은 딱 봐도 고전인데, taskcraft 입장에서 흥미로운 건 이 논문이 풀고 있는 문제가 사실 embodiment 무관 표현이라는 목표의 아주 축소된 버전이라는 점이다. VAE는 이미지의 픽셀 디테일을 벗겨내고 압축된 $$z$$를 만드는데, 이게 taskcraft가 원하는 사람 손과 로봇 관절 사이의 표면적 차이를 벗겨낸 latent와 방향성이 닮아 있다고 느껴진다.

<details class="tc-faq">
<summary>KL 정규화가 만드는 압축이 정말로 embodiment와 무관한 표현을 만들어낼까</summary>
<div class="tc-faq__body" markdown="1">

이 논문의 목적함수만 보면 그런 보장은 없고, 재구성에 필요한 정보만 남기라는 압력일 뿐이라서 taskcraft에 그대로 가져다 쓰려면 별도의 unsupervised한 해석성 제약이 더 필요해 보인다

</div>
</details>

<details class="tc-faq">
<summary>재파라미터화 트릭을 정적 이미지가 아니라 시퀀스 형태의 latent dynamics에 적용하면 어떤 문제가 생길까</summary>
<div class="tc-faq__body" markdown="1">

매 타임스텝마다 노이즈를 독립적으로 재파라미터화하면 시간축에 걸친 일관성이 깨질 수 있어 DreamerV3류 모델들이 어떻게 이 문제를 우회했는지 다음에 따로 짚어볼 필요가 있다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 human demonstration 인코더와 embodiment별 디코더를 이 논문의 AEVB 구조 그대로 학습시킬 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

구조는 거의 동형이지만 taskcraft는 재구성이 아니라 여러 디코더 간의 일관성이 목표라서 손실함수 자체를 재구성 오차에서 다른 형태로 바꿔야 할 것 같다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/1312.6114v11" target="_blank" rel="noopener">Auto-Encoding Variational Bayes</a> · arxiv</span></div>
</div>
</div>
