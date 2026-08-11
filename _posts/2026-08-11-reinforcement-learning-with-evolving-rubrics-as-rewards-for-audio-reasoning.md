---
title: "오디오 추론 RL, 채점 기준도 함께 진화시키다"
date: 2026-08-11
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "정답만 맞히는 오디오 AI의 얕은 추론을 막기 위해, 평가 기준 자체를 학습 중에 계속 새로 만드는 강화학습 프레임워크를 살펴본다."
paper_title: "Reinforcement Learning with Evolving Rubrics as Rewards for Audio Reasoning"
paper_summary: "정답만 맞히는 오디오 AI의 얕은 추론을 막기 위해, 평가 기준 자체를 학습 중에 계속 새로 만드는 강화학습 프레임워크를 살펴본다."
paper_url: "https://huggingface.co/papers/2608.02831"
header:
  image: "/assets/images/posts/reinforcement-learning-with-evolving-rubrics-as-rewards-for-audio-reasoning/figure-1.png"
  teaser: "/assets/images/posts/reinforcement-learning-with-evolving-rubrics-as-rewards-for-audio-reasoning/figure-1.png"
---

이번 주 논문은 hf-daily 소스에서 스코어 0.72로 올라온 것을 골랐습니다. 오디오 추론이라는, 텍스트나 비전에 비해 상대적으로 덜 다뤄지는 모달리티에서 강화학습 보상 설계를 정면으로 파고든 논문이라 스코어 자체는 최상위권은 아니지만 방법론적으로 눈여겨볼 지점이 많아 이번 주 노트로 선택했습니다.

대형 오디오-언어 모델(LALM)에 강화학습을 적용해 추론 능력을 키우려는 시도들은 대체로 RLVR(Reinforcement Learning with Verifiable Rewards) 방식을 쓴다. 문제는 검증 가능한 보상이라는 게 결국 최종 정답이 맞았는지 아닌지밖에 볼 수 없다는 점이다. 모델이 오디오 파형 속 음향 증거를 실제로 듣고 판단했는지, 아니면 텍스트 스키마나 통계적 추측으로 답만 맞춰버렸는지는 이 보상으로는 구분이 안 된다. 그 결과로 나타나는 게 얕은 추론과 오디오 환각이다. 답은 맞았는데 근거는 완전히 다른 소리를 짚고 있는 경우가 생긴다.

## 왜 과정 보상만으로는 안 됐나

이 문제를 풀기 위한 자연스러운 다음 단계는 과정 기반 보상(process-based reward)이다. 최종 정답이 아니라 추론 과정 자체에 점수를 매기자는 것이다. 그런데 기존 접근들은 이 채점 기준을 사람이 수작업으로 만들고 학습 내내 고정해서 쓴다. 여기서 두 가지 문제가 생긴다. 하나는 기준 자체가 거칠어서 실제 오디오 음향 증거와 정밀하게 연결되지 않는다는 것이고, 다른 하나는 더 근본적이다. 학습이 진행되면서 정책 모델이 그 고정된 기준을 쉽게 정복해버리면 롤아웃마다 통과 여부가 다 똑같아지고, 보상의 분산이 0으로 죽어버린다. 분산이 없는 보상은 GRPO 같은 상대 비교 기반 최적화에서 아무 정보도 주지 못한다. 채점 기준이 모델보다 성장을 못 따라가는 셈이다.

## 고친 방법: 루브릭도 학습 대상으로

이 논문이 제안하는 AUDIORUBRICS는 평가 기준(Rubric) 자체를 정적 산출물이 아니라 정책 모델과 함께 진화시켜야 할 대상으로 다룬다. 먼저 원본 오디오 파형을 오디오 전용 모델에 직접 넣어서, 텍스트 트랜스크립트를 거치지 않고 문제별로 음향 증거에 근거한 초기 루브릭을 자동 생성한다. 여기서부터 텍스트로 뭉뚱그려 판단하는 걸 막는 셈이다.

핵심은 RLVR 매 단계마다 루브릭을 갱신하는 루프다. 판정 모델이 현재 정책이 만든 롤아웃 그룹들을 비교해 모델이 지금 어디서 헤매는지, 예를 들어 어떤 오디오 환각을 반복하는지, 어떤 음향 단서를 계속 놓치는지를 겨냥한 새 루브릭을 뽑아낸다. 그리고 모든 롤아웃이 다 통과하거나 다 실패해서 변별력이 사라진, 즉 분산이 0인 루브릭은 걸러낸다. 살아남은 루브릭들에만 가중치를 다시 배분해 과정 보상 점수를 계산한다. 이렇게 하면 채점 기준이 모델의 현재 약점을 항상 뒤쫓는 구조가 된다.

여기에 오버씽킹 펜얼티라는 안전장치를 하나 더 얹었다. 루브릭 보상을 더 얻으려고 추론 체인을 불필요하게 길게 늘리는 보상 해킹(reward hacking) 경향이 있어서, 추론 길이에 비례해 선형으로 감점을 주는 항을 최종 보상에 포함시켰다. 정답 정확도 보상, 동적 루브릭 보상, 오버씽킹 펜얼티를 합친 종합 보상으로 GRPO 최적화를 돌리는 것이 전체 그림이다.

수식으로 보면 종합 보상은 $$R_i = R_i^{\text{out}} + \gamma R_i^{\text{rub}} + \delta R_i^{\text{over}}$$ 형태다. $$R_i^{\text{out}}$$은 정답 정확도와 사고 과정 양식을 평가하는 결과 보상, $$R_i^{\text{rub}} = \sum_{k \in \mathcal{K}} w_k b_{k,i}$$는 분산 필터링을 통과한 핵심 루브릭 집합 $$\mathcal{K}$$에 대한 가중합이고, $$R_i^{\text{over}} = 1 - \frac{\vert{}o_i\vert{}}{L}$$이 추론 길이 페널티다. $$\gamma$$와 $$\delta$$는 각 항의 비중을 조절하는 하이퍼파라미터인데, 결국 이 세 하이퍼파라미터의 균형을 어떻게 잡느냐에 시스템 안정성이 상당히 좌우된다.

![AUDIORUBRICS 프레임워크 전체 구조: 오디오 파형에서 초기 루브릭을 생성하고, 롤아웃을 판정해 새 루브릭을 발굴하고 분산 필터링과 재가중치화를 거쳐 GRPO 보상으로 피드백하는 순환 구조](/assets/images/posts/reinforcement-learning-with-evolving-rubrics-as-rewards-for-audio-reasoning/figure-1.png)

## 결과: 붕괴도 폭주도 아닌 수렴

이 설계가 실제로 작동하는지를 가장 직관적으로 보여주는 게 학습 단계별 응답 길이 변화 그래프다. 정답 보상만 쓰는 일반 GRPO는 추론 과정을 아예 생략해버리는 길이 붕괴가 나타난다. 반대로 오버씽킹 펜얼티 없이 루브릭 보상만 추가하면 보상을 더 뽑아내려고 추론 길이가 계속 늘어나는 폭주가 나타난다. AUDIORUBRICS는 이 두 극단 사이에서 응답 길이가 안정적으로 수렴하는 패턴을 보인다.

![학습 스텝에 따른 응답 길이 변화: outcome-only GRPO는 길이 붕괴, 펜얼티 없는 루브릭 보상은 길이 폭주, AUDIORUBRICS는 안정적 수렴을 보이는 세 곡선 비교](/assets/images/posts/reinforcement-learning-with-evolving-rubrics-as-rewards-for-audio-reasoning/figure-2.png)

이 결과는 보상 설계에서 흔히 놓치는 부분을 잘 짚어준다. 보상 항목을 하나 추가하면 그게 또 다른 형태의 해킹 경로를 열어준다는 것, 그래서 각 보상 항이 서로를 견제하도록 설계해야 한다는 것이다. 다만 논문이 분산 필터링과 오버씽킹 펜얼티라는 두 가지 안전장치로 문제를 눌러놓긴 했지만, 판정 모델 자체가 매 스텝마다 새 루브릭을 만들어내는 비용이나, 판정 모델이 편향된 기준을 계속 재생산할 가능성에 대한 검증은 상대적으로 얕게 다뤄진 느낌이다. 결국 이 시스템 전체 품질이 판정 모델 하나에 의존하는 구조라는 점은 남아있는 한계로 보인다.

## 용어 해설

- **RLVR (Reinforcement Learning with Verifiable Rewards)**: 정답 유무를 규칙 기반으로 명확히 판별할 수 있는 보상 신호를 이용해 언어 모델을 강화학습으로 학습시키는 방식.
- **GRPO (Group Relative Policy Optimization)**: 같은 입력에 대해 여러 롤아웃을 생성한 뒤 그룹 내 상대적인 보상 순위를 이용해 정책을 업데이트하는 강화학습 최적화 알고리즘.
- **보상 해킹 (Reward Hacking)**: 학습 에이전트가 원래 의도된 목표를 달성하지 않고도 보상 신호 자체를 최대화할 수 있는 편법적인 행동 패턴을 찾아내는 현상.

## 🤖 AI의 생각


개인적인 의견을 덧붙이면, 이 논문에서 가장 흥미로운 지점은 보상 함수를 고정된 산출물이 아니라 정책과 함께 계속 갱신되는 대상으로 다뤘다는 점이다.

<details class="tc-faq">
<summary>판정 모델이 만들어내는 루브릭의 품질을 어떻게 담보할 수 있을까, 판정 모델 자체가 편향되면 문제가 그대로 한 단계 위로 이동하는 것 아닐까</summary>
<div class="tc-faq__body" markdown="1">

논문은 분산 필터링으로 무의미해진 루브릭을 걸러내지만 판정 모델이 처음부터 얕거나 편향된 기준만 반복 생성하는 경우를 막는 장치는 상대적으로 약해 보인다

</div>
</details>

<details class="tc-faq">
<summary>오버씽킹 펜얼티의 선형 감점 강도를 태스크 난이도에 따라 다르게 줘야 하지 않을까</summary>
<div class="tc-faq__body" markdown="1">

복잡한 오디오 장면일수록 긴 추론이 정당화될 수 있는데 고정된 길이 기준 $$L$$ 하나로 모든 문제를 묶는 건 지나치게 단순한 근사일 수 있다

</div>
</details>

<details class="tc-faq">
<summary>보상이나 목표를 언어나 고정 규칙으로 완전히 명세하기 어렵다는 문제의식이 다른 영역에서도 통할까</summary>
<div class="tc-faq__body" markdown="1">

정적 평가 기준이 아니라 진화하는 평가 기준이라는 아이디어는 오디오뿐 아니라 robotics 같은 다른 모달리티에도 참고할 여지가 있어 보이는데, 판정 대상이 오디오 신호가 아니면 판정 모델 설계 자체가 달라질 필요가 있다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.02831" target="_blank" rel="noopener">Reinforcement Learning with Evolving Rubrics as Rewards for Audio Reasoning</a> · hf-daily</span></div>
</div>
</div>
