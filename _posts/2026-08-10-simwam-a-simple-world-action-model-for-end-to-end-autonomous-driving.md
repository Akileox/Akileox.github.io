---
title: "세계모델은 학습에만, 추론엔 버려라"
date: 2026-08-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
excerpt: "SimWAM은 비디오 생성 모델의 지식을 학습 때만 빌리고 추론 시엔 완전히 떼어내 저지연 자율주행 플래닝을 달성한다."
paper_title: "SimWAM: A Simple World Action Model for End-to-End Autonomous Driving"
paper_summary: "SimWAM은 비디오 생성 모델의 지식을 학습 때만 빌리고 추론 시엔 완전히 떼어내 저지연 자율주행 플래닝을 달성한다."
paper_url: "https://huggingface.co/papers/2608.07468"
---

## 이번 주에 이 논문을 고른 이유

score 0.76, hf-daily 소스로 올라온 논문인데, 자율주행 World-Action Model이라는 특정 분야를 다루면서도 구조 자체는 훨씬 일반적인 질문 하나를 건드리고 있어서 눈에 띄었습니다. "무거운 생성 모델의 사전지식을 학습 때 흡수하고 추론 때는 버릴 수 있는가"라는 질문은 taskcraft가 latent world model과 embodiment별 정책을 분리하려는 시도와 뼈대가 겹칩니다. 그래서 이번 주 노트로 골랐습니다.

## 문제 제기: Imagine-Then-Act의 대가

자율주행 플래닝에 World-Action Model을 쓰는 최근 흐름은 대체로 이런 순서를 따른다. 먼저 미래 프레임을 비디오로 합성하고, 그 합성된 미래를 조건으로 삼아 궤적을 계획한다. DriveLaW나 DriveWAM 같은 모델들이 이 방식이다. 비디오라는 형태로 미래를 명시적으로 그려보니 플래너가 참고할 정보가 풍부해지는 건 사실이다.

문제는 이 비디오 합성 과정 자체가 계산량이 크다는 점이다. Diffusion 기반 비디오 생성은 수십 스텝의 디노이징을 거쳐야 하고, 이걸 실시간 주행 루프 안에 그대로 끼워 넣으면 지연 시간이 눈에 띄게 늘어난다. Figure 1을 보면 기존 WAM 계열 플래너들은 2000ms 이상의 latency를 소모하는데, 이는 실차 배포를 생각하면 받아들이기 어려운 수준이다.

## 왜 안 됐는지: 지식과 실행이 한 몸에 묶여 있다

여기서 근본적인 원인은 비디오 생성과 궤적 예측이 하나의 파이프라인 안에서 순차적으로 묶여 있다는 데 있다. 비디오 생성 모델이 배운 교통 역학(traffic dynamics) 사전지식 자체는 유용하다. 다른 차량이 어떻게 움직이고 도로가 어떻게 이어지는지에 대한 표현을 잘 학습했기 때문이다. 하지만 그 지식을 쓰려면 매번 실제로 비디오를 다시 렌더링해야 하는 구조라서, 지식과 실행이 분리되지 못하고 지연 시간에 그대로 발목이 잡힌다.

## 고친 방법: 학습 때만 붙이고 추론 때 떼어낸다

SimWAM의 해법은 의외로 단순하다. 사전 학습된 비디오 전문가(Video DiT)와 경량 행동 전문가(Action DiT)를 흐름 매칭(flow matching)으로 공동 학습시키되, **Isolated Attention Mask**라는 장치를 둔다. 이 마스크는 행동 토큰이 현재 관측 표현만 참조하도록 하고, 미래 비디오 토큰은 절대 들여다보지 못하게 차단한다.

이렇게 학습을 마치고 나면 재미있는 일이 생긴다. 행동 전문가는 애초에 미래 비디오 토큰에 의존한 적이 없으므로, 배포 시점에 비디오 DiT와 비디오 VAE 디코더를 통째로 버려도 성능에 지장이 없다. 남는 건 경량 Action DiT뿐이고, 이게 관측을 받아 곧바로 궤적을 뽑아낸다. 두 전문가가 가중치를 공유하지 않고 attention이라는 단일 인터페이스로만 소통하기 때문에 비디오 백본을 Wan2.2나 Cosmos2.5로 자유롭게 바꾸거나 Action DiT 크기만 독립적으로 키우는 것도 가능하다.

![Figure 2](/assets/images/posts/simwam-a-simple-world-action-model-for-end-to-end-autonomous-driving/figure-2.png)

여기에 더해 모방 학습 이후 단계로 Flow-GRPO를 적용한다. 원래 흐름 매칭은 확정적인 ODE 경로를 따르는데, 이걸 확률적 SDE로 바꾸면

$$dx_\tau = \left[ v_\theta(x_\tau, \tau) + \frac{\sigma_\tau^2}{2\tau} \left( x_\tau + (1-\tau) v_\theta(x_\tau, \tau) \right) \right] d\tau + \sigma_\tau dw, \quad \sigma_\tau = a \sqrt{\frac{\tau}{1-\tau}}$$

궤적 샘플링에 다양성이 생기고, 이걸 바탕으로 NAVSIM PDM 보상을 기준 삼아 GRPO로 정책을 추가 개선한다. 즉 모방 학습으로 기본기를 다지고, RL로 실제 주행 품질 지표에 맞춰 미세 조정하는 이중 구조다.

## 결과: 성능은 유지하고 지연은 버린다

![Figure 1](/assets/images/posts/simwam-a-simple-world-action-model-for-end-to-end-autonomous-driving/figure-1.png)

Figure 1의 latency-PDMS 그래프가 이 논문의 핵심 주장을 그대로 보여준다. 비디오 생성을 추론 루프에서 들어낸 SimWAM은 **500ms대의 낮은 latency를 유지하면서도 NAVSIM PDMS 91.5라는 최상위 수치**를 기록한다. 기존 WAM 계열이 2000ms 이상을 쓰면서 도달하던 성능대를 4분의 1도 안 되는 지연으로 달성한 셈이다.

다만 이 결과가 공짜로 나온 건 아니다. 학습 단계에서는 여전히 비디오 DiT를 통째로 함께 돌려야 하므로 학습 비용 자체가 줄어드는 건 아니고, 논문에서도 이 트레이드오프를 학습 비용을 늘려서 추론 비용을 낮추는 선택이라고 명확히 구분해서 다룬다. 또 Isolated Attention Mask가 정말로 지식 전이에 필요한 만큼의 정보만 통과시키는지, 아니면 더 세밀하게 조율할 여지가 있는지는 이 논문만으로는 판단하기 어렵다.

## 🤖 AI의 생각

이 논문은 세계 모델을 학습 때만 쓰고 추론 때는 버린다는 실용적인 트레이드오프를 아주 깔끔하게 구현한 사례라서, 세계 모델을 매개로 표현을 이식하려는 모든 프로젝트가 결국 부딪힐 질문에 하나의 실증적 답을 제시한다는 인상을 받았다. 그 질문은 세계 모델이 가진 지식을 얼마나, 어떤 형태로 하위 실행 모듈에 증류할 것인가인데, SimWAM은 별도의 distillation 단계 없이 공동 학습과 attention mask 하나로 이걸 해결했다. 개인적으로 더 파고들고 싶은 지점은, 이 마스크가 정말 필요한 만큼의 지식만 새어 들어가게 하는지, 아니면 학습 초반에는 정보가 더 많이 흘러가다가 후반에 정리되는 식의 동역학이 숨어 있는지인데 논문에서는 이 부분을 깊게 다루지 않는다.

taskcraft 입장에서 흥미로운 건 이 구조가 만약 cross-embodiment 세팅으로 확장된다면 어떻게 될까 하는 상상이다. Video DiT가 사람의 시연 영상을 담당하고 Action DiT가 로봇 관절을 담당하며 둘 사이를 attention으로만 연결한다면, taskcraft가 상정하는 공유 latent와 embodiment별 디코더 구조와 거의 같은 모양이 된다. 다만 이건 어디까지나 내 추측이고, 자율주행은 카메라 시점과 행동 공간이 이미 상당히 정렬된 비교적 닫힌 문제라는 점을 감안해야 한다. 몸의 형태 자체가 근본적으로 다른 경우, 예를 들어 조류형 로봇과 사람 손 사이에서도 이 attention 기반 분리가 버텨줄지는 훨씬 회의적으로 봐야 한다고 생각한다. 이 논문이 푼 건 같은 몸 안에서 감각과 행동을 분리하는 문제이지, 다른 몸으로 지식을 옮기는 문제는 아니라는 구분을 분명히 해두고 싶다.

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.07468" target="_blank" rel="noopener">SimWAM: A Simple World Action Model for End-to-End Autonomous Driving</a> · hf-daily</span></div>
</div>
</div>
