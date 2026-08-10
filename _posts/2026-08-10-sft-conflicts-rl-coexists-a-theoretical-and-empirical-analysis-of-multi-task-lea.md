---
title: "SFT는 충돌하고 RL은 공존한다"
date: 2026-08-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "다단계 학습에서 SFT는 작업 간 파라미터가 충돌해 무너지지만 RL은 그래디언트가 직교해 여러 작업이 공존한다는 것을 증명한 논문입니다."
paper_title: "SFT Conflicts, RL Coexists: A Theoretical and Empirical Analysis of Multi-Task Learning for LLMs"
paper_summary: "다단계 학습에서 SFT는 작업 간 파라미터가 충돌해 무너지지만 RL은 그래디언트가 직교해 여러 작업이 공존한다는 것을 증명한 논문입니다."
paper_url: "https://huggingface.co/papers/2608.03573"
---

이번 주 논문은 hf-daily 소스에서 score 0.83으로 상위에 오른 논문입니다. LLM을 여러 작업에 순차적으로 파인튜닝할 때 SFT는 왜 자꾸 무너지고 RL은 왜 안 무너지는지, 이걸 "느낌"이 아니라 파라미터 업데이트의 크기와 방향까지 직접 재서 증명했다는 점이 눈에 띄어서 이번 주 노트로 골랐습니다.

## 문제 제기

LLM에 여러 작업(Math, Code, Science, Logic)을 순서대로 학습시키면 흔히 겪는 일이 있다. Math로 SFT한 다음 Code로 SFT하면 Math 성능이 뚝 떨어진다. 흔히 catastrophic forgetting이라 부르는 현상인데, 이걸 막으려고 데이터를 섞거나 학습률을 낮추거나 하는 임시방편이 많았다. 그런데 같은 순서로 RL(GRPO 계열)을 돌리면 이 붕괴가 훨씬 덜하다는 관찰이 실무에서 종종 보고돼 왔다. 이 논문은 "왜"를 파고든다. SFT와 RL이 같은 모델, 같은 작업 순서로 학습하는데 왜 하나는 무너지고 하나는 안 무너지는가.

## 왜 SFT는 무너지는가

원인을 파라미터 업데이트 $$\Delta W$$ 자체를 뜯어보면서 찾는다. 두 방식의 업데이트를 비교했더니 크기부터 차이가 컸다. SFT의 업데이트 $$L_2$$ norm은 평균 7.4 수준이고, RL은 $$3 \times 10^{-2}$$ 수준으로 100배 이상 작았다. 그리고 서로 다른 작업(예를 들어 Math와 Code) 사이의 업데이트 방향을 코사인 유사도로 재보면, SFT는 $$0.95 \sim 1.0$$ 혹은 반대 부호로 $$-0.97$$까지 나온다. 즉 두 작업의 업데이트 방향이 사실상 같은 축 위에 있다는 뜻이다. 한 작업의 업데이트가 다른 작업이 이미 자리 잡은 방향을 그대로 밀어버리니 순서대로 학습하면 이전 작업이 밀려나는 게 당연하다.

이걸 이론적으로 설명하는 부분이 이 논문의 핵심이다. SFT는 결국 정답 궤적을 그대로 따라가려는 off-policy 방식이라, 스코어 함수(로그 그래디언트)의 절대 크기 자체가 크다. 두 작업 간 간섭의 상한이 각 작업 스코어 함수 크기의 곱, $$M_i \cdot M_j$$로 결정되는데(Norm-limited), 이 $$M$$ 값 자체가 크니 간섭도 클 수밖에 없다는 걸 정리(Theorem 4.5)로 못박는다.

## RL은 왜 안 무너지는가

RL(GRPO)의 경우 그룹 내 여러 롤아웃의 보상에서 그룹 평균을 빼는 advantage 정규화가 핵심 열쇠다. 이 정규화는 단순히 분산을 줄이는 트릭으로 알려져 있었는데, 이 논문은 여기에 대수적으로 중요한 부작용이 있다는 걸 보여준다. Advantage 값들의 합이 0이 되는 영합(zero-sum) 성질 때문에, 서로 다른 롤아웃들이 공유하는 공통 그래디언트 방향 $$\bar{S}(x)$$가 계산 과정에서 그냥 사라진다. 남는 건 각 롤아웃이 그룹 평균에서 얼마나 벗어났는지를 나타내는 잔차 $$\delta S$$뿐이다.

여기에 더해 RL은 on-policy다. 외부 정답을 억지로 따라가는 게 아니라 현재 정책이 스스로 생성한 롤아웃을 학습 신호로 쓰기 때문에, 정책과 데이터 분포 사이의 괴리(KL divergence)가 크지 않고 잔차 변동성도 작게 유지된다. 그 결과 RL의 간섭 상한은 그룹 내 롤아웃 분산 $$V_i \cdot V_j$$로 결정되는 Variance-limited 형태가 되고, 이 $$V$$ 값이 SFT의 $$M$$ 값보다 훨씬 작으니 자연히 작업 간 간섭도 훨씬 작아진다. 실제로 측정한 RL 업데이트 간 코사인 유사도는 $$10^{-5}$$ 수준, 거의 직교(orthogonal)에 가까웠다.

## Parallel-RL: 이 통찰을 실전에 쓰기

작업별 RL 업데이트가 서로 직교한다면, 굳이 순서대로 학습시킬 필요가 없다는 발상으로 이어진다. 논문은 각 작업을 완전히 독립적으로 병렬 학습시킨 뒤, 얻어진 파라미터 변화량 $$\Delta W_1, \dots, \Delta W_N$$을 그냥 더하거나 TIES, SVD 같은 기존 모델 병합 기법으로 합치는 Parallel-RL을 제안한다. 여기에 전체 데이터의 5%만 써서 가볍게 사후 적응(post-merge adaptation)을 더한 Adapted Parallel-RL 버전은 단일 작업 전용 모델 대비 102.8%의 성능을 보존했다. 순차 학습이 만드는 충돌을 아예 우회하면서 학습 비용도 병렬화로 줄인 셈이다.

물론 이 실험은 작업 수가 4개(Math, Science, Code, Logic) 수준에 머물러 있다. 작업이 10개, 20개로 늘어나도 업데이트 방향들이 여전히 서로 직교한 채 남아 있을지는 이 논문만으로는 확신하기 어렵다. 고차원 공간이라 해도 방향의 개수가 늘면 우연히 겹치는 쌍이 생길 가능성은 있고, 논문도 이 스케일 문제를 명시적으로 다루지는 않았다.

## 용어 해설

- **Advantage Function**: 강화학습에서 특정 행동이 평균적인 행동보다 얼마나 더 나은지를 나타내는 값. 보상에서 baseline(기준값)을 뺀 형태로, 학습 신호의 분산을 줄이는 데 쓰인다.
- **On-policy / Off-policy**: On-policy는 현재 정책이 직접 생성한 데이터로 그 정책을 업데이트하는 방식이고, Off-policy는 다른 정책이나 외부에서 수집한 데이터로 정책을 업데이트하는 방식이다.
- **GRPO (Group Relative Policy Optimization)**: 같은 입력에 대해 여러 개의 응답(rollout)을 생성한 뒤, 그룹 내 평균 보상을 기준으로 각 응답의 상대적 우열을 매겨 정책을 업데이트하는 강화학습 알고리즘.

## 🤖 AI의 생각


이 논문의 재미는 결과 자체보다 "왜"를 설명하는 방식에 있다고 생각합니다. 흔히 경험적으로만 알려져 있던 현상에 코사인 유사도와 간섭 상한이라는 구체적인 수치와 증명을 붙였다는 점이 특히 인상적입니다.

<details class="tc-faq">
<summary>RL의 간섭 상한이 그룹 내 분산에 의해 결정된다면, rollout 개수 $$G$$를 늘려서 분산 $$V_i$$를 줄이면 작업 간 공존이 더 강해질까, 아니면 잔차 신호 자체가 약해져서 학습이 무뎌지는 트레이드오프가 생길까 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

논문은 $$G$$를 바꿔가며 이 트레이드오프를 직접 실험하지 않아서, 분산을 줄이는 것과 학습 신호를 유지하는 것 사이의 균형점이 어디인지는 후속 연구로 남아 있어 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft에서 여러 embodiment의 디코더를 하나의 공유 latent 위에서 순차적으로 파인튠하는 상황이 생기면, 이 논문이 말하는 BC(=SFT류) 방식의 파라미터 충돌이 그대로 재현될지 확인해볼 만하다.</summary>
<div class="tc-faq__body" markdown="1">

아직 taskcraft 파일럿은 단일 embodiment 스케일이라 직접 검증은 이르지만, 이 논문이 쓴 파라미터 delta의 pairwise 코사인 유사도 측정법 자체는 BC와 PPO 디코더를 비교할 때 그대로 진단 도구로 가져다 쓸 수 있을 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>Parallel-RL의 병합 방식이 작업 수가 4개에서 10개, 20개로 늘어나도 여전히 직교성을 유지할지, 아니면 고차원 공간에서도 우연한 방향 겹침이 누적될 위험이 있는지 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

이 논문 실험이 4개 작업 수준에 머물러 있어서 스케일이 커졌을 때의 검증은 빠져 있고, 작업 수가 늘수록 우연한 겹침 확률도 무시할 수 없을 것 같아 후속 검증이 필요해 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.03573" target="_blank" rel="noopener">SFT Conflicts, RL Coexists: A Theoretical and Empirical Analysis of Multi-Task Learning for LLMs</a> · hf-daily</span></div>
</div>
</div>
