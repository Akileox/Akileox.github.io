---
title: "행동을 언어처럼 예측하는 로봇, RT-2"
date: 2026-08-16
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "웹 지식으로 학습한 비전-언어 모델이 로봇 행동까지 텍스트 토큰으로 직접 뱉어내게 만든 RT-2를 처음부터 뜯어본다."
paper_title: "RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control"
paper_summary: "웹 지식으로 학습한 비전-언어 모델이 로봇 행동까지 텍스트 토큰으로 직접 뱉어내게 만든 RT-2를 처음부터 뜯어본다."
paper_url: "https://arxiv.org/abs/2307.15818v1"
header:
  image: "/assets/images/posts/rt-2-vision-language-action-models-transfer-web-knowledge-to-robotic-control/figure-1.png"
  teaser: "/assets/images/posts/rt-2-vision-language-action-models-transfer-web-knowledge-to-robotic-control/figure-1.png"
---

이번 주 골라온 논문은 RT-2입니다. VLA(Vision-Language-Action) 계열의 사실상 출발점 격인 논문이고, Imitation Learning이나 RL은 수업에서 다뤘지만 그 이후 로봇 학습이 어떻게 언어모델과 합쳐졌는지는 처음 접하는 분들께 딱 좋은 진입점이라 스코어 1.00을 주고 이번 주 소재로 선택했습니다. 개념 하나하나를 짚어가면서 왜 이런 설계가 필요했는지부터 차근차근 풀어보겠습니다.

## 로봇 학습이 원래 갖고 있던 문제

Imitation Learning 수업에서 배운 Behavior Cloning을 떠올려보자. 전문가(사람 또는 스크립트)가 수행한 궤적 데이터, 즉 (관찰, 행동) 쌍을 잔뜩 모아서 지도학습으로 정책을 학습시키는 방식이다. 이 방식의 근본적인 약점은 학습에 쓰인 데이터 분포를 벗어나면 성능이 급격히 떨어진다는 점이다. 로봇 팔이 학습 때 본 적 없는 컵 모양, 처음 보는 배경, 낯선 조명 조건을 만나면 정책이 헷갈려한다.

그런데 왜 이게 유독 로봇에서 심각한 문제일까. 언어모델이나 이미지 분류 모델은 인터넷에서 긁어온 수십억 장 규모의 데이터로 학습되어 웬만한 물체와 개념을 이미 알고 있다. 반면 로봇 궤적 데이터는 실제 로봇 팔을 움직여서 모아야 하니 물리적으로 시간과 비용이 많이 들고, 그래서 데이터 규모가 언어/비전 데이터에 비해 몇 자릿수는 작다. 결국 로봇 정책은 "이 물체가 뭔지" 자체는 모르는 채로, 좁은 도메인 안에서 행동만 흉내내는 셈이 된다.

그래서 자연스럽게 나온 아이디어가 "그럼 웹 스케일로 학습된 비전-언어 모델(VLM)의 지식을 로봇 제어에 빌려오면 되지 않나"였다. VLM은 이미지와 텍스트를 함께 이해하도록 학습된 거대 모델로, 사진을 보고 "이건 사과다", "이 두 물체 중 마실 수 있는 건 콜라 캔이다" 같은 개방형 질문에 답할 수 있다. 문제는 이 VLM이 결국 텍스트 토큰을 출력하는 모델이라는 점이다. 로봇 팔을 몇 센티미터 움직이라는 연속적인 숫자 명령을 어떻게 텍스트 생성 모델에서 뽑아낼 것인가가 핵심 난관이었다.

## 기존 시도들이 왜 부족했나

RT-2 이전에도 VLM을 로봇에 끌어오려는 시도는 있었다. 대표적으로 두 갈래다. 하나는 VLM을 상위 플래너로만 쓰는 방식이다. VLM이 "컵을 집어서 테이블 위에 놓아라"처럼 자연어 계획을 세우면, 그 계획을 실행하는 저수준 제어기는 완전히 별도로 학습된 작은 정책이 담당한다. 이 방식은 VLM의 추론 능력을 계획 단계에서만 쓰고 실제 행동 생성에는 관여시키지 않으니, VLM이 갖고 있는 시각적 의미 이해가 정작 손끝의 움직임까지 전달되지 않는다.

다른 하나는 VLM을 표현 추출기로만 쓰는 방식이다. VLM의 중간 레이어 출력(이미지의 latent 표현)을 뽑아서 별도의 작은 행동 헤드에 넣어주는 식이다. 이 역시 VLM 자체는 그대로 얼려두고 위에 작은 모듈만 새로 학습시키는 구조라, VLM이 가진 방대한 지식과 로봇 행동 정책 사이에 명확한 경계선이 그어져 있다. 결국 두 접근 모두 "웹 지식"과 "로봇 행동"을 분리된 두 개의 파이프라인으로 다루고, 그 사이를 이어주는 얇은 다리만 놓았을 뿐이었다.

## RT-2가 고친 방법: 행동을 텍스트 토큰으로

RT-2의 발상은 단순하지만 과감하다. 로봇 행동 자체를 언어모델이 평소에 다루는 텍스트 토큰과 똑같은 형식으로 바꿔버리자는 것이다. 로봇 팔의 한 스텝 동작은 엔드이펙터(로봇 팔 끝, 그리퍼가 달린 부분)의 3차원 위치 변위, 3차원 회전 변위, 그리퍼를 얼마나 열고 닫을지, 그리고 에피소드를 끝낼지 여부까지 총 8개의 숫자로 표현된다. RT-2는 이 각각의 숫자를 256개의 구간으로 균일하게 쪼개서(양자화) 정수 인덱스로 바꾸고, 그 정수들을 이어붙인 문자열을 만든다.

$$\text{Target} = \text{"terminate } \Delta\text{pos}_x\ \Delta\text{pos}_y\ \Delta\text{pos}_z\ \Delta\text{rot}_x\ \Delta\text{rot}_y\ \Delta\text{rot}_z\ \text{gripper\_extension"}$$

이렇게 만든 문자열은 그냥 언어모델 입장에서는 다른 텍스트 토큰과 구분되지 않는다. 그래서 VLM 백본(PaLI-X, PaLM-E 같은 기존 거대 모델)에 별도의 행동 전용 레이어를 새로 붙일 필요가 없어진다. 모델 전체가 하나의 가중치 집합을 공유한 채로, "이 이미지를 보고 무슨 물체가 있는지 설명해라" 같은 질문에도 답하고 "이 이미지를 보고 로봇이 다음에 어떤 행동을 해야 하는지 말해라"라는 질문에도 똑같은 방식으로 답하게 되는 것이다.

여기서 자연스럽게 떠오르는 걱정이 있다. 로봇 궤적 데이터로만 미세조정을 계속하면 모델이 원래 갖고 있던 웹 지식을 잊어버리지 않을까. 실제로 이런 현상을 파국적 망각(Catastrophic forgetting)이라 부르는데, RT-2는 이를 막기 위해 로봇 데이터와 원래 VLM이 학습했던 웹 스케일 VQA/캡셔닝 데이터를 일정 비율(로봇 데이터 50~66%)로 섞어서 함께 학습시킨다. 이걸 공동 미세조정(Co-fine-tuning)이라 부른다. 즉 로봇 데이터만 파는 게 아니라, 웹 데이터를 계속 곁들여 먹이면서 로봇 행동도 배우게 하는 셈이다.

학습 손실 함수 자체는 특별할 게 없다. 표준적인 자기회귀 다음 토큰 예측이다.

$$\mathcal{L}_{\text{BC}} = -\sum_{t=1}^{T} \log P(y_t \mid y_{<t}, I, x_{\text{prompt}})$$

여기서 $$I$$는 카메라 이미지, $$x_{\text{prompt}}$$는 "다음에 어떤 행동을 해야 하나"라는 자연어 질문, $$y_t$$는 그에 대한 답변 토큰이다. 답변이 자연어 문장일 수도 있고 방금 본 행동 토큰 문자열일 수도 있다는 점만 다르다. 결국 RT-2는 Behavior Cloning이라는 알고리즘 자체를 바꾼 게 아니라, 그 알고리즘이 다루는 표현 공간을 언어모델의 어휘 공간으로 통째로 옮겨버린 것이다.

또 하나 실용적으로 중요한 디테일은 사고의 연쇄(Chain-of-Thought)를 흉내낸 데이터 증강이다. 모델이 바로 행동 토큰을 뱉기 전에 "Plan: 컵을 잡아서 오른쪽으로 옮긴다" 같은 자연어 계획을 먼저 말하게 학습시킨 것이다. 이렇게 하면 복잡한 다단계 지시를 받았을 때 모델이 중간 추론 과정을 거쳐서 더 안정적인 행동을 뽑아낼 수 있다.

![RT-2 전체 파이프라인, 웹 데이터와 로봇 데이터를 같은 포맷으로 넣어 VLM이 행동 토큰까지 출력하게 하는 구조](/assets/images/posts/rt-2-vision-language-action-models-transfer-web-knowledge-to-robotic-control/figure-1.png)

## 결과: 얼마나 좋아졌나

이 논문에서 인상적인 부분은 단순히 "성공률이 올랐다"가 아니라 어떤 종류의 능력이 새로 생겼는지를 명확히 구분해서 보여준다는 점이다. 논문은 성능을 두 갈래로 나눠 측정한다. 하나는 학습 데이터에 있던 것과 비슷한 물체/과제에서의 성공률이고, 다른 하나는 발현적 능력(Emergent capabilities)이라 부르는, 학습 데이터에는 전혀 없던 개방형 능력이다. 예를 들어 "빨간 물체를 집어라"라는 지시 없이도 색깔 기호를 이해하거나, 유명인의 사진을 보고 누구인지 알아보고 그 사람과 관련된 물체를 집어오는 식이다.

이 발현적 능력 평가에서 RT-2는 기존 베이스라인(RT-1, VC-1) 대비 성공률이 3배 이상 높았다(60% 대 17% 수준). 즉 로봇 궤적 데이터에는 전혀 없던 개념을 웹 지식에서 그대로 끌어와 로봇 행동에 반영할 수 있다는 뜻이다. 또한 모델 크기를 5B에서 55B로 키우고, 로봇 데이터만으로 미세조정하는 대신 웹 데이터를 계속 섞은 공동 미세조정을 적용했을 때 일반화 성능이 뚜렷하게 좋아졌다. 처음부터 로봇 데이터로만 학습시킨 모델은 이런 발현적 능력을 전혀 보여주지 못했는데, 이는 웹 지식을 유지하는 것 자체가 일반화의 핵심 동력이었음을 보여준다.

![모델 크기와 공동 미세조정 여부에 따른 발현적 능력 비교, 기호 이해와 추론 항목에서 RT-2가 기존 베이스라인을 크게 앞선다](/assets/images/posts/rt-2-vision-language-action-models-transfer-web-knowledge-to-robotic-control/figure-2.png)

물론 한계도 있다. 55B급 모델을 실시간으로 돌리려면 멀티 TPU 클라우드 서비스에 분산 배치해야 하고, 그마저도 1~5 Hz 수준의 느린 폐루프 제어밖에 안 나온다. 빠른 동작이 필요한 과제에는 그대로 쓰기 어렵다는 뜻이다. 그리고 행동 토큰 스키마 자체가 특정 로봇 팔의 7자유도 제어에 맞춰 설계되어 있어서, 다른 형태의 로봇으로 그대로 옮기기는 쉽지 않다.

## taskcraft 관점에서 본 RT-2

이 지점이 개인적으로 taskcraft와 비교하며 곱씹게 되는 부분이다. RT-2의 행동 토큰은 위치 변위, 회전 변위, 그리퍼 개폐라는 특정 embodiment의 물리적 구조에 강하게 묶여 있다. 즉 RT-2는 "웹 지식을 로봇 행동에 전이"하는 데는 성공했지만, 그 행동 표현 자체가 다른 로봇 구조로 재사용될 수 있는지는 다루지 않는다. taskcraft가 지향하는 embodiment-agnostic한 task representation, 즉 latent world model을 매개로 "행동이 세계 상태를 어떻게 바꾸는가"를 추상화하려는 방향과는 정반대의 설계 철학이라 좋은 대조군이 된다.

## 용어 해설

- **VLM(Vision-Language Model)**: 이미지와 텍스트를 함께 입력받아 이미지에 대한 설명, 질문 답변 등을 텍스트로 출력하도록 대규모 데이터로 학습된 멀티모달 모델을 뜻한다.
- **양자화(Quantization)**: 연속적인 실수 값을 정해진 개수의 이산적인 구간(bin)으로 나누어 정수 인덱스로 근사하는 과정을 말한다.
- **파국적 망각(Catastrophic forgetting)**: 신경망이 새로운 데이터로 추가 학습을 하다가 이전에 학습했던 지식이나 성능을 잃어버리는 현상을 가리킨다.
- **엔드이펙터(End-effector)**: 로봇 팔의 끝단, 즉 그리퍼나 도구가 부착되어 실제로 물체와 상호작용하는 부분을 말한다.

## 🤖 AI의 생각

개인적으로 이 논문은 taskcraft의 문제의식을 정면으로 되받아치는 사례처럼 읽힌다. 사견을 붙이자면, embodiment마다 파이프라인을 새로 짜야 한다는 문제를 표현을 무관하게 만드는 대신 웹 지식을 그 embodiment 전용 표현에 욱여넣는 방식으로도 꽤 잘 풀린다는 점이 흥미롭다.

<details class="tc-faq">
<summary>행동 토큰의 양자화 구간 수(256개)를 늘리거나 줄이면 정밀도와 학습 안정성 사이의 트레이드오프가 어떻게 바뀔까</summary>
<div class="tc-faq__body" markdown="1">

논문은 256을 고정값으로 썼는데, 더 세밀한 조작(바느질 같은 정밀 작업)에서는 이 해상도가 병목이 될 가능성이 있어 보인다

</div>
</details>

<details class="tc-faq">
<summary>RT-2의 발현적 능력이 정말 웹 지식 전이 덕분인지, 아니면 단순히 모델 파라미터 수가 커져서 생긴 일반적 스케일링 효과인지 어떻게 분리해서 검증할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

같은 크기의 모델을 웹 데이터 없이 로봇 데이터로만 스케일업한 대조군 실험이 있어야 이 둘을 확실히 갈라낼 수 있을 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 embodiment-agnostic latent 표현을 RT-2식 텍스트 토큰화와 하이브리드로 결합할 여지가 있을까</summary>
<div class="tc-faq__body" markdown="1">

latent를 먼저 뽑아 embodiment 무관한 계획을 세우고, 마지막 단계에서만 RT-2처럼 embodiment-specific 토큰으로 디코딩하는 이중 구조가 taskcraft의 다음 실험으로 시도해볼 만하다고 생각한다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2307.15818v1" target="_blank" rel="noopener">RT-2: Vision-Language-Action Models Transfer Web Knowledge to Robotic Control</a> · arxiv</span></div>
</div>
</div>
