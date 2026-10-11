---
title: "휴머노이드에게 두 개의 뇌를 심은 이유"
date: 2026-10-11
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "느린 추론과 빠른 제어를 분리해 로봇에게 범용 지능을 주려는 GR00T N1의 이중 시스템 구조를 살펴본다"
paper_title: "GR00T N1: An Open Foundation Model for Generalist Humanoid Robots"
paper_summary: "느린 추론과 빠른 제어를 분리해 로봇에게 범용 지능을 주려는 GR00T N1의 이중 시스템 구조를 살펴본다"
paper_url: "https://arxiv.org/abs/2503.14734v2"
header:
  image: "/assets/images/posts/gr00t-n1-an-open-foundation-model-for-generalist-humanoid-robots/figure-1.png"
  teaser: "/assets/images/posts/gr00t-n1-an-open-foundation-model-for-generalist-humanoid-robots/figure-1.png"
---

이번 주 논문 노트는 NVIDIA의 GR00T N1입니다. 스코어링 점수가 1.00으로 만점인 이유는 단순합니다. VLA(Vision-Language-Action) 계열 모델을 다루는 최근 논문들 중에서도 아키텍처 설계의 동기가 가장 명확하게 서술되어 있고, Imitation Learning과 RL을 수업에서 익힌 사람이 "그다음 흐름"으로 넘어가기에 가장 좋은 디딤돌이기 때문입니다. 오픈 가중치로 공개되어 있다는 점도 커뮤니티 임팩트 측면에서 가산점이 됩니다. 오늘은 이 논문을 기초부터 따라가 보겠다.

## VLA가 뭐길래

수업에서 배운 Behavior Cloning은 (관측, 액션) 쌍으로 정책을 지도학습시키는 가장 기본적인 모방학습 방법이다. 문제는 로봇이 "컵을 집어라" 같은 언어 지시를 이해하고, 한 번도 본 적 없는 물체 배치에서도 일반화하려면 단순한 관측-액션 매핑만으로는 부족하다는 점이다. 여기서 VLA는 Vision(시각), Language(언어), Action(행동)을 하나의 모델이 end-to-end로 다루게 만든 구조를 말한다. 이미지와 텍스트 지시를 입력받아 로봇의 모터 명령을 직접 뱉어내는 정책인 셈이다. RT-2 이후 이 흐름이 본격화되었고, GR00T N1은 이 VLA 아이디어를 휴머노이드라는 훨씬 복잡한 플랫폼에 적용한 사례다.

## 휴머노이드 로봇이 유독 어려운 이유

문제는 세 가지로 요약된다. 첫째, 데이터가 절대적으로 부족하다. 웹에는 텍스트와 이미지가 넘쳐나지만, 특정 휴머노이드 하드웨어로 실제 텔레오퍼레이션 데이터를 모으려면 사람이 직접 로봇을 조작해야 하고, 이렇게 모인 데이터는 다른 로봇(다른 자유도, 다른 액추에이터)에는 그대로 쓸 수 없다. 논문은 이를 "Data Island" 문제라고 부른다. 둘째, 환경을 이해하고 계획을 세우는 느린 추론과, 관절을 부드럽게 움직이는 빠른 제어는 요구되는 주기 자체가 다르다. 전자는 사람이 봐도 초 단위로 생각해도 되지만, 후자는 수십에서 수백 Hz로 반응해야 끊기지 않는 동작이 나온다. 셋째, 유튜브나 Ego4D 같은 1인칭 비디오는 양은 많지만 액션 라벨이 없다. 사람 손이 컵을 어떻게 움직였는지는 보이는데, 그걸 로봇의 토크 명령으로 변환할 방법이 없다는 뜻이다.

## 왜 기존 방법으로는 안 됐나

Diffusion Policy는 이 중 제어 문제에 대한 좋은 답이었다. 노이즈에서 출발해 조건부 확산 과정으로 액션 시퀀스를 생성하는 방식이라, 멀티모달한 동작 분포(같은 목표를 달성하는 여러 궤적)를 잘 표현한다. 그런데 Diffusion Policy는 보통 하나의 로봇, 하나의 태스크 세트에 맞춰 처음부터 정책을 학습해야 한다. 데이터 섬 문제를 전혀 풀어주지 못한다는 뜻이다. 또한 DDPM 방식의 확산 모델은 샘플링에 여러 스텝이 필요해서, 120Hz로 모터를 구동해야 하는 상황에서는 추론 속도 자체가 병목이 된다. VLM을 그대로 액션 생성에 끌어다 쓰는 접근들은 반대로 느린 추론 쪽은 잘 풀지만, 고빈도 제어 루프와 자연스럽게 엮이지 않는다는 문제가 있었다.

## GR00T N1의 해법: 느린 뇌와 빠른 뇌

GR00T N1은 이 간극을 아예 두 개의 모듈로 쪼개서 풀었다. 심리학자 카너먼의 "빠른 사고와 느린 사고" 비유를 그대로 가져와, System 2와 System 1이라는 이름을 붙였다.

System 2는 NVIDIA의 Eagle-2 VLM(1.34B 파라미터)이다. 카메라 이미지와 언어 지시문을 받아 10Hz로 "지금 뭘 봐야 하고 뭘 해야 하는지"를 이해한다. 흥미로운 설계 포인트는 VLM의 마지막 레이어가 아니라 12번째 레이어의 임베딩을 꺼내 쓴다는 점이다. 끝까지 다 계산하지 않고 중간에서 끊어 쓰는 셈인데, 이렇게 하면 추론 지연이 줄어들면서도 성공률은 오히려 유지되거나 더 좋아졌다고 한다. 끝단 레이어로 갈수록 텍스트 생성에 특화된 표현이 되는 반면, 중간 레이어가 로봇 제어에 필요한 시각언어 정보를 더 잘 들고 있다는 해석이 가능하다.

System 1은 플로우 매칭(Flow Matching) 기반의 확산 트랜스포머(DiT)다. System 2가 내놓은 임베딩 $$\varphi_t$$와 로봇의 현재 proprioception 상태 $$q_t$$를 조건으로 받아 120Hz로 액션 청크를 생성한다. 학습 손실은 다음과 같다.

$$\mathcal{L}_{\mathrm{fm}}(\theta) = \mathbb{E}_{\tau} \left[ \lVert V_{\theta}(\varphi_t, A_t^\tau, q_t) - (\epsilon - A_t) \rVert^2 \right]$$

여기서 $$A_t$$는 길이 16짜리 액션 청크, $$\epsilon$$은 가우시안 노이즈, $$A_t^\tau = \tau A_t + (1-\tau)\epsilon$$은 노이즈와 실제 액션을 섞은 중간 상태다. 모델은 이 중간 상태에서 실제 액션 방향으로 가는 속도 벡터장을 예측하도록 학습된다. DDPM과 다른 점은 노이즈를 단계적으로 제거하는 대신 노이즈와 데이터 사이를 잇는 직선 경로의 속도장을 직접 배운다는 것이고, 그 덕분에 추론 시에는 오일러 적분을 단 $$K=4$$ 스텝만 돌려도 충분하다.

두 모듈을 잇는 방식도 단순하다. Mixture of Experts 같은 복잡한 라우팅 대신 크로스 어텐션(Cross-Attention)으로 System 2의 토큰을 System 1이 참조하게 했다. 그리고 로봇마다 자유도와 액추에이터 사양이 다른 문제는, 로봇별 전용 MLP 인코더와 디코더를 둬서 서로 다른 차원의 상태·액션을 공통 임베딩 공간으로 사상하는 방식으로 우회했다. 이 구조 덕분에 외팔 로봇부터 양팔 휴머노이드까지 같은 가중치로 다룰 수 있다는 게 저자들의 주장이다.

![GR00T N1의 데이터 피라미드 구조](/assets/images/posts/gr00t-n1-an-open-foundation-model-for-generalist-humanoid-robots/figure-1.png)

## 데이터 피라미드: 데이터가 없으면 만들어 쓴다

아키텍처만큼이나 눈여겨봐야 할 부분이 데이터 전략이다. 실제 로봇 텔레오퍼레이션 데이터는 Fourier GR-1 로봇으로 모은 88.4시간이 전부다. 이것만으로는 파운데이션 모델을 학습하기엔 턱없이 부족하다. 그래서 피라미드 구조로 데이터를 쌓았다.

맨 아래층은 Ego4D, EPIC-KITCHENS 같은 대규모 인간 1인칭 비디오다. 여기엔 액션 라벨이 없으니, VQ-VAE 기반의 잠재 액션(Latent Action, LAPA 방식)을 추출해서 이걸 가상의 로봇 embodiment로 취급하고 플로우 매칭 타깃으로 학습에 끼워 넣는다. 가운데 층은 합성 데이터인데, 물리 시뮬레이션(DexMimicGen)으로 54만 건의 궤적을 생성하고, 추가로 이미지 투 비디오 생성 모델(WAN2.1-I2V)을 미세조정해 827시간 분량의 "Neural Trajectories"를 만들었다. 이 신경망이 생성한 비디오에는 당연히 액션 라벨이 없으니, 역동역학 모델(Inverse Dynamics Model)과 잠재 액션을 다시 동원해 유사 액션(Pseudo-action)을 붙인다. 맨 위층이 바로 그 88.4시간짜리 실제 로봇 데이터로, 양은 적지만 하드웨어에 정확히 들어맞는 가장 신뢰할 수 있는 신호 역할을 한다.

![GR00T N1의 이중 시스템 아키텍처](/assets/images/posts/gr00t-n1-an-open-foundation-model-for-generalist-humanoid-robots/figure-3.png)

## 결과와 한계

이 논문 노트에는 구체적인 벤치마크 성공률 표까지는 정리되어 있지 않다. 다만 구조적으로 짚을 만한 한계는 분명하다. 하단과 중간층 데이터에 붙는 잠재 액션, 유사 액션은 모두 근사치다. 역동역학 모델이나 VQ-VAE가 잘못 추정한 액션이 섞여 들어가면, 모델이 실제로는 존재하지 않는 동작 패턴을 "배워버릴" 위험이 있다. 또한 로봇별 MLP 인코더로 임베딩 공간을 맞춘다고 해서 하드웨어 차이에서 오는 동역학적 차이(힘, 관성, 지연)까지 완전히 사라지는 건 아니다. 즉 cross-embodiment 일반화는 표현 공간 수준에서는 성립해도, 실제 물리 법칙 수준에서는 여전히 각 로봇에 대한 fine-tuning이 필요할 가능성이 높다. 12번째 레이어를 쓰는 설계나 $$K=4$$ 스텝 같은 숫자들도 특정 하드웨어, 특정 태스크 세트에서 튜닝된 값이라, 다른 세팅에서 그대로 재현될지는 별개 검증이 필요해 보인다.

## 용어 해설

- **Flow Matching**: 노이즈 분포와 데이터 분포를 잇는 경로의 속도장(vector field)을 직접 학습해서, 확산 모델보다 적은 스텝으로 샘플을 생성하는 생성 모델 학습 방법이다.
- **잠재 액션(Latent Action)**: 액션 라벨이 없는 비디오에서 연속된 두 프레임의 변화를 보고, 그 변화를 설명하는 저차원 벡터를 비지도 학습(주로 VQ-VAE)으로 뽑아내는 기법이다.
- **크로스 어텐션(Cross-Attention)**: 한 시퀀스(쿼리)가 다른 시퀀스(키, 값)의 정보를 참조해서 가중합을 만드는 어텐션 연산으로, 서로 다른 두 모달리티나 모듈을 연결할 때 자주 쓰인다.
- **고유수용성 감각(Proprioception)**: 로봇이나 생물이 외부 센서 없이도 자기 관절 각도, 힘, 자세 등을 스스로 감지하는 내부 감각 신호를 말한다.

## 🤖 AI의 생각


개인적인 감상을 덧붙이자면, 이 논문의 진짜 기여는 새로운 모델링 트릭보다 "어디서 데이터를 구해 올 것인가"라는 질문에 끝까지 답하려 한 태도에 있다고 본다. 아키텍처는 사실 기존 VLM과 Diffusion Policy를 영리하게 이어붙인 수준이지만, 데이터 피라미드는 로보틱스 특유의 데이터 기근을 정면으로 다룬다.

<details class="tc-faq">
<summary>12번째 레이어 임베딩이 다른 로봇, 다른 VLM 백본에서도 똑같이 최적일까?</summary>
<div class="tc-faq__body" markdown="1">

레이어 번호 자체보다는 "텍스트 생성 특화 전 단계의 시각언어 표현"이라는 상대적 위치가 핵심일 가능성이 높아서, 백본이 바뀌면 재탐색이 필요해 보인다.

</div>
</details>

<details class="tc-faq">
<summary>Pseudo-action으로 학습된 Neural Trajectories가 실제 robot action 분포와 얼마나 가까운지 어떻게 검증할 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

논문 노트에는 정량적 검증이 없어서, IDM 예측 오차나 실제 로봇에서의 downstream 성공률 차이로 간접 추정하는 수밖에 없어 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 task와 trajectory를 합성해서 늘리는 프로젝트 입장에서 이 데이터 피라미드 전략은 어떤 시사점이 있을까?</summary>
<div class="tc-faq__body" markdown="1">

비디오 생성 모델로 궤적을 늘리더라도 라벨 없는 합성 데이터에는 반드시 IDM이나 latent action 같은 보조 라벨링 장치를 같이 설계해야 학습에 바로 쓸 수 있다는 점이 taskcraft의 데이터 생성 파이프라인 설계에도 그대로 적용될 만하다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2503.14734v2" target="_blank" rel="noopener">GR00T N1: An Open Foundation Model for Generalist Humanoid Robots</a> · arxiv</span></div>
</div>
</div>
