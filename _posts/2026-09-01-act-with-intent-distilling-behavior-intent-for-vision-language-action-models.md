---
title: "행동복제가 놓친 것, 로봇의 의도"
date: 2026-09-01
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "VLA 모델이 액션만 따라 하고 그 행동의 목적은 배우지 못한다는 문제를 teacher VLM 증류로 푼 논문을 읽었다"
paper_title: "Act with Intent: Distilling Behavior Intent for Vision-Language-Action Models"
paper_summary: "VLA 모델이 액션만 따라 하고 그 행동의 목적은 배우지 못한다는 문제를 teacher VLM 증류로 푼 논문을 읽었다"
paper_url: "https://huggingface.co/papers/2608.23478"
header:
  image: "/assets/images/posts/act-with-intent-distilling-behavior-intent-for-vision-language-action-models/figure-1.png"
  teaser: "/assets/images/posts/act-with-intent-distilling-behavior-intent-for-vision-language-action-models/figure-1.png"
---

## 이번 주에 이 논문을 고른 이유

이번 주 hf-daily 소스에서 스코어 1.00으로 최상위에 오른 논문입니다. VLA(Vision-Language-Action) 모델 학습에서 행동 복제(behavior cloning)의 근본적 한계를 짚고, 그 해법으로 teacher VLM의 해석을 액션 디코더에 증류하는 구조를 제안했다는 점에서 taskcraft가 다루는 "행동에서 의미를 뽑아내는 문제"와 방법론적으로 맞닿아 있어 이번 주 소재로 선정했습니다.

## 행동은 따라 했는데, 왜 필요한지는 모른다

로봇 조작 정책을 학습시키는 가장 기본적인 방법은 시연 데이터로 행동 복제를 하는 것이다. 관측과 언어 지시를 넣으면 시연자가 낸 모터 명령이 나오도록 지도한다. 최근에는 여기에 미래 프레임 예측, 궤적 예측, 모션 필드, 포인트 트랙 같은 future-based supervision을 얹어서 정책이 "다음에 무슨 일이 일어날지"까지 예측하도록 만드는 흐름도 늘었다.

문제는 이 두 방식 모두 "이 행동이 지금 이 지시 하에서 왜 필요한가"라는 국소적 목적(local objective)은 어디에도 명시적으로 담지 않는다는 점이다. 예를 들어 컵을 향해 접근해서 파지하는 동작은 겉보기에는 거의 똑같지만, 어떤 과제에서는 "옮기기 위한 준비"이고 어떤 과제에서는 "쏟아지지 않게 고정하기 위한 준비"일 수 있다. 반대로 목적이 같아도 실행 방식은 물체 위치나 장애물에 따라 완전히 다르게 실현될 수 있다. 행동 복제도 future supervision도 결국 "무엇을 했는가"와 "다음에 무엇이 될 것인가"만 볼 뿐, "왜"는 암묵적으로 남겨둔다.

저자들은 액션 디코더가 이 intent를 별도의 표현으로 복원하도록 만들어야 한다고 주장한다. 그래야 겉모습이 비슷한 행동들이 서로 다른 목적을 가질 때 이를 구분하고, 목적이 같은데 실행이 다른 경우를 하나의 표현 아래 묶을 수 있다는 것이다.

## teacher VLM이 사후에 해석한 것을 디코더 속으로 증류한다

이 문제를 풀기 위해 제안된 INDI(Intention Distillation)는 크게 세 부분으로 이루어진다.

먼저 학습 시에는 고정된 teacher VLM(Cosmos-Reason2-8B)이 현재 관측, 지시, 대략적인 액션 요약, 그리고 실제로 실행된 구간의 비디오를 한꺼번에 보고 "이 구간에서 무엇이 바뀌었고 어떤 목적을 수행했는지"를 자기회귀적으로 생성한다. 이 생성 과정에서 나온 중간 레이어(18층) 은닉 상태를 intent 타깃으로, 최종 레이어에서 만들어진 목적문의 은닉 상태를 텍스트 목적 타깃으로, 구간이 끝난 시점 관측을 별도 비전 인코더로 인코딩한 것을 시각적 결과 타깃으로 뽑아둔다.

그다음 실제 배포될 VLA 디코더 내부에는 학습 가능한 clean intent 쿼리와 self-conditioned grounding 행들을 추가한다. 디코더 중간 레이어(예: GR00T-N1.7의 32블록 중 16블록, π0.5의 18레이어 중 9레이어)에서 이 intent 쿼리들의 상태를 teacher가 만든 intent 타깃과 코사인 거리로 정렬시킨다. 즉 배포 시에는 teacher가 없어도, 디코더가 스스로 그 자리에서 intent를 복원해내도록 학습시키는 것이다.

마지막으로 이 복원된 intent가 실제로 쓰이도록 강제하는 두 가지 장치가 붙는다. 하나는 이진 게이트 $$\alpha \sim \text{Bernoulli}(0.5)$$로, 학습 중 절반의 경우 액션 행이 VLA 문맥에 직접 접근하지 못하게 막아서 정보가 반드시 intent 쿼리를 거쳐 흘러가도록 만든다. 다른 하나는 intent-mismatch loss인데, 배치 안에서 다른 샘플의 intent를 섞어 넣었을 때 다운스트림 손실이 실제 intent를 썼을 때보다 더 커지도록 힌지 손실로 강제한다. 이게 없으면 intent 쿼리가 그냥 존재하기만 하고 실제 예측에는 아무 영향을 안 주는 지름길이 생길 수 있는데, 이 손실이 그 지름길을 막는다.

![INDI의 개념 비교와 성능 향상 막대그래프](/assets/images/posts/act-with-intent-distilling-behavior-intent-for-vision-language-action-models/figure-1.png)

Figure 1은 이 구조 차이가 실제로 얼마나 큰 성능 차이로 이어지는지 보여준다. SimplerEnv-Bridge에서 GR00T-N1.7 베이스라인이 64.3%인데, future-based supervision을 추가해도 68.0%로 소폭만 오르는 반면 INDI를 적용하면 84.7%까지 뛴다. RoboCasa-Kitchen에서도 64.1%에서 65.8%, 70.3%로 같은 순서의 향상이 관찰된다. 흥미로운 점은 future supervision 자체는 그렇게 크게 기여하지 못한다는 것인데, 이는 "무엇이 될지"를 더 자세히 알려주는 것보다 "왜 필요한지"를 알려주는 것이 정책 학습에 더 중요한 신호라는 저자들의 주장을 뒷받침한다.

![INDI 아키텍처, teacher가 만든 intent를 디코더가 복원하는 구조](/assets/images/posts/act-with-intent-distilling-behavior-intent-for-vision-language-action-models/figure-2.png)

Figure 2를 보면 이 구조에서 배포 시에 남는 게 무엇인지 명확해진다. 왼쪽의 teacher VLM과 타깃 생성 파이프라인은 점선으로 표시되어 학습 전용임을 나타내고, 오른쪽의 디코더만 실제로 배포된다. 즉 무거운 teacher는 학습이 끝나면 완전히 사라지고, 디코더가 스스로 intent를 복원하는 능력만 남는다는 게 이 논문의 핵심 실용적 장점이다.

## 한계와 남는 질문

다만 이 방법이 정말로 "의미론적 목적"을 학습한 것인지, 아니면 teacher VLM이 잘 표현하는 특정 패턴(주로 텍스트로 서술 가능한 조작 목적)을 잘 흉내내게 된 것인지는 구분하기 어렵다. teacher 자체가 언어로 서술하기 애매한 미묘한 물리적 목적(예: 미끄러지지 않게 힘을 조절하는 이유)을 얼마나 잘 포착하는지는 논문에서 깊게 검증되지 않는다. 또한 intent 타깃이 여전히 같은 로봇 팔, 같은 관측 공간 안에서 정의되기 때문에, 신체 구조가 다른 로봇 사이에서도 이 intent 표현이 전이 가능한지는 별개의 질문으로 남는다.

## 용어 해설

- **증류(distillation)**: 크고 무거운 모델(teacher)이 만든 출력이나 중간 표현을 작고 가벼운 모델(student)이 모방하도록 학습시켜, 배포 시에는 student만 사용하면서도 teacher의 능력 일부를 이어받게 하는 학습 기법.
- **Flow matching**: Diffusion 모델처럼 노이즈에서 데이터로 가는 변환을 학습하되, 노이즈 제거 과정 대신 노이즈와 데이터를 잇는 벡터장(속도장)을 직접 회귀하도록 학습하는 생성 모델링 방법.
- **Self-conditioning**: 생성 모델이 자기 자신의 이전 예측(주로 노이즈가 덜 낀 깨끗한 추정치)을 다시 입력으로 받아 다음 예측의 조건으로 쓰는 기법으로, 반복적 생성 과정에서 정보 손실을 줄이는 데 쓰인다.
- **힌지 손실(hinge loss)**: 두 값의 차이가 일정 마진(margin) 이상이 되도록 강제하되, 이미 마진을 만족하면 손실이 0이 되는 형태의 손실 함수로, 대조 학습이나 랭킹 학습에서 흔히 쓰인다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인적인 감상을 덧붙이면, 이 논문에서 가장 인상적인 부분은 성능 수치보다 intent-mismatch loss 같은 "복원된 표현이 실제로 쓰이는지 검증하는 장치"를 따로 마련했다는 점이다. latent를 정렬만 시키고 끝내는 논문이 많은데, 여기서는 그 latent가 실제로 다운스트림 예측을 좌우하는지까지 손실 함수로 강제한 게 방법론적으로 꽤 신뢰가 간다.

<details class="tc-faq">
<summary>teacher VLM이 언어로 서술하기 힘든 미묘한 물리적 목적(마찰, 힘 조절 등)도 intent로 잘 포착할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문은 이 부분을 깊게 검증하지 않아서, teacher의 서술 능력 자체가 intent 품질의 상한선이 될 가능성이 있어 보인다

</div>
</details>

<details class="tc-faq">
<summary>이 intent 표현은 로봇 팔이 아니라 손이나 이족 보행처럼 다른 embodiment에서도 재사용될 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문의 teacher가 여전히 같은 관측/액션 공간을 전제하므로 이 검증은 빠져 있고, taskcraft가 채워야 할 지점으로 남는다

</div>
</details>

<details class="tc-faq">
<summary>intent-mismatch loss처럼 latent 병목이 실제로 쓰이는지 강제하는 장치를 taskcraft의 world model latent에도 그대로 적용할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

게이팅과 mismatch 손실 조합은 embodiment가 달라도 재구성 손실만 흉내내는 지름길을 막는 정칙화로 그대로 옮겨볼 만한 아이디어로 보인다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.23478" target="_blank" rel="noopener">Act with Intent: Distilling Behavior Intent for Vision-Language-Action Models</a> · hf-daily</span></div>
</div>
</div>
