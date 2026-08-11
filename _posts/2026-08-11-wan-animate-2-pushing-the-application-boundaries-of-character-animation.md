---
title: "캐릭터 애니메이션이 스켈레톤을 버린 이유"
date: 2026-08-11
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "중간 표현 없이 드라이빙 비디오를 직접 처리해 정체성 손실 없이 카메라 시점까지 바꾸는 캐릭터 애니메이션 프레임워크"
paper_title: "Wan-Animate-2: Pushing the Application Boundaries of Character Animation"
paper_summary: "중간 표현 없이 드라이빙 비디오를 직접 처리해 정체성 손실 없이 카메라 시점까지 바꾸는 캐릭터 애니메이션 프레임워크"
paper_url: "https://arxiv.org/abs/2608.06009v2"
header:
  image: "/assets/images/posts/wan-animate-2-pushing-the-application-boundaries-of-character-animation/figure-1.png"
  teaser: "/assets/images/posts/wan-animate-2-pushing-the-application-boundaries-of-character-animation/figure-1.png"
---

이번 주는 Wan-Animate-2를 골랐습니다. arXiv에서 스코어 1.00을 받은 논문인데, 단순히 화질이 좋아졌다는 식의 개선이 아니라 "모션을 어떻게 표현해서 다른 몸으로 옮길 것인가"라는 구조적인 질문에 정면으로 답하려는 시도라서 눈여겨볼 만했다. 로보틱스 쪽 world model 작업을 하는 입장에서도 embodiment가 다른 대상 간에 행동을 전이하는 문제는 늘 고민거리라, 애니메이션 분야에서 같은 딜레마를 어떻게 풀었는지 살펴볼 가치가 있었다.

## 문제 제기: 모션을 어떻게 표현할 것인가

캐릭터 애니메이션에서 사람이 찍힌 드라이빙 비디오를 보고 다른 캐릭터가 똑같이 움직이게 만드는 작업을 생각해보자. 가장 먼저 떠올릴 방법은 스켈레톤이나 3D SMPL 같은 명시적 모션 표현을 뽑아서 옮기는 것이다. 하지만 이 방식은 두 가지 문제를 안고 있다. 추출 과정 자체에서 오류가 쉽게 생기고, 원본 캐릭터와 타깃 캐릭터의 체형이 다르면 정체성이 흔들리는 identity drift 현상이 나타난다. 팔다리 비율이 다른 캐릭터에 사람의 관절 각도를 그대로 옮기면 부자연스러운 움직임이 나오는 것과 같은 맥락이다.

그럼 반대로 학습된 latent 특징으로 모션을 암묵적으로 넘기면 어떨까. 이 경우엔 압축 병목 때문에 표정의 미세한 변화나 손가락 움직임처럼 고주파 성분이 뭉개진다. 인컨텍스트 러닝(ICL) 방식으로 중간 표현 자체를 아예 피하는 접근도 있지만, 참조 비디오와 타깃 비디오의 모든 토큰 사이에 풀 셀프 어텐션을 걸다 보니 연산 복잡도가 토큰 수의 제곱으로 폭증한다. 여기에 더해 기존 방법들은 대체로 카메라 시점이 드라이빙 비디오에 고정되어버려서, 생성된 영상의 앵글을 자유롭게 바꿀 수도 없었다.

정리하면 명시적 표현은 체형 차이에 약하고, 암묵적 표현은 디테일을 잃고, ICL은 비효율적이다. 세 갈래 모두 나름의 이유로 막혀 있었던 셈이다.

## 왜 안 됐는가: 노이즈 간섭과 연산량

이 논문이 지목한 근본 원인은 두 가지다. 하나는 참조 비디오와 타깃 비디오를 하나의 흐름으로 섞어 처리하면, 디노이징 중인 타깃의 노이즈가 깨끗한 참조 정보와 뒤섞여 서로 간섭한다는 점이다. 다른 하나는 참조와 타깃의 모든 토큰을 서로 대응시키려다 보니 연산량이 감당할 수 없이 커진다는 점이다. ICL 방식의 셀프 어텐션 복잡도를 식으로 쓰면 다음과 같다.

$$O((N_r + N_l)^2)$$

여기서 $$N_r$$은 참조 토큰 수, $$N_l$$은 타깃 잠재 토큰 수다. 비디오는 프레임마다 토큰이 쌓이니 이 항이 순식간에 커진다. 게다가 실시간 스트리밍까지 생각하면 문제가 하나 더 붙는다. 디퓨전 모델은 원래 여러 단계를 거쳐 노이즈를 제거하는 구조라 지연 시간이 크고, 학습 때는 정답 프레임을 참조해서 다음을 예측하는 teacher forcing을 쓰지만 실제 추론에서는 자기가 만든 프레임을 다시 입력으로 써야 한다. 이 둘 사이의 간극에서 오차가 누적되는 노출 편향(exposure bias) 문제가 실시간 생성을 더 어렵게 만든다.

## 고친 방법: 흐름을 나누고, 대응되는 것만 잇는다

Wan-Animate-2는 고품질용 Base 모델과 실시간용 Lite 모델 두 갈래로 문제를 풀었다.

Base 모델의 핵심은 Dual-Branch DiT다. 노이즈가 섞인 타깃 잠재 흐름(타임스텝 $$t$$)과 노이즈 없는 참조 흐름(타임스텝을 $$0$$으로 고정)을 처음부터 분리해서, 서로 다른 성질의 두 신호가 섞이지 않게 만들었다. 대신 QKV 투영은 공유해서 정보 교환 통로는 열어둔다.

![Wan-Animate-2 Dual-Branch DiT 아키텍처: 노이즈 섞인 타깃 흐름과 참조 흐름이 공유 QKV로 들어가고 Time-Align RoPE와 Sparse-Ref Attention이 적용되는 구조](/assets/images/posts/wan-animate-2-pushing-the-application-boundaries-of-character-animation/figure-1.png)

두 흐름을 위치 정보 측면에서 정렬해주는 게 Time-Align RoPE다. 참조와 타깃 토큰을 프레임 단위로 결합해 위치 인코딩을 걸되, 두 비디오의 해상도가 다르면 참조 토큰에 동적 공간 오프셋을 부여해서 위치 인덱스가 겹치는 문제를 막는다.

그리고 연산량 문제는 Sparse-Ref Attention으로 해결했다. 발상은 단순한데, 타깃 프레임의 토큰이 참조 비디오 전체를 뒤질 필요 없이 시간적으로 대응되는 참조 프레임의 토큰만 보면 된다는 것이다. 이렇게 하면 복잡도가 다음처럼 줄어든다.

$$O(N_r \times N_l) \rightarrow O(N_l)$$

여기에 언리얼 엔진으로 만든 합성 다중 시점 데이터셋을 학습시킨 Viewpoint LoRA를 교차 어텐션에 붙여서, "right 60-degree view" 같은 자연어 프롬프트만으로 카메라 시점을 바꿀 수 있게 했다. 드라이빙 비디오의 시점에 묶여 있던 기존 한계를 텍스트 조건으로 풀어낸 셈이다.

실시간용 Lite 모델은 접근이 다르다. 비디오 잠재 공간을 8프레임 청크로 쪼개고 Causal Attention Mask를 써서 이전 청크를 보고 다음 청크를 생성하도록 학습한다. 다만 이것만으로는 앞서 말한 노출 편향이 남기 때문에, 단계별 예측 오차를 Error Buffer에 기록해뒀다가 다음 학습의 컨텍스트에 주입하는 방식으로 학습과 추론 사이의 간극을 메웠다. 마지막으로 디노이징 단계 수 자체를 DMD(Distribution Matching Distillation) 기반으로 대폭 줄이고, 14B 규모 모델을 감당하기 위해 자율 회귀 전방 패스는 그래디언트 없이 미리 점수를 계산해두고 청크 단위로만 역전파를 수행해 메모리 사용량을 청크 크기 수준으로 눌렀다.

![Wan-Animate-2-Lite의 자율 회귀 추론 파이프라인: 8프레임 청크 단위로 이전 결과가 다음 청크의 조건으로 주입되어 24fps로 순차 생성되는 구조](/assets/images/posts/wan-animate-2-pushing-the-application-boundaries-of-character-animation/figure-2.png)

## 결과와 한계

Base 모델은 명시적 모션 표현 없이도 identity drift 없이 정체성을 유지하면서 표정과 손가락 같은 고주파 디테일까지 살렸고, 카메라 시점을 자연어로 바꾸는 기능까지 얹었다. Lite 모델은 이 모든 걸 24 fps 실시간 스트리밍으로 압축해냈다. 다만 이 논문이 최종적으로 만들어내는 것은 "다른 신체가 실제로 그 동작을 수행하는 결과"가 아니라 "다른 외형의 캐릭터가 그 동작을 하는 것처럼 보이는 영상"이라는 점은 분명히 짚어둘 필요가 있다. 물리적 실행 가능성이나 접촉 동역학 같은 것은 애초에 이 문제 설정 밖에 있다.

## 용어 해설

- **DiT (Diffusion Transformer)**: 이미지를 패치 단위 토큰으로 쪼개어 Transformer 구조로 처리하는 디퓨전 모델. 타임스텝과 조건 정보를 함께 넣어 노이즈를 단계적으로 제거하며 이미지나 비디오를 생성한다.
- **RoPE (Rotary Position Encoding)**: 토큰의 위치 정보를 회전 변환 형태로 어텐션 연산에 주입하는 위치 인코딩 기법. 상대적 위치 관계를 자연스럽게 표현할 수 있어 긴 시퀀스에서도 잘 작동한다.
- **LoRA (Low-Rank Adaptation)**: 사전학습된 대형 모델의 원래 가중치는 고정한 채, 저차원 행렬 두 개를 추가로 학습해 적은 파라미터로 모델을 특정 작업에 맞게 미세조정하는 기법.
- **노출 편향 (Exposure Bias)**: 학습 때는 정답 데이터를 입력으로 주지만 추론 때는 모델이 스스로 만든 결과를 다음 입력으로 써야 해서, 학습과 추론 조건이 어긋나 오차가 누적되는 현상.

## 🤖 AI의 생각

로보틱스가 아니라 애니메이션 분야에서 나온 논문인데도 "행동을 어떻게 표현해야 서로 다른 몸에 잘 옮겨붙는가"라는 질문을 정면으로 다루고 있어서 인상적이었다. 다만 아래 내용은 사실 요약이 아니라 순전히 내 해석임을 밝혀둔다.

<details class="tc-faq">
<summary>명시적 모션과 암묵적 latent 사이의 트레이드오프를 이 논문보다 더 근본적으로 없앨 방법이 있을까</summary>
<div class="tc-faq__body" markdown="1">

Sparse-Ref Attention처럼 대응 관계를 국소적으로만 연결하는 중간 지점의 표현을 더 정교하게 설계하면 압축 손실과 identity drift를 동시에 줄일 여지가 있어 보인다

</div>
</details>

<details class="tc-faq">
<summary>Viewpoint LoRA처럼 합성 데이터로 조건을 늘리는 전략이 실제 물리 환경에서도 통할까</summary>
<div class="tc-faq__body" markdown="1">

언리얼 엔진 합성 데이터가 시각적 다양성은 주지만 물리적 타당성까지 보장하진 않으므로 taskcraft처럼 실제 동작이 필요한 도메인에서는 sim-to-real 격차를 별도로 검증해야 할 것이다

</div>
</details>

<details class="tc-faq">
<summary>Sparse-Ref Attention의 시간 정렬 아이디어를 taskcraft의 latent 상태 전이 표현에 그대로 적용할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

비디오는 프레임이라는 명확한 시간 축이 있어 대응 관계를 잡기 쉽지만, 로봇 시연 데이터는 embodiment마다 시간 스케일과 자유도가 달라 대응 기준을 정의하는 것 자체가 별도의 연구 문제가 될 것 같다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2608.06009v2" target="_blank" rel="noopener">Wan-Animate-2: Pushing the Application Boundaries of Character Animation</a> · arxiv</span></div>
</div>
</div>
