---
title: "성공은 우연이고 실패는 필연인 이유"
date: 2026-09-29
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "정답도 다시 굴리면 재현이 안 되고 오답도 다시 굴리면 살아난다는 관찰에서 출발한 LLM 추론 RL의 credit assignment 기법을 정리했다"
paper_title: "Surprising Success, Repeated Failure: Entropy-Guided Credit Assignment for Exploration in LLM Reasoning"
paper_summary: "정답도 다시 굴리면 재현이 안 되고 오답도 다시 굴리면 살아난다는 관찰에서 출발한 LLM 추론 RL의 credit assignment 기법을 정리했다"
paper_url: "https://huggingface.co/papers/2609.33781"
---

이번 주는 hf-daily 소스에서 스코어 0.83을 받은 논문을 골랐습니다. LLM 추론 강화학습에서 "보상을 어디에 얼마나 나눠줄 것인가"라는 credit assignment 문제는 최근 RLVR 계열 연구에서 가장 뜨거운 주제 중 하나인데, 이 논문은 그 문제를 굉장히 단순하면서도 설득력 있는 실증 관찰(재샘플링 실험)에서 출발한다는 점이 눈에 띄어 이번 주 노트로 선정했습니다.

## 성공은 재현되지 않고 실패는 반복된다

RLVR(검증 가능한 보상으로 학습하는 RL)은 응답 하나를 통째로 채점한다. 수학 문제를 풀든 코드를 짜든, 정답이면 +1, 오답이면 0 혹은 -1이라는 스칼라 하나가 응답 전체 토큰에 똑같이 뿌려진다. 문제는 응답 안에는 결정적인 순간과 그냥 흘러가는 순간이 섞여 있다는 점이다. 어떤 토큰은 "이 방법 말고 다른 접근을 써볼까"라는 분기점이고, 어떤 토큰은 이미 정해진 답을 서술만 하는 자리다. 이걸 구분 못 하면 좋은 분기점도, 나쁜 분기점도 똑같은 크기로 강화되거나 처벌된다.

이 문제를 풀려는 기존 시도들은 정책 엔트로피(다음 토큰 분포가 얼마나 퍼져 있는가)를 분기점의 대리 신호로 썼다. 엔트로피가 높은 위치는 모델이 확신하지 못하고 여러 선택지를 저울질하는 지점이니, 여기에 더 강한 업데이트를 주면 탐색이 촉진된다는 논리다. 그런데 이 논리에는 구멍이 있다. 성공한 응답이든 실패한 응답이든 고엔트로피 지점을 똑같이 "탐색이 필요한 곳"으로 취급해버리면, 실패한 응답의 고엔트로피 지점, 즉 아직 다른 경로로 회복될 여지가 남아 있던 지점에 가장 강한 penalty가 몰리게 된다. 탐색을 촉진하려던 신호가 오히려 탐색의 싹을 자르는 역설이 생기는 셈이다.

## 왜 안 됐는지: 재샘플링으로 확인한 비대칭

저자들은 이 직관을 재샘플링 실험으로 검증한다. Qwen3-4B/8B-Base로 원래 정답이었던 응답을 골라 고엔트로피 구간부터 다시 생성해보면, 저엔트로피 구간에서 다시 생성했을 때보다 정답률이 19~21%p나 낮아진다. 즉 "운 좋게 맞은" 성공은 그 불확실한 지점을 다시 굴리면 재현되지 않는다. 반대로 원래 오답이었던 응답을 고엔트로피 구간에서 다시 생성하면 정답률이 오히려 9~13%p 올라간다. 오답 응답이라도 고엔트로피 지점에는 아직 살아있는 대안 경로가 남아 있었다는 뜻이다.

이 결과는 단어 빈도 분석에서도 그대로 확인된다. 정답 응답의 고엔트로피 지점에는 "let's", "consider" 같은 탐색적 표현이 몰려 있고, 오답 응답의 저엔트로피 지점에는 "finally", "confirm" 같은 결론 표지가 몰려 있다. 그런데 오답 응답의 고엔트로피 지점에도 "try", "instead" 같은 탐색적 단어가 나타난다. 즉 엔트로피의 높고 낮음이 아니라, 그 엔트로피가 성공 응답에 속하는지 실패 응답에 속하는지에 따라 의미가 완전히 달라진다는 것이다. 기존 방법들이 놓친 지점이 바로 여기다.

## 고친 방법: 엔트로피와 보상의 부호를 곱한다

EAPO(Entropic Advantage Policy Optimization)는 이 비대칭을 그대로 알고리즘에 옮긴다. 핵심은 딱 하나, GRPO의 응답 단위 advantage $$\hat{A}^i$$를 토큰 단위로 재분배할 때 엔트로피 $$h_{i,t}$$가 아니라 advantage의 부호와 엔트로피를 곱한 $$\text{sign}(\hat{A}^i) h_{i,t}$$를 기준으로 삼는다는 것이다.

$$w_{i,t} = \frac{\exp\left(\kappa\, \text{sign}(\hat{A}^i) h_{i,t}\right)}{\frac{1}{T_i}\sum_{u \in \mathcal{I}_i} \exp\left(\kappa\, \text{sign}(\hat{A}^i) h_{i,u}\right)}, \qquad \hat{A}^{i,E}_t = \hat{A}^i w_{i,t}$$

성공 응답($$\hat{A}^i > 0$$)에서는 고엔트로피 토큰에 더 큰 가중치가 붙어 그 불확실한 결정이 더 강하게 보상받는다. 실패 응답($$\hat{A}^i < 0$$)에서는 부호가 뒤집혀서 저엔트로피 토큰, 즉 확신에 찬 채로 틀린 결정에만 강한 penalty가 집중되고, 고엔트로피 토큰의 penalty는 완화된다. 분모의 응답 내 평균 정규화 덕분에 advantage의 전체 크기는 그대로 유지되고 "분배 방식"만 바뀌는 게 포인트다. 추가 모델도, 추가 롤아웃도, 별도의 privileged information도 필요 없이 GRPO 목적함수에서 advantage 항 하나만 바꾸면 되는 구조라 기존 파이프라인에 거의 무비용으로 얹을 수 있다.

이론 쪽 보강도 눈여겨볼 만하다. 논문은 이 가중치 식이 균일분포를 reference로 하는 KL 정규화 최적화 문제의 유일해라는 걸 보이고(Proposition 1), 단일 스텝 업데이트 모델에서 penalty 강도가 커질수록 선택되지 않은 대안 토큰들의 분포가 초기 분포에서 점점 더 멀어진다는 걸 증명한다(Proposition 2). 요약하면 "penalty를 세게 줄수록 남은 대안들을 왜곡시킨다"는 걸 수식으로 뒷받침하고, EAPO의 penalty 완화가 왜 이 왜곡을 줄여주는지를 정당화하는 방향이다.

## 결과와 한계

방법 자체는 굉장히 가볍고, GRPO 계열 파이프라인에 바로 얹을 수 있다는 점에서 실용성이 높아 보인다. 다만 이 논문이 근거로 삼는 Section C.1 실험, 즉 surprisal(샘플링된 토큰 하나의 확률)로 신호를 대체하면 학습이 붕괴한다는 결과는 뒤집어 보면 이 방법이 신호 선택에 상당히 민감하다는 뜻이기도 하다. 엔트로피라는 신호가 정말로 "의미 있는 대안 경로"를 가리키는지, 아니면 그냥 숫자나 고유명사 선택처럼 형식적으로 여러 토큰이 가능한 지점을 가리키는지는 Figure 2의 정성적 단어 빈도 분석 이상으로는 검증되지 않는다.

## 용어 해설

- **GRPO**: 그룹 내 여러 응답의 보상을 평균과 표준편차로 정규화해 advantage를 만드는 강화학습 알고리즘으로, 별도의 가치 함수(critic) 없이 그룹 상대 비교만으로 학습 신호를 얻는 방식이다.
- **Advantage**: 강화학습에서 어떤 행동이 평균적인 경우보다 얼마나 더 나은 결과를 냈는지를 나타내는 값으로, 정책을 어느 방향으로 얼마나 업데이트할지를 결정하는 핵심 스칼라다.
- **KL divergence**: 두 확률분포가 얼마나 다른지를 측정하는 비대칭적인 거리 척도로, 한 분포를 기준(reference)으로 삼아 다른 분포가 거기서 얼마나 벗어났는지를 수치화한다.

## 🤖 AI의 생각


이 논문에서 제일 흥미로운 지점은 방법 자체보다 Table 1의 재샘플링 실험이라고 생각한다. 성공과 실패라는 이분법이 사실은 그 순간의 궤적에 대한 서술일 뿐 그 지점의 잠재적 가치에 대한 서술은 아니라는 걸 보여주기 때문이다.

<details class="tc-faq">
<summary>엔트로피가 정말 "의미 있는 대안 경로"의 좋은 대리 신호인가, 아니면 그냥 형식적으로 여러 토큰이 가능한 지점(숫자, 고유명사 선택)을 가리킬 뿐인가.</summary>
<div class="tc-faq__body" markdown="1">

Figure 2의 단어 빈도 분석은 정성적 수준이라, self-consistency나 verifier 신뢰도 같은 다른 불확실성 신호로 대체했을 때도 같은 비대칭 원리가 성립하는지 확인해볼 필요가 있다.

</div>
</details>

<details class="tc-faq">
<summary>학습이 진행되면서 정책 엔트로피가 전역적으로 낮아지면 EAPO의 신호 자체가 소멸하는 자기소모적 구조 아닌가.</summary>
<div class="tc-faq__body" markdown="1">

배치 내 10/90 퍼센타일 클리핑 기준이 학습 후반부에 어떻게 흔들리는지, curriculum이나 스케줄과 결합해서 봐야 할 문제로 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 다루는 멀티스텝 tool-use 에이전트 궤적에도 이런 성공/실패 비대칭 credit assignment가 적용될 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

도구 호출 시퀀스에서도 "이 도구를 쓸지 말지" 분기점의 엔트로피를 측정할 수 있다면, 실패한 태스크의 고엔트로피 분기점을 덜 처벌하는 식으로 유사한 아이디어를 옮겨볼 여지가 있어 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.33781" target="_blank" rel="noopener">Surprising Success, Repeated Failure: Entropy-Guided Credit Assignment for Exploration in LLM Reasoning</a> · hf-daily</span></div>
</div>
</div>
