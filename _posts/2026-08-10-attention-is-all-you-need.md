---
title: "어텐션 하나로 충분했던 순간"
date: 2026-08-10
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "RNN과 CNN을 걷어내고 셀프 어텐션만으로 시퀀스를 처리한 트랜스포머 구조를 기초부터 정리한다."
paper_title: "Attention Is All You Need"
paper_summary: "RNN과 CNN을 걷어내고 셀프 어텐션만으로 시퀀스를 처리한 트랜스포머 구조를 기초부터 정리한다."
paper_url: "https://arxiv.org/abs/1706.03762v7"
header:
  image: "/assets/images/posts/attention-is-all-you-need/figure-1.png"
  teaser: "/assets/images/posts/attention-is-all-you-need/figure-1.png"
---

이번 주 논문 노트로 Attention Is All You Need를 골랐습니다. score=1.00으로 스코어링됐는데, 사실 이 논문은 2017년에 나온 고전이라 "이번 주 트렌드"라고 부르기엔 조금 애매합니다. 하지만 최근 VLA나 world model 계열 논문들을 읽다 보면 결국 인코더 구조의 밑바탕에는 이 논문에서 나온 self-attention이 항상 깔려 있다는 걸 계속 마주치게 됩니다. taskcraft 프로젝트에서 latent world model을 설계하려면 이 구조를 제대로 이해하고 넘어가야 한다는 판단이 들어서, 이번 주는 최신 흐름을 좇기보다 기초를 다지는 쪽으로 방향을 잡았습니다.

## 순차 계산이라는 병목

2017년 이전까지 시퀀스를 다루는 모델의 기본값은 RNN이었다. LSTM이든 GRU든, 문장이나 시계열 데이터를 처리할 때는 토큰을 하나씩 순서대로 읽으면서 hidden state를 갱신해 나가는 방식을 썼다. 이 방식의 문제는 두 가지였다.

첫째, 구조 자체가 순차적이라서 GPU의 장점인 병렬 연산을 시간 축에서는 쓸 수 없었다. 열 번째 토큰을 처리하려면 아홉 번째 토큰의 결과가 반드시 먼저 나와야 하니, 시퀀스가 길어질수록 학습 시간이 그대로 늘어난다.

둘째, 장거리 의존성 문제다. 문장 맨 앞의 단어와 맨 뒤의 단어가 서로 관련이 있어도, RNN은 그 정보를 hidden state 하나에 계속 압축해서 들고 가야 한다. 거리가 멀어질수록 정보가 희석되거나 gradient가 소실되어서, 먼 단어 사이의 관계를 학습하기가 어려워진다. CNN 기반 모델들(ConvS2S, ByteNet)은 병렬 처리는 가능했지만, 먼 토큰 사이의 관계를 보려면 레이어를 여러 겹 쌓아야 했다. 결국 둘 다 근본적인 해결책은 아니었다.

이 논문의 답은 단순하다. 순환도, 합성곱도 다 빼고 어텐션만 남기자는 것이다.

## Self-Attention이 뭔가: Query, Key, Value부터

self-attention의 직관은 이렇다. 시퀀스 안의 모든 토큰이 서로 한 번씩 직접 대화를 나누도록 만든다. RNN처럼 이웃한 토큰끼리만 순서대로 정보를 넘기는 게 아니라, 문장 맨 앞의 단어와 맨 뒤의 단어가 중간 단계를 거치지 않고 곧바로 서로의 관련도를 계산한다.

이걸 계산하기 위해 각 토큰은 세 개의 벡터로 표현된다. Query는 "내가 지금 무엇을 찾고 있는가"를 나타내는 벡터, Key는 "나는 이런 정보를 갖고 있다"를 나타내는 식별용 벡터, Value는 실제로 전달할 내용 벡터다. 논문의 수식은 다음과 같다.

$$\text{Attention}(Q, K, V) = \text{softmax}\left(\frac{QK^T}{\sqrt{d_k}}\right)V$$

$$QK^T$$는 모든 Query와 모든 Key 사이의 내적을 한꺼번에 계산해서, 토큰 쌍마다 얼마나 관련이 있는지를 점수로 만든다. 이 점수를 softmax에 통과시켜 확률 분포처럼 만든 다음, 그 가중치로 Value들을 가중합하면 각 토큰이 다른 모든 토큰의 정보를 자기 위치에 맞게 섞어서 가져오는 셈이 된다.

## 왜 하필 $$\sqrt{d_k}$$로 나누는가

식을 보면 굳이 $$\sqrt{d_k}$$로 나누는 스케일링이 들어가 있다. 이건 순전히 수치적인 이유다. Key 벡터의 차원 $$d_k$$가 커지면 내적 $$QK^T$$의 값도 커지는데, softmax는 입력값이 커질수록 특정 항목에 확률이 몰리면서 나머지 항목의 gradient가 거의 0에 가까워지는 경향이 있다. 즉 학습이 잘 안 되는 영역으로 쉽게 밀려난다는 뜻이다. 차원 수의 제곱근으로 나눠서 내적값의 분산을 일정하게 유지해 주면 이 문제를 피할 수 있다. 별거 아닌 것 같은 트릭이지만, 이게 없으면 큰 모델에서 학습이 잘 안 된다.

## 여러 개의 시선으로 보기: Multi-Head Attention

attention을 한 번만 계산하면 토큰 사이의 관계를 딱 한 가지 기준으로만 보게 된다. 하지만 언어든 영상이든 토큰 사이 관계는 여러 층위로 존재한다. 문법적 관계, 의미적 관계, 위치적 관계가 동시에 있을 수 있다. 그래서 논문은 Query, Key, Value를 여러 개의 서로 다른 부분공간으로 나눠 투영한 뒤, 각각에서 독립적으로 attention을 계산하고 결과를 이어붙인다.

$$\text{MultiHead}(Q, K, V) = \text{Concat}(\text{head}_1, \dots, \text{head}_h)W^O$$

논문에서는 $$h=8$$개의 head를 썼다. 각 head가 서로 다른 관점에서 토큰 관계를 학습하고, 마지막에 이걸 합쳐서 다시 하나의 표현으로 만드는 구조다.

## 순서 정보는 어떻게 알려주나: Positional Encoding

여기서 한 가지 문제가 생긴다. self-attention은 모든 토큰을 동시에, 순서 무관하게 다 연결해서 계산하기 때문에 원래는 "몇 번째 토큰인지"를 전혀 모른다. RNN은 순서대로 읽으니 위치 정보가 구조에 자연히 들어가지만, attention은 그런 게 없다. 그래서 이 논문은 입력 임베딩에 위치 정보를 담은 벡터를 더해준다.

$$PE_{(pos, 2i)} = \sin\left(\frac{pos}{10000^{2i/d_{model}}}\right), \quad PE_{(pos, 2i+1)} = \cos\left(\frac{pos}{10000^{2i/d_{model}}}\right)$$

차원마다 주기가 다른 sin, cos 함수를 쓰는 이유는, 이렇게 하면 두 위치 사이의 상대적 거리가 선형 변환으로 표현될 수 있어서 모델이 "몇 칸 떨어져 있다"는 관계를 더 쉽게 학습할 수 있기 때문이다. 학습 가능한 위치 임베딩을 쓰는 대안도 있었지만, 이 방식은 학습 데이터보다 긴 시퀀스에도 그대로 적용할 수 있다는 장점이 있다.

## 전체 구조 조립하기

이 모든 조각을 합치면 인코더, 디코더 각각 6개의 동일한 레이어를 쌓은 구조가 된다. 각 레이어는 multi-head self-attention과 position-wise feed-forward network로 구성되고, 그 사이사이에 residual connection과 layer normalization이 들어간다. 디코더에는 한 가지가 더 추가되는데, 미래 토큰을 미리 보면 안 되니까 self-attention 계산에서 아직 나오지 않은 미래 위치를 마스킹으로 가려버리는 masked self-attention을 쓴다. 그리고 인코더의 출력을 받아서 참조하는 encoder-decoder attention 레이어가 하나 더 들어간다.

![트랜스포머 전체 아키텍처, 인코더와 디코더의 레이어 구조](/assets/images/posts/attention-is-all-you-need/figure-1.png)

이 그림을 보면 입력이 임베딩과 positional encoding을 더한 채로 인코더 스택에 들어가고, 인코더의 최종 출력이 디코더의 각 레이어에 반복적으로 참조되는 흐름을 확인할 수 있다.

![Scaled Dot-Product Attention과 Multi-Head Attention의 내부 연산 흐름](/assets/images/posts/attention-is-all-you-need/figure-2.png)

왼쪽은 $$Q, K, V$$가 내적, 스케일링, 마스킹, softmax를 거쳐 최종적으로 Value와 다시 곱해지는 과정을 보여주고, 오른쪽은 이 과정이 여러 head로 나뉘어 병렬로 돌고 마지막에 다시 합쳐지는 구조를 보여준다.

## 남는 한계

이 구조가 만능은 아니다. self-attention은 모든 토큰 쌍의 관계를 다 계산하기 때문에 연산량이 시퀀스 길이의 제곱에 비례해서 늘어난다. 문장 수준에서는 문제가 안 되지만, 영상 프레임처럼 시퀀스가 매우 길어지는 도메인에서는 이게 그대로 병목이 된다. 또 positional encoding이 절대 위치를 명시적으로 인코딩하는 방식이라서, 입력 도메인이 바뀌었을 때 "위치"라는 개념 자체가 달라지면 그대로 재사용하기 어려운 지점도 있다. 이런 부분들은 이후 논문들(상대적 위치 인코딩, sparse attention 등)에서 계속 보완되어 왔다.

## taskcraft와의 관련성

이 논문 자체는 embodiment나 imitation learning을 다루지 않는다. 하지만 taskcraft가 상정하는 latent world model의 구현체는 결국 이 아키텍처에 크게 의존할 가능성이 높다. VPT의 IDM이나 STEVE-1의 정책 백본, GROOT의 영상 인코더 모두 self-attention 기반 구조를 쓰고 있고, taskcraft에서 "영상에서 행동이 세계 상태를 어떻게 바꾸었는가"를 latent로 뽑아내려면 시퀀스 전체에 걸친 장거리 의존성(먼 프레임 간 관계)을 다뤄야 하는데, 이건 이 논문이 풀었던 문제와 같은 종류다. 다만 taskcraft의 핵심 질문인 embodiment 무관 표현 이식은 트랜스포머가 직접 답을 주는 문제는 아니고, 트랜스포머는 그 표현을 뽑아내는 도구 중 하나일 뿐이다.

## 용어 해설

- **기울기 소실(Gradient vanishing)**: 신경망을 역전파로 학습시킬 때 층이나 시간 축을 거칠수록 gradient 값이 점점 작아져서 앞쪽(혹은 먼 과거) 파라미터가 거의 학습되지 않는 현상.
- **자기회귀적(Auto-regressive)**: 모델이 출력을 생성할 때 이전 시점까지 만들어진 출력을 다시 입력으로 사용해서 다음 출력을 순차적으로 만들어내는 방식.
- **잔차 연결(Residual Connection)**: 어떤 레이어의 출력에 그 레이어의 입력을 그대로 더해주는 구조로, 깊은 네트워크에서도 gradient가 잘 흐르도록 돕는다.
- **레이어 정규화(Layer Normalization)**: 한 샘플 내에서 레이어의 출력값들을 평균과 분산 기준으로 정규화해 학습을 안정시키는 기법.

## 🤖 AI의 생각


이 논문은 이미 8년 가까이 지난 고전이지만, taskcraft 노트를 다시 읽고 나니 왜 지금 다시 봐야 했는지 알 것 같다는 느낌이 든다. 이건 사실 요약이 아니라 개인적인 인상이다.

<details class="tc-faq">
<summary>self-attention의 $$O(n^2)$$ 연산량 문제는 영상 프레임 시퀀스처럼 긴 입력을 다루는 world model 설계에서 얼마나 심각한 제약이 될까</summary>
<div class="tc-faq__body" markdown="1">

프레임 수가 수백 개를 넘어가면 그대로 병목이 될 수 있어서 sparse attention이나 windowed attention 같은 대안을 taskcraft 인코더 설계 단계에서 미리 검토해볼 가치가 있다고 본다.

</div>
</details>

<details class="tc-faq">
<summary>positional encoding이 절대 위치를 명시적으로 주입하는 방식인데, embodiment마다 시간 축의 의미(프레임 속도, 동작 스케일)가 다른 taskcraft 상황에도 그대로 맞을까</summary>
<div class="tc-faq__body" markdown="1">

그대로 가져다 쓰기엔 다소 부적합해 보이고, 오히려 상대적 위치나 이벤트 기반 인코딩 쪽으로 변형이 필요할 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>multi-head attention에서 각 head가 실제로 서로 다른 종류의 관계를 학습하는지는 어떻게 검증할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

attention weight를 head별로 시각화해서 어떤 head가 문법적 관계에, 어떤 head가 위치적 관계에 반응하는지 비교해보는 사후 분석이 가능할 것 같다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/1706.03762v7" target="_blank" rel="noopener">Attention Is All You Need</a> · arxiv</span></div>
</div>
</div>
