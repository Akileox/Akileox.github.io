---
title: "운전 지시도 의심하는 자율주행 AI"
date: 2026-10-15
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "위험한 내비게이션 지시를 가상으로 먼저 실행해보고 판단하는 자율주행 VLA, SafeDriveVLA를 기초부터 풀어본다."
paper_title: "SafeDriveVLA: Navigation-Conditioned World Model Dreaming for Conflict-Aware End-to-End Autonomous Driving"
paper_summary: "위험한 내비게이션 지시를 가상으로 먼저 실행해보고 판단하는 자율주행 VLA, SafeDriveVLA를 기초부터 풀어본다."
paper_url: "https://arxiv.org/abs/2610.20609v1"
header:
  image: "/assets/images/posts/safedrivevla-navigation-conditioned-world-model-dreaming-for-conflict-aware-end-/figure-1.png"
  teaser: "/assets/images/posts/safedrivevla-navigation-conditioned-world-model-dreaming-for-conflict-aware-end-/figure-1.png"
---

이번 주 논문은 arxiv 스코어 0.33으로 상위권은 아니지만, 자율주행이라는 구체적인 응용 영역에서 VLA(Vision-Language-Action) 모델의 가장 근본적인 약점, 즉 "지시를 곧이곧대로 따르다 사고 낸다"는 문제를 정면으로 다루고 있어 학습 자료로 골랐습니다. IL과 RL의 고전 알고리즘은 배웠지만 VLA 계열 모델이 실제로 어떤 구조로 동작하고 어디서 깨지는지는 접할 기회가 많지 않았을 텐데, 이 논문은 그 간극을 메우기 좋은 사례입니다.

## VLA가 운전을 배운다는 것

먼저 용어부터 정리하자. VLA(Vision-Language-Action) 모델은 카메라 이미지와 텍스트 지시를 함께 입력받아 로봇 팔의 관절 각도나 자동차의 조향, 가속 같은 저수준 행동을 직접 출력하는 신경망이다. 기존의 로보틱스나 자율주행은 인지(Perception), 예측(Prediction), 계획(Planning), 제어(Control)를 각각 별도 모듈로 나누어 만들었는데, VLA는 이 전체 파이프라인을 하나의 거대한 신경망이 end-to-end로 떠맡는다. 입력에서 출력까지 중간 단계를 사람이 설계하지 않고 신경망이 통째로 학습하기 때문에 종단간(E2E) 학습이라고 부른다.

자율주행에서 이 구조가 왜 매력적이냐면, 사람이 일일이 "신호등이 빨간불이면 정지한다" 같은 규칙을 코딩하지 않아도 전문가 주행 데이터를 모방 학습(Behavior Cloning)시키는 것만으로 운전을 배우기 때문이다. 수업에서 배운 BC의 기본 아이디어 그대로다. 전문가가 운전하는 영상과 그때의 조향·속도를 쌍으로 모아서, "이 장면에서는 이렇게 운전한다"를 지도학습으로 외우게 한다.

문제는 자율주행차가 단순히 "알아서 잘 가는" 것만으로는 부족하고, 탑승자나 관제 시스템이 "다음 사거리에서 좌회전해", "속도 좀 올려" 같은 내비게이션 명령을 실시간으로 내릴 수 있어야 한다는 점이다. 기존 모듈형 스택에서는 행동 플래너(Behavior Planner)라는 규칙 기반 모듈이 이 명령과 주변 상황(보행자, 신호, 장애물)을 사람이 짠 로직으로 조율했다. 그런데 VLA는 이 조율 과정까지 신경망 내부에 통째로 녹여 넣어야 한다. 그리고 이게 생각보다 잘 안 된다.

## 명령을 곧이곧대로 따르면 생기는 일

![기존 모듈형 자율주행 스택과 VLA의 명령 처리 구조 비교](/assets/images/posts/safedrivevla-navigation-conditioned-world-model-dreaming-for-conflict-aware-end-/figure-1.png)

논문이 지적하는 첫 번째 문제는 단순하다. 전방에 장애물이 있거나 신호가 빨간불인데도 사람이 "직진해", "속도를 올려" 같은 지시를 내리면, 기존 VLA 모델은 이를 그대로 실행해서 충돌을 일으킨다. BC로 학습한 모델은 "이 지시가 주어지면 이런 행동을 했다"는 데이터의 상관관계만 외웠을 뿐, "이 지시를 지금 실행하면 위험하다"는 판단 로직을 명시적으로 배운 적이 없기 때문이다.

두 번째 문제는 더 당혹스럽다. 기존 VLA 모델(SimLingo)을 오픈루프로 테스트하면, 즉 녹화된 주행 영상에 대해 한 스텝씩 예측값만 비교하면 차선 변경 성공률이 83%로 꽤 그럴듯해 보인다. 그런데 같은 모델을 폐루프로, 즉 모델이 낸 출력을 실제로 시뮬레이터에 반영하고 다음 프레임을 다시 모델에 먹이는 방식으로 돌리면 차선 변경 성공률이 0%로 추락한다. 오픈루프 평가는 "정답과 비슷한 출력을 냈는가"만 보기 때문에 누적 오차나 분포 변화(distribution shift) 문제를 전혀 잡아내지 못한다는, IL 수업에서 배웠을 DAgger의 교훈이 여기서도 고스란히 재현된 셈이다.

세 번째는 평가 체계 자체의 공백이다. 모든 지시를 의심하고 일단 멈추는 보수적인 모델은 "충돌 안 함" 지표에서 유리하고, 반대로 모든 지시를 무조건 따르는 모델은 "지시 추종성" 지표에서 유리하다. 두 지표를 따로 재면 극단적인 두 전략이 모두 좋은 점수를 받는 역설이 생긴다. 그래서 저자들은 둘을 동시에 압박하는 벤치마크부터 새로 만들었다. 교통을 다 치우고 순수 지시 추종만 재는 CARLA-F, 위험한 순간에 위험한 지시를 주입하는 B2D-C와 NavSim-C, 아예 탈옥하듯 규칙 위반을 강제하는 B2D-Adv까지 네 종류다.

## 고친 방법: 실행하기 전에 먼저 꿈꿔본다

여기서 이 논문의 핵심 아이디어가 나온다. 이름하여 "내비게이션 조건부 월드 모델 Dreaming"이다. RL 수업에서 Dreamer 계열 알고리즘을 들어봤다면 감이 올 텐데, World Model이란 에이전트가 실제 환경과 상호작용하지 않고도 "이 행동을 하면 다음 상태가 어떻게 될까"를 예측하는 내부 시뮬레이터다. Dreamer는 이 월드 모델 안에서 가상의 rollout을 돌려 정책을 학습했는데, SafeDriveVLA는 이 아이디어를 정책 학습이 아니라 판단 보조 장치로 가져다 쓴다.

구체적으로는 이렇다. 먼저 V-JEPA 2라는 비디오 인코더로 연속된 두 프레임을 latent 벡터로 압축한다. 이 latent 공간 위에서 동작하는 별도의 트랜스포머 디코더가 "현재 상태에서 어떤 행동을 하면 2.5초 뒤 latent 상태가 어떻게 바뀌는가"를 예측하도록 사전 학습된다. 여기서 중요한 포인트는, 이 가상 시뮬레이션에 넣는 행동이 전문가가 실제로 한 행동이 아니라 현재 내비게이션 명령(직진, 좌회전, 차선 변경 등)에 대응하는 대표적인 행동 패턴, 즉 "액션 앵커(Action Anchor)"라는 점이다. 액션 앵커는 2만 개의 전문가 궤적에서 각 명령 유형별로 상대 좌표계 위치 변화량과 헤딩 변화량을 평균 내서 미리 고정해 둔 값이다.

즉 모델은 "지금 받은 이 명령을 실제로, 엄격하게 실행한다면 2.5초 뒤 세상이 어떻게 보일까"를 사전 학습된 월드 모델로 미리 상상(dreaming)해보고, 그 결과로 나온 미래 latent 표현을 "월드 토큰"으로 변환해서 VLM(Vision-Language Model) 본체에 추가 입력으로 넣는다. VLM은 현재 카메라 영상, 텍스트 지시, 자차 상태뿐 아니라 이 월드 토큰까지 보고 판단을 내리게 된다.

![모드 토큰을 강제로 주입했을 때와 모델이 스스로 예측한 모드에서의 주행 궤적 분포 비교](/assets/images/posts/safedrivevla-navigation-conditioned-world-model-dreaming-for-conflict-aware-end-/figure-2.png)

이 판단을 더 명시적으로 만들기 위해 저자들은 행동 토큰을 내뱉기 전에 먼저 주행 모드 토큰을 하나 예측하게 만든다. 즉시 실행을 뜻하는 `$$<STRICT>$$`, 감속하며 양보하는 `$$<CAUTIOUS>$$`, 명령을 거부하고 정지하는 `$$<FALLBACK>$$` 세 가지다. 전문가 데이터를 이 세 모드로 재라벨링하고 Cross-Entropy 손실로 직접 지도학습시킨다. 전체 손실 함수는

$$\mathcal{L} = \mathcal{L}_{\text{action}} + \mathcal{L}_{\text{mode}} + \mathcal{L}_{\text{path}} + 0.5\mathcal{L}_{\text{text}}$$

로, 행동 토큰 손실과 모드 토큰 손실, 보조 경로 예측 손실, 텍스트 손실을 더한 형태다. 핵심은 $$\mathcal{L}_{\text{mode}}$$가 별도의 손실 항으로 분리되어 있다는 점이다. 이 덕분에 모델은 "충돌 여부를 판단하는 일"과 "저수준 조향·속도를 생성하는 일"을 서로 다른 토큰에서 따로 책임지게 된다. 위 그림에서 전방에 정차 차량이 가로막은 상황을 보면, 모델 스스로는 `$$<FALLBACK>$$` 모드를 98.8% 확률로 예측해 정지하지만, 실험적으로 `$$<STRICT>$$`나 `$$<CAUTIOUS>$$` 모드를 강제로 주입하면 똑같은 입력에서도 위험하게 전진하는 궤적이 나온다. 모드 토큰 하나가 전체 주행 정책의 분포를 실제로 바꾸는 손잡이 역할을 하고 있다는 뜻이다.

## 결과와 솔직한 한계

새로 만든 벤치마크들에서 SafeDriveVLA는 지시 추종성과 충돌 회피를 동시에 개선했다고 보고한다. 특히 폐루프 환경에서 기존 모델이 0%까지 떨어졌던 지시 추종 실패를 월드 토큰과 모드 토큰 분리로 상당 부분 복구했고, 위험한 지시가 주입된 상황에서 `$$<FALLBACK>$$`을 정확히 골라내는 비율도 높게 나왔다. 다만 이 구조에는 분명한 비용이 따른다. 월드 모델을 별도로 사전 학습해야 하고, 추론 시점마다 액션 앵커 선택과 가상 rollout, latent 투영까지 거쳐야 하므로 순수 BC 기반 VLA보다 파이프라인이 훨씬 무거워진다. 또한 액션 앵커 자체가 전문가 데이터의 평균값이라는 점에서, 이례적인 도로 구조나 학습 데이터에 없던 명령 유형에 대해서는 월드 모델의 dreaming 결과가 신뢰하기 어려운 값을 낼 가능성도 남아 있다. 논문도 이를 완전히 해소했다기보다는, "판단 근거를 명시적인 토큰과 가상 시뮬레이션으로 분리했을 때 얼마나 나아지는가"를 보여주는 데 집중한 느낌이다.

## 용어 해설

- **Closed-loop 평가 (폐루프 평가)**: 모델의 출력을 실제 환경(혹은 시뮬레이터)에 반영해 다음 상태를 바꾸고, 그 바뀐 상태를 다시 모델에 입력하는 식으로 평가하는 방식. 녹화된 데이터에 대해 한 스텝씩만 예측을 비교하는 open-loop 평가와 달리 누적 오차와 분포 변화 문제를 드러낸다.
- **Latent 공간**: 원본 이미지나 비디오처럼 차원이 크고 중복 정보가 많은 데이터를 신경망이 압축해 표현하는 저차원 벡터 공간. 이 공간 안에서의 거리나 변화가 실제 의미 있는 특징 변화에 대응하도록 학습된다.
- **자기회귀(Autoregressive) 생성**: 이전에 생성한 출력(토큰)을 다음 출력을 만들 때의 입력으로 다시 사용하는 방식. 문장을 한 단어씩 이어 쓰는 언어 모델의 생성 방식이 대표적이며, 이 논문에서는 행동 토큰과 모드 토큰을 순차적으로 이어서 생성하는 데 쓰인다.
- **Action Tokenization**: 연속적인 조향각이나 속도 값을 유한한 개수의 이산적인 코드(코드북 인덱스)로 양자화해서, 언어 모델처럼 토큰 시퀀스로 다루는 방식.

## 🤖 AI의 생각


이 글은 논문 내용을 정리한 것이고, 여기부터는 개인적인 의견이다. 행동을 직접 생성하기 전에 latent 공간에서 미래를 미리 시뮬레이션해보고 그 결과를 판단 근거로 쓴다는 아이디어는 단순하지만 효과적인 절충안으로 보인다. BC의 "데이터에 없으면 모른다"는 근본적 한계를 완전히 없애지는 못해도, 적어도 위험 판단이라는 좁은 역할만큼은 별도 토큰과 별도 신호로 떼어내서 지도학습을 확실히 걸었다는 점이 설계적으로 깔끔하다.

<details class="tc-faq">
<summary>액션 앵커가 전문가 궤적의 평균값이라면, 데이터에 드물게 등장하는 애매한 명령(예: 비보호 좌회전)에 대해서도 dreaming 결과가 믿을 만할까</summary>
<div class="tc-faq__body" markdown="1">

논문에서 다루지 않은 부분인데, 앵커 자체의 분산이 크거나 다봉분포(multimodal)인 명령에 대해서는 평균 하나로 대표시키는 방식이 오히려 잘못된 미래를 예측하게 만들 위험이 있어 보인다

</div>
</details>

<details class="tc-faq">
<summary>모드 토큰을 세 가지(`\(<STRICT>\)`, `\(<CAUTIOUS>\)`, `\(<FALLBACK>\)`)로 고정한 설계가 실제 도로의 연속적인 위험도를 충분히 표현할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

이산적인 세 단계로 나눈 것이 학습 안정성에는 유리하지만, 경계에 걸친 애매한 상황에서는 모드 간 전환이 突변적으로 일어나 승차감이나 안전 마진에 영향을 줄 수 있을 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 가정하는 latent world model 기반 행동 평가 구조와 이 논문의 dreaming 메커니즘이 거의 닮아 있는데, 이걸 로봇 조작 같은 다른 도메인에 그대로 옮기면 뭐가 가장 먼저 깨질까</summary>
<div class="tc-faq__body" markdown="1">

자율주행은 액션 공간이 조향·속도 정도로 단순해 앵커를 평균 내기 쉽지만, 조작 태스크는 행동 공간이 훨씬 고차원이라 "대표 앵커"를 정의하는 단계 자체가 먼저 막힐 가능성이 크다고 본다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2610.20609v1" target="_blank" rel="noopener">SafeDriveVLA: Navigation-Conditioned World Model Dreaming for Conflict-Aware End-to-End Autonomous Driving</a> · arxiv</span></div>
</div>
</div>
