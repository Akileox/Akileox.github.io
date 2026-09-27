---
title: "몸이 달라도 함께 배우면 강해진다"
date: 2026-09-27
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "22개 로봇, 100만 궤적을 한데 모아 학습시켰더니 서로 다른 로봇끼리도 성능이 좋아졌다는 이야기입니다"
paper_title: "Open X-Embodiment: Robotic Learning Datasets and RT-X Models"
paper_summary: "22개 로봇, 100만 궤적을 한데 모아 학습시켰더니 서로 다른 로봇끼리도 성능이 좋아졌다는 이야기입니다"
paper_url: "https://arxiv.org/abs/2310.08864v9"
header:
  image: "/assets/images/posts/open-x-embodiment-robotic-learning-datasets-and-rt-x-models/figure-1.png"
  teaser: "/assets/images/posts/open-x-embodiment-robotic-learning-datasets-and-rt-x-models/figure-1.png"
---

## 이번 주 논문을 고른 이유

안녕하세요. 이번 주 스코어링에서 만점(1.00)을 받은 논문은 Open X-Embodiment입니다. 21개 기관이 협력해 22개 서로 다른 로봇 플랫폼의 데이터를 하나로 모으고, 그 위에서 RT-1-X와 RT-2-X라는 모델을 학습시킨 프로젝트입니다. Imitation Learning이나 RL은 수업에서 익숙하게 다뤘어도, "로봇마다 팔 길이도 다르고 카메라 위치도 다른데 그걸 어떻게 한 모델로 학습시키지?"라는 질문은 강의실에서 잘 안 나오는 질문입니다. 그런데 최근 VLA(Vision-Language-Action) 흐름의 뿌리를 거슬러 올라가면 결국 이 논문이 나옵니다. 오늘은 이 논문을 기초부터 따라가면서, 왜 이런 시도가 필요했고 무엇을 어떻게 고쳤는지 짚어보겠습니다.

## 로봇마다 따로 배우는 게 왜 문제였나

먼저 배경을 정리하자. Imitation Learning의 가장 기본적인 형태인 Behavior Cloning은 사람이나 teleoperation으로 수집한 (관측, 액션) 쌍을 지도학습으로 흉내 내는 방식이다. 문제는 이 관측과 액션의 형식이 로봇마다 완전히 다르다는 점이다. 어떤 로봇은 팔이 6자유도, 어떤 로봇은 7자유도이고, 카메라도 손목에 달린 것과 고정 삼각대에 달린 것이 섞여 있다. 그래서 지금까지 로보틱스 연구는 "이 로봇, 이 환경, 이 작업"에 맞춰 매번 처음부터 데이터를 모으고 모델을 새로 학습시키는 파편화된 방식으로 흘러왔다.

이건 NLP나 Computer Vision이 걸어온 길과 정반대다. 그쪽 분야는 웹 스케일 데이터를 모아 하나의 거대한 모델을 사전학습시키고, 그 지식을 다운스트림 작업에 전이(transfer)시키는 방식으로 크게 도약했다. 로보틱스에서는 왜 이게 안 됐을까. 이 논문은 그 이유를 두 가지로 짚는다.

## 왜 그동안 안 됐나

첫째는 단순한 데이터 부족이다. 텍스트나 이미지는 인터넷에 널려 있지만, 로봇 궤적 데이터는 실제 로봇 팔을 움직여서 얻어야 하는 물리적 자원이라 한 연구실이 모을 수 있는 양에 한계가 있다. 둘째는 이 논문이 embodiment gap이라고 부르는 신체적 격차다. 로봇마다 센서의 위치와 각도, 관절 수, 제어 주기(Hz), 엔드이펙터의 자유도와 제어 방식(위치 제어인지 속도 제어인지, 절대 좌표인지 상대 좌표인지)이 다 다르다. 이걸 그냥 한 배치(batch)에 섞어 넣고 학습시키면 모델 입장에서는 서로 모순되는 신호를 받는 셈이 되어 제대로 학습이 안 된다.

그래서 이 논문의 핵심 질문은 이렇게 정리된다. 서로 다른 로봇 형태와 환경에서 수집된 이종(X-embodiment) 데이터를 한곳에 모아 고용량 Transformer 기반 정책을 공동 학습시켰을 때, 정말로 단일 로봇 전용 모델보다 뛰어난 긍정적 전이가 일어날 수 있는가.

## 어떻게 풀었나: 데이터셋을 한곳에 모으고 좌표계를 맞추다

첫 단계는 데이터를 물리적으로 모으는 작업이다. Open X-Embodiment(OXE) 데이터셋은 21개 기관이 참여해 22개 로봇 플랫폼, 60개 개별 데이터셋을 RLDS라는 표준 포맷으로 통합한 결과물이다. 100만 개 이상의 실제 로봇 궤적, 527개의 스킬, 16만 개 이상의 태스크를 담고 있다. 규모 자체가 기존 로보틱스 데이터셋과는 자릿수가 다르다.

문제는 이걸 그냥 합친다고 끝이 아니라는 점이다. 관측과 액션의 형식을 맞추는 정렬(alignment) 작업이 필요하다. 이 논문은 여기서 아주 정교한 latent 표현을 새로 설계하는 대신, 상당히 직접적인 방법을 택한다. 관측 쪽에서는 로봇마다 여러 대의 카메라 중 대표 RGB 뷰 하나를 골라 동일한 해상도로 리사이징한다. 액션 쪽에서는 엔드이펙터(EEF)의 7자유도 제어 벡터 $$(x, y, z, \text{roll}, \text{pitch}, \text{yaw}, \text{gripper})$$와 에피소드 종료 신호 1개로 통일한다. 각 데이터셋의 액션 값 범위는 사전에 정규화해서 학습하고, 실제로 로봇에 적용할 때는 그 로봇 환경에 맞춰 역정규화한다. 즉 좌표계와 단위가 다른 문제를 우아하게 푸는 대신, 학습 전후에 스케일을 맞춰주는 방식으로 우회한 것이다.

![RT-1-X와 RT-2-X 모델 구조, 이종 로봇의 이미지와 언어 입력이 공유 백본을 거쳐 이산화된 액션으로 출력되는 파이프라인](/assets/images/posts/open-x-embodiment-robotic-learning-datasets-and-rt-x-models/figure-1.png)

## RT-X 모델: 크기가 다른 두 형제

이렇게 정렬된 데이터 위에서 두 종류의 모델을 학습시킨다. 하나는 기존에 있던 RT-1을 그대로 이종 데이터에 학습시킨 RT-1-X이고, 다른 하나는 대규모 Vision-Language Model 위에 얹은 RT-2-X다.

| 항목 | RT-1-X | RT-2-X |
|---|---|---|
| 파라미터 수 | 35M | 5B 또는 55B |
| 백본 | EfficientNet + Transformer | PaLI-X 기반 VLM (ViT + UL2) |
| 언어 입력 처리 | Universal Sentence Encoder | VLM 자체 토크나이저 |
| 액션 출력 방식 | 이산 토큰 분류 | 텍스트 토큰으로 변환된 액션 |

RT-1-X는 최근 15장의 이미지 시퀀스를 EfficientNet으로 임베딩하고, 텍스트 명령어는 Universal Sentence Encoder로 인코딩한 뒤 FiLM 레이어로 두 정보를 합쳐 Transformer 디코더에 넣는다. RT-2-X는 여기서 한발 더 나아간다. 웹 스케일 이미지와 텍스트로 이미 사전학습된 거대 VLM을 백본으로 쓰고, 로봇 액션 값을 아예 텍스트 토큰 문자열로 바꿔서 VLM의 언어 생성 과정 안에 밀어 넣는다. 즉 "이 상황에서 팔을 어디로 움직여야 하는가"라는 질문을, 모델 입장에서는 "다음 텍스트 토큰이 무엇인가"라는 익숙한 문제로 바꿔버리는 것이다.

이 액션 토큰화 방식을 조금 더 구체적으로 보자. 8차원 액션 벡터

$$a = [x, y, z, \text{roll}, \text{pitch}, \text{yaw}, \text{gripper}, \text{terminate}] \in \mathbb{R}^8$$

의 각 축 $$d$$는 데이터셋별 최소, 최대 범위로 정규화된 뒤 $$B=256$$개의 이산 구간(bin) 중 하나로 양자화된다. 모델은 각 축에 대해 256개 bin에 대한 소프트맥스 확률 분포 $$\hat{y}_d \in \mathbb{R}^B$$를 출력하고, 정답 bin의 원핫 벡터 $$y_d$$와의 표준 교차 엔트로피 손실

$$\mathcal{L} = - \sum_{d=1}^{8} \sum_{k=1}^{B} y_{d,k} \log \hat{y}_{d,k}$$

을 최소화하도록 학습된다. 액션을 연속값 회귀가 아니라 이산 분류로 다루는 이유는, 로봇 팔이 같은 목표를 여러 경로로 도달할 수 있는 다봉(multimodal) 분포 문제를 회귀 손실로는 잘 표현하지 못하기 때문이다. RT-2-X는 여기서 한 걸음 더 나아가, 이 256개의 bin 인덱스를 LLM의 자연어 어휘 토큰과 그대로 일치시킨다. 그러면 로봇 제어 학습이 언어 모델의 토크나이저 안에서 완전히 동일한 방식으로 처리되고, 결과적으로 웹에서 배운 시각언어 지식이 로봇 액션 예측으로 자연스럽게 흘러 들어갈 통로가 생긴다.

## 결과: 작은 데이터셋일수록 더 크게 웃었다

이렇게 학습시킨 모델이 정말 도움이 되는지 확인한 실험이 흥미롭다. Kitchen Manipulation, Cable Routing, NYU Door Opening, Autolab UR5, Robot Play라는 다섯 개의 비교적 소규모 데이터셋 환경에서, 각 환경 전용으로 학습한 기존 모델과 RT-1을 그 환경 데이터로만 학습시킨 버전, 그리고 전체 이종 데이터로 공동 학습시킨 RT-1-X의 성공률을 비교했다.

![소규모 데이터셋 다섯 개 환경에서 기존 전용 모델, 단일 RT-1, 이종 학습 RT-1-X의 성공률을 비교한 막대그래프](/assets/images/posts/open-x-embodiment-robotic-learning-datasets-and-rt-x-models/figure-2.png)

결과는 다섯 개 중 네 개 도메인에서 RT-1-X가 기존 전용 모델을 앞섰고, 전체 평균 성공률은 기존 방식 대비 50% 향상을 기록했다. 주목할 점은 이 향상 폭이 데이터가 적은 환경일수록 더 크게 나타났다는 것이다. 즉 자기 로봇의 데이터만으로는 부족했던 부분을, 전혀 다른 형태의 로봇들이 남긴 궤적이 채워준 셈이다. 이게 바로 이 논문이 말하는 긍정적 전이(positive transfer)의 실증적 증거다.

## 그래서 어디까지 통했나, 그리고 한계

여기서 솔직히 짚어야 할 지점이 있다. 이 논문이 다루는 22개 로봇은 결국 다 팔과 그리퍼로 구성된 매니퓰레이터라는 공통점을 갖고 있다. 관측을 대표 RGB 뷰 하나로 뭉개고, 액션을 7자유도 EEF 좌표로 억지로 통일하는 coarse alignment가 통한 것도, 애초에 이들이 형태적으로 크게 다르지 않았기 때문이라고 볼 수 있다. 만약 로봇이 사족보행이거나 다관절 구조가 근본적으로 다르다면, 이 정도의 정렬 방식으로 충분할지는 불확실하다. 또한 이 논문은 여전히 순수한 Behavior Cloning 기반이라, 학습 데이터에 없는 완전히 새로운 물리적 상호작용에는 취약할 수밖에 없다는 지도학습 특유의 한계도 그대로 안고 간다.

## 용어 해설

- **Embodiment Gap**: 로봇마다 관절 수, 센서 위치, 제어 주기, 자유도 등 물리적 구조와 제어 방식이 달라서 발생하는 격차. 한 로봇에서 배운 정책을 다른 로봇에 그대로 적용하기 어렵게 만드는 근본 원인이다.
- **End-effector (EEF)**: 로봇 팔의 맨 끝에 달려 실제로 물체와 접촉하는 부분. 그리퍼나 흡착 패드 등이 해당하며, 로봇 제어에서는 관절 각도 대신 EEF의 위치와 방향을 기준으로 액션을 표현하는 경우가 많다.
- **Vision-Language-Action (VLA) 모델**: 이미지와 텍스트를 함께 입력받아 로봇 제어 액션을 직접 출력하는 모델. 대규모 Vision-Language Model의 사전학습 지식을 로봇 제어에 재활용하려는 접근이다.
- **긍정적 전이 (Positive Transfer)**: 서로 다른 작업이나 도메인의 데이터를 함께 학습했을 때, 개별적으로 학습했을 때보다 성능이 더 좋아지는 현상. 반대로 성능이 나빠지면 부정적 전이라고 부른다.

## 🤖 AI의 생각


사실 정리는 여기까지고, 이건 순전히 개인적인 감상인데 이 논문이 보여준 건 정교한 이론적 해법이 아니라 꽤 무식하고 실용적인 해법이 통했다는 점이 인상적이다.

<details class="tc-faq">
<summary>관측을 RGB 뷰 하나로, 액션을 7자유도 EEF로 뭉개는 이 coarse alignment는 로봇 형태가 훨씬 더 이질적일 때도 버틸 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

이 논문의 22개 로봇이 모두 팔형 매니퓰레이터라는 공통 구조를 갖고 있어서 통했을 가능성이 높고, 사족보행이나 조류형처럼 근본적으로 다른 형태에서는 이 정도 정렬로는 부족할 것 같다는 게 개인적 추측이다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 전제하는 embodiment-agnostic latent 표현이 정말 필요한 걸까, 아니면 RT-X처럼 좌표계만 맞추고 데이터와 모델 크기로 밀어붙이는 게 더 효율적일까</summary>
<div class="tc-faq__body" markdown="1">

지금 taskcraft가 다루려는 로봇들이 팔+그리퍼 범주를 벗어난다면 RT-X식 coarse alignment의 한계가 먼저 드러날 가능성이 크고, 그 지점이 정교한 latent 표현이 정말 필요해지는 경계선이라고 본다.

</div>
</details>

<details class="tc-faq">
<summary>RT-2-X처럼 액션을 아예 자연어 토큰 어휘에 편입시키는 방식이 taskcraft의 파이프라인에도 적용 가능할까</summary>
<div class="tc-faq__body" markdown="1">

taskcraft가 다루는 로봇들의 액션 공간이 EEF 좌표계로 환원되지 않는 경우가 많다면, 액션을 언어 토큰으로 억지로

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2310.08864v9" target="_blank" rel="noopener">Open X-Embodiment: Robotic Learning Datasets and RT-X Models</a> · arxiv</span></div>
</div>
</div>
