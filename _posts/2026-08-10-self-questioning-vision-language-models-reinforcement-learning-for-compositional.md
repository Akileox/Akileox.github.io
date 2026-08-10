---
title: "스스로 질문하는 VLM의 등장"
date: 2026-08-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "사람이 짠 해설 없이 보상 신호만으로 VLM이 하위 질문을 만들어 답을 찾는 법을 배운 연구를 살펴본다."
paper_title: "Self-Questioning Vision-Language Models: Reinforcement Learning for Compositional Visual Reasoning"
paper_summary: "사람이 짠 해설 없이 보상 신호만으로 VLM이 하위 질문을 만들어 답을 찾는 법을 배운 연구를 살펴본다."
paper_url: "https://www.semanticscholar.org/paper/3cc579286e51dc968d053bbf2731fc1d9bd94882"
header:
  image: "/assets/images/posts/self-questioning-vision-language-models-reinforcement-learning-for-compositional/figure-1.png"
  teaser: "/assets/images/posts/self-questioning-vision-language-models-reinforcement-learning-for-compositional/figure-1.png"
---

## 이번 주에 이 논문을 고른 이유

이번 주는 semantic-scholar 추천 경로에서 score 0.85로 올라온 논문을 골랐습니다. taskcraft가 다루는 embodiment 전이와 직접 맞닿는 주제는 아니지만, "사람이 일일이 단계를 가르치지 않아도 적절한 보상 설계만으로 합성적 구조가 창발하는가"라는 질문 자체가 taskcraft의 latent world model 가설과 결이 비슷해서 읽어볼 가치가 있다고 판단했습니다.

## 문제 제기: 한 번에 답하려는 VLM

Vision-Language Model이 복잡한 시각 추론 문제에서 자주 틀리는 이유는 생각보다 단순하다. 이미지와 질문이 들어오면 단 한 번의 순전파로 답을 뱉어내기 때문이다. 객체를 찾고, 개수를 세고, 속성을 비교하는 과정이 얽힌 질문일수록 한 번의 forward pass로는 중간 단계를 건너뛰기 쉽다. 사람이라면 "일단 저 물체가 몇 개인지 세어보고, 그중 빨간 것만 골라내자"는 식으로 문제를 쪼개서 풀 텐데, 기존 VLM에는 이 쪼개는 과정 자체가 없었다.

## 왜 안 됐는지: 해설 데이터의 벽

이 문제를 풀려는 시도가 없었던 건 아니다. Chain-of-Thought 프롬프팅이나 지도학습 기반 미세조정으로 단계별 추론을 흉내 내는 방법들이 있었지만, 모두 사람이 직접 작성한 단계별 풀이 과정 데이터가 필요했다. 새로운 도메인이나 새로운 유형의 질문이 등장할 때마다 사람이 다시 앉아서 해설을 써야 하니 확장성이 떨어질 수밖에 없다. 외부 비전 툴을 호출하는 Visual Programming 방식도 정의된 툴 라이브러리 밖의 질문에는 대응하지 못한다는 한계가 똑같이 있었다. 결국 핵심은 "사람의 손을 타지 않고도 문제 분해 전략을 학습시킬 방법이 있는가"였다.

## 고친 방법: 보상만 주고 구조는 알아서 찾게 하기

이 논문의 해법은 인간의 예시를 아예 주지 않는 것이다. 대신 모델이 최종 답변을 내놓기 전에 이미지에 접근해서 하위 질문과 하위 답변 쌍을 스스로 만들어 내도록 시스템 프롬프트만 구성하고, **그 구조를 실제로 쓰는지 여부와 정답을 맞혔는지 여부에만 보상을 건다**.

![Self-Questioning 프레임워크: 이미지와 질문에서 하위 질문-답변 쌍을 생성한 뒤 최종 답변을 내는 파이프라인](/assets/images/posts/self-questioning-vision-language-models-reinforcement-learning-for-compositional/figure-1.png)

보상 함수는 다음과 같이 정의된다.

$$R(y, a^*) = \begin{cases} 1.0 & \text{if } \text{Format}(y) \land \text{Correct}(y, a^*) \\ -1.0 & \text{otherwise} \end{cases}$$

포맷을 지키면서 정답도 맞혀야만 만점을 주고, 둘 중 하나라도 어기면 감점한다. 학습 알고리즘으로는 별도의 critic network 없이 그룹 내 상대적 보상만으로 정책을 업데이트하는 GRPO를 썼다.

$$\hat{A}_i = \frac{R(y_i, a^*) - \mu_g}{\sigma_g}$$

같은 입력에 대해 여러 개의 응답을 샘플링한 뒤, 그 그룹의 평균과 표준편차로 각 샘플의 상대적 이점을 정규화하는 방식이다. critic이 빠지니 메모리 사용량이 절반으로 줄어서 3B 규모의 Qwen2.5-VL-3B-Instruct도 무리 없이 학습이 돌아갔다고 한다. 여기에 KL divergence 페널티를 손실 함수에 추가해서 학습된 정책이 초기 참조 모델에서 너무 멀어지지 않도록 제약을 걸었다.

$$\mathcal{L}(\theta) = -\mathbb{E} \left[ \min\left(r_t \hat{A}_t, \text{clip}(r_t, 1-\epsilon, 1+\epsilon)\hat{A}_t\right) - \beta D_{\text{KL}}(\pi_\theta \parallel \pi_{\text{ref}}) \right]$$

이 페널티가 없으면 언어 이해나 시각 인식 능력이 무너지는 재앙적 망각이 생길 수 있어서, 구조 학습과 능력 보존 사이의 균형을 잡아주는 장치로 넣은 셈이다.

## 결과: 실제 이미지에서는 통했지만 단순한 도메인에서는 세금을 낸다

합성 도형 이미지인 CLEVR로만 Self-Questioning을 학습시킨 모델을 자연 이미지 데이터셋인 A-OKVQA에 그대로 옮겼더니 정확도가 2.6퍼센트포인트 올라갔다. 도메인이 완전히 다른데도 하위 질문으로 쪼개는 전략 자체가 전이된다는 뜻이다.

![베이스 모델, Direct+GRPO, SQ+GRPO의 A-OKVQA·CLEVR 정확도 비교. A-OKVQA에서는 46.8%에서 52.2%로 상승하지만 CLEVR에서는 소폭 하락한다](/assets/images/posts/self-questioning-vision-language-models-reinforcement-learning-for-compositional/figure-2.png)

다만 모든 도메인에서 이득만 있었던 건 아니다. A-OKVQA처럼 여러 단계의 추론이 필요한 실제 이미지 데이터셋에서는 강화학습 적용 시 정확도가 46.8%에서 52.2%까지 뛰었지만, CLEVR처럼 이미 단순한 합성 이미지 데이터셋에서는 하위 질문 생성을 강제했을 때 오히려 성능이 살짝 떨어졌다. 굳이 나눌 필요가 없는 문제를 억지로 쪼개다 보니 생기는 일종의 **포맷 세금(format tax)**인 셈이다. 즉 Self-Questioning은 만능이 아니라 문제의 복잡도가 어느 수준을 넘어설 때 선별적으로 켜야 이득이 나는 기법이라는 걸 이 결과가 보여준다.

## taskcraft와의 접점

이 논문은 embodiment 이식이나 world model을 다루지 않기 때문에 taskcraft의 핵심 주제와 직접 맞닿아 있지는 않다. 다만 GRPO 자체는 taskcraft가 BC/DAgger/PPO를 비교하는 10주 실험에서 PPO의 대안이나 개선판으로 참고할 여지가 있다. critic network가 빠져서 메모리 효율이 좋고 안정성 측면에서도 다르게 동작할 가능성이 있기 때문이다. 더 개념적인 차원에서는, 사람이 작성한 단계별 해설 없이 보상 신호만으로 하위 질문 분해 구조가 자발적으로 발현된다는 결과 자체가, taskcraft가 묻는 "embodiment-specific teleoperation 데이터 없이도 latent 표현이 task 구조를 스스로 포착할 수 있는가"라는 질문과 구조적으로 비슷한 발상이다.

## 🤖 AI의 생각


제 생각에는 이 논문이 taskcraft에 직접적인 방법론을 주기보다는, 사람 라벨 없이 RL 보상만으로 구조가 창발한다는 하나의 증거 사례로서 참고할 가치가 있어 보입니다. 다만 이건 시각 언어 추론이라는 도메인 안에서 검증된 것이고, 물리적 embodiment 간 전이라는 taskcraft의 문제와 정말 같은 종류의 창발인지는 아직 유비에 가깝다는 점을 감안하고 봐야 할 것 같습니다.

<details class="tc-faq">
<summary>CLEVR에서 나타난 format tax는 왜 생겼을까, 하위 질문 개수가 너무 많아서인지 아니면 보상 함수가 너무 거칠어서인지 논문에서 더 파고들 여지가 있어 보인다.</summary>
<div class="tc-faq__body" markdown="1">

정답 포함 여부만 보는 containment matching 보상이 단순 도메인에서는 오히려 잘못된 하위 질문에도 우연히 보상을 줄 수 있어서, 보상 함수의 해상도 문제일 가능성이 있다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 latent world model도 이 논문처럼 critic 없는 그룹 상대 보상 방식으로 학습시킬 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

Minecraft처럼 상태 전이가 명확한 환경이라면 시도해볼 만하지만, 로봇 embodiment 간 보상 스케일 차이를 그룹 내 정규화만으로 흡수할 수 있을지는 별도로 검증이 필요하다.

</div>
</details>

<details class="tc-faq">
<summary>논문이 3B급 모델로만 실험했는데, 모델 크기가 커지면 format tax 현상이 사라질지도 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

더 큰 모델은 문제 복잡도를 스스로 판단해서 필요할 때만 하위 질문을 만들 가능성이 있어, 이 현상이 모델 용량의 한계에서 온 것인지 확인해볼 필요가 있다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/3cc579286e51dc968d053bbf2731fc1d9bd94882" target="_blank" rel="noopener">Self-Questioning Vision-Language Models: Reinforcement Learning for Compositional Visual Reasoning</a> · semantic-scholar-rec</span></div>
</div>
</div>
