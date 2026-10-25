---
title: "감각과 운동이 통하면 로봇도 잘한다"
date: 2026-10-25
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "로봇 정책을 실제로 돌려보지 않고도, 시각·언어 표현과 동작 궤적의 정렬도만으로 성공률을 예측하는 지표를 살펴본다."
paper_title: "Sensorimotor Alignment: A Platonic Proxy for Embodied Policy Success"
paper_summary: "로봇 정책을 실제로 돌려보지 않고도, 시각·언어 표현과 동작 궤적의 정렬도만으로 성공률을 예측하는 지표를 살펴본다."
paper_url: "https://arxiv.org/abs/2610.31670v1"
header:
  image: "/assets/images/posts/sensorimotor-alignment-a-platonic-proxy-for-embodied-policy-success/figure-1.png"
  teaser: "/assets/images/posts/sensorimotor-alignment-a-platonic-proxy-for-embodied-policy-success/figure-1.png"
---

## 이번 주 이 논문을 고른 이유

이번 주 스코어링 점수는 0.50으로 그리 높지 않습니다. 하지만 이 논문을 고른 이유는 점수보다 시의성에 있습니다. VLA(Vision-Language-Action) 모델이 쏟아지는 지금, "이 모델이 로봇 팔을 실제로 잘 움직일지"를 매번 로봇을 돌려서 확인하는 건 비효율적입니다. 이 논문은 그 문제에 정면으로 답을 시도하면서, 동시에 저희 taskcraft 프로젝트가 다루는 "좋은 embodied representation이란 무엇인가"라는 질문과도 맞닿아 있어 학습 목적 글로 다루기 좋다고 판단했습니다.

## 폐루프 평가라는 벽

Imitation Learning이나 RL을 수업에서 배울 때는 보통 정책을 시뮬레이터에 올려 직접 굴려보고 성공률을 센다. 이걸 폐루프(closed-loop) 평가라고 부른다. 로봇이 매 스텝 관측을 받아 행동을 내고, 그 행동이 다시 다음 관측에 영향을 주는 루프를 실제로 돌려야 하기 때문이다. 문제는 VLA 시대로 오면서 평가 대상이 폭발적으로 늘었다는 점이다. 거대한 Vision-Language 모델을 로봇 데이터로 파인튜닝한 모델이 매주 쏟아지는데, 이걸 전부 실물 로봇이나 고비용 시뮬레이터에 올려서 성공률을 재는 건 시간과 장비 양쪽에서 감당이 안 된다.

그래서 연구자들은 흔히 우회로를 택해왔다. VQA 벤치마크 점수나 비디오 생성 품질(FID, SSIM, PSNR) 같은, 모델을 굴리지 않고도 계산할 수 있는 오프라인 지표로 대신 줄 세우는 것이다. 그런데 이 논문이 지적하는 핵심은, 이런 지표들이 로봇 제어 성공률과 상관관계가 거의 없거나 심지어 음의 상관관계를 보인다는 점이다. 이미지를 잘 설명하고 비디오를 예쁘게 생성하는 능력과, 실제로 그리퍼를 열고 물체를 집는 능력은 서로 다른 문제라는 뜻이다. 수동적으로 세상을 지각하는 능력(passive perception)과 행동을 조직화하는 능력(action-oriented representation)이 다르다는 건데, 생각해보면 당연하다. 사람도 그림을 잘 설명한다고 손놀림이 좋은 건 아니다.

## 플라톤적 표현 가설을 로봇에 옮기면

이 논문은 최근 표현 학습 쪽에서 화제가 된 PRH(Platonic Representation Hypothesis, 플라톤적 표현 가설)를 끌어온다. PRH의 주장은, 서로 다른 모달리티(이미지, 텍스트 등)로 학습한 모델이라도 데이터와 모델 규모가 커지면 결국 하나의 공통된 물리적 현실을 가리키는 표현으로 수렴한다는 것이다. 플라톤의 동굴 비유처럼, 벽에 비친 그림자(이미지)와 소리(언어)는 서로 다른 감각 정보지만 결국 같은 동굴 밖 실재를 가리킨다는 식이다.

이 논문의 아이디어는 단순하다. 로봇의 물리적 동작 궤적도 또 하나의 "현실의 투영"으로 보자는 것이다. 카메라가 찍은 이미지와 지시문이 현실을 감각적으로 투영한 것이라면, 로봇 팔이 실제로 그린 궤적은 같은 현실을 운동학적으로 투영한 것이다. 그렇다면 능력 있는 로봇 정책일수록, 두 투영이 서로 "같은 구조"를 가지고 있어야 한다는 가설을 세울 수 있다. 비슷하게 생긴 장면과 지시문을 받았을 때는 비슷한 궤적을 그려야 한다는 뜻이다. 이 일치도를 저자들은 Sensorimotor Alignment, 줄여서 $$S^2$$라고 부른다.

![PRH 개념도와 VLA 모델들의 S^2 vs 성공률 상관관계](/assets/images/posts/sensorimotor-alignment-a-platonic-proxy-for-embodied-policy-success/figure-1.png)

## S²는 구체적으로 어떻게 재는가

측정 방식 자체는 복잡하지 않다. 먼저 감각 표현을 만든다. 임의의 시점에서 찍힌 RGB 이미지 프레임과 그 순간의 자연어 지시문을 인코더에 통과시켜 하나의 벡터 $$z_i$$로 압축한다. 다음은 운동 표현이다. 같은 시점 이후 $$h$$ 스텝 동안 엔드이펙터(로봇 팔 끝, 보통 그리퍼가 달린 부분)가 실제로 그린 궤적을 기록한다. 3차원 위치, 회전을 나타내는 쿼터니언, 그리퍼 개폐 상태까지 포함한 벡터 시퀀스다.

이제 두 공간에서 각각 "이웃"을 찾는다. 감각 공간에서는 코사인 거리로 가장 비슷한 장면 $$k$$개를 찾고, 운동 공간에서는 DTW(Dynamic Time Warping) 거리로 가장 비슷한 궤적 $$k$$개를 찾는다. 만약 감각적으로 비슷하다고 판단된 에피소드들이 운동적으로도 비슷한 궤적을 그렸다면, 두 이웃 집합이 많이 겹칠 것이다. 이 겹침 비율이 기본 직관이다.

$$m_{\text{NN}} = \frac{1}{Nk} \sum_{i=1}^N \left\vert{} \mathcal{N}_{\text{sensory}}(i) \cap \mathcal{N}_{\text{motor}}(i) \right\vert{}$$

여기서 $$\mathcal{N}_{\text{sensory}}(i)$$와 $$\mathcal{N}_{\text{motor}}(i)$$는 각각 감각 공간과 운동 공간에서 에피소드 $$i$$와 가장 가까운 $$k$$개의 이웃 집합이다. 실제 논문에서는 이 단순 겹침 비율을 CKNNA(centered kernel nearest neighbor alignment)라는, 서로 다른 표현 공간의 정렬도를 재는 데 쓰이던 통계적 도구로 정규화해서 최종 $$S^2$$를 만든다. 핵심은, 이 전체 과정이 추가 학습 없이 기존 오프라인 데이터셋만으로 계산 가능하다는 점이다(training-free). 모델을 시뮬레이터나 로봇에 올릴 필요가 전혀 없다.

## 결과: 실제로 맞아떨어지는가

핵심 결과는 12종의 VLA 모델에 대해 오프라인에서 계산한 $$S^2$$와, SimplerEnv 시뮬레이터에서 실제로 폐루프 롤아웃을 돌려 측정한 성공률 사이의 상관관계다. 피어슨 상관계수 $$r = +0.717$$, $$p = 0.004$$로 꽤 강한 양의 선형 관계가 나왔다. 즉 로봇을 한 번도 굴리지 않고도, $$S^2$$ 값만 보고 어떤 모델이 더 잘 움직일지 상당히 정확하게 줄 세울 수 있다는 뜻이다.

더 흥미로운 건 이 상관관계가 한 가지 세팅에만 국한되지 않는다는 점이다. 논문은 파인튜닝된 VLA뿐 아니라 파인튜닝 이전의 순수 VLM 백본($$r=+0.681$$), 비디오 기반 World Action Model($$r=+0.718$$), 심지어 실물 Franka 로봇에서 수집한 정책 데이터($$r=+0.831$$)까지 네 가지 서로 다른 영역에서 일관되게 비슷한 수준의 상관관계를 재현한다.

![네 가지 다른 모델군과 실로봇 환경에서의 S^2 대 성공률 상관관계](/assets/images/posts/sensorimotor-alignment-a-platonic-proxy-for-embodied-policy-success/figure-2.png)

이건 두 가지를 동시에 말해준다. 하나는 $$S^2$$가 특정 시뮬레이터나 특정 네트워크 구조에 붙은 트릭이 아니라는 것. 다른 하나는, 파인튜닝을 시작하기도 전에 VLM 백본 단계에서부터 "이 모델이 로봇 행동 쪽으로 잘 이어질 잠재력이 있는지"를 가늠할 수 있다는 것이다. 모델 선정 단계의 비용을 크게 줄일 수 있는 지점이다.

## 한계와 taskcraft와의 접점

다만 이 논문이 다루는 정렬은 어디까지나 하나의 신체(엔드이펙터 궤적) 안에서의 정렬이다. 서로 다른 로봇 팔, 혹은 인간 시연과 로봇 실행처럼 신체 구조 자체가 다른 경우에 감각과 운동이 정렬되는지는 다루지 않는다. 또한 상관계수가 0.7에서 0.8 사이로 강하긴 해도 완벽한 예측은 아니라서, 오프라인 지표가 폐루프 평가를 완전히 대체한다고 보기는 이르다. 저자들도 이를 "스크리닝용 프록시"로 자리매김한다.

taskcraft 프로젝트 입장에서는 이 지표가 특히 유용해 보인다. taskcraft는 latent world model을 매개로 삼아 특정 로봇의 관절 구조에 덜 종속적인 task representation을 뽑아내려는 가설을 세우고 있는데, 그렇게 뽑아낸 latent가 실제로 "행동 실행에 쓸모 있는 표현"인지, 아니면 시각적으로만 그럴듯한 겉치레인지를 로봇을 굴리지 않고도 먼저 걸러낼 도구가 될 수 있다. 다만 이 논문 자체가 서로 다른 embodiment 간의 이식 가능성까지 검증한 건 아니라서, taskcraft가 풀어야 할 "신체가 바뀌어도 정렬이 유지되는가"라는 질문은 여전히 열려 있다.

## 용어 해설

- **DTW (Dynamic Time Warping)**: 길이가 다르거나 속도가 다른 두 시계열 데이터를 비교할 때, 시간축을 유연하게 늘이거나 줄여가며 가장 비슷한 대응 구간을 찾아 거리를 계산하는 알고리즘이다. 같은 동작이라도 빠르게 하거나 천천히 한 두 궤적을 공평하게 비교할 때 쓴다.
- **HSIC (Hilbert-Schmidt Independence Criterion)**: 두 변수(또는 두 표현 공간) 사이의 통계적 독립성 여부를 커널 함수를 이용해 측정하는 지표로, 값이 클수록 두 변수가 서로 관련이 있다는 뜻이다. 선형 상관계수로는 잡기 힘든 비선형적인 연관성까지 포착할 수 있다.
- **쿼터니언 (quaternion)**: 3차원 공간에서 물체의 회전을 표현하는 4개의 숫자 조합으로, 오일러 각이나 회전 행렬보다 계산이 안정적이어서 로봇공학과 컴퓨터 그래픽스에서 널리 쓰인다.
- **k-최근접 이웃 (k-Nearest Neighbors)**: 어떤 데이터 포인트에 대해, 정의된 거리 기준으로 가장 가까운 $$k$$개의 데이터를 찾는 방법이다. 이 논문에서는 두 표현 공간 각각에서 이웃을 찾아 그 겹침 정도로 정렬도를 잰다.

## 🤖 AI의 생각

제 생각에 이 논문은 평가 지표 논문의 외피를 쓰고 있지만, 실은 "좋은 embodied representation이란 무엇인가"에 대한 하나의 조작적 정의를 제시하고 있다고 느껴집니다. 감각과 운동의 이웃 구조가 겹쳐야 한다는 조건은, 표현 학습 단계에서부터 명시적인 보조 목표(auxiliary objective)로 써볼 여지가 있어 보입니다.

<details class="tc-faq">
<summary>\(S^2\)가 높은 모델이 실제로 왜 더 잘 작동하는지, 즉 상관관계 너머의 인과적 메커니즘은 무엇일까</summary>
<div class="tc-faq__body" markdown="1">

논문은 상관관계만 보였을 뿐 왜 정렬도가 성공률의 원인이 되는지는 설명하지 않아서, 정렬도를 직접 손실 함수로 넣어 학습했을 때도 같은 효과가 나는지 확인해볼 가치가 있다.

</div>
</details>

<details class="tc-faq">
<summary>서로 다른 로봇 팔이나 인간 시연처럼 신체 구조가 아예 다른 경우에도 \(S^2\) 같은 정렬 지표를 정의할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

taskcraft가 풀려는 embodiment 간 이식 문제와 정확히 맞닿는 지점인데, 이 경우 운동 표현 자체를 신체에 독립적인 좌표계로 먼저 정규화해야 DTW 비교가 의미를 가질 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>파인튜닝 전 VLM 백본 단계에서도 상관관계가 유지된다면, \(S^2\)를 모델 선택이 아니라 사전학습 데이터 큐레이션 기준으로 쓸 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

어떤 사전학습 데이터가 감각 운동 정렬을 더 잘 만드는지 거꾸로 추적하면, 로봇 파인튜닝에 가장 유리한 비디오 데이터셋을 미리 골라내는 데 쓸 수 있을 것 같다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2610.31670v1" target="_blank" rel="noopener">Sensorimotor Alignment: A Platonic Proxy for Embodied Policy Success</a> · arxiv</span></div>
</div>
</div>
