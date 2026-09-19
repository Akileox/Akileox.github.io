---
title: "파인튜닝 없이 로봇 조작하는 VLM"
date: 2026-09-19
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "별도 학습 없이 프롬프트 문맥만으로 로봇을 움직이는 GPT-Policy를 통해 In-Context Learning의 물리적 한계를 살펴본다."
paper_title: "In-Context Robot Learning with VLM Agents"
paper_summary: "별도 학습 없이 프롬프트 문맥만으로 로봇을 움직이는 GPT-Policy를 통해 In-Context Learning의 물리적 한계를 살펴본다."
paper_url: "https://huggingface.co/papers/2609.19138"
header:
  image: "/assets/images/posts/in-context-robot-learning-with-vlm-agents/figure-1.png"
  teaser: "/assets/images/posts/in-context-robot-learning-with-vlm-agents/figure-1.png"
---

이번 주 노트로 hf-daily에서 score 0.90을 받은 "In-Context Robot Learning with VLM Agents"를 골랐습니다. 로봇 정책 학습 쪽 논문들이 대부분 "더 많은 데이터로 더 큰 정책을 학습하자"는 방향인데, 이 논문은 아예 정책 파라미터 업데이트 자체를 없애고 범용 VLM의 In-Context Learning만으로 폐루프 로봇 제어가 어디까지 가능한지 실측했다는 점에서 taskcraft가 고민하는 embodiment-agnostic 학습 문제와 접점이 많아 이번 주 소재로 선택했습니다.

## 왜 로봇은 매번 다시 배워야 하는가

로봇 정책이 실환경에 배포되면 반드시 학습 데이터에 없던 물체, 조명, 배치를 만난다. Behavior Cloning으로 학습한 정책은 이런 새로운 상황마다 시연을 다시 모으고 그래디언트 업데이트를 다시 돌려야 한다. 문제는 이 재학습 사이클이 너무 느리다는 것이다. 로봇을 새로운 작업에 투입하고 싶을 때마다 데이터 수집, 학습, 배포를 반복하는 건 실용적이지 않다. 그래서 나온 아이디어가 test-time에 파라미터를 건드리지 않고 맥락(context)만 바꿔서 행동을 바꾸는 것, 즉 In-Context Learning이다.

## VLM이 로봇을 움직일 수 있을까라는 질문이 검증되지 않았던 이유

GPT급 범용 비전-언어 모델은 이미지 이해와 추론에서는 뛰어나지만, 그게 곧 물리적 접촉 작업까지 이어진다는 보장은 없다. 고수준에서 "병뚜껑을 돌려서 딴다"는 계획을 세우는 것과, 실제로 그리퍼를 몇 밀리미터 단위로 접근시키고 회전축을 맞추는 저수준 제어는 완전히 다른 난이도의 문제다. VLM이 텍스트로 그럴듯한 행동 계획을 뱉어도, 그걸 안전하게 로봇 관절 궤적으로 바꾸는 다리가 없으면 하드웨어가 부서지거나 작업이 실패한다. 기존 연구들은 VLM을 고수준 플래너로만 쓰고 저수준 제어는 별도 학습된 정책에 맡기는 식이었는데, 이 논문은 그 경계를 아예 없애고 VLM이 직접 툴을 호출해 폐루프로 로봇을 제어하게 만들었다.

## GPT-Policy, 맥락을 프롬프트에 욱여넣기

핵심 아이디어는 단순하다. 인간 시연 비디오, 로봇 궤적과 액션 시퀀스, 목표 이미지, 이전 상호작용 이력까지 전부 "맥락 컴파일러(Context Compiler)"가 파싱해서 하나의 구조화된 프롬프트로 만든다. 파라미터가 고정된 VLM은 이 프롬프트와 현재 관측값을 보고 `move_to`, `move_eef_chunk`, `set_gripper` 같은 파라미터화된 툴 호출을 생성한다. 여기까지는 다른 VLM-as-planner 연구들과 비슷하지만, 이 논문의 차별점은 그 다음에 있는 "제약 실행 하네스(Constrained Execution Harness)"다. VLM이 뱉은 목표 자세를 곧이곧대로 실행하지 않고, 선형 보간과 SLERP로 경로를 매끄럽게 만든 뒤 역기구학(IK) 해를 계산하고, 그 해가 위치·자세 오차 허용 범위 안에 있는지 검증한다. 여기서 걸리면 Ruckig 알고리즘으로 저크(jerk)와 가속도 한계를 넘지 않게 시간 스케일링을 다시 계산해서 실행한다. 즉 VLM은 "어디로 가야 하는지"를 판단하고, 하네스는 "그걸 물리적으로 안전하게 가는 법"을 책임지는 역할 분담이다.

![GPT-Policy 전체 아키텍처, VLM 정책과 제약 실행 하네스의 폐루프 구조](/assets/images/posts/in-context-robot-learning-with-vlm-agents/figure-1.png)

수식으로 보면 이 루프는 다음과 같이 정리된다.

$$a_t \sim \pi_\theta(\cdot \mid T, c_t, o_t, f_{t-1})$$
$$(o_{t+1}, f_t) = \mathcal{E}(a_t, o_t)$$

작업 지시문 $$T$$, 맥락 $$c_t$$, 현재 관측값 $$o_t$$, 직전 실행 피드백 $$f_{t-1}$$을 모두 받아서 VLM $$\pi_\theta$$가 행동 $$a_t$$를 뽑고, 실행 인터페이스 $$\mathcal{E}$$가 이를 물리적으로 수행한 뒤 다음 관측값과 피드백을 되돌려주는 구조다. 여기서 $$\theta$$는 절대 업데이트되지 않는다. 학습이 일어나는 곳은 파라미터가 아니라 매 스텝 갱신되는 맥락 $$c_t$$와 피드백 $$f_{t-1}$$이다.

## 비디오만으로는 부족했다

논문에서 가장 흥미로운 실험 결과는 병뚜껑을 돌려 따는 양손(bimanual) 작업에서 나왔다. 인간 비디오나 로봇 비디오만 맥락으로 준 경우와, 여기에 정렬된 로봇 액션 수치까지 같이 준 경우를 비교했더니 그리퍼의 기울기와 접근 방향 오차가 확연히 달랐다.

![병뚜껑 따기 작업에서 비디오만 제공된 경우와 비디오+액션이 함께 제공된 경우의 엔드이펙터 자세 오차 비교](/assets/images/posts/in-context-robot-learning-with-vlm-agents/figure-7.png)

비디오만 있을 때는 VLM이 프레임 간의 미세한 힘과 자세 전이를 추정하는 데 한계가 있어서 오차가 크게 남았지만, 수치 액션 레퍼런스가 결합되자 오차가 $$10^{-3}$$ 수준까지 떨어졌다. 이건 VLM이 "무엇을 해야 하는지"는 시각 정보만으로도 꽤 잘 파악하지만, "얼마나 정밀하게 해야 하는지"는 명시적인 수치 신호 없이는 잘 못 잡아낸다는 뜻이다. 접촉이 민감한 작업일수록 시각적 시연만으로는 부족하고, 정량화된 액션 정보가 반드시 함께 있어야 한다는 게 실험으로 확인된 셈이다.

## 남는 한계

이 방식의 가장 큰 한계는 결국 "학습을 없앤 게 아니라 다른 데로 떠넘긴 것"이라는 점이다. VLM의 파라미터는 고정되어 있으니, 실제로 작업을 잘 해내는지 여부는 그 VLM이 사전학습 단계에서 얼마나 물리적 상호작용에 대한 암묵적 지식을 갖췄는지에 전적으로 의존한다. 논문에서 다룬 작업들도 병뚜껑 따기, 블록 배치, 틱택토처럼 비교적 구조화된 시나리오들이라, VLM의 사전지식이 부족한 완전히 낯선 도메인(예: 특이한 재질의 변형체 조작)에서도 같은 수준의 성능이 나올지는 미지수다. 또한 제약 실행 하네스가 IK 잔차와 저크 한계를 검증해준다고 해도, 이는 어디까지나 물리적 안전성을 보장하는 것이지 작업 성공률 자체를 보장하지는 않는다. VLM이 애초에 틀린 목표 자세를 제시하면 하네스는 그걸 안전하게 실행할 뿐이다.

## 용어 해설

- **역기구학(Inverse Kinematics, IK)**: 로봇 팔 끝단(엔드이펙터)이 원하는 위치와 자세를 갖도록 각 관절의 각도를 역산하는 계산 과정이다.
- **SLERP(Spherical Linear Interpolation)**: 두 회전(자세) 사이를 구면 위의 최단 경로를 따라 부드럽게 보간하는 방법으로, 3D 회전을 자연스럽게 연결할 때 쓰인다.
- **저크(Jerk)**: 가속도의 시간 변화율로, 로봇 움직임에서 저크가 너무 크면 급격한 힘 변화로 하드웨어에 무리가 가거나 진동이 발생한다.
- **In-Context Learning(ICL)**: 모델 파라미터를 갱신하지 않고, 프롬프트에 주어진 예시나 맥락 정보만으로 새로운 작업에 적응하는 능력을 말한다.

## 🤖 AI의 생각


파라미터 업데이트를 아예 걷어내고 맥락 구성과 실행 하네스 설계에 문제를 몰아넣은 접근이 인상적이지만, 이건 결국 VLM의 사전학습 지식이 얼마나 좋은가에 성능을 통째로 위임하는 전략이라는 인상도 동시에 듭니다.

<details class="tc-faq">
<summary>비디오+액션 조합이 접촉 작업에 효과적이었다면, 액션 없이 비디오만으로도 정밀도를 높일 수 있는 중간 지점, 예를 들어 힘 센서 신호나 접촉 이벤트 마커를 프롬프트에 추가하는 방식은 얼마나 도움이 될까</summary>
<div class="tc-faq__body" markdown="1">

논문이 다룬 건 시각+수치 좌표뿐이라, 촉각이나 힘 정보 같은 다른 모달리티를 맥락에 얼마나 효과적으로 텍스트화할 수 있는지는 열린 문제로 남아있다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 human video에서 embodiment-agnostic latent를 명시적으로 학습해 이식하는 방향이라면, GPT-Policy처럼 latent 없이 원본 비디오를 그대로 VLM 프롬프트에 흘려 넣는 방식과 비교했을 때 어느 쪽이 새로운 로봇 하드웨어로의 전이에 더 유리할지</summary>
<div class="tc-faq__body" markdown="1">

latent를 학습하면 특정 embodiment 편향을 명시적으로 제거할 수 있지만 재학습 비용이 들고, GPT-Policy식 접근은 재학습은 없지만 VLM이 embodiment 차이를 스스로 얼마나 잘 무시하는지에 기대야 해서 서로 다른 트레이드오프를 가진다.

</div>
</details>

<details class="tc-faq">
<summary>제약 실행 하네스가 안전성은 보장해도 작업 성공률은 보장하지 않는다면, VLM이 애초에 잘못된 목표를 낼 확률을 줄이기 위해 피드백 루프에 실패 원인을 더 구조화해서 되돌려주는 방법이 있을까</summary>
<div class="tc-faq__body" markdown="1">

지금은 단순한 실행 결과와 오류 메시지를 피드백으로 주는데, 실패 시점의 이미지 차분이나 오차 방향 같은 더 구체적인 진단 정보를 다음 프롬프트에 포함시키면 VLM의 자기 교정 정확도가 올라갈 여지가 있어 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.19138" target="_blank" rel="noopener">In-Context Robot Learning with VLM Agents</a> · hf-daily</span></div>
</div>
</div>
