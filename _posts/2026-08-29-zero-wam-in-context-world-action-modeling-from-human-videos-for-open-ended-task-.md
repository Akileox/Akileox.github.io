---
title: "사람 시연 영상으로 로봇을 가르치는 법"
date: 2026-08-29
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "인간 시연 비디오를 인컨텍스트 프롬프트로 써서 로봇이 처음 보는 작업도 해내게 만드는 Zero-WAM을 리뷰합니다."
paper_title: "Zero-WAM: In-Context World-Action Modeling from Human Videos for Open-Ended Task Generalization"
paper_summary: "인간 시연 비디오를 인컨텍스트 프롬프트로 써서 로봇이 처음 보는 작업도 해내게 만드는 Zero-WAM을 리뷰합니다."
paper_url: "https://huggingface.co/papers/2608.26103"
header:
  image: "/assets/images/posts/zero-wam-in-context-world-action-modeling-from-human-videos-for-open-ended-task-/figure-1.png"
  teaser: "/assets/images/posts/zero-wam-in-context-world-action-modeling-from-human-videos-for-open-ended-task-/figure-1.png"
---

이번 주는 로봇 조작 학습에서 오랫동안 풀리지 않던 문제, 즉 로봇이 한 번도 학습하지 않은 작업(Zero-shot task)을 어떻게 일반화시킬 것인가를 정면으로 다룬 논문이 눈에 띄어 골랐습니다. HF Daily 소스에서 score 0.93이라는 높은 관심도를 보였고, 언어 지시 대신 인간 시연 비디오를 프롬프트로 쓴다는 아이디어가 taskcraft 프로젝트에서 다루는 "태스크를 어떻게 표현하고 전달할 것인가"라는 질문과도 맞닿아 있어 이번 주 논문 노트로 정리합니다.

## 언어로는 부족했던 것

로봇에게 새로운 작업을 시키는 가장 흔한 방법은 자연어 지시다. "컵을 집어서 접시 옆에 놓아라" 같은 문장을 VLA(Vision-Language-Action) 모델에 넣으면 되지 않을까 싶지만, 실제로는 언어가 담을 수 있는 정보량이 생각보다 얕다. 물체를 어느 각도로 쥐어야 하는지, 중간에 손이 어떤 경로를 지나야 하는지, 작업이 몇 단계로 구성되어 있는지 같은 공간적, 시간적 디테일은 문장 몇 개로 압축되지 않는다. 반면 사람이 그 작업을 수행하는 영상 한 클립에는 이 정보가 전부 들어 있다. 그래서 이 논문은 언어 대신, 혹은 언어와 함께 인간 시연 비디오를 인컨텍스트(In-Context) 프롬프트로 로봇에게 던져주자는 방향을 잡는다.

## 왜 그동안 안 됐는가

아이디어 자체는 새롭지 않다. 문제는 두 가지였다. 첫째, 인간 비디오와 로봇이 실제로 실행 가능한 액션 궤적이 쌍으로 존재하는 데이터가 거의 없다. 같은 작업을 사람이 손으로 하는 영상과 로봇 팔이 하는 영상을 동시에, 그것도 수천 개 태스크에 걸쳐 모으는 건 현실적으로 비용이 감당 안 되는 수준이다. 둘째, 설령 그런 데이터를 어찌어찌 만들었다 해도 학습 과정에서 모델이 지름길을 찾는 문제가 생긴다. 로봇의 과거 행동 이력과 텍스트 지시어만 봐도 다음에 뭘 해야 할지 대충 맞출 수 있다면, 모델은 굳이 인간 비디오를 참고하지 않고도 손실을 줄일 방법을 찾아버린다. 테스트 시점에 정작 중요한 인간 비디오 프롬프트가 있어도 무시하는 셈이다. 이 숏컷(Shortcut) 학습 문제는 인컨텍스트 학습을 쓰는 대부분의 방법이 공통으로 겪는 함정이다.

## 고친 방법: 데이터를 만들고, 지름길을 막는다

이 논문이 첫 번째 문제에 대해 내놓은 답은 HumanGen이라는 자동 데이터 생성 파이프라인이다. 사람이 일일이 촬영하는 대신, 이미 존재하는 로봇 궤적 비디오를 VLM(Gemini, Qwen 등)으로 분석해 어떤 작업인지, 시작과 끝 상태가 어떻게 바뀌는지 파싱한다. 그 다음 이미지 편집 모델로 첫 프레임을 인간 손과 인간 환경처럼 바꾸고, 비디오 생성 모델(Wan, Kling AI)로 같은 시맨틱을 갖는 인간 시연 비디오를 합성한다. 품질이 떨어지는 결과는 다시 VLM으로 걸러낸다. 이 과정을 거쳐 8,600여 개 태스크, 74.2K 쌍의 인간-로봇 데이터셋을 확보했다. 사람을 시켜서 찍는 대신 생성 모델로 찍는다는 발상의 전환인 셈이다.

![Zero-WAM 전체 아키텍처, 언어 또는 인간 비디오 프롬프트로부터 미래 로봇 비디오를 상상하고 액션을 디코딩하는 파이프라인](/assets/images/posts/zero-wam-in-context-world-action-modeling-from-human-videos-for-open-ended-task-/figure-1.png)

두 번째 문제, 숏컷 학습을 막기 위한 장치가 IFP(In-Context Future Chunk Prediction)다. 핵심은 모델이 로봇 이력만 보고 게으르게 답을 내지 못하게, 비디오 트랜스포머의 중간 표현 $$\phi^{i+1}$$ 하나로부터 시간적으로 여러 스텝 떨어진 미래 로봇 비디오 청크들을 동시에 예측하도록 강제하는 보조 손실을 거는 것이다. 이때 보조 모듈에는 인간 비디오 $$h$$를 직접 주지 않는다. 그러면 모델 입장에서는 인간 비디오가 담고 있는 장기적인 작업 진행 정보를 어떻게든 $$\phi^{i+1}$$ 안에 압축해 넣어야만 먼 미래를 맞출 수 있게 된다. 인간 비디오를 무시하고 싶어도 무시할 수 없게 구조로 강제하는 방식이다.

아키텍처 관점에서는 Diffusion Policy가 액션 시퀀스를 노이즈에서 바로 denoising하던 것을 한 단계 확장했다고 볼 수 있다. Zero-WAM은 액션을 곧바로 생성하지 않고, 먼저 flow matching으로 미래 로봇 비디오를 생성한 뒤 그 생성된 비디오로부터 역동역학(Inverse Dynamics)으로 액션을 뽑아낸다.

$$p_\theta(x^{i+1}, a^{i+1} \mid x^{\le i}, a^{\le i}, c) = p_\theta^{\text{vid}}(x^{i+1} \mid [h, x^{\le i}], a^{\le i}, \ell) \cdot p_\theta^{\text{act}}(a^{i+1} \mid x^{\le i}, a^{\le i}, x^{i+1}, \ell)$$

이 식에서 비디오 모델은 인간 비디오 $$h$$를 접두사 메모리로 참조하지만, 액션 모델은 인간 비디오에 직접 어텐션하지 않고 방금 생성된 로봇 비디오 $$x^{i+1}$$만 보고 액션을 디코딩한다. 액션이 언어나 인간 비디오라는 추상적 신호가 아니라 시각적으로 그럴듯한 미래 상태에 근거해서 나오도록 만든 구조다. 인간 비디오와 로봇 비디오가 같은 VAE latent space를 공유하다 보니 토큰들이 뒤섞일 위험이 있는데, 이건 RoPE의 높이 축에 오프셋 $$\Delta H$$를 줘서 인간 비디오 토큰을 로봇 다중 뷰가 차지하는 좌표 범위 밖으로 밀어내는 방식으로 해결했다.

![HumanGen 데이터 생성 파이프라인, VLM 파싱부터 이미지 편집, 비디오 생성, 품질 검증까지의 흐름](/assets/images/posts/zero-wam-in-context-world-action-modeling-from-human-videos-for-open-ended-task-/figure-2.png)

## 결과와 남는 의문

정리한 논문 노트에는 아키텍처와 데이터 파이프라인, 손실 함수 설계까지는 상세히 나와 있는데, 정작 벤치마크 수치나 기존 방법 대비 성공률 비교 같은 정량적 결과는 담겨 있지 않았다. 이 부분은 솔직히 아쉬운 지점이다. 태스크 균형 사전학습에 6,000개 이상의 태스크, 에폭당 약 400K 궤적을 썼다는 규모감은 확인할 수 있지만, HumanGen으로 합성한 인간 비디오가 실제 사람 시연만큼 로봇 정책에 유효한 신호를 주는지, 아니면 생성 모델 특유의 아티팩트가 오히려 학습을 방해하지는 않는지는 숫자 없이는 판단하기 어렵다. 다음에 볼 기회가 있다면 이 부분의 ablation을 더 파봐야 할 것 같다.

## 용어 해설

- **Flow Matching**: 노이즈에서 시작해 목표 데이터 분포로 점진적으로 변환하는 벡터 필드를 학습하는 생성 모델 학습 방식으로, Diffusion 모델의 대안으로 최근 많이 쓰인다.
- **Inverse Dynamics**: 로봇공학에서 원하는 상태 변화(위치, 속도 등)가 주어졌을 때 그 변화를 만들어내기 위해 필요한 힘이나 토크, 즉 액션을 역산하는 문제를 뜻한다.
- **RoPE (Rotary Positional Embedding)**: 트랜스포머에서 토큰의 순서 정보를 회전 변환 형태로 임베딩에 주입하는 위치 인코딩 기법이다.
- **Mixture-of-Transformers (MoT)**: 하나의 큰 트랜스포머 안에서 서로 다른 모달리티나 데이터 유형마다 별도의 파라미터 서브셋을 두어 처리하는 구조를 말한다.

## 🤖 AI의 생각


인간 시연 비디오를 데이터로 취급하지 않고 프롬프트로 취급했다는 발상 자체는 신선하지만, 이 노트만으로는 그 아이디어가 실제로 작동하는지 판단할 근거가 부족하다는 점을 짚고 싶습니다. 아래는 이 논문을 더 파고들 때, 그리고 taskcraft 프로젝트와 연결 지을 때 생각해볼 지점들입니다.

<details class="tc-faq">
<summary>HumanGen으로 생성한 인간 비디오가 실제 사람이 찍은 시연 영상과 얼마나 다른 분포를 가질지, 그 도메인 갭이 로봇 정책 학습에 실제로 문제를 일으키는지 정량적으로 확인됐을까</summary>
<div class="tc-faq__body" markdown="1">

논문 노트에는 생성 파이프라인과 VLM 품질 검증 단계만 있고 실제 도메인 갭을 측정한 지표는 보이지 않아서, 이 부분이 후속 검증에서 가장 먼저 확인해야 할 지점으로 보인다.

</div>
</details>

<details class="tc-faq">
<summary>IFP의 미래 청크 예측 스트라이드 $$s$$나 예측 개수 $$K$$를 어떻게 정했는지, 이 하이퍼파라미터가 숏컷 억제 효과에 얼마나 민감한지</summary>
<div class="tc-faq__body" markdown="1">

너무 가까운 미래만 예측하면 숏컷이 여전히 통할 것이고 너무 먼 미래를 강요하면 학습이 아예 안 될 것 같은데, 이 균형점을 잡는 ablation이 논문에 있는지 원문에서 더 확인해보고 싶다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft에서 태스크를 언어 대신 시연 영상이나 궤적 형태로 표현하는 방식을 도입하면 태스크 정의의 모호함을 줄일 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

언어 지시가 놓치는 공간적, 시간적 디테일을 시연 데이터가 보완한다는 이 논문의 문제의식은 taskcraft가 태스크를 어떻게 명세할지 고민할 때도 그대로 참고할 만하다고 본다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.26103" target="_blank" rel="noopener">Zero-WAM: In-Context World-Action Modeling from Human Videos for Open-Ended Task Generalization</a> · hf-daily</span></div>
</div>
</div>
