---
title: "학습 없이 로봇을 움직인 VLM"
date: 2026-09-22
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "파인튜닝 없이 인터페이스 설계와 인컨텍스트 러닝만으로 VLM의 지능을 로봇 조작에 전이한 RoboDawn을 읽었습니다."
paper_title: "Transferring the Intelligence of VLMs to Robotic Control"
paper_summary: "파인튜닝 없이 인터페이스 설계와 인컨텍스트 러닝만으로 VLM의 지능을 로봇 조작에 전이한 RoboDawn을 읽었습니다."
paper_url: "https://huggingface.co/papers/2609.22966"
header:
  image: "/assets/images/posts/transferring-the-intelligence-of-vlms-to-robotic-control/figure-1.png"
  teaser: "/assets/images/posts/transferring-the-intelligence-of-vlms-to-robotic-control/figure-1.png"
---

이번 주는 HF Daily에서 score 0.83으로 상위권에 오른 논문을 골랐습니다. VLA(Vision-Language-Action) 모델을 만드는 흔한 방식, 그러니까 로봇 데이터를 잔뜩 모아 VLM을 파인튜닝하는 접근을 정면으로 뒤집는 논문이라 골랐습니다. "그 파인튜닝이 정말 필요한가"라는 질문 하나로 설계 전체를 다시 짠 게 흥미로웠습니다.

## 문제: 로봇을 똑똑하게 만들수록 로봇이 멍청해진다

VLM은 인터넷 규모의 이미지와 텍스트로 학습되면서 일반적인 시각 이해와 추론 능력을 갖추게 됐다. 이걸 로봇에 옮기려는 시도가 VLA고, 최근엔 월드 모델까지 붙인 WAM(World-Action Model) 계열도 나왔다. 방법은 대체로 비슷하다. 로봇 팔이 물건을 집고 옮기는 데이터를 모으고, 그 데이터로 VLM 파라미터를 다시 학습시킨다.

문제는 두 군데서 터진다. 첫째, 로봇 데이터는 인터넷 데이터에 비하면 턱없이 작고, 하드웨어마다 그리퍼 구조나 카메라 배치가 달라서 한 로봇에서 모은 데이터가 다른 로봇에 잘 안 옮겨간다. 둘째가 더 뼈아픈데, 이 좁은 로봇 데이터로 파인튜닝을 하고 나면 VLM이 원래 갖고 있던 범용 추론 능력과 복잡한 지시어 따라가는 능력이 눈에 띄게 퇴화한다는 보고가 이어졌다. 똑똑하게 만들려고 학습을 더 시켰는데 오히려 멍청해지는 역설이다.

이 논문은 그래서 아예 다른 길을 택한다. VLM 파라미터는 건드리지 않는다(frozen). 대신 로봇이 VLM의 언어를 알아듣게 만들자는 쪽으로 문제를 뒤집는다.

## 고친 방법 1: 사람이 이해하는 이산 명령어와 GIP

연속적인 모터 토크나 엔드이펙터 좌표를 직접 예측하게 시키면 VLM 입장에서는 익숙하지 않은 출력 공간을 새로 배워야 한다. RoboDawn은 대신 `move`, `rotate`, `gripper`, `home`, `wait`, `done` 같은, 사람이 게임패드나 원격조종에서 쓰는 것과 비슷한 이산 모션 프리미티브를 액션 공간으로 정의한다. VLM은 이미 이런 종류의 언어적 지시를 세상에서 숱하게 봐왔으니 별도 학습 없이도 바로 사용할 수 있다.

여기서 애매함이 생기는 지점은 "어디를 기준으로 옮길 것인가"다. 그리퍼의 두 손가락 끝을 각각 기준으로 삼으면 모델마다 해석이 갈릴 수 있다. 논문은 두 손가락 끝의 중간점을 GIP(Gripper Interaction Point)로 고정하고, 모든 이동과 회전, 화면 위의 시각적 주석을 이 한 점을 기준으로 통일한다. 사소해 보이지만 이런 기준점 하나가 모호성을 없애서 이후 파이프라인 전체를 단순하게 만든다.

## 고친 방법 2: 학습 대신 인컨텍스트 러닝

이산 명령어만으로는 부족한 부분이 있다. "10만큼 옮겨라"라고 했을 때 그게 몇 mm인지, 회전 명령을 줬을 때 그리퍼가 실제로 어떻게 움직이는지는 언어 규칙만으로 설명하기 어렵다. 논문은 이걸 파인튜닝이 아니라 인컨텍스트 데모(ICL)로 채운다. 데모를 두 종류로 나누는데, 기본 명령어 하나하나가 실제로 어떤 물리적 효과를 내는지 보여주는 프리머 데모($$D_{\text{prim}}$$)와, 실제 과업을 처음부터 끝까지 푸는 과정을 보여주는 태스크 데모($$D_{\text{task}}$$)다. 이 둘을 합쳐서 컨텍스트에 넣어주면 놀랍게도 단 하나의 데모(1-shot)만으로도 성능이 크게 뛴다. 파라미터는 그대로 두고 프롬프트에 실린 몇 장의 예시가 인터페이스 정렬 역할을 대신하는 셈이다.

## 고친 방법 3: 실패해도 다시 시도하는 폐루프 구조

아래 그림은 RoboDawn이 매 라운드마다 무엇을 보고 무엇을 결정하는지 보여준다.

![RoboDawn 폐루프 프레임워크 개요](/assets/images/posts/transferring-the-intelligence-of-vlms-to-robotic-control/figure-1.png)

전면, 손목, 탑다운 세 방향의 시각 관찰과 로봇 상태, 이전 라운드의 실행 피드백, 누적된 메모리가 매번 VLM에 다시 입력된다. VLM은 이걸 바탕으로 스크래치패드 형태의 추론을 거쳐 `left move x 10` 같은 이산 커맨드를 뱉고, 저수준 모션 플래너가 이걸 실제 안전한 궤적으로 바꿔 실행한다. 실행 결과는 다시 피드백으로 돌아와 메모리에 쌓인다.

이 구조를 하나의 흐름으로 정리하면 이렇다.

$$(y_t, a_t) = \pi_\theta (L, E, D; I_t, x_t, F_{t-1}, M_t)$$

$$(s_{t+1}, F_t) = \mathcal{E}_P (s_t, a_t)$$

$$(I_{t+1}, x_{t+1}) = \mathcal{O}_P (s_{t+1})$$

$$M_{t+1} = \mathcal{U} (M_t, a_t, y_t, F_t, x_{t+1})$$

여기서 $$\pi_\theta$$는 파라미터가 고정된 VLM, $$\mathcal{E}_P$$는 명령을 실제 물리 궤적으로 바꿔 실행하는 연산자, $$\mathcal{O}_P$$는 물리 상태로부터 다음 시각 및 로봇 상태 관찰을 생성하는 관찰 연산자, $$\mathcal{U}$$는 실행 결과를 다시 메모리에 쌓는 연산자다. 핵심은 어느 항에도 학습 가능한 파라미터가 새로 들어가지 않는다는 점이다. 대신 $$F_{t-1}$$과 $$M_t$$라는 두 채널이 매 라운드 "방금 뭐가 잘 안 됐는지"를 VLM에게 다시 알려주면서 다음 결정을 고쳐나간다.

## 결과: 명령어 예산을 늘리면 성능이 계속 올라간다

RoboDojo 벤치마크에서 에피소드당 허용 명령어 수를 60에서 240까지 늘려가며 성공률을 측정한 결과가 아래 그림이다.

![명령어 예산에 따른 test-time scaling 결과](/assets/images/posts/transferring-the-intelligence-of-vlms-to-robotic-control/figure-2.png)

제로샷은 23.7%에서 35.7%로, 원샷은 31.2%에서 47.2%로, 예산이 늘어날수록 두 조건 모두 꾸준히 우상향한다. 이 결과가 의미 있는 이유는, 단 한 번의 궤적으로 끝을 봐야 하는 오픈루프 방식이라면 예산을 더 줘도 성능이 오를 이유가 없기 때문이다. 예산이 늘수록 좋아진다는 건, 실패했을 때 그 실패를 피드백으로 받아서 자세를 재조정하고 다시 시도하는 능력이 실제로 작동하고 있다는 뜻이다. 파인튜닝된 정책이 아니라 frozen VLM이 이 복구를 해낸다는 게 이 실험의 핵심 주장이다.

## 남는 한계

다만 GIP 자체가 여전히 그리퍼라는 특정 형태의 엔드이펙터를 전제로 설계돼 있다. 두 손가락의 중간점이라는 정의는 평행 그리퍼에는 잘 맞지만, 다지(multi-finger) 핸드나 흡착식 그리퍼처럼 구조가 다른 엔드이펙터로 그대로 옮기긴 어려워 보인다. 또 이산 명령어의 세분성(granularity)을 데모로 채워야 한다는 건, 결국 새로운 로봇이나 새로운 작업 유형을 만날 때마다 소량이나마 다시 데모를 준비해야 한다는 의미이기도 하다. 완전한 zero-shot이 아니라 "적은 데모로 충분한" 수준이라는 점은 정직하게 짚고 넘어갈 부분이다.

## 용어 해설

- **VLA(Vision-Language-Action) 모델**: 시각 입력과 언어 지시를 함께 받아 로봇의 행동(관절 각도, 엔드이펙터 위치 등)을 직접 출력하도록 학습된 모델을 말한다.
- **인컨텍스트 러닝(In-Context Learning, ICL)**: 모델 파라미터를 갱신하지 않고, 프롬프트에 몇 개의 예시를 함께 넣어주는 것만으로 모델이 그 예시의 패턴을 따라 하도록 유도하는 방식이다.
- **모션 프리미티브(Motion Primitive)**: 로봇의 복잡한 움직임을 이동, 회전, 파지 같은 더 작고 재사용 가능한 기본 동작 단위로 쪼갠 것을 말한다.
- **폐루프(Closed-loop) 제어**: 시스템이 명령을 내린 뒤 그 결과(센서 관찰이나 실행 피드백)를 다시 받아 다음 명령에 반영하는 제어 방식으로, 결과를 확인하지 않고 명령만 내보내는 개루프(open-loop) 제어와 대비된다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인적인 인상을 남기면, 이 논문은 "모델을 더 학습시키자"가 당연시되는 흐름에서 "인터페이스를 더 잘 설계하면 학습이 줄어든다"는 반례를 꽤 설득력 있게 보여준 사례로 읽힌다.

<details class="tc-faq">
<summary>GIP 같은 형태 종속적인 기준점 없이도 이 프레임워크가 다지 핸드나 흡착 그리퍼에서 동작할 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

논문은 평행 그리퍼 기준으로만 검증했고, 기준점 정의를 엔드이펙터 형태마다 새로 설계해야 한다는 점에서 아직 완전히 embodiment-agnostic하다고 보긴 어렵다.

</div>
</details>

<details class="tc-faq">
<summary>명령어 예산을 늘려서 성능이 오른다는 결과는 결국 추론 비용(추론 시간, API 호출 수)이 늘어난다는 뜻인데, 이 트레이드오프는 어느 선까지 정당화될까?</summary>
<div class="tc-faq__body" markdown="1">

논문에서 예산 240까지의 곡선만 보여주는데, 실제 작업 현장에서는 지연 시간 제약이 더 타이트할 수 있어 이 test-time scaling이 어디서 한계에 부딪히는지가 더 궁금하다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 latent world model 학습으로 풀려는 embodiment 문제를, 이 논문처럼 "학습을 최소화하는" 방향으로도 접근할 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

데이터가 희소한 embodiment일수록 이 논문의 접근처럼 기준점과 이산 인터페이스를 잘 설계해 frozen 모델에 얹는 쪽이 latent representation을 처음부터 학습하는 것보다 비용 대비 효율적일 수 있어 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.22966" target="_blank" rel="noopener">Transferring the Intelligence of VLMs to Robotic Control</a> · hf-daily</span></div>
</div>
</div>
