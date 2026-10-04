---
title: "보는 뇌와 움직이는 손을 분리한 CogACT"
date: 2026-10-04
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "VLA 모델이 행동을 언어 토큰처럼 다루다 겪은 한계를, Diffusion 기반 전용 액션 모듈로 풀어낸 CogACT를 살펴본다."
paper_title: "CogACT: A Foundational Vision-Language-Action Model for Synergizing Cognition and Action in Robotic Manipulation"
paper_summary: "VLA 모델이 행동을 언어 토큰처럼 다루다 겪은 한계를, Diffusion 기반 전용 액션 모듈로 풀어낸 CogACT를 살펴본다."
paper_url: "https://arxiv.org/abs/2411.19650v1"
header:
  image: "/assets/images/posts/cogact-a-foundational-vision-language-action-model-for-synergizing-cognition-and/figure-1.png"
  teaser: "/assets/images/posts/cogact-a-foundational-vision-language-action-model-for-synergizing-cognition-and/figure-1.png"
---

## 들어가며

이번 주 블로그에서 다룰 논문은 arXiv에 공개된 CogACT입니다. 스코어링 점수가 1.00으로 가장 높게 나온 논문이라 골랐습니다만, 점수를 떠나서도 이 논문은 VLA(Vision-Language-Action) 모델이라는 개념 자체가 최근 몇 년간 로봇 학습 분야에서 왜 중요해졌는지, 그리고 그 안에서 지금 어떤 설계 논쟁이 벌어지고 있는지를 한 편으로 압축해서 보여주는 좋은 사례입니다. Imitation Learning이나 RL은 수업에서 배웠지만 VLA라는 용어는 처음이라는 분이라도 따라올 수 있도록, 기초 개념부터 짚으면서 풀어보겠습니다.

## VLA가 뭐고 왜 토큰화에 갇혀 있었나

Imitation Learning 수업에서 배운 Behavior Cloning을 떠올려보면, 입력은 로봇의 관측(카메라 이미지, 관절 상태)이고 출력은 다음에 취할 액션이다. 전통적으로는 이 매핑을 CNN이나 작은 MLP로 학습했다. 그런데 최근 몇 년 사이 흐름이 바뀌었다. "로봇이 언어 지시를 이해하고 다양한 물체와 환경에 일반화하려면, 애초에 인터넷 규모의 이미지와 텍스트로 학습된 거대 Vision-Language Model의 지능을 빌려오자"는 아이디어가 등장했고, 이렇게 시각, 언어, 행동을 한 모델에 합친 것이 Vision-Language-Action(VLA) 모델이다.

문제는 "행동을 어떻게 출력하느냐"였다. RT-2나 OpenVLA 같은 초기 대형 VLA들은 언어 모델의 익숙한 방식을 그대로 썼다. 즉 로봇의 연속적인 7차원 제어 신호(엔드 이펙터의 위치 변화, 회전 변화, 그리퍼 개폐)를 마치 텍스트 토큰처럼 이산 구간으로 쪼개서, 언어 모델이 다음 단어를 예측하듯 다음 액션 토큰을 예측하게 만들었다. 이 방식은 구현이 간단하고 언어 모델 인프라를 그대로 재활용할 수 있다는 장점이 있지만, 치명적인 단점이 있다. 연속적인 물리량을 유한한 구간으로 양자화하는 순간 정밀도가 떨어진다. 병에 뚜껑을 정확히 맞추거나 작은 부품을 집는 것처럼 밀리미터 단위의 정밀함이 필요한 작업에서 이 손실이 바로 성공률 하락으로 이어진다.

또 하나의 문제는 다중 모드(Multimodality)다. 같은 과업이라도 로봇이 취할 수 있는 궤적은 하나가 아니다. 장애물을 왼쪽으로 피해도 되고 오른쪽으로 피해도 된다. RoboFlamingo처럼 단순 회귀(MLP, LSTM)로 액션을 예측하면 이런 다양한 가능성을 평균 내버려서 애매하고 비현실적인 중간 동작이 나온다. Diffusion Policy 같은 연구가 이 다중 모드 문제를 diffusion 생성 방식으로 풀어낸 적이 있지만, 그 모델들은 대체로 작고 좁은 태스크에 특화되어 있어서 VLM의 일반화 능력과는 결합되지 못했다.

## CogACT의 해법: 인지와 행동을 분리하기

CogACT가 던지는 제안은 간결하다. "보는 뇌"와 "움직이는 손"을 아예 다른 모듈로 떼어놓자는 것이다. 먼저 DINOv2와 SigLIP으로 이미지를 토큰화하고, 이 시각 토큰과 텍스트 지시문, 그리고 학습 가능한 전용 인지 토큰을 LLaMA-2 기반 언어 모델에 함께 넣는다. 언어 모델은 causal attention을 거쳐 작업 수행에 필요한 고수준 결정을 하나의 벡터, 논문 표기로는 $$f_t^c$$ 로 압축해서 뱉어낸다. 여기까지는 기존 VLA와 크게 다르지 않다.

차이는 그다음부터다. 이 $$f_t^c$$ 를 받아서 실제 7차원 액션을 만드는 일은 언어 모델이 아니라 별도의 Diffusion Transformer(DiT)가 맡는다. DiT는 13M짜리 작은 버전부터 308M짜리 큰 버전까지 세 가지 크기로 실험되었는데, 이 모듈이 가우시안 노이즈에서 출발해 $$f_t^c$$ 를 조건으로 삼아 여러 단계의 디노이징을 거치면서 현재와 미래 15스텝 분량의 연속 액션 시퀀스를 복원한다.

![CogACT 전체 아키텍처, Vision과 Language 모듈이 인지 특징을 추출하고 Diffusion Transformer 액션 모듈이 이를 조건으로 연속 액션을 생성하는 구조](/assets/images/posts/cogact-a-foundational-vision-language-action-model-for-synergizing-cognition-and/figure-2.png)

이 구조가 좋은 이유는 두 가지다. 첫째, 이산 토큰화를 거치지 않으니 정밀도 손실이 없다. 둘째, diffusion 생성 방식이라 다중 모드 분포를 자연스럽게 표현한다. 왼쪽으로 피하는 궤적과 오른쪽으로 피하는 궤적이 둘 다 유효한 샘플로 나올 수 있다는 뜻이다. 그리고 VLM과 DiT 전체를 Open X-Embodiment의 2,250만 프레임으로 end-to-end 학습시켜서, VLM의 일반화 능력이 액션 모듈에도 흘러들어가게 만들었다.

## 궤적을 이어붙일 때 생기는 새로운 문제

그런데 액션을 한 번에 15스텝씩 묶어서 예측하는 방식(Action Chunking)을 쓰면 추론 시 새로운 골칫거리가 생긴다. 로봇은 매 스텝마다 새로 관측을 받고 다시 15스텝을 예측하기 때문에, 같은 시점 $$t$$에 대해 과거 여러 시점에서 예측된 값들이 쌓이게 된다. 기존 방법들은 이 과거 예측값들을 단순히 고정 가중치로 평균 내서(temporal ensemble) 부드럽게 만들었는데, 여기서 다중 모드 문제가 다시 고개를 든다. 어떤 예측은 "왼쪽으로 피하는 모드"고 어떤 예측은 "오른쪽으로 피하는 모드"라면, 이 둘을 그냥 평균 내버리면 로봇은 둘 다 아닌 이상한 중간 동작을 하게 된다.

CogACT는 여기에 Adaptive Action Ensemble(AAE)이라는 장치를 추가한다. 현재 관측에서 예측한 액션과 과거 관측에서 예측한 액션 사이의 코사인 유사도를 계산해서, 비슷한 모드끼리만 더 큰 가중치로 섞는 방식이다.

$$\hat{a}_t = \sum_{k=0}^{K} w_k^{\text{ada}} \cdot a_{t\vert{}o_{t-k}}$$

$$w_k^{\text{ada}} = \exp(\alpha \cdot \langle a_{t\vert{}o_t}, a_{t\vert{}o_{t-k}} \rangle)$$

여기서 $$\langle \cdot, \cdot \rangle$$ 는 두 액션 벡터의 코사인 유사도이고, $$\alpha$$ 는 민감도를 조절하는 값(기본 0.1)이다. 쉽게 말해 "지금 예측과 방향이 비슷한 과거 예측만 신뢰하고, 방향이 다른 과거 예측은 가중치를 낮춰서 섞는다"는 것이다. 작지만 다중 모드 생성 모델을 실시간 제어에 끌어다 쓸 때 반드시 마주치는 문제를 구체적으로 짚고 고친 부분이라 실용적으로 눈에 띈다.

## 결과: 스케일링이 액션 모듈에서도 통하는가

결과를 보면 두 가지가 두드러진다. 첫째, SIMPLER 시뮬레이션 벤치마크에서 CogACT는 55B 규모의 RT-2-X 대비 18%포인트 높은 성공률을 보였고, 비슷한 체급인 7B급 OpenVLA 대비로는 실세계 환경에서 55%포인트 이상 앞섰다. 둘째, DiT 액션 모듈의 파라미터 수를 13M, 89M, 308M으로 늘려갈수록 성공률이 파라미터 수의 로그값에 비례해서 거의 선형으로 증가했다.

![다양한 로봇 플랫폼에서의 성공률 비교와 액션 모듈 파라미터 크기에 따른 스케일링 곡선](/assets/images/posts/cogact-a-foundational-vision-language-action-model-for-synergizing-cognition-and/figure-1.png)

이 두 번째 관찰이 개인적으로 더 흥미롭다. 전체 VLM(수십억 파라미터)을 더 키우는 것보다, 수백 메가바이트짜리 전용 액션 모듈을 키우는 쪽이 제어 성능 향상에 훨씬 효율적인 투자라는 뜻이기 때문이다. 다만 이 논문이 회전 표현으로 Euler Angle을 그대로 썼다는 점은 짚어둘 만하다. Euler Angle은 짐벌락 같은 불연속성 문제가 알려져 있는데, 6D rotation 같은 연속 표현을 쓰는 다른 VLA 연구들과 비교하면 상대적으로 단순한 선택을 유지한 셈이고, 이 부분이 성능에 어떤 영향을 줬는지는 논문에서 따로 분석되지 않는다.

## 용어 해설

- **Diffusion Model(확산 모델)**: 데이터에 점진적으로 노이즈를 섞은 뒤, 그 과정을 거꾸로 되돌리는 디노이징을 학습해서 새로운 데이터를 생성하는 모델이다. 이미지 생성에서 많이 쓰이지만 이 논문처럼 연속적인 수치 시퀀스(로봇 액션) 생성에도 적용할 수 있다.
- **Action Chunking(액션 청킹)**: 한 스텝씩 액션을 예측하는 대신, 미래 여러 스텝 분량의 액션을 한 번에 묶어서 예측하는 방식이다. 예측 효율은 좋아지지만 매 시점마다 새로 예측된 값들을 어떻게 이어붙일지(앙상블)가 별도 문제로 남는다.
- **코사인 유사도**: 두 벡터가 가리키는 방향이 얼마나 비슷한지를 측정하는 값으로, 두 벡터 사이 각도의 코사인 값으로 계산한다. 값이 1에 가까우면 방향이 거의 같고, 0에 가까우면 방향이 서로 무관하다는 뜻이다.

## 🤖 AI의 생각


사실 요약과는 별개로 제 의견을 덧붙이면, 이 논문은 화려한 새 아이디어 하나라기보다 "VLA 설계에서 무엇을 분리하고 무엇을 합칠지"에 대한 꽤 설득력 있는 엔지니어링 정리처럼 읽힙니다.

<details class="tc-faq">
<summary>DiT 액션 모듈을 308M보다 더 키우면 스케일링 곡선이 계속 선형으로 이어질까, 아니면 어딘가에서 포화될까</summary>
<div class="tc-faq__body" markdown="1">

논문은 세 개 지점만 찍어서 직선처럼 보이지만 포화 지점을 찾으려면 더 큰 액션 모듈 실험이 필요해 보인다

</div>
</details>

<details class="tc-faq">
<summary>Adaptive Action Ensemble의 코사인 유사도 기반 가중치는 모드가 두 개보다 많을 때도 잘 작동할까</summary>
<div class="tc-faq__body" markdown="1">

논문 실험은 대체로 양자택일 수준의 모호함만 다뤄서 더 복잡한 다중 모드 상황에서의 검증은 비어 있다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 목표로 하는 사람 시연 기반 embodiment-agnostic 표현 학습과의 관계는 무엇일까</summary>
<div class="tc-faq__body" markdown="1">

로봇 고유의 7차원 액션 공간을 인지 벡터와 분리한 것처럼, 사람 시연에서도 저수준 신체 좌표와 고수준 의도 벡터를 비슷하게 나눠볼 여지가 있어 보인다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2411.19650v1" target="_blank" rel="noopener">CogACT: A Foundational Vision-Language-Action Model for Synergizing Cognition and Action in Robotic Manipulation</a> · arxiv</span></div>
</div>
</div>
