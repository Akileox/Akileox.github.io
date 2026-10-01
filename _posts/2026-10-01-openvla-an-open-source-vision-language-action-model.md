---
title: "OpenVLA, 로봇도 언어모델처럼 배운다"
date: 2026-10-01
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "7B 오픈소스 모델이 55B 폐쇄형 모델을 이긴 비결, 액션을 토큰으로 바꾼 VLA 구조를 따라가본다"
paper_title: "OpenVLA: An Open-Source Vision-Language-Action Model"
paper_summary: "7B 오픈소스 모델이 55B 폐쇄형 모델을 이긴 비결, 액션을 토큰으로 바꾼 VLA 구조를 따라가본다"
paper_url: "https://arxiv.org/abs/2406.09246v3"
header:
  image: "/assets/images/posts/openvla-an-open-source-vision-language-action-model/figure-1.png"
  teaser: "/assets/images/posts/openvla-an-open-source-vision-language-action-model/figure-1.png"
---

이번 주는 OpenVLA를 골랐습니다. Imitation Learning이나 RL은 수업에서 배웠지만 최근 로봇 정책들이 왜 전부 거대 언어모델을 끌어다 쓰기 시작했는지 궁금했던 분들께 딱 맞는 논문이라고 판단했습니다. 특히 이 논문은 아키텍처, 학습 데이터, 가중치까지 전부 공개한 거의 유일한 대규모 VLA라서, 블랙박스로 남아있던 RT-2 계열 모델의 내부를 들여다볼 좋은 기회라고 봤습니다.

## VLA가 뭔지부터

Imitation Learning 수업에서 배운 Behavior Cloning은 결국 상태를 받아 액션을 내놓는 policy를 지도학습으로 흉내내는 것이다. 문제는 policy가 학습 데이터에 없던 배경, 물체, 지시어를 만나면 쉽게 무너진다는 점이었다. 반면 CLIP, SigLIP, Llama 2 같은 비전 언어 모델들은 인터넷 규모 데이터로 사전학습되어서 본 적 없는 물체나 문장도 어느 정도 이해한다. 그럼 이 두 세계를 합치면 어떨까라는 질문에서 나온 게 Vision Language Action, 줄여서 VLA 모델이다. 이미지와 언어 지시를 입력으로 받아서 로봇 액션을 출력하는, 거대한 비전 언어 모델을 로봇 제어용으로 바꾼 것이라고 생각하면 된다.

RT-2가 이 아이디어로 강력한 일반화 성능을 보여줬지만 구글 내부에서 폐쇄적으로 학습된 모델이라 아키텍처도 학습 절차도 공개되지 않았다. 게다가 이미 학습된 VLA를 새로운 로봇이나 새로운 태스크에 효율적으로 파인튜닝하는 방법도 거의 연구되지 않은 상태였다. 소비자용 GPU 한 장으로 파인튜닝할 수 있는지조차 아무도 확인하지 않았다는 뜻이다. OpenVLA는 이 두 가지 빈틈, 폐쇄성과 파인튜닝 효율성을 동시에 메우려는 논문이다.

## 아키텍처: 로봇 액션을 토큰으로 바꾼다

![OpenVLA 전체 파이프라인: 970k 로봇 에피소드로 학습된 VLM이 사용자 지시를 받아 로봇 액션을 생성하는 과정](/assets/images/posts/openvla-an-open-source-vision-language-action-model/figure-1.png)

전체 그림을 먼저 보면, 970k개의 실세계 로봇 시연 데이터가 비전 인코더와 Llama 2 7B로 구성된 기반 VLM에 들어가 파인튜닝되고, 최종적으로 "Wipe the table" 같은 문장 지시를 받으면 로봇 액션을 뱉어내는 폐루프 제어기가 된다. 핵심은 이 모든 과정이 기존 언어모델 학습 파이프라인을 거의 그대로 재사용한다는 점이다.

그럼 로봇 액션을 어떻게 언어모델에 집어넣을까. Llama 2는 원래 토큰을 예측하도록 만들어진 모델이라 숫자 벡터인 로봇 액션을 그대로 받을 수 없다. OpenVLA는 각 액션 차원마다 학습 데이터의 1번째 분위수부터 99번째 분위수 구간을 256개 구간으로 쪼갠다.

$$\text{bin width}_d = \frac{Q_{99}(a_d) - Q_{1}(a_d)}{256}$$

min값과 max값을 그대로 쓰지 않고 분위수를 쓴 이유는 간단하다. 어쩌다 한 번 튀는 이상치 액션값이 있으면 min max 기준으로는 구간 폭이 쓸데없이 넓어져서 정작 대부분의 정상적인 액션들이 몇 개 안 되는 구간에 몰리게 된다. 분위수 기준으로 자르면 이런 왜곡을 피할 수 있다. 이렇게 이산화한 256개 구간을, Llama 토크나이저 어휘 중에서 거의 쓰이지 않는 마지막 256개 토큰 자리에 덮어씌워서 액션 토큰으로 재활용한다. 학습은 이 액션 토큰 부분에 대해서만 교차 엔트로피 손실을 건다.

$$\mathcal{L} = -\sum_{i \in \text{action tokens}} \log p_\theta(y_i \mid y_{<i}, I, \ell)$$

여기서 $$I$$는 입력 이미지, $$\ell$$은 언어 지시, $$y_i$$는 $$i$$번째 액션 토큰이다. 결국 일반 언어모델의 다음 토큰 예측과 수식 형태는 똑같고, 손실을 액션 토큰에만 거는 부분만 다르다. 이게 핵심이다. 복잡한 전용 헤드를 새로 만들지 않고, 언어모델이 이미 잘하는 다음 토큰 예측 그 자체를 로봇 제어에 그대로 가져다 쓴 것이다.

![OpenVLA 아키텍처: DINOv2와 SigLIP 특징을 결합해 Llama 2 임베딩 공간으로 매핑하고 액션 토큰을 생성하는 구조](/assets/images/posts/openvla-an-open-source-vision-language-action-model/figure-2.png)

비전 쪽을 보면 SigLIP 특징과 DINOv2 특징을 채널 단위로 붙여서 쓴다. SigLIP만 쓰면 의미적인 이해는 되지만 공간적인 정보가 약하고, DINOv2를 더하면 물체의 위치나 깊이 같은 공간 추론 능력이 올라간다는 걸 ablation으로 확인했다고 한다. 이 결합된 특징을 2계층 MLP로 Llama 2의 임베딩 공간에 투영해서 언어 토큰들과 나란히 넣어준다.

## 왜 이렇게까지 해야 했나: 숨겨진 디테일들

모델이 잘 작동하기까지 생각보다 미묘한 설계 결정들이 많았다. 논문은 이 부분을 꽤 솔직하게 풀어놓는다.

먼저 백본 선택. LLaVA 스타일 백본이 IDEFICS-1보다 다중 객체 태스크에서 35% 더 좋았고, SigLIP과 DINOv2를 결합한 Prismatic 백본이 다시 LLaVA보다 10% 좋았다. 해상도는 224픽셀이면 충분했다. 384픽셀을 쓰면 연산량이 3배로 늘어나는데 성능은 그대로였다.

가장 눈에 띄는 발견은 비전 인코더를 고정하지 않고 같이 파인튜닝해야 한다는 점이다. 일반적인 VLM 학습에서는 비전 인코더를 freeze하는 게 보통이고 그래야 사전학습된 시각 표현이 안 망가진다는 통념이 있는데, VLA에서는 거꾸로였다. 비전 인코더까지 같이 학습해야 공간 디테일을 로봇 제어에 맞게 세밀하게 조정할 수 있었던 것 같다.

학습 기간도 의외로 길었다. 27 에폭을 돌려야 액션 토큰 예측 정확도가 95%를 넘기고, 그 이후부터 실제 로봇 위에서의 성능이 꾸준히 올라갔다. 언어모델 fine-tuning 치고는 상당히 긴 학습 시간인데, 로봇 액션이라는 완전히 새로운 "언어"를 모델이 배우는 데 그만큼 시간이 걸린다는 뜻으로 읽힌다.

데이터는 Open X-Embodiment의 70개 이상 데이터셋 중에서 3인칭 카메라와 단일팔 엔드이펙터 제어를 쓰는 것만 걸러서 970k개 궤적으로 추렸다. DROID 데이터셋은 처음엔 10% 비중으로 섞었다가, 학습이 생각만큼 진행되지 않아서 마지막 3분의 1 구간에서는 아예 빼버렸다고 한다. 이런 중간 조정을 숨기지 않고 밝힌 점도 눈에 띈다.

## 효율적인 파인튜닝과 양자화

OpenVLA가 공개한 또 하나의 축은 "이 큰 모델을 어떻게 싸게 쓸 수 있는가"다. 전체 파라미터를 다 업데이트하는 full fine-tuning, 마지막 레이어만, 비전 인코더 고정, 비전과 토큰 임베딩과 마지막 레이어만 묶어서 업데이트하는 샌드위치 방식, 그리고 LoRA를 비교했다. 결과는 LoRA가 전체 파라미터의 1.4%만 건드리면서 full fine-tuning과 동등한 성능을 냈다는 것이다. 즉 소비자용 GPU 한 장으로도 새 로봇이나 새 태스크에 적응시킬 수 있다는 뜻이다.

추론 쪽에서는 bfloat16, int8, int4 세 가지 정밀도를 비교했다. int4로 양자화해도 bfloat16과 성능이 동등했고 메모리는 절반 이하로 줄었다. int8은 추론 속도가 느려지면서 실시간 제어 환경에서는 성능이 떨어졌는데, 블로킹 제어로 속도 영향을 제거하면 세 정밀도가 다시 동등해졌다. 양자화 자체의 품질 문제가 아니라 속도 문제였다는 걸 따로 확인해둔 점이 꼼꼼하다.

## 결과와 한계

BridgeData V2의 WidowX 로봇 평가에서 OpenVLA는 시각적, 운동적, 물리적 일반화와 언어 그라운딩 전 범주에서 RT-2-X(55B, 폐쇄형)를 능가했다. 파라미터 수로는 7배 작은 모델이 더 큰 모델을 이긴 셈이다. 다만 의미적 일반화 범주에서는 RT-2-X가 더 나았는데, 이건 RT-2-X가 더 큰 언어모델 백본의 세상 지식을 더 잘 활용했기 때문으로 추정된다.

Diffusion Policy, Octo와 비교한 실험도 흥미롭다. 단일 지시, 단일 태스크처럼 좁은 설정에서는 Diffusion Policy가 여전히 강했지만, 다중 물체와 다중 지시가 섞인 복잡한 태스크에서는 사전학습된 OpenVLA가 가장 우수했다. 대규모 로봇 사전학습의 이점이 태스크가 다양하고 복잡할수록 더 커진다는 뜻으로 읽힌다. 거꾸로 말하면 태스크가 단순하고 데이터가 충분하면 굳이 7B짜리 VLA를 안 써도 된다는 이야기이기도 하다.

한계도 분명하다. 학습 데이터가 단일팔 엔드이펙터, 3인칭 카메라라는 공통분모로 미리 걸러져 있어서, 액션 공간 자체가 근본적으로 다른 로봇, 가령 다리로 걷는 로봇이나 날개로 움직이는 로봇에는 이 토큰화 전략이 그대로 적용될지 검증되지 않았다. 또한 27 에폭이라는 긴 학습 시간과 970k 궤적이라는 데이터 규모는 여전히 상당한 컴퓨팅 자원을 요구한다.

## 용어 해설

- **Vision Language Action(VLA) 모델**: 이미지와 자연어 지시를 함께 입력받아 로봇 액션을 출력하도록 만든 모델 계열로, 비전 언어 모델을 로봇 제어용으로 확장한 형태다.
- **LoRA(Low Rank Adaptation)**: 거대 모델 전체를 다시 학습시키는 대신 각 레이어에 작은 저차원 행렬만 추가로 학습시켜 적은 파라미터로 파인튜닝하는 기법이다.
- **양자화(Quantization)**: 모델 파라미터를 32비트나 16비트 같은 고정밀 숫자 대신 8비트, 4비트처럼 더 적은 비트로 표현해서 메모리와 연산량을 줄이는 기법이다.
- **분위수(Quantile)**: 데이터를 크기 순으로 정렬했을 때 특정 비율 지점에 해당하는 값으로, 예를 들어 99번째 분위수는 데이터의 99%가 그 값 이하에 분포한다는 뜻이다.

## 🤖 AI의 생각


이 논문은 액션 공간이 비슷한 로봇들 사이에서 "표현을 공유하면 일반화가 된다"는 명제를 꽤 설득력 있게 증명한 사례로 보인다. 다만 이게 성립하는 전제 조건, 즉 단일팔 엔드이펙터와 3인칭 카메라라는 공통분모로 데이터를 미리 걸러냈다는 점이 오히려 눈에 밟힌다.

<details class="tc-faq">
<summary>액션 공간 자체가 근본적으로 다른 로봇, 가령 보행 로봇에도 이 분위수 기반 256-bin 토큰화 전략이 그대로 통할까</summary>
<div class="tc-faq__body" markdown="1">

단일팔의 연속적인 Δx, Δθ 공간과 달리 보행은 접촉과 타이밍이 핵심이라 같은 이산화 방식으로는 정보 손실이 더 클 가능성이 높아 보인다

</div>
</details>

<details class="tc-faq">
<summary>비전 인코더를 freeze하지 않고 같이 학습해야 성능이 오른다는 발견은 다른 world model 기반 접근에도 적용될까</summary>
<div class="tc-faq__body" markdown="1">

사전학습된 표현을 그대로 쓰기보다 로봇 상호작용 데이터로 재조정하는 과정이 필수적이라는 신호로 읽혀서 taskcraft의 latent world model에도 비슷한 재조정 단계가 필요할 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 풀려는 서로 다른 형태의 embodiment 간 이식 문제는 OpenVLA 방식으로 완화될 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

OpenVLA는 비슷한 embodiment들 사이의 일반화이고 taskcraft는 근본적으로 다른 형태 간 이식이라 출발점이 달라 LoRA식 저비용 적응 정도만 참고할 수 있을 듯하다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2406.09246v3" target="_blank" rel="noopener">OpenVLA: An Open-Source Vision-Language-Action Model</a> · arxiv</span></div>
</div>
</div>
