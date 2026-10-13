---
title: "정밀한 지각이 로봇 행동의 토대가 될까"
date: 2026-10-13
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "그라운딩 성능을 끌어올린 파운데이션 모델이 자율주행과 로봇 조작 전이에서도 앞선다는 걸 보여준 논문입니다"
paper_title: "GroundingPI: A Grounding Foundation Model towards Physical Intelligence with Visual Primitives"
paper_summary: "그라운딩 성능을 끌어올린 파운데이션 모델이 자율주행과 로봇 조작 전이에서도 앞선다는 걸 보여준 논문입니다"
paper_url: "https://huggingface.co/papers/2609.39601"
header:
  image: "/assets/images/posts/groundingpi-a-grounding-foundation-model-towards-physical-intelligence-with-visu/figure-1.png"
  teaser: "/assets/images/posts/groundingpi-a-grounding-foundation-model-towards-physical-intelligence-with-visu/figure-1.png"
---

## 이번 주 논문을 고른 이유

이번 주는 HuggingFace Daily Papers에서 score 1.00으로 가장 높은 관심을 받은 논문을 골랐습니다. GroundingPI는 "지각을 더 정밀하게 만들면 로봇 조작과 자율주행 같은 물리적 과업에 실제로 도움이 되는가"라는, taskcraft 작업을 하면서 계속 마주치는 질문과 정확히 맞닿아 있어서 이번 주 첫 번째로 다뤄볼 만하다고 판단했습니다.

## 문제: 지각이 행동 학습의 병목이 된다

VLA(Vision-Language-Action)든 WAM(World-Action Model)이든, 로봇이나 자율주행 에이전트에 붙는 백본은 대부분 범용 비전-언어 모델이나 비디오 생성 모델에서 물려받는다. 문제는 이 백본들이 "물체를 분류한다"에는 강하지만 "이 물체가 정확히 어디에 있고, 타겟이 밀집된 장면 속 어느 것인지"를 픽셀 단위로 짚어내는 능력, 즉 그라운딩에는 특화되어 있지 않다는 점이다. 그런데 로봇이 뭔가를 집으려면 결국 위치를 정확히 알아야 한다. 저자들은 이 간극이 행동 학습의 병목이 되고, 희소한 로봇 데이터가 지각까지 같이 배워야 하는 이중 부담을 지게 된다고 지적한다.

## 왜 범용 백본으로는 안 됐나

범용 VLM은 "저게 컵이다"는 맞히지만 "컵의 손잡이가 정확히 몇 픽셀 지점인가"는 애매하게 답하는 경우가 많다. 특히 작은 물체나 겹쳐 있는 물체일수록 이 오차가 커진다. 논문은 Section C.3에서 이걸 수치로 보여주는데, 폭 $$w$$인 타겟 박스가 수평으로 $$\delta$$만큼 어긋났을 때 IoU는

$$\text{IoU}(\delta)=\frac{w-\vert{}\delta\vert{}}{w+\vert{}\delta\vert{}}$$

로 떨어진다. 즉 IoU 기준 $$\tau$$를 만족하려면 $$\vert{}\delta\vert{} \le w\frac{1-\tau}{1+\tau}$$여야 하는데, 타겟이 작을수록(작은 $$w$$) 허용 오차도 선형으로 줄어든다. 범용 VLM이 대충 맞는 위치를 말해도 벤치마크 점수는 그럭저럭 나올 수 있지만, 작은 물체를 집어야 하는 로봇 조작에서는 이 "대충"이 치명적이라는 뜻이다. 비디오 생성 기반 WAM도 사정이 비슷하다. 미래 프레임을 그럴듯하게 생성하는 것과, 지금 이 물체의 정확한 좌표를 뱉는 것은 다른 능력이다.

## 고친 방법: 좌표를 토큰으로, 3단계로 학습

GroundingPI는 MoonViT-V2 비전 인코더에 Qwen3-4B 디코더를 붙이고, 좌표를 1000개로 양자화된 토큰(`<0>`부터 `<999>`까지)으로 취급한다. 정규화된 좌표 $$v$$는

$$q(v) = \left\lfloor \frac{999v + 500}{1000} \right\rfloor$$

로 토큰 인덱스에 매핑되고, 박스는 4개, 점은 2개의 좌표 토큰으로 표현된다. 이 좌표 토큰은 의미 라벨, 프로토콜 마커와 같은 어휘 공간을 공유하기 때문에 "이게 뭔지"와 "어디 있는지"를 같은 자기회귀 디코딩 과정 안에서 처리한다.

학습은 세 단계로 쌓는다. 먼저 프로젝터 정렬과 멀티모달 공동 사전학습으로 기본 비전-언어 능력을 확보하고, 두 번째 단계에서 탐지, GUI, 참조 표현 그라운딩, OCR, 레이아웃, 밀집 포인팅/카운팅 등 8개 공간 태스크 그룹에 대해 teacher-forcing 지도학습을 한다. 마지막으로 GRPO 강화학습을 붙이는데, 그라운딩에는 세트 완전성 보상과 엄격한 로컬라이제이션 보상을 7대3 비율로 결합하고, OCR에는 텍스트와 기하 위치가 일치하는지를 보는 별도 보상을 쓴다. 단순히 "맞았다/틀렸다"가 아니라 "빠뜨린 물체는 없는가", "박스가 얼마나 딱 맞는가"를 나눠서 보상하는 설계가 특징이다.

데이터 쪽에서는 여러 교사 모델의 예측을 필드 단위(아이템, 박스, 마스크, 속성)로 합치고 필수 필드 통과 여부를 검증한 뒤 받아들이는 반복적 주석 파이프라인을 썼다. 한 모델의 예측만 믿기보다 여러 모델의 합의를 거르는 방식으로, 이 자체도 라벨 품질을 올리는 중요한 축이다.

![GroundingPI 개요: 지각 벤치마크 레이더 차트와 Basic Perception first, Action Learning next 파이프라인](/assets/images/posts/groundingpi-a-grounding-foundation-model-towards-physical-intelligence-with-visu/figure-1.png)

## 결과: 지각 벤치마크부터 로봇 조작 전이까지

지각 쪽에서는 44개 베이스라인, 34개 벤치마크를 돌려 거의 모든 축에서 Rex-Omni, LocateAnything 같은 기존 그라운딩 특화 모델을 능가했다고 보고한다. 그런데 이 논문이 진짜 하고 싶은 얘기는 그 다음이다. Action DiT 구조, 행동 데이터, 학습 예산을 모두 고정하고 백본만 바꿔치기하는 통제된 실험을 자율주행(nuScenes)과 로봇 조작(RoboTwin2.0, RoboCasa-GR1)에서 진행했는데, GroundingPI를 백본으로 쓴 경우가 6개 조작 설정 중 5개, 특히 4개의 OOD(분포 밖) 설정 전부에서 1위를 차지했다. RoboTwin2.0의 Clean2Random OOD 설정에서는 2위 모델 대비 상대적으로 24.8% 높은 성공률을 보였다. 단순히 벤치마크 점수가 높은 모델이 아니라, 분포가 바뀐 상황에서도 흔들리지 않는 지각이 실제로 행동 학습 전이에 도움이 된다는 걸 숫자로 보여준 셈이다.

흥미로운 지점은 선형 어텐션 기반 백본(RynnBrain1.1)이 완전 어텐션 기반 백본(RynnBrain)보다 조작 태스크에서 뒤처진다는 관찰이다. 저자들은 게이트 값이 누적되며 지수적으로 감쇠하는 메커니즘을 가설로 제시하는데, 과거 정보를 얼마나 오래 "기억"하느냐가 조작 성공률과 관련이 있을 수 있다는 암시다.

![9개 패널로 비교한 자율주행 및 로봇 조작 전이 성능, 백본별 비교 막대그래프](/assets/images/posts/groundingpi-a-grounding-foundation-model-towards-physical-intelligence-with-visu/figure-2.png)

## 한계와 생각해볼 지점

통제된 비교라고는 하지만 Action DiT 구조와 행동 데이터가 고정되어 있다는 것은, GroundingPI가 그 특정 디코더 구조와 궁합이 잘 맞아서 좋은 결과가 나왔을 가능성을 완전히 배제하기 어렵다는 뜻이기도 하다. 또한 4B 파라미터 규모의 백본을 이렇게 세분화된 3단계로 학습시키는 비용은 결코 작지 않은데, 이 비용 대비 실제 로봇 성능 향상폭이 다른 경량화된 그라운딩 보완 방식(예: 별도 디텍터를 VLA에 붙이는 방식)보다 더 효율적인지는 논문에서 직접 비교하지 않는다.

## 용어 해설

- **GRPO(Group Relative Policy Optimization)**: 같은 프롬프트에 대해 여러 응답을 샘플링하고, 그룹 내 상대적인 보상 순위를 기준으로 정책을 업데이트하는 강화학습 알고리즘이다. 별도의 가치 함수 없이 그룹 내 비교만으로 어드밴티지를 추정한다.
- **IoU(Intersection over Union)**: 두 영역(보통 바운딩 박스)이 얼마나 겹치는지를 겹친 면적을 합친 면적으로 나눠 측정하는 지표로, 객체 탐지 성능을 평가할 때 널리 쓰인다.
- **Teacher-forcing**: 자기회귀 모델을 학습시킬 때 이전 단계의 실제 정답(모델이 생성한 값이 아니라)을 다음 단계 입력으로 넣어주는 학습 방식이다.
- **Cross-attention**: 서로 다른 두 시퀀스(예: 백본의 특징과 디코더의 쿼리) 사이에서 하나가 다른 하나의 정보를 참조하도록 계산하는 어텐션 방식이다.

## 🤖 AI의 생각


사실 요약과는 별개로 제 소감을 덧붙이면, 이 논문이 보여준 "지각 벤치마크 1등이 곧 행동 전이 1등은 아니지만 꽤 상관관계가 있더라"는 결과는 당연해 보이면서도 실제로 통제 실험으로 증명한 사례가 드물었다는 점에서 의미가 크다고 생각합니다.

<details class="tc-faq">
<summary>Action DiT와 행동 데이터를 고정한 비교가 다른 디코더 구조(예: 토큰 기반 자기회귀 행동 생성)에서도 같은 순위를 유지할까?</summary>
<div class="tc-faq__body" markdown="1">

논문은 flow-matching 기반 디코더 하나만 검증했기 때문에, 디코더 종류를 바꿔도 GroundingPI 백본이 1위를 지키는지는 별도 실험이 필요해 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 사람 시연에서 뽑은 embodiment-agnostic task representation과 GroundingPI의 좌표 토큰 표현을 같이 쓰면 어떤 일이 생길까?</summary>
<div class="tc-faq__body" markdown="1">

taskcraft의 latent task representation이 "무엇을 해야 하는가"를 담당하고 GroundingPI의 좌표 토큰이 "어디에 있는가"를 담당하는 식으로 역할을 나눠 붙이면 서로 보완될 여지가 있어 보인다.

</div>
</details>

<details class="tc-faq">
<summary>선형 어텐션 백본이 조작 태스크에서 뒤처지는 현상이 게이트 감쇠 때문이라는 가설은 다른 태스크(자율주행 등)에서도 같은 패턴으로 나타날까?</summary>
<div class="tc-faq__body" markdown="1">

논문은 조작 태스크에서만 이 현상을 보고했는데, 장기 기억이 덜 중요한 자율주행 같은 짧은 지평선 태스크에서는 선형 어텐션의 불리함이 덜할 수도 있다는 생각이 든다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.39601" target="_blank" rel="noopener">GroundingPI: A Grounding Foundation Model towards Physical Intelligence with Visual Primitives</a> · hf-daily</span></div>
</div>
</div>
