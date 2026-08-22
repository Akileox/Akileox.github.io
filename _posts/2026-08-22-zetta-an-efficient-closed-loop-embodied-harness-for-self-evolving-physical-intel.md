---
title: "VLA는 얼리고 하네스만 진화시키다"
date: 2026-08-22
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "정책은 동결한 채 런타임 크리틱과 복구 스킬로 구성된 하네스만 폐루프로 진화시키는 Zetta를 살펴본다."
paper_title: "Zetta $ζ$: An Efficient Closed-Loop Embodied Harness for Self-Evolving Physical Intelligence"
paper_summary: "정책은 동결한 채 런타임 크리틱과 복구 스킬로 구성된 하네스만 폐루프로 진화시키는 Zetta를 살펴본다."
paper_url: "https://arxiv.org/abs/2608.16590v1"
header:
  image: "/assets/images/posts/zetta-an-efficient-closed-loop-embodied-harness-for-self-evolving-physical-intel/figure-1.png"
  teaser: "/assets/images/posts/zetta-an-efficient-closed-loop-embodied-harness-for-self-evolving-physical-intel/figure-1.png"
---

이번 주에는 로봇 조작 정책의 "실시간 실패 교정" 문제를 정면으로 다룬 논문을 골랐습니다. arXiv에서 스코어 1.00으로 최상위에 오른 이유는 명확한데, VLA(Vision-Language-Action) 정책을 파인튜닝하지 않고도 그 바깥에 감시와 복구 레이어를 씌워 자기 진화(self-evolving)시킨다는 설계가, 최근 임바디드 AI 커뮤니티가 계속 부딪혀온 지연시간과 일반화 사이의 트레이드오프에 실질적인 답을 주는 방향이기 때문입니다. 정책 자체는 건드리지 않는다는 원칙이 특히 눈에 띄어서 이번 주 노트로 선택했습니다.

## VLA는 왜 "그때그때" 실패를 못 고칠까

로봇 팔이 물체를 잡다가 살짝 미끄러지거나 접촉이 불안정해지는 상황은 조작 태스크에서 흔하다. 문제는 지금의 VLA 기반 에이전트 대부분이 이런 순간을 실시간으로 감지하지 못한다는 점이다. 대개 에피소드가 끝난 뒤에야 성공/실패를 판정하고 사후 반추(post-hoc reflection)로 다음 시도를 개선하거나, 아예 정적으로 짜인 스킬 시퀀스를 그대로 실행한다. 밀리초 단위로 벌어지는 물리적 오차는 이런 사후적 대응 사이클보다 훨씬 빠르게 지나가버린다.

그렇다고 LLM이나 VLM 기반 에이전트를 고주파 제어 루프 안에 그대로 욱여넣기도 어렵다. 추론 지연시간이 너무 크기 때문이다. 결국 실패가 나면 사람이 저수준 제어 파라미터를 하나씩 손으로 고치는 식으로 대응하게 되는데, 이 방식은 두 가지 문제를 낳는다. 하나는 ad-hoc한 수정이 그 상황에만 맞춰지면서 일반화 성능을 깎아먹는 오버피팅이고, 다른 하나는 사람이 개입해야 하는 한계 때문에 긴 꼬리(long-tail) 실패 케이스를 확장성 있게 처리할 수 없다는 점이다.

## 정책은 얼리고, 하네스만 진화시킨다

Zetta가 택한 해법은 발상의 전환에 가깝다. 기저 VLA 정책 $$\pi$$와 이를 감독하는 온라인 중재자 $$\mathcal{A}_{orch}$$의 파라미터는 아예 동결($$\nabla_\theta = 0$$)해버린다. 대신 그 위를 감싸는 하네스 $$\mathcal{H} = \{C, R, \mathcal{T}\}$$만을 최적화 대상으로 삼는다. 여기서 $$C$$는 액션 주기 수준으로 궤적을 모니터링해 실패 징후와 모드 전환을 제안하는 런타임 크리틱, $$R$$은 인과적 실패 원인별로 정리된 복구 전략 라이브러리, $$\mathcal{T}$$는 진화 과정에서 동적으로 합성되는 실행 도구 모음(GraspGen, 모션 플래너, 임피던스 제어기 등)이다.

목적식은 다음과 같이 정리된다.

$$\max_{\mathcal{H}} J(\mathcal{H}) = \mathbb{E}_{g, s_0 \sim \mathcal{D}} [\text{Success}(\tau) \mid \pi, \mathcal{A}_{orch}, \mathcal{H}]$$

정책도 중재자도 고정한 채 하네스의 코드와 파라미터 공간만 바꾸면서 성공률을 끌어올린다는 뜻이다. 파인튜닝처럼 모델 내부를 헤집지 않으니 기존 정책이 가진 일반화 능력을 훼손할 위험이 원천적으로 줄어든다.

실제 실행 중에는 크리틱이 궤적 $$\tau_{0:t}$$를 보고 실패 증거 $$e_t$$와 추천 모드 $$\hat{\sigma}_t$$를 담은 제안 $$P_t = \langle e_t, \hat{\sigma}_t \rangle$$를 올리고, 중재자가 이를 검증해서 VLA 실행을 계속할지($$\sigma_t = 0$$) 복구 툴로 전환할지($$\sigma_t > 0$$)를 결정한다. 복구 동작이 끝나면 아무 때나 VLA에게 제어권을 넘기는 게 아니라, 원인 결함이 해소됐고 접촉력이나 토크 진동이 안정 임계값 이상으로 가라앉았는지를 확인하는 재진입 계약 $$\Psi(s_t)$$를 만족해야만 넘긴다. 이 부분이 은근히 중요한데, 복구 후 성급하게 정책을 되돌리면 같은 실패가 곧바로 재발할 수 있기 때문이다.

## 3단계로 나뉜 진화 루프

하네스를 어떻게 진화시키는지는 세 개의 시간 스케일이 분리된 루프로 설명된다.

![Zetta 진화 프레임워크: 병렬 롤아웃과 3단계 오프라인 진화(Phase I~III)가 폐루프로 순환하는 구조](/assets/images/posts/zetta-an-efficient-closed-loop-embodied-harness-for-self-evolving-physical-intel/figure-1.png)

Phase I(Loop 1)에서는 고주파 크리틱이 지켜보는 상태로 대량의 롤아웃을 돌리며 성공 궤적 기준선 $$I_{succ}$$과 비교해 실패 매니페스트를 모은다. Phase II(Loop 2)에서는 이 실패들을 조기 관측 분기점(Earliest Observable Divergence, EOD) 기준으로 클러스터링한다.

$$t_{EOD} = \min \{ t \mid \text{dist}(s_t, s_t^{ref}) > \epsilon, \; s_t^{ref} \in I_{succ}(\mu_t) \}$$

같은 마일스톤에서 성공 궤적 분포로부터 처음 벗어난 시점을 잡아내는 방식인데, 이렇게 하면 겉보기 증상이 아니라 메커니즘 단위로 실패를 묶을 수 있다. 이후 평가, 크리틱, 상태, 계획, 복구, 파라미터 순으로 내려가는 하향식 계층 인과 진단을 거쳐 최소 개입 수리 패치를 뽑아낸다. 여기서 "최소 개입"이라는 조건이 핵심인데, 손 닿는 대로 여러 곳을 고치면 결국 앞서 지적한 오버피팅 문제가 재발하기 때문이다. Phase III(Loop 3)에서는 이렇게 나온 패치들을 메커니즘 단위 불변량으로 통합하고, 과거 실패 100% 해결과 미확인 테스트셋 검증을 모두 통과해야만 영구 스킬 메모리에 반영한다. 검증을 게이트로 세워둔 덕분에 진화가 누적될수록 성능이 퇴화하는 걸 막는다.

## 병목은 결국 인프라였다

이 정도로 촘촘한 진화 루프를 돌리려면 롤아웃 자체를 엄청나게 많이, 빠르게 반복해야 한다. 논문이 Z-Infra라는 이름으로 별도 인프라 스택을 붙인 이유가 여기에 있다. VLA를 비전-언어 부분과 Action Expert로 쪼개서 CUDA IPC로 통신시키고 Prefix MLP에는 W8A8 양자화를 적용해 지연시간을 53% 줄였고, MuJoCo의 불변 모델 구조는 공유하되 가변 상태만 포크하는 방식으로 시뮬레이터 환경을 대량 병렬화했다.

![동시성 증가에 따른 에피소드 지연시간과 롤아웃 처리량 비교. Z-Infra 적용 시 처리량이 동시성 64에서 분당 35.1 에피소드까지 선형적으로 확장된다](/assets/images/posts/zetta-an-efficient-closed-loop-embodied-harness-for-self-evolving-physical-intel/figure-2.png)

결과는 확실히 인프라가 병목이었다는 걸 보여준다. Z-Infra 없이는 동시성이 늘수록 지연시간이 34초에서 112초로 급증하다 결국 메모리 부족(OOM)으로 죽는 반면, Z-Infra를 적용하면 95초 이하로 안정적으로 유지되고 처리량은 동시성 16 기준 베이스라인 대비 최대 12.8배까지 벌어진다. 자기 진화라는 아이디어가 아무리 그럴듯해도, 하루에 돌릴 수 있는 롤아웃 수가 적으면 실제로는 무용지물이라는 점을 이 그림이 잘 보여준다.

## 남는 의문

다만 Phase II의 하향식 인과 진단, 즉 평가부터 파라미터까지 내려가며 원인을 짚어내는 과정이 지금은 상당히 사람이 설계한 계층 구조와 휴리스틱에 의존하는 것처럼 읽힌다. 이 진단 절차 자체가 새로운 로봇 형태나 전혀 다른 실패 양상에도 자동으로 적응할지는 논문만으로는 확신하기 어렵다. 또한 정책을 완전히 동결한다는 전제가 강점이자 동시에 한계이기도 하다. 정책 자체가 애초에 특정 태스크를 구조적으로 못 푸는 경우라면, 아무리 하네스를 잘 진화시켜도 복구 스킬로 땜질하는 데는 분명 한계가 있을 것이다.

## 용어 해설

- **폐루프(Closed-Loop) 제어**: 시스템의 현재 출력을 실시간으로 관측해서 다음 입력을 조정하는 제어 방식. 반대로 관측 없이 미리 정해진 순서대로만 동작을 실행하면 개루프(Open-Loop)라고 부른다.
- **VLA(Vision-Language-Action) 모델**: 이미지나 영상 같은 시각 정보와 언어 지시를 함께 입력받아 로봇의 행동(관절 각도, 그리퍼 개폐 등)을 직접 출력하도록 학습된 모델.
- **양자화(Quantization)**: 모델의 가중치나 활성값을 32비트 부동소수점 대신 8비트 정수 같은 더 적은 비트로 표현해서 연산 속도와 메모리 사용량을 줄이는 기법. W8A8은 가중치와 활성값 모두 8비트로 표현한다는 뜻이다.
- **Covariate Shift**: 학습 데이터가 만들어진 상황과 실제 배포 환경에서 모델이 마주치는 입력 분포가 서로 달라지는 현상. 정책이 스스로 만든 궤적을 따라가다 보면 학습 때는 못 본 상태에 자주 놓이게 되는 게 대표적인 예다.

## 🤖 AI의 생각

이 논문은 개인적으로 "VLA를 더 잘 학습시키자"는 최근 흐름과는 결이 다른 방향이라 흥미롭게 읽었다. 정책 내부를 계속 고치는 대신 바깥에 감시와 복구 레이어를 쌓아 시스템 전체의 신뢰성을 올리는 쪽에 베팅했는데, 이건 소프트웨어 공학에서 서비스에 서킷 브레이커나 재시도 레이어를 얹는 발상과 닮아서 로봇 정책도 결국 모델 성능이 아니라 시스템 엔지니어링 문제로 수렴할 수 있다는 신호로 읽힌다.

<details class="tc-faq">
<summary>Phase II의 하향식 인과 진단 절차가 완전히 새로운 embodiment나 처음 보는 실패 유형에도 사람 개입 없이 자동으로 확장될 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문에서는 계층 구조(평가-크리틱-상태-계획-복구-파라미터)가 상당히 고정된 형태로 제시되어 있어서, 진단 계층 자체를 데이터로부터 학습시키는 후속 연구가 필요해 보인다

</div>
</details>

<details class="tc-faq">
<summary>정책을 완전히 동결하는 전제가 태스크 자체를 구조적으로 못 푸는 경우에도 유효할까</summary>
<div class="tc-faq__body" markdown="1">

복구 스킬은 어디까지나 정책이 얼추 맞는 방향으로 가다가 어긋난 걸 교정하는 역할이라, 정책이 애초에 잘못된 전략을 학습했다면 하네스만으로는 한계가 있을 것 같다

</div>
</details>

<details class="tc-faq">
<summary>"정책은 고정하고 그 위 레이어만 진화시킨다"는 원칙이 taskcraft가 지향하는 embodiment-agnostic latent 설계와 어떤 식으로 맞닿을 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

taskcraft가 공유 latent 인터페이스는 고정하고 embodiment별 디코더만 바꾸는 것처럼, Zetta도 정책이라는 공유 인터페이스는 건드리지 않고 하네스라는 얇은 레이어만 갈아끼운다는 점에서 왜 인터페이스를 고정하는 설계가 유리한지에 대한 방증 사례로 참고할 만하다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2608.16590v1" target="_blank" rel="noopener">Zetta $ζ$: An Efficient Closed-Loop Embodied Harness for Self-Evolving Physical Intelligence</a> · arxiv</span></div>
</div>
</div>
