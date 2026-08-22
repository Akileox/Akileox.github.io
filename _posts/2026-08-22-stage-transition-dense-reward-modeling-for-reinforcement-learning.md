---
title: "영상만으로 조밀 보상을 설계하는 법"
date: 2026-08-22
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "전문가 시연 비디오에서 단계 전이와 진행도를 동시에 읽어 장기 과제용 조밀 보상을 자동 설계하는 STDR을 정리했다."
paper_title: "Stage-Transition Dense Reward Modeling for Reinforcement Learning"
paper_summary: "전문가 시연 비디오에서 단계 전이와 진행도를 동시에 읽어 장기 과제용 조밀 보상을 자동 설계하는 STDR을 정리했다."
paper_url: "https://www.semanticscholar.org/paper/23c9942eea2300ef32f8c218b259910653c70104"
header:
  image: "/assets/images/posts/stage-transition-dense-reward-modeling-for-reinforcement-learning/figure-1.png"
  teaser: "/assets/images/posts/stage-transition-dense-reward-modeling-for-reinforcement-learning/figure-1.png"
---

이번 주는 로봇 조작 강화학습에서 오랫동안 발목을 잡아온 "보상 설계" 문제를 정면으로 다룬 논문을 골랐습니다. semantic-scholar 추천 경로에서 스코어 0.85로 올라온 이 논문은, 사람이 짜는 보상 함수 없이 전문가 시연 비디오만으로 조밀하고 단조 증가하는 보상 신호를 자동 생성한다는 점에서 제가 최근 보고 있는 로봇 학습 파이프라인들과 직결되는 문제의식을 가지고 있어 이번 주 1순위로 선택했습니다.

## 희소 보상이 못 하는 일

장기 수평선 로봇 조작 과제, 예를 들어 물건을 집어서 옮기고 정확한 자세로 내려놓는 다단계 작업을 강화학습으로 풀려면 결국 보상 함수가 필요하다. 가장 단순한 선택은 성공했을 때만 1을 주는 희소 보상(Sparse reward)인데, 이건 탐색 초기에 에이전트가 우연히 성공 궤적을 밟을 확률이 거의 0에 가깝기 때문에 학습이 사실상 멈춘다. 그래서 실무에서는 사람이 물체와 그리퍼 사이 거리, 목표 자세와의 오차 같은 걸 손으로 조합해 조밀 보상(Dense reward)을 설계하는데, 이게 물체 배치나 환경이 조금만 바뀌어도 깨진다는 게 고질적인 문제다.

이 문제를 풀려는 최근 시도가 VIP, LIV 같은 비디오 기반 표현 학습 보상들이다. 전문가 비디오와 현재 상태 사이의 잠재 공간 유사도를 보상으로 쓰는 방식인데, 이 논문이 지적하는 지점은 명확하다. 이런 유사도 기반 보상에는 시간적 방향성이라는 개념이 없다. "지금이 전체 과제의 어느 지점인가"를 모르니 중간 단계에서 보상이 정체(Plateau)되기 쉽고, 더 심각하게는 물체를 놓치는 등 실제로는 실패한 상황에서도 시각적으로 비슷하기만 하면 높은 보상을 잘못 주는 보상 해킹(Reward hacking)이 발생한다.

## 왜 유사도만으로는 안 되는가

핵심은 전문가 비디오 자체가 비구조화된 데이터라는 점이다. 서브태스크 경계도, 진행도 레이블도 없다. 그래서 기존 방법들은 어쩔 수 없이 전체 궤적을 하나의 연속 공간에 욱여넣고 거리로만 판단했고, 그 결과 "지금 몇 번째 단계를 수행 중인가"라는 이산적이고 논리적인 정보를 놓쳤다. 물체를 집는 단계와 옮기는 단계는 시각적으로 겹치는 프레임이 꽤 있는데, 유사도 기반 보상은 이 둘을 구분할 근거가 없다.

## VLM으로 단계를 나누고, 이중 구조로 채점한다

이 논문(STDR)의 해법은 두 층위로 보상을 나누는 것이다. 먼저 오프라인 단계에서 VLM(논문에서는 Qwen3 계열 사용)에게 전문가 비디오를 보여주고 $$K$$개의 기능적 단계로 나누게 해서 의사 레이블을 만든다. 사람이 직접 라벨링하지 않고 VLM의 시맨틱 이해력을 빌려오는 약지도(Weak supervision) 방식이다.

이 의사 레이블을 가지고 두 개의 모델을 학습시킨다.

| 모듈 | 입력/메커니즘 | 산출 | 역할 |
|---|---|---|---|
| Stage Discriminator | 슬라이딩 윈도우 관측 + LSTM | 거시적 단계 인덱스 $$k_t$$ | 지금이 전체 과제의 몇 번째 단계인지 판단 |
| Intra-Stage Progress Estimator | FiLM으로 단계 임베딩 조건화된 ResNet | 단계 내 진행도 $$p_{\text{prog}} \in [0,1]$$ | 같은 단계 안에서 얼마나 진행됐는지 미세하게 채점 |

이 둘을 합쳐서 $$R_{\text{stage}} = k_t/K$$와 $$R_{\text{prog}} = p_{\text{prog}}/K$$를 더한다. 진행도 보상을 $$K$$로 나눠주는 이유가 재밌는 설계 포인트인데, 이렇게 스케일을 눌러주면 같은 단계 안에서 진행도가 아무리 흔들려도 그 값이 다음 단계의 최소 보상값을 절대 넘지 못한다. 즉 전체 보상 곡선이 이산적인 단계 전이를 기준으로는 반드시 계단식으로 단조 증가하도록 수학적으로 보장된다. 유사도 기반 보상이 정체나 역행을 겪던 지점을 구조적으로 막아버린 셈이다.

![STDR 프레임워크의 오프라인 학습과 온라인 보상 계산 파이프라인](/assets/images/posts/stage-transition-dense-reward-modeling-for-reinforcement-learning/figure-1.png)

진행도 추정기를 학습시킬 때도 단순 회귀 손실만 쓰지 않는다. 시간 순서를 지키도록 강제하는 순위 손실, 인접 스텝 간 진행도가 절대 뒤로 가지 않게 하는 등위 정규화, 강/약 증강 이미지 사이 예측이 흔들리지 않게 하는 시각 일관성 손실까지 네 가지 손실을 함께 쓴다. 진행도라는 값 자체가 "시간이 지나면 반드시 늘어나야 한다"는 강한 사전 지식을 가지고 있는데, 이걸 손실 함수 설계 단계에서부터 명시적으로 박아넣은 것이다.

## OOD에서 보상 해킹을 막는 방법

여기서 끝나면 여전히 문제가 남는다. 학습 데이터에 없던 상태, 예를 들어 물체가 손에서 미끄러진 상황에 에이전트가 놓이면 판별기와 추정기 모두 신뢰할 수 없는 값을 뱉는다. 이 논문은 여기에 세 겹짜리 방어막을 둔다.

첫째, VAE로 관측 시퀀스를 재구성해서 오차가 임계값을 넘으면 아예 단계를 0으로 리셋해버린다. 둘째, 마할라노비스 거리로 지금 상태가 전문가 분포에서 얼마나 벗어났는지를 재서, 가까우면 회귀 기반 진행도를, 멀면 최근접 이웃 검색 기반의 보수적인 진행도를 쓰도록 게이트 함수로 부드럽게 섞고, 거리가 특정 임계값을 넘으면 진행도를 아예 0으로 깎는다. 셋째, 그리퍼가 물체를 실제로 안정적으로 쥐었는지를 별도 MLP로 검증해서, 파지에 실패했는데도 다음 단계로 보상이 새는 "단계 누수"를 막는다.

결과를 보면 이 설계가 실제로 작동한다. 성공 궤적에서는 STDR 보상이 VIP, LIV와 달리 정체 구간 없이 시간에 따라 매끄럽게 선형 증가한다. 반대로 물체를 떨어뜨리는 실패 궤적에서는 기존 표현 학습 기반 보상들이 여전히 높은 점수를 잘못 주는 반면, STDR은 즉시 낮은 보상으로 떨어뜨려서 실패를 정확히 감지한다.

![성공/실패 비디오에서 STDR과 VIP, LIV의 보상 곡선 비교](/assets/images/posts/stage-transition-dense-reward-modeling-for-reinforcement-learning/figure-3.png)

## 남는 한계

솔직히 이 구조는 VLM이 단계를 얼마나 잘 나눠주는지에 전체 성능이 크게 좌우된다는 뜻이기도 하다. 단계 수 $$K$$를 몇 개로 잡을지, VLM이 애매하게 경계를 그은 구간을 어떻게 다룰지는 여전히 사람 손이 개입할 여지가 있다. 또 손실 항이 네 개(회귀, 순위, 등위, 일관성)나 되기 때문에 가중치 $$\lambda$$들을 태스크마다 다시 튜닝해야 할 가능성이 높고, Stage Discriminator와 Progress Estimator를 둘 다 온라인으로 굴려야 하니 추론 비용도 유사도 기반 방법보다 늘어난다. 논문이 실제 로봇 실험으로 검증한 태스크 종류가 제한적이라, 단계 경계가 훨씬 모호한 과제(연속적인 변형 조작 같은)에도 이 이산 단계 가정이 잘 통할지는 별개의 질문으로 남는다.

## 용어 해설

- **FiLM (Feature-wise Linear Modulation)**: 신경망 중간 특징 맵에 조건 정보(여기서는 단계 임베딩)로부터 계산한 스케일과 편향값을 곱하고 더해서, 같은 백본이 조건에 따라 다른 특징에 집중하도록 만드는 조건화 기법.
- **마할라노비스 거리(Mahalanobis distance)**: 데이터의 분산과 변수 간 상관관계를 고려해서 한 점이 특정 분포로부터 얼마나 떨어져 있는지를 재는 거리 척도. 단순 유클리드 거리와 달리 분포의 모양을 반영한다.
- **OOD(Out-of-Distribution)**: 모델이 학습에 사용한 데이터 분포 범위를 벗어난 입력을 가리키는 말. 이런 입력에서는 모델 출력을 신뢰하기 어렵다.
- **VAE(Variational Autoencoder)**: 입력을 저차원 잠재 공간으로 압축했다가 다시 복원하도록 학습하는 생성 모델. 복원 오차가 크면 입력이 학습 데이터와 이질적이라는 신호로 쓸 수 있다.

## 🤖 AI의 생각


이 논문의 흥미로운 지점은 보상 학습을 "표현 유사도 문제"가 아니라 "시간 구조를 가진 분류 및 회귀 문제"로 재정의했다는 점이라고 생각합니다. 다만 이건 제 해석이고, 논문이 직접 그렇게 프레이밍하지는 않았다는 점은 짚어둡니다.

<details class="tc-faq">
<summary>VLM이 단계를 잘못 나눈 의사 레이블에 대해 이 시스템이 어느 정도까지 버틸 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문은 정확한 세그멘테이션을 전제로 실험했지만, 실제로는 노이즈가 섞인 라벨에서 Stage Discriminator가 얼마나 무너지는지에 대한 강건성 분석이 빠져 있어 이 부분이 궁금하다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 서로 다른 embodiment 간 정책을 옮기는 상황에서 이 단계 정의를 그대로 재사용할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

단계 경계가 embodiment에 무관한 태스크 의미 단위로 정의된다면 재사용 가능성이 있어 보이지만, 그리퍼 파지 조절기처럼 특정 하드웨어에 결합된 모듈은 embodiment가 바뀌면 다시 학습해야 할 것 같다

</div>
</details>

<details class="tc-faq">
<summary>단계 수 $$K$$를 고정하지 않고 과제 복잡도에 따라 동적으로 조절하는 방향은 시도해볼 만할까</summary>
<div class="tc-faq__body" markdown="1">

연속적인 변형 조작처럼 이산 단계 가정이 어색한 과제로 이 프레임워크를 확장하려면 결국 $$K$$ 자체를 학습 가능한 값으로 만드는 시도가 필요해 보인다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/23c9942eea2300ef32f8c218b259910653c70104" target="_blank" rel="noopener">Stage-Transition Dense Reward Modeling for Reinforcement Learning</a> · semantic-scholar-rec</span></div>
</div>
</div>
