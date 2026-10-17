---
title: "월드 모델과 액션 모델은 서로 돕는가"
date: 2026-10-17
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "비디오 예측과 로봇 액션 생성을 하나의 인과적 구조에 섞어 학습하면 둘 다 좋아진다는 OneWM의 실험 노트"
paper_title: "Can World and Action Modeling Benefit Each Other under Unified Causal Training?"
paper_summary: "비디오 예측과 로봇 액션 생성을 하나의 인과적 구조에 섞어 학습하면 둘 다 좋아진다는 OneWM의 실험 노트"
paper_url: "https://huggingface.co/papers/2610.21095"
header:
  image: "/assets/images/posts/can-world-and-action-modeling-benefit-each-other-under-unified-causal-training/figure-1.png"
  teaser: "/assets/images/posts/can-world-and-action-modeling-benefit-each-other-under-unified-causal-training/figure-1.png"
---

이번 주 HF Daily 스코어 1.00으로 가장 먼저 눈에 띈 논문입니다. 로보틱스와 World Model을 같이 보는 입장에서, "비디오 예측과 액션 생성을 한 모델에 묶으면 서로 도움이 되는가"라는 질문 자체가 taskcraft가 깔고 가는 전제와 거의 겹쳐서 이번 주 노트로 골랐습니다.

## 문제의식: 월드 모델과 액션 모델은 왜 따로 놀았나

로봇 정책을 학습할 때 미래 비디오를 예측하는 World Model과, 실제로 관절이나 엔드이펙터를 움직이는 Action Model은 오랫동안 각자의 길을 걸어왔다. 비디오 예측이 정책 학습에 보조 신호로 쓰인 사례는 많지만, "정답 액션이 주어졌을 때 미래 비디오를 정확히 예측하는 순방향 동역학 학습이 액션 생성 자체를 더 잘하게 만드는가"라는 인과적 방향의 질문에는 명확한 답이 없었다. 반대 방향, 즉 액션을 잘 생성하도록 학습하는 과정이 월드 모델의 예측 충실도를 높이는지도 마찬가지로 불분명했다. 이 논문 OneWM은 이 두 방향의 상호 이점을 한 번에 검증하겠다는 문제의식에서 출발한다.

## 왜 기존 Causal World-Action Model은 안 됐나

기존 Causal WAM(World-Action Model) 계열은 비디오와 액션을 같은 transformer 안에서 동시에 디노이징하는 구조를 많이 썼다. 그런데 이 과정에서 두 가지 설계 중 하나를 택해야 했다. 하나는 비디오 토큰이 액션 토큰을 아예 못 보게 마스킹하는 방식이고, 다른 하나는 비디오가 액션을 보되 그 액션이 항상 노이즈가 낀 상태로만 주어지는 방식이다. 전자는 비디오 예측이 액션의 인과적 영향을 전혀 학습하지 못하고, 후자는 노이즈 낀 액션을 조건으로 물리적으로 정확한 미래를 예측해야 하니 학습 신호 자체가 왜곡된다. 결국 둘 다 "정답 액션이 주어졌을 때 깨끗하게 순방향 동역학만 학습"하는 구간을 모델에게 제공하지 못했던 것이다.

아래 그림은 이 차이를 어텐션 맵으로 보여준다.

![기존 World Model, WAM, OneWM의 액션 조건화 방식 비교](/assets/images/posts/can-world-and-action-modeling-benefit-each-other-under-unified-causal-training/figure-1.png)

왼쪽은 순수 World Model로 액션이 항상 clean하게 주어지지만 정책 학습 신호가 없고, 오른쪽은 일반적인 WAM으로 비디오가 액션을 아예 못 보거나(Masked) 노이즈 낀 액션만 보는(Noisy) 구조다. 가운데가 OneWM인데, 비디오가 보는 액션의 clean 비율을 0%부터 100%까지 자유롭게 섞을 수 있다는 게 핵심 차이다.

## 고친 방법: 인과적 Mixture-of-Transformers와 가변 액션 컨디셔닝

OneWM은 사전학습된 비디오 생성 모델(Wan2.2-TI2V-5B)을 기반으로 Video DiT와 Action DiT를 별도 파라미터로 분리하고, 그 사이를 시간적 인과 마스크(Causal Mask)가 적용된 Joint Attention으로 연결한다. 이게 Mixture-of-Transformers 구조다. 미래 비디오 토큰의 쿼리는 지금까지의 비디오와 액션을 모두 참조할 수 있고($$S_v(i) = V_{\le i} \cup A_{\le i}$$), 반대로 액션 토큰의 쿼리는 초기 관측 $$V_0$$와 이전 액션 $$A_{\le i}$$만 참조한다($$S_a(i) = V_0 \cup A_{\le i}$$). 액션 쪽이 미래 비디오를 못 보게 막아둔 이유는 명확한데, 실제 로봇 배치 시점에 매번 미래를 "상상"해서 생성할 여유가 없기 때문이다. 이 비대칭 가시성 설계 덕분에 추론할 때는 비디오 생성 없이 액션만 빠르게 뽑을 수 있다.

그리고 이 논문의 진짜 아이디어는 액션 컨디셔닝을 배치 단위로 섞는다는 점이다. 액션 청크 길이 $$K$$ 중 clean prefix 길이 $$r$$을 샘플마다 다르게 주는데, 세 모드로 정리하면 이렇다.

| 모드 | 조건 | 학습되는 것 |
|---|---|---|
| All-clean ($$r=K$$) | 액션 전체가 정답으로 주어짐 | 순수 순방향 동역학(미래 비디오 예측) |
| Partial-clean ($$0<r<K$$) | 앞부분은 clean, 뒷부분은 noisy | noisy suffix에 대한 액션 디노이징 |
| All-noisy ($$r=0$$) | 액션 전체가 노이즈 상태 | 표준 WAM 형태의 동시 학습 |

기본 비율은 All-noisy 60%, Partial-clean 25%, All-clean 15%로 섞었다. 핵심 손실은 flow matching 형태로

$$L_v = \mathbb{E}\left[ w_v(\tau_v) \\vert{}\hat{u}_v - (\epsilon_v - z^{\text{fut}})\\vert{}^2_{\text{mean}} \right]$$

$$L_a = \mathbb{E}_{r < K}\left[ w_a(\tau_a) \\vert{}\hat{u}_{a, >r} - (\epsilon_{a, >r} - a_{>r})\\vert{}^2_{\text{mean}} \right]$$

로 정의되고, 둘을 $$L = \lambda_v L_v + \lambda_a L_a$$로 묶어 동시에 최적화한다. $$r=K$$일 때는 디노이징할 액션이 없으니 $$L_a$$가 자연히 빠진다는 점이 수식에 반영돼 있다. 결국 한 배치 안에서 어떤 샘플은 "정답 액션을 보고 미래를 정확히 그려보는 연습"을, 다른 샘플은 "노이즈에서 액션을 복원하는 연습"을 하는 셈이고, 같은 causal joint attention 공간을 공유하기 때문에 두 연습의 표현이 서로 흘러 들어간다.

![OneWM 전체 아키텍처: Video DiT와 Action DiT가 Causal Joint Attention으로 연결된 구조](/assets/images/posts/can-world-and-action-modeling-benefit-each-other-under-unified-causal-training/figure-2.png)

이 구조도를 보면 VAE로 인코딩된 프레임이 Video DiT로, 임베딩된 액션 청크가 Action DiT로 들어가고 둘 다 언어 지시문을 크로스 어텐션으로 받는다. 하단 패널에 All Clean, Clean Prefix/Noisy Suffix, All Noisy 세 가지 입력 구성이 나란히 그려져 있는데, 이게 바로 위 표의 세 모드가 같은 forward pass 안에서 섞여 들어가는 그림이다.

## 결과와 한계

논문은 all-clean 비중을 늘릴수록 순방향 동역학 예측(미래 비디오의 물리적 정확도)이 좋아지고, 반대로 all-noisy 비중이 일정 수준 이상 유지돼야 정책 쪽 일반화 성능이 유지된다는 trade-off를 관찰했다고 설명한다. 즉 저자들이 찾은 60/25/15라는 비율은 그 자체로 하나의 튜닝 결과이지, 원리적으로 유도된 최적점은 아니다. 또한 액션 표현이 특정 로봇의 액션 공간에 묶여 있어서, 다른 embodiment로 그대로 옮겨 쓰기는 어렵다. 비디오 쪽도 사전학습된 대형 비디오 생성 모델에 의존하기 때문에, 그 백본의 품질이 전체 시스템 성능의 상한을 사실상 정하는 구조라는 점도 한계로 남는다.

## 용어 해설

- **Mixture-of-Transformers**: 하나의 큰 transformer를 쓰는 대신 모달리티별로 파라미터를 분리한 여러 transformer를 두고, 특정 지점(어텐션 등)에서만 서로 정보를 교환하게 만드는 구조.
- **Flow Matching**: 노이즈 분포에서 데이터 분포로 가는 결정론적 속도장(velocity field)을 직접 회귀하도록 학습시켜 생성 모델을 만드는 기법으로, diffusion 모델의 대안 또는 확장으로 쓰인다.
- **Causal Mask**: 어텐션 연산에서 미래 시점의 토큰이 과거 시점의 토큰만 참조하고 반대로는 참조하지 못하도록 막는 마스크로, 시계열이나 순차 생성 모델에서 시간 순서를 지키기 위해 쓴다.

## 🤖 AI의 생각


사실관계 요약과는 별개로, 개인적으로는 이 논문이 "비율 튜닝"이라는 다소 공학적인 해법으로 깔끔한 인과적 질문에 답했다는 점이 인상적이면서도 조금 아쉽습니다.

<details class="tc-faq">
<summary>all-clean, partial-clean, all-noisy 비율이 로봇 task 난이도나 데이터 양에 따라 달라져야 하는지 확인했을까</summary>
<div class="tc-faq__body" markdown="1">

논문에서는 고정 비율로 전체 실험을 돌린 것으로 보이는데, task별로 최적 비율이 달라진다면 그 자체가 또 다른 연구 주제가 될 만하다.

</div>
</details>

<details class="tc-faq">
<summary>비대칭 가시성 설계(\(S_v, S_a\))를 taskcraft의 사람 영상 latent와 embodiment별 디코더 사이에도 그대로 가져올 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

구조는 닮아 있지만 taskcraft는 액션 공간 자체가 embodiment마다 다르다는 전제가 있어서, 액션 쪽 가시성 제한보다 latent transition 쪽 공유 범위를 어떻게 설계할지가 더 핵심 질문이 될 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>사전학습된 비디오 생성 모델에 의존하는 구조가 다른 도메인(예: 조작이 아닌 보행 로봇)으로도 잘 옮겨질까</summary>
<div class="tc-faq__body" markdown="1">

비디오 백본이 사람이나 조작 중심 데이터로 학습됐다면 보행처럼 전신 동역학이 중요한 도메인에서는 순방향 예측 품질이 떨어질 가능성이 있어 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2610.21095" target="_blank" rel="noopener">Can World and Action Modeling Benefit Each Other under Unified Causal Training?</a> · hf-daily</span></div>
</div>
</div>
