---
title: "0.5% 파라미터가 100%를 이기는 이유"
date: 2026-10-20
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "거대 모델을 통째로 미세조정하지 않고도 참조 이미지 분할 성능을 끌어올린 양방향 어댑터 구조를 들여다봤습니다."
paper_title: "0.5%>100%: Bidirectional Reciprocal Learning for Referring Image Segmentation"
paper_summary: "거대 모델을 통째로 미세조정하지 않고도 참조 이미지 분할 성능을 끌어올린 양방향 어댑터 구조를 들여다봤습니다."
paper_url: "https://www.semanticscholar.org/paper/3f39d4ce53bd7495b77f76874b70668410e7f47a"
header:
  image: "/assets/images/posts/0-5-100-bidirectional-reciprocal-learning-for-referring-image-segmentation/figure-1.png"
  teaser: "/assets/images/posts/0-5-100-bidirectional-reciprocal-learning-for-referring-image-segmentation/figure-1.png"
---

이번 주 논문은 "0.5%>100%: Bidirectional Reciprocal Learning for Referring Image Segmentation"입니다. Semantic Scholar 추천 경로에서 스코어 0.85로 올라온 논문인데, 숫자부터 자극적인 제목이 눈길을 끌었습니다. 파라미터의 0.5% 미만만 학습해서 전체를 다 미세조정한 모델과 멀티모달 LLM까지 앞선다는 주장은 검증해볼 가치가 있다고 판단해서 이번 주 노트로 골랐습니다.

참조 이미지 분할(Referring Image Segmentation, RIS)은 "빨간 가방을 든 사람"같은 자연어 표현을 읽고, 이미지 안에서 그 표현이 가리키는 객체의 픽셀을 정확히 분할해내는 작업이다. 최근에는 DINOv3 같은 강력한 시각 파운데이션 모델(VFM)을 백본으로 끌어다 쓰는 흐름이 대세인데, 여기서 바로 문제가 시작된다.

## 문제 제기: 통째로 미세조정하면 생기는 부작용

VFM을 RIS에 가져다 쓰는 가장 직관적인 방법은 전체 파라미터를 새로 미세조정(Full Fine-Tuning)하는 것이다. 그런데 이렇게 하면 두 가지 비용이 든다. 첫째는 단순한 계산 비용, 둘째는 더 심각한데 사전학습 때 애써 쌓아둔 단일 모달 시각 표현이 망가지는 파멸적 망각(Catastrophic Forgetting)이다. 모델이 RIS 태스크에 맞춰 적응하는 과정에서 원래 가지고 있던 범용적인 시각 이해 능력 자체가 흐려지는 셈이다.

그래서 등장한 대안이 파라미터 효율적 미세조정(PEFT), 즉 어댑터를 붙이고 백본은 얼려두는 방식이다. 그런데 기존 어댑터들도 한계가 있었다. 시각과 언어 스트림에 각각 독립적인 어댑터를 붙이는 경우는 두 모달리티가 서로 대화를 하지 못하고, 언어 정보를 시각 피처 변조를 위한 정적인 가이드로만 쓰는 단방향 융합 방식은 한 방향으로만 정보가 흐른다. 즉 시각적 증거를 보고 모호한 텍스트 표현을 다시 정제하는 피드백 루프가 없었다. "그 사람 옆에 있는 가방"처럼 모호한 표현은 이미지를 봐가면서 의미가 점점 좁혀져야 하는데, 기존 구조는 그걸 한 번에 끝내버리고 되돌아보지 않는 셈이다.

## 왜 안 됐는지: 일방향 융합의 구조적 한계

아래 그림이 이 문제를 잘 정리해준다.

![기존 PEFT 방식과 BRL 패러다임 비교](/assets/images/posts/0-5-100-bidirectional-reciprocal-learning-for-referring-image-segmentation/figure-1.png)

(a)는 시각과 언어 어댑터가 서로 독립적으로 붙어있는 구조고, (b)는 언어가 시각 쪽으로만 흘러들어가는 단방향 구조다. 두 구조 모두 공통적으로, 언어 표현이 한 번 정해지면 그 다음부터는 고정값 취급을 받는다. 시각 정보가 언어 쪽으로 되돌아가서 "이 표현이 맞는 뜻인지" 다시 체크해주는 경로가 아예 없다. 결국 모달리티 간 불일치가 레이어를 거칠수록 누적되는 구조적 문제가 생긴다.

## 고친 방법: 토큰 수준과 채널 수준에서 양방향으로 주고받기

저자들이 제안한 Bidirectional Reciprocal Learning(BRL)은 VFM과 언어 모델(LFM)을 둘 다 동결한 채, 각 트랜스포머 레이어에 두 종류의 가벼운 어댑터를 잔차(residual) 형태로 끼워 넣는다.

하나는 셀프 어텐션 블록에 붙는 Reciprocal Attention Adapter(RAA)다. 보통 크로스 어텐션은 한쪽 모달리티의 쿼리로 다른 쪽의 값을 가져오는 1차 연산에서 끝나는데, RAA는 여기서 한 단계 더 나아간다. 시각에서 언어로 가는 어텐션 맵과 언어에서 시각으로 가는 어텐션 맵을 각각 구한 다음, 이 두 맵을 서로 곱해서 2차 상호 어텐션 맵을 만든다. 시각 토큰이 언어 공간을 한 번 경유했다가 다시 시각 토큰으로 돌아오는 경로를 만드는 셈이라, 언어적 맥락이 반영된 공간 구조를 시각 쪽이 학습하고, 동시에 언어 토큰도 시각적 증거를 경유해서 자기 의미를 재평가하게 된다.

다른 하나는 피드포워드 블록에 붙는 Reciprocal Gate Adapter(RGA)다. 이쪽은 토큰 하나하나가 아니라 채널 전체의 전역 통계를 보는 역할을 한다. 각 모달리티의 피처를 전역 평균 풀링해서 하나의 요약 벡터로 만들고, 시그모이드를 거친 게이팅 신호를 상대 모달리티의 채널에 곱한다. 언어의 전역 의미와 일치하는 시각 채널은 증폭되고, 관계없는 배경 잡음성 채널은 억제되는 식이다.

![BRL 프레임워크의 세부 아키텍처](/assets/images/posts/0-5-100-bidirectional-reciprocal-learning-for-referring-image-segmentation/figure-3.png)

이 그림에서 보이듯, RAA는 레이어 내부에서 공간적 정렬을 담당하고 RGA는 전역적인 의미 보정을 담당하도록 역할이 분업되어 있다. 둘 다 백본 자체를 건드리지 않고 다운 프로젝션, 상호작용, 업 프로젝션으로 이어지는 작은 bottleneck 구조만 추가하기 때문에, 전체 파라미터의 0.5% 미만만 학습 대상이 된다. 그리고 흥미롭게도 손실 함수는 별도의 대조 학습(Contrastive Learning) 항 없이 표준적인 BCE와 Dice 분할 손실만 사용한다. 저자들의 주장은, 모달리티 정렬이라는 문제 자체를 대조 손실로 풀 필요 없이 RAA와 RGA라는 구조적 상호작용으로 이미 흡수했다는 것이다.

## 결과와 남는 질문

논문에서 밝힌 수치대로라면 1.5M에서 1.8M 수준의 파라미터만 학습해서 전체 미세조정 모델과 MLLM 기반 접근을 모두 능가했다고 한다. 백본을 완전히 얼린 채로 이 정도 결과를 낸다는 건, PEFT 어댑터의 설계를 "독립적으로 적응시키는 것"에서 "서로 피드백을 주고받게 만드는 것"으로 바꾼 효과가 꽤 크다는 뜻이다.

다만 아쉬운 지점도 있다. RAA의 2차 어텐션 맵은 어텐션 행렬끼리 곱하는 연산이라 토큰 수가 많아지면 계산 비용이 어떻게 늘어나는지, 그리고 레이어를 몇 개나 선택해서 어댑터를 붙여야 최적인지에 대한 민감도 분석은 상세히 다뤄지지 않았다. 또한 이 논문이 RIS라는 비교적 좁은 태스크에 한정돼 있어서, 이 상호 어댑터 구조가 다른 비전-언어 태스크, 예를 들어 VQA나 grounding 같은 데서도 같은 효율을 내는지는 별도로 확인이 필요해 보인다.

## 용어 해설

- **파라미터 효율적 미세조정(PEFT)**: 거대 사전학습 모델의 대부분 가중치는 고정(동결)한 채, 일부 작은 모듈이나 파라미터만 추가로 학습시켜 적은 비용으로 특정 태스크에 적응시키는 방법론.
- **잔차 연결(Residual Connection)**: 어떤 블록의 출력에 그 블록의 입력을 그대로 더해주는 구조로, 깊은 신경망에서 정보 손실과 기울기 소실 문제를 줄이기 위해 널리 쓰인다.
- **대조 학습(Contrastive Learning)**: 비슷한 쌍(positive)은 임베딩 공간에서 가깝게, 다른 쌍(negative)은 멀게 만들도록 학습하는 방식으로, 서로 다른 모달리티나 뷰 사이의 정렬을 학습할 때 자주 쓰인다.
- **파멸적 망각(Catastrophic Forgetting)**: 신경망이 새로운 태스크를 학습하면서 기존에 학습했던 지식이나 능력을 잃어버리는 현상.

## 🤖 AI의 생각

이 논문은 로봇이나 embodiment 문제와는 거리가 먼 순수 비전-언어 분할 연구이지만, 설계 철학 하나는 눈여겨볼 만하다고 생각합니다. 거대한 두 사전학습 모델을 완전히 동결한 채 얇은 어댑터만으로 양방향 피드백을 만든다는 발상은, 결국 서로 다른 표현 공간을 잇는 다리를 어떻게 설계할 것인가의 문제로 요약됩니다.

<details class="tc-faq">
<summary>RAA의 2차 어텐션 맵 합성이 레이어가 깊어질수록 노이즈를 증폭시키지는 않는지, 저자들이 몇 번째 레이어에만 붙였는지에 대한 민감도 분석이 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

논문에 명시된 레이어 선택(예: 5, 8, 11번째)이 경험적 튜닝 결과일 가능성이 높아 보이고, 레이어 수를 늘렸을 때 성능이 포화되는지 아니면 과적합으로 꺾이는지는 추가 실험으로 확인해볼 지점이다.

</div>
</details>

<details class="tc-faq">
<summary>대조 손실 없이도 모달리티 정렬이 된다는 주장이 RIS보다 더 어려운 open-vocabulary grounding 태스크에서도 유지될까.</summary>
<div class="tc-faq__body" markdown="1">

분할이라는 조밀한 픽셀 단위 지도 신호 자체가 이미 강한 정렬 힌트를 주기 때문일 수 있어서, 약한 지도 신호만 있는 태스크에서는 대조 손실의 역할이 다시 필요해질 수도 있다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft에서 사람 시연 영상의 latent world model과 특정 embodiment 정책을 연결할 때 이런 동결 백본 플러스 경량 상호 어댑터 구조를 가져다 쓸 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

두 표현 공간을 각각 보존하면서 얇은 다리만 학습시킨다는 아이디어 자체는 재사용할 만하지만, RIS는 정적인 이미지 텍스트 쌍이고 taskcraft는 시계열 행동 데이터라서 RAA의 2차 어텐션을 시간 축으로 어떻게 확장할지는 별도로 고민해야 할 문제다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/3f39d4ce53bd7495b77f76874b70668410e7f47a" target="_blank" rel="noopener">0.5%>100%: Bidirectional Reciprocal Learning for Referring Image Segmentation</a> · semantic-scholar-rec</span></div>
</div>
</div>
