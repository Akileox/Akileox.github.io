---
title: "RoboTok, 웹 영상 속 손동작을 검색하다"
date: 2026-09-05
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "DTW로 학습한 3D 손 궤적 임베딩으로 인터넷 영상에서 로봇 시연에 쓸 손동작을 검색하는 데이터 엔진을 소개합니다."
paper_title: "RoboTok: An Internet-Scale Data Engine for Human Demonstration Retrieval and Dexterous Manipulation Learning"
paper_summary: "DTW로 학습한 3D 손 궤적 임베딩으로 인터넷 영상에서 로봇 시연에 쓸 손동작을 검색하는 데이터 엔진을 소개합니다."
paper_url: "https://huggingface.co/papers/2609.03199"
header:
  image: "/assets/images/posts/robotok-an-internet-scale-data-engine-for-human-demonstration-retrieval-and-dext/figure-1.png"
  teaser: "/assets/images/posts/robotok-an-internet-scale-data-engine-for-human-demonstration-retrieval-and-dext/figure-1.png"
---

이번 주에는 로봇 학습 데이터 확보 문제를 다루는 RoboTok을 골랐습니다. 로봇 조작 정책 학습에서 데이터 부족은 거의 만성적인 병목인데, 이 논문은 그 해법을 유튜브 규모의 인간 시연 영상에서 찾으면서도 단순히 "비슷해 보이는 영상"이 아니라 "실제로 손이 비슷하게 움직이는 영상"을 검색한다는 점에서 방법론적으로 신선했습니다. taskcraft가 다루는 embodiment-agnostic task representation 문제와도 접점이 커서 스코어를 높게 매겼습니다.

## 문제 제기: 로봇은 데이터가 없는데 인터넷엔 손이 넘친다

로봇 조작 정책을 학습시키려면 다양한 과제와 객체, 환경을 아우르는 시연이 필요하다. 그런데 로봇으로 직접 시연을 모으는 건 비싸고 느리다. 반면 인터넷에는 HowTo100M이나 Action100M 같은 코퍼스에 사람이 물건을 다루는 영상이 수억 개 쌓여 있다. 문제는 이 영상들이 카메라 시점도 제각각이고, 장면 배경도 다르고, 사람 몸이 프레임 밖으로 잘려 있기 일쑤라는 점이다. "칼질하는 영상을 찾아줘" 같은 질의에 대해, 겉모습이 비슷한 부엌 장면을 찾는 것과 실제로 손목과 손가락이 칼질과 유사한 궤적을 그리는 클립을 찾는 것은 전혀 다른 문제다.

## 왜 안 됐는지: 시각적 유사성은 모션 유사성이 아니다

기존 로봇 데이터 검색 연구들(FlowRetrieval, HAND, STRAP 등)은 대체로 고정된 로봇 데이터셋 안에서, 그것도 2D 이미지 공간의 optical flow나 의미(semantic) 라벨을 기준으로 검색했다. 이 방식들을 그대로 웹 영상에 적용하면 두 가지가 무너진다. 첫째, 2D 이미지 공간 표현은 카메라 시점이 바뀌면 완전히 다른 값이 되어 버리므로, 똑같은 칼질 동작도 촬영 각도가 다르면 다른 걸로 인식된다. 둘째, 의미 라벨(예: "cutting")에 의존하면 라벨은 같아도 실제 손동작의 리듬과 궤적 형태는 완전히 다를 수 있다는 걸 놓친다. 즉 "무엇을 하는가"는 맞아도 "어떻게 움직이는가"가 다른 걸 걸러내지 못한다. 이 논문이 Figure 4에서 보여주는 비교가 정확히 이 지점을 짚는데, 베이스라인들은 "칼질" 질의에 대해 상당수 무관한 클립을 상위로 올리는 반면, RoboTok은 top-3 전부 실제로 같은 동작을 검색해낸다.

## 고친 방법: 3D 손 궤적을 뽑고, DTW로 순위를 매기고, 임베딩으로 증류한다

핵심 아이디어는 명확하다. 시각적 외형이나 의미 라벨 대신, 손이 3D 공간에서 어떻게 움직이는지 그 자체를 비교 기준으로 삼는다.

먼저 데이터 파이프라인이다. Action100M에서 4에서 8초 길이의 정적 카메라 클립을 걸러낸 뒤, WiLoR로 3D 손 키포인트를, MoGe-2로 메트릭 깊이를 뽑아 카메라 좌표계에서 손의 3D 자세를 복원한다. 여기서 흥미로운 디테일은 몸통(torso) 프레임 추정이다. 웹 영상은 사람 몸 전체가 안 보이는 경우가 많은데, 손목 프레임만으로 몸통 좌표계를 추정하는 경량 모델을 따로 학습시켜서, 몸이 안 보여도 손 궤적을 자기중심(egocentric) 좌표계로 정규화할 수 있게 만들었다.

![RoboTok 데이터 파이프라인: 클립 필터링, 3D 손 복원, 자기중심 좌표 정규화](/assets/images/posts/robotok-an-internet-scale-data-engine-for-human-demonstration-retrieval-and-dext/figure-2.png)

다음은 유사도 오라클이다. 정규화된 두 궤적 사이에 Dynamic Time Warping(DTW)을 계산해서 실행 속도 차이에 흔들리지 않는 모션 유사도를 정의한다.

$$
\text{DTW}(x_i, x_j) = \min_{\pi \in \Pi} \sum_{(t,u) \in \pi} d(x_i^t, x_j^u), \quad d(x_i^t, x_j^u) = \lVert x_i^t - x_j^u \rVert_2
$$

문제는 DTW 자체가 계산 비용이 커서 코퍼스 전체를 대상으로 매번 계산할 수는 없다는 점이다. 그래서 DTW 순위를 학습 시점에만 지도 신호로 쓰고, 추론 시에는 값싼 벡터 내적으로 근사하도록 임베딩을 학습시킨다. cross-attention 인코더가 프레임별 궤적을 pooling해서 단위 초구 위의 정규화 벡터로 사상하고, 오라클이 유도하는 순위가 임베딩 공간의 내적 순위로 그대로 보존되도록 set loss와 rank loss를 함께 최적화한다.

$$
s(i,j) > s(i,k) \implies \langle \Gamma(x_i), \Gamma(x_j) \rangle > \langle \Gamma(x_i), \Gamma(x_k) \rangle
$$

학습이 끝나면 코퍼스 전체를 한 번씩만 인코딩해 인덱스에 저장해두고, 질의는 코사인 유사도 최근접 이웃 검색만으로 처리한다. 새 클립이 들어와도 재학습 없이 인코더 forward pass 한 번이면 인덱스에 추가된다는 점이 실용적으로 크다.

## 결과: 라벨 없이도 동작별로 군집이 생긴다

가장 인상적인 확인은 t-SNE 시각화(Figure 1)다. 이 임베딩은 어떤 명시적 의미 라벨도 학습에 쓰지 않았는데, "믹싱", "코바늘뜨기", "드릴링", "그리기" 같은 카테고리로 자연스럽게 군집이 나뉜다. 즉 모션 구조 자체가 의미 구조와 상당 부분 겹친다는 걸 데이터가 보여준 셈이다.

![RoboTok 임베딩의 t-SNE 시각화, 라벨 없이도 동작별로 군집이 형성됨](/assets/images/posts/robotok-an-internet-scale-data-engine-for-human-demonstration-retrieval-and-dext/figure-1.png)

정량적으로도 Recall@k, DTW cost@k, CKNNA@k를 k=1부터 500까지 늘려가며 확인했을 때, 학습이 실제로 최적화한 범위는 local k=20 수준인데도 훨씬 넓은 범위까지 DTW 오라클과 순위가 일치했다. 이는 단순 암기가 아니라 일반화된 모션 구조를 학습했다는 근거로 제시된다. 다운스트림으로는 검색된 인간 시연을 로봇 상태 공간에 정합한 뒤, 현재 로봇 상태와 시연 상태 뱅크 사이 k-NN 거리를 potential-based reward로 써서 PPO 정책 학습에 붙였고, VTDexManip 벤치마크에서 검증했다.

## 한계

다만 짚어야 할 지점도 있다. 손 자세만으로 모션 유사도를 정의하다 보니, 손이 접촉하는 객체의 형태나 물리적 특성(단단함, 마찰)은 표현에 전혀 반영되지 않는다. 겉보기 궤적은 비슷해도 실제로는 완전히 다른 물성의 물체를 다루는 시연이 섞여 들어올 여지가 있다. 또한 몸통 프레임 추정 모델 자체가 SMPL-H 기반 근사이기 때문에, 극단적으로 몸이 가려진 영상에서는 정규화 오차가 누적될 수 있다. 그리고 최종적으로 로봇 embodiment로 옮기는 단계는 여전히 손 자세를 로봇 상태 공간에 정합하는 수작업에 가까운 과정에 의존하고 있어서, 이 부분의 자동화는 아직 남은 숙제로 보인다.

## 용어 해설

- **Dynamic Time Warping (DTW)**: 두 시계열 데이터가 서로 다른 속도로 진행되더라도 형태가 비슷한지 비교하기 위해, 시간축을 비선형적으로 늘리거나 줄여 정렬한 뒤 누적 거리를 계산하는 알고리즘이다.
- **Potential-based reward shaping**: 강화학습에서 원래의 sparse reward에 상태 간 거리나 진행도 같은 보조 신호(potential function)를 더해, 학습 속도를 높이면서도 최적 정책 자체는 바뀌지 않도록 보장하는 보상 설계 기법이다.
- **Cross-attention**: Transformer의 attention 메커니즘 중 하나로, 서로 다른 두 시퀀스(예: 질의 벡터와 입력 시퀀스) 사이의 관련성을 계산해 한쪽이 다른 쪽 정보를 선택적으로 참조하도록 만드는 구조다.

## 🤖 AI의 생각


개인적인 감상으로는, 이 논문의 진짜 기여는 "웹 영상에서 손동작을 뽑았다"는 것보다 "DTW라는 비싼 오라클을 학습 시점에만 쓰고 추론은 벡터 내적으로 때운다"는 증류 설계 자체라고 본다. 이 패턴은 다른 도메인의 검색 문제에도 그대로 옮겨 쓸 수 있을 것 같다.

<details class="tc-faq">
<summary>손 궤적만으로 정의된 유사도가 접촉하는 객체의 물성 차이를 놓친다는 한계를 논문이 어떻게 완화할 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

손 궤적 임베딩에 객체 인식이나 접촉 힘 추정 결과를 보조 채널로 붙여 DTW 오라클 자체를 다중 모달로 확장하는 방향이 자연스러워 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 Latent World Model이 추구하는 embodiment-agnostic 표현과 RoboTok의 순위 보존 임베딩을 합치면 무엇을 얻을 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

검색으로 찾은 유사 시연을 world model의 학습 신호로 바로 흘려넣어, 검색과 예측을 분리하지 않고 한 파이프라인으로 묶는 실험이 흥미로울 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>potential-based reward로 시연 매니폴드를 따르게 하는 방식이 GAIL 같은 적대적 모방학습보다 정말 안정적인지는 어떻게 검증할 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

동일한 검색 시연 뱅크를 두고 GAIL 판별자 기반 보상과 k-NN 거리 기반 보상을 같은 정책 학습 세팅에서 직접 맞대결시키는 ablation이 필요해 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.03199" target="_blank" rel="noopener">RoboTok: An Internet-Scale Data Engine for Human Demonstration Retrieval and Dexterous Manipulation Learning</a> · hf-daily</span></div>
</div>
</div>
