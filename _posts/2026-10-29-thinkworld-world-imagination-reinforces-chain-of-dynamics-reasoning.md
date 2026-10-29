---
title: "사람 비디오로 로봇을 가르치는 ThinkWorld"
date: 2026-10-29
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "액션 라벨 없는 인간 비디오로도 로봇을 가르치는 ThinkWorld의 Chain-of-Dynamics와 World-Imagination RL을 풀어본다."
paper_title: "ThinkWorld: World Imagination Reinforces Chain-of-Dynamics Reasoning"
paper_summary: "액션 라벨 없는 인간 비디오로도 로봇을 가르치는 ThinkWorld의 Chain-of-Dynamics와 World-Imagination RL을 풀어본다."
paper_url: "https://arxiv.org/abs/2610.37666v1"
header:
  image: "/assets/images/posts/thinkworld-world-imagination-reinforces-chain-of-dynamics-reasoning/figure-1.png"
  teaser: "/assets/images/posts/thinkworld-world-imagination-reinforces-chain-of-dynamics-reasoning/figure-1.png"
---

이번 주 스코어는 0.33으로 그리 높은 편은 아닙니다. 다만 이 논문을 고른 이유는 점수보다 방향성에 있습니다. VLA 이후 흐름을 처음 접하는 분들이 가장 궁금해할 질문, "결국 로봇 학습도 데이터가 전부인데 그 데이터를 어떻게 늘릴 것인가"에 대해 ThinkWorld가 꽤 구체적인 답을 제시하고 있어서 이번 주 학습 자료로 선택했습니다. Imitation Learning 수업에서 배운 Behavior Cloning의 한계, 즉 "시연 데이터만큼만 잘한다"는 문제를 VLA 시대에 어떻게 넘어서려 하는지 보여주는 좋은 사례라고 생각합니다.

## VLA가 막힌 지점

VLA(Vision-Language-Action model)는 이름 그대로 이미지와 언어 지시를 입력받아 로봇 액션을 직접 출력하는 모델이다. 보통 대규모로 사전학습된 VLM(Vision-Language Model)을 backbone으로 쓰고, 그 위에 액션을 생성하는 head나 expert 네트워크를 붙이는 식으로 구성된다. 본질적으로는 수업에서 배운 Behavior Cloning의 연장선인데, 다른 점은 입력 공간이 훨씬 넓은 시각언어 표현이고 사전학습된 지식을 끌어쓴다는 점이다.

문제는 BC의 근본적 한계가 그대로 따라온다는 것이다. 정책이 잘하는 범위는 결국 시연 데이터가 보여준 범위를 넘지 못한다. 그런데 로봇 시연 데이터는 실제 로봇 팔과 사람이 붙어서 하나하나 조작해야 모을 수 있는, 지독하게 비싼 데이터다. 반면 유튜브나 egocentric 카메라로 찍은 인간 비디오는 훨씬 풍부하고 다양한 행동을 담고 있지만, "이 순간 로봇 관절이 어떻게 움직여야 하는가"에 해당하는 액션 라벨이 없다.

이 간극을 메우려는 기존 시도는 크게 두 갈래로 나뉜다.

| 접근 | 장점 | 한계 |
|---|---|---|
| Latent action model | 라벨 없는 비디오를 그대로 학습에 쓸 수 있음 | 프레임 간 변화에서 "전환 변수"를 추론하다 보니 카메라 흔들림이나 조명 변화 같은, 실제로는 제어 불가능한 요소까지 액션으로 착각하는 underconstrained 문제가 생긴다 |
| 손 포즈 직접 복원 | 물리적으로 명확한 손목 궤적을 얻음 | 정확한 포즈 추정과 사람손과 로봇 end-effector 사이의 대응 관계(correspondence)를 만드는 데 비용이 크다 |

즉 "데이터를 얼마나 확장할 수 있는가"와 "액션을 얼마나 명확하게 그라운딩할 수 있는가"가 서로 trade-off 관계에 있었다는 것이 이 논문이 짚는 출발점이다.

## Chain-of-Dynamics: 사람과 로봇이 같은 언어를 쓰게 만들기

ThinkWorld의 첫 번째 아이디어는 사람과 로봇의 움직임을 아예 같은 표현 공간으로 밀어 넣는 것이다. 로봇의 end-effector(팔 끝에 달린 그리퍼 등 실제로 작업을 수행하는 말단 장치) 델타 움직임과, 사람 손목이 영상 속에서 움직인 궤적을 모두 "정규화된 델타 end-effector 표현"이라는 공통 포맷으로 바꾼 뒤, FAST 스타일의 모션 토크나이저로 이산 토큰 시퀀스로 변환한다. 이 토큰 시퀀스가 바로 Chain-of-Dynamics, 줄여서 CoD다.

핵심은 이 토큰화가 로봇이든 사람이든 상관없이 "움직임 그 자체"를 기술한다는 점이다. 기존 액션 토큰화가 특정 로봇의 액션 공간에 묶여 있던 것과 달리, CoD는 embodiment와 무관하게 공유되는 모션 언어가 된다. VLM(이 논문에서는 Qwen3-VL-2B)은 언어 지시, 과거 시각 히스토리, 현재 관측을 조건으로 이 CoD를 자기회귀적으로 생성하도록 학습된다.

여기까지만 보면 "결국 또 다른 형태의 액션 토큰 예측 아닌가" 싶을 수 있다. 실제로 저자들도 CoD 생성만으로는 충분하지 않다고 본다. 토큰을 맞게 뱉는 것과, 그 토큰이 실제로 세계에 의미 있는 변화를 일으키는지는 별개 문제이기 때문이다.

## 미래를 상상하게 만들기: World Model을 그라운딩에 끌어온다

그래서 CoD 뒤에 학습 가능한 "미래 쿼리(future query)" 토큰을 하나 더 붙인다. 이 쿼리는 방금 생성된 CoD를 조건으로, 약 1초 뒤 세계가 어떻게 보일지를 예측하도록 학습된다. 다만 미래 프레임을 픽셀 단위로 통째로 생성하는 대신, DINOv3라는 self-supervised vision 모델이 뽑아낸 특징 공간에서 예측을 한다. 포토리얼리스틱한 이미지를 그릴 필요 없이, 구조적인 변화(물체가 어디로 옮겨갔는지, 서랍이 열렸는지 같은)만 비교할 수 있으면 충분하기 때문이다.

이 단계의 지도학습 손실은 CoD 생성에 대한 교차엔트로피 항과, 예측된 미래 특징과 실제 관측된 미래 특징 사이의 거리 항의 합으로 구성된다.

$$\mathcal{L}_{\text{SFT}} = \mathcal{L}_{\text{CoD}} + \lambda_{\text{future}}\mathcal{L}_{\text{future}}$$

여기서 $$\mathcal{L}_{\text{CoD}}$$는 생성된 CoD 토큰이 실제 모션 토크나이저가 만든 정답 토큰과 얼마나 일치하는지를 측정하는 표준적인 분류 손실이고, $$\mathcal{L}_{\text{future}}$$는 CoD가 예측한 미래 상태가 실제 미래 관측과 얼마나 가까운지를 측정한다. 이 두 번째 항이 있어야 CoD가 "그럴듯한 토큰"이 아니라 "실제로 미래를 바꾸는 토큰"이 되도록 그라운딩된다.

아래 그림은 이 전체 구조를 보여준다. 위쪽 Stage (a)가 지금 설명한 지도학습 단계, 즉 사람과 로봇의 움직임을 Unified Motion Space로 통일하고 CoD와 미래 특징을 함께 학습하는 과정이다.

![ThinkWorld의 두 단계 아키텍처. Stage (a)는 CoD 생성과 DINO 특징 공간에서의 미래 예측을 함께 학습하는 지도학습 단계이고, Stage (b)는 학습 시에만 쓰이는 World-Imagination RL 단계로 여러 CoD 후보의 상상된 미래와 실제 미래를 비교해 보상을 계산한다](/assets/images/posts/thinkworld-world-imagination-reinforces-chain-of-dynamics-reasoning/figure-1.png)

## 라벨 없이도 배우는 법: World-Imagination RL

여기까지는 여전히 지도학습이라 페어링된 로봇/사람 데이터가 필요하다. ThinkWorld가 진짜 하고 싶은 일은 액션 라벨이 전혀 없는 인간 비디오도 활용하는 것인데, 이를 위해 두 번째 단계인 World-Imagination RL을 둔다.

방법은 이렇다. 학습된 모델로 하나의 상황에 대해 여러 개(논문에서는 4개)의 CoD 후보를 샘플링한다. 각 후보에 대해 미래 시각 특징을 예측하고, 이를 실제로 관측된 미래 프레임의 DINO 특징과 비교해서 보상을 계산한다.

$$r_{\text{world}} = \text{sim}(\mathbf{z}_{t+\Delta}, \mathbf{z}_{t+\Delta}^{\star})$$

이때 그냥 전체 패치를 평균해서 유사도를 재면, 화면 대부분을 차지하는 정적인 배경이 보상을 지배해버리는 문제가 생긴다. 그래서 현재 프레임과 미래 프레임 사이 DINO 특징 변화량이 큰 패치, 즉 실제로 움직임이 일어난 영역에 더 큰 가중치를 줘서 보상을 계산한다. 이렇게 얻은 보상으로 GRPO(Group Relative Policy Optimization, 같은 상황에서 샘플링한 여러 후보들끼리 상대적으로 비교해서 더 나은 쪽을 강화하는 정책 최적화 기법)를 이용해 VLM을 업데이트한다. 동시에 지도학습 단계에서 배운 모션 그라운딩을 잊지 않도록 replay loss를 함께 쓴다.

$$\mathcal{L}_{\text{RL}} = \lambda_{\text{GRPO}}\mathcal{L}_{\text{GRPO}}(r_{\text{world}}) + \lambda_{\text{replay}}\mathcal{L}_{\text{replay}}$$

이 설계의 진짜 노림수는 보상 계산에 액션 라벨이 전혀 필요 없다는 점이다. 필요한 것은 전후 관측 프레임뿐이다. 즉 사람이 찍은 egocentric 비디오라도 "그 전 1초와 그 후 1초"만 있으면 CoD 추론을 다듬는 학습 신호로 쓸 수 있다. 배포 단계에서는 CoD 생성과 미래 예측 과정 자체를 들어내고 action expert만 추론에 사용하므로, 추가로 생긴 지연시간도 없다고 저자들은 보고한다.

## 결과: atomic skill만 배운 로봇이 복합 작업을 해낸다

가장 인상적인 결과는 "구성 전이(skill composition transfer)"다. 로봇 데이터에는 open, put-left, put-right, close 같은 개별 atomic skill만 있고, 서랍을 열고 왼쪽에 물건을 놓고 닫는 전체 시퀀스는 사람 시연으로만 존재한다. ThinkWorld는 이 전체 시퀀스를 한 번도 로봇 데이터로 본 적이 없는데도 단일 rollout으로 성공적으로 수행한다. CoD라는 공유 공간이 사람 쪽에서 배운 "작업을 어떤 순서로 구성하는가"에 대한 지식을 로봇 실행으로 넘겨준 것으로 해석된다.

![ThinkWorld 개요. 서로 다른 embodiment의 시연이 Unified Motion Space를 공유하고, atomic skill들로부터 로봇 데이터에 없던 복합 시퀀스를 OOD 상황에서 성공적으로 수행하는 과정](/assets/images/posts/thinkworld-world-imagination-reinforces-chain-of-dynamics-reasoning/figure-2.png)

정량적으로는 OOD(분포 밖) 상황에서 88.9%의 성공률을 보여 비교 대상 방법들보다 뚜렷하게 앞선다고 보고된다. t-SNE로 CoD 임베딩을 들여다보면 사람과 로봇 데이터가 embodiment가 아니라 모션의 방향성(여는 동작인지 닫는 동작인지 등)을 중심으로 클러스터링되는데, 이는 CoD 공간이 실제로 "누가 움직였는가"보다 "어떻게 움직였는가"를 더 중요하게 인코딩하고 있다는 증거로 제시된다.

## 남는 한계

다만 모든 것이 깔끔하게 풀린 것은 아니다. 1단계 지도학습을 위해서는 여전히 사람 손목 궤적과 로봇 end-effector 궤적을 같은 정규화된 포맷으로 맞춰주는 전처리가 필요하고, 이 정규화 과정에 embodiment 간 기하학적 차이(팔 길이, 그리퍼 형태 등)를 완전히 지울 수 있는지는 명확하지 않다. 또한 World-Imagination RL이 라벨 없는 데이터를 쓸 수 있다고는 해도, action expert 자체의 최종 학습은 여전히 지도학습 틀 안에 있어서, "라벨 의존성을 완전히 없앤다"는 것을 지지하는 직접적인 증거는 논문에 명시적으로 제시되지 않는다. DINO 특징 공간이 실제로 조작에 중요한 미세한 접촉/힘 정보까지 잘 담아내는지도 추가로 검증이 필요한 지점으로 보인다.

## 용어 해설

- **GRPO (Group Relative Policy Optimization)**: 하나의 입력 상황에 대해 여러 개의 출력을 동시에 샘플링한 뒤, 그 그룹 안에서 상대적으로 더 좋은 출력에는 더 큰 가중치를,

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2610.37666v1" target="_blank" rel="noopener">ThinkWorld: World Imagination Reinforces Chain-of-Dynamics Reasoning</a> · arxiv</span></div>
</div>
</div>
