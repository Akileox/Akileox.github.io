---
title: "강화학습을 정책 대신 데이터 공장으로 쓰기"
date: 2026-10-27
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "RL이 하나의 정답에 수렴해버리는 문제를, 모방학습 데이터를 다양하게 뽑아내는 생성기로 뒤집은 DiscoDemo를 읽었습니다."
paper_title: "DiscoDemo: Discovering Efficient and Diverse Robot Demonstrations for Imitation Learning"
paper_summary: "RL이 하나의 정답에 수렴해버리는 문제를, 모방학습 데이터를 다양하게 뽑아내는 생성기로 뒤집은 DiscoDemo를 읽었습니다."
paper_url: "https://huggingface.co/papers/2610.32093"
header:
  image: "/assets/images/posts/discodemo-discovering-efficient-and-diverse-robot-demonstrations-for-imitation-l/figure-1.png"
  teaser: "/assets/images/posts/discodemo-discovering-efficient-and-diverse-robot-demonstrations-for-imitation-l/figure-1.png"
---

이번 주 논문 노트를 고를 때 가장 눈에 들어온 점은 score=0.82라는 수치보다 문제의식의 날카로움이었습니다. 로봇 매니퓰레이션에서 behavior cloning용 데이터를 모으는 일이 왜 이렇게 비싼지, 그리고 그 비용을 줄이겠다고 들고 온 RL이 왜 오히려 더 좁은 데이터를 만들어내는지를 정면으로 짚은 논문이라 이번 주 블로그 소재로 선택했습니다.

모방학습, 그중에서도 behavior cloning(BC)은 사람이 준 시연을 그대로 흉내 내는 방식이라 구조가 단순하고 학습이 안정적이다. 문제는 이 단순함이 그대로 약점이 된다는 점이다. 정책이 학습 데이터에 없던 상태로 한 발짝만 벗어나도 복구할 방법을 모르는, 이른바 distribution shift에 취약하다. 그래서 BC의 성능은 결국 "얼마나 다양한 성공 시연을 모았는가"에 달려 있는데, 그 다양한 시연을 사람이 직접 모으려면 비용이 어마어마하다.

그래서 시뮬레이션에서 자동으로 데이터를 만들려는 시도들이 나왔다. 모션 플래닝은 특정 작업 구조에 종속적이라 범용성이 떨어지고, MimicGen류는 기존 시연을 기하학적으로 재배치하는 방식이라 접촉이 많은 작업에서는 성공적인 상호작용 자체가 깨져버린다. DAgger나 PGDG 같은 섭동 기반 방법들은 기존 궤적 주변에 지원 영역을 넓히는 데는 효과적이지만, 결국 주어진 행동에 anchor된 채로만 움직인다. 그렇다면 RL로 직접 풀게 하면 되지 않을까 싶지만, 여기서 이 논문이 지적하는 진짜 문제가 나온다. task reward만으로 학습된 RL 정책은 신뢰할 수 있는 해법을 하나 찾는 순간 더 탐색할 이유가 없어진다. 그 뒤로 아무리 많은 롤아웃을 돌려도 state-action 분포는 한 곳에만 몰린다. RL이 "효율적인 해법"은 잘 찾지만 "다양한 해법"은 찾지 않는다는 뜻이다.

![RL 기반 데이터 생성 방법들의 한계를 비교한 그림](/assets/images/posts/discodemo-discovering-efficient-and-diverse-robot-demonstrations-for-imitation-l/figure-1.png)

DiscoDemo는 이 지점을 "생성기 자체가 작업을 푸는 여러 방법을 발견하게 만들면 되지 않을까"라는 질문으로 뒤집는다. 핵심은 RL을 배포할 정책으로 쓰지 않고, 끝까지 데이터를 뽑아내는 생성기로만 쓴다는 설계 결정이다. 먼저 소수의 인간 시연에서 출발해 병렬 역방향 커리큘럼으로 task-solving 행동을 부트스트랩한다. 각 시연마다 독립적인 커리큘럼 경계를 두고, 성공률이 임계값(0.9)을 넘으면 경계를 점점 작업 초반부로 당겨서 더 긴 구간을 스스로 풀게 만드는 방식이다. 이 자체는 sparse reward 환경에서 학습을 쉽게 만들어주는 장치다.

진짜 핵심은 여기에 latent skill $$z$$로 조건화된 diversity reward를 얹는다는 점이다. METRA에서 가져온 아이디어로, state 공간에서의 변위가 샘플링된 $$z$$와 정렬될수록 보상을 주는 식이다. 그런데 다양성 보상을 아무 데나 붙이면 안 된다는 걸 저자들도 알고 있다. 왜 안 되는지 생각해보면, sparse task reward 위에 다양성 보상을 그냥 더하면 다양성 신호가 과제 자체를 푸는 신호를 압도해버려서 애초에 성공조차 못 하는 정책이 나온다. 그래서 SMERL의 구조적 다양성 원칙을 빌려와 success gate $$y(\tau)$$를 둔다. 성공한 궤적들 사이에서만 다양성을 보상하고, 실패한 궤적이 다양해지는 건 애초에 막는 것이다. 여기에 접촉량이나 insertion force를 벌점화하는 safety shaping까지 더해서 물리적으로 말이 안 되는 상호작용을 걸러낸다.

학습이 끝나면 이 생성기를 고정해놓고, 원래 초기상태 분포에서 매 rollout마다 다른 $$z$$를 샘플링해 시연을 쏟아낸다. 이후 safety filter로 접촉 크기나 삽입력, 컨테이너 변위 기준을 통과한 궤적만 골라 정적 데이터셋으로 만들고, 이 데이터로 DROID에서 사전학습된 $$\pi_{0.5}$$ VLA를 특권 정보 없이 fine-tuning한다.

![DiscoDemo의 3단계 파이프라인: 생성기 학습, 시연 생성, 다운스트림 모방학습](/assets/images/posts/discodemo-discovering-efficient-and-diverse-robot-demonstrations-for-imitation-l/figure-2.png)

결과는 이 설계 의도를 숫자로 뒷받침한다. Figure 1의 막대 표에서 4개 과제 평균 성공률 DiscoDemo는 62%를 기록해 2위인 P-RFCL(40%)을 크게 앞섰다. TCP 궤적을 겹쳐 그린 분석에서도 모션 플래닝과 RFCL 계열은 좁은 패턴 하나를 반복하고, MimicGen과 PGDG는 사람 시연 주변의 변형에 머무는 반면 DiscoDemo만 뚜렷하게 구별되는 여러 궤적 다발을 만들어냈다. 더 흥미로운 지점은 이 다양성이 공짜가 아니라는 걸 저자들이 직접 확인했다는 점이다. coverage와 diversity가 둘 다 가장 높으면서도 생성 수율(yield)은 여전히 높고, 궤적 길이는 짧고, GPU 비용도 낮았다. 즉 다양성을 확보한다고 해서 탐색이 무분별해지거나 비효율적이 되는 게 아니라는 걸 수치로 보여준 셈이다.

물론 한계도 분명하다. latent skill $$z$$는 같은 embodiment, 같은 시뮬레이션 환경 안에서 서로 다른 해법을 구분하는 용도로만 쓰이기 때문에, 애초에 생성기가 풀 수 없는 과제 구조라면 다양성을 아무리 늘려도 해법 자체가 나오지 않는다. 또한 safety filter의 기준(접촉량, insertion force 임계값 등)이 과제마다 수작업으로 튜닝되는 부분이라, 새로운 과제로 확장할 때 이 기준 설계 자체가 병목이 될 수 있어 보인다.

## 용어 해설

- **Behavior Cloning**: 전문가의 시연 데이터(상태와 행동 쌍)를 지도학습처럼 그대로 모방하도록 정책을 학습시키는 모방학습 방법이다.
- **Distribution Shift**: 학습 때 본 데이터 분포와 실제 적용 시 마주치는 데이터 분포가 달라져서 모델 성능이 떨어지는 현상이다.
- **Latent Skill**: 정책의 행동 양식을 구분짓는, 사람이 직접 정의하지 않고 저차원 벡터로 표현되는 잠재적인 행동 패턴이다.
- **Curriculum Learning**: 쉬운 문제부터 점진적으로 어려운 문제로 난이도를 올려가며 학습시켜, 처음부터 어려운 문제를 푸는 것보다 효율적으로 학습하게 만드는 전략이다.

## 🤖 AI의 생각


RL을 배포용 정책이 아니라 다양성을 가진 데이터 생성기로만 쓴다는 발상 전환이 이 논문의 진짜 기여라고 생각합니다. 다만 아래는 논문이 명시적으로 답하지 않은, 더 파고들면 흥미로울 지점들에 대한 개인적인 의견입니다.

<details class="tc-faq">
<summary>success-gated diversity reward가 과제 난이도가 매우 높아 애초에 성공 궤적 자체가 드문 상황에서도 잘 작동할까</summary>
<div class="tc-faq__body" markdown="1">

게이트가 성공 여부에 의존하는 구조라 성공 샘플이 희박하면 다양성 신호를 받을 기회 자체가 줄어들어, sparse reward 환경에서의 병목이 다양성 학습 단계로 그대로 옮겨갈 가능성이 있다고 본다

</div>
</details>

<details class="tc-faq">
<summary>latent skill \(z\)가 포착하는 다양성이 "물리적으로 다른 해법"인지 "같은 해법의 스타일 변주"인지 어떻게 구분할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

Figure 4의 TCP 궤적 다발이 질적 증거이긴 하지만, 해법의 본질적 차이와 표면적 변주를 가르는 정량 지표가 있었다면 재현성 측면에서 더 설득력이 있었을 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 latent world model 매개 구조와 이 논문의 \(\phi_\psi\) 기반 diversity reward가 형식적으로 닮았다면, world model의 latent를 skill 조건화에 재사용할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

taskcraft가 이미 환경 표현을 학습한다면 별도의 METRA 표현을 새로 학습하는 대신 그 latent 공간 위에서 diversity reward를 정의하는 식으로 두 아이디어를 엮어볼 여지가 있다고 생각한다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2610.32093" target="_blank" rel="noopener">DiscoDemo: Discovering Efficient and Diverse Robot Demonstrations for Imitation Learning</a> · hf-daily</span></div>
</div>
</div>
