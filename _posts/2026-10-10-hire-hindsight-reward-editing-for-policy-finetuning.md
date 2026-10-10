---
title: "학습 없이 보상을 고치는 HiRE의 발상"
date: 2026-10-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "로봇 정책 파인튜닝의 보상 설계 문제를, 신경망 학습 없이 밀도 비 추정만으로 풀어낸 HiRE를 읽었다."
paper_title: "HiRE: Hindsight Reward Editing for Policy Finetuning"
paper_summary: "로봇 정책 파인튜닝의 보상 설계 문제를, 신경망 학습 없이 밀도 비 추정만으로 풀어낸 HiRE를 읽었다."
paper_url: "https://www.semanticscholar.org/paper/36448cf08024b9ba02275382a4b91de66abe6563"
header:
  image: "/assets/images/posts/hire-hindsight-reward-editing-for-policy-finetuning/figure-1.png"
  teaser: "/assets/images/posts/hire-hindsight-reward-editing-for-policy-finetuning/figure-1.png"
---

이번 주 논문 노트는 semantic-scholar의 추천 목록에서 가장 높은 점수(0.85)를 받은 HiRE: Hindsight Reward Editing for Policy Finetuning입니다. 로봇 정책을 RL로 파인튜닝할 때 누구나 한 번은 부딪히는 보상 설계 문제를, 별도의 신경망 학습 없이 닫힌 형식(closed-form) 수식 하나로 풀겠다는 접근이 눈에 띄어 골랐습니다.

## 희소 보상이 낳는 두 가지 함정

사전 학습된 로봇 정책을 RL로 더 잘 만들려면 보상이 필요하다. 가장 간단한 방법은 작업 성공 여부만 알려주는 희소 보상(sparse reward)이다. 문제는 여기서 시작한다. 물체를 집고, 옮기고, 끼워 맞추는 다단계 작업에서 중간 과정이 얼마나 잘 되고 있는지에 대한 신호가 전혀 없으니, 장기 호라이즌 작업일수록 신용 할당(credit assignment)이 거의 불가능해진다.

그래서 많은 연구가 DINOv2나 CLIP 같은 파운데이션 비전 모델로 상태와 목표 이미지의 유사도를 재서 보상으로 쓴다. 이 방식은 인터넷 규모 데이터로 학습된 덕분에 처음 보는 장면에도 잘 일반화되지만, 정작 로봇 물리 제어 데이터로는 한 번도 학습된 적이 없다는 치명적인 약점을 갖는다. 논문은 이를 "control-unaware"라고 부른다. 로봇이 물체를 놓쳐서 빈 배경만 카메라에 잡혀도, 배경이 목표 이미지와 시각적으로 비슷하면 높은 보상을 주는 거짓 양성(trap state)이 생긴다. 반대로 올바르게 물체에 접근하는 중간 동작인데도 최종 목표 프레임과 그림이 다르다는 이유만으로 낮은 보상을 주는 거짓 음성도 생긴다. 학습 기반 보상 모델(RoboMeter, GCR 등)로 이 문제를 고치려는 시도도 있었지만, 비디오 레이블링 비용이 크고 온라인에서 그래디언트를 업데이트하다 보면 가치 함수가 붕괴하는 불안정성이 따라왔다.

## 왜 기존 방법들이 안 됐는가

핵심은 "시각적으로 비슷하다"와 "제어상 진전이 있다"가 다른 개념이라는 점이다. 파운데이션 인코더는 이미지 공간의 유사도를 잘 포착하지만, 그 유사도가 로봇 팔이 실제로 목표에 가까워지고 있는지는 보장해주지 않는다. 그렇다고 매번 새로운 보상 신경망을 학습시키면 또 다른 불안정성(가치 함수 붕괴, 보상 해킹)을 끌어들이게 된다. 즉 "파운데이션 모델의 일반화 능력은 유지하면서 제어 인지적(control-aware) 신호로 바꾸는" 중간 지점이 비어 있었다.

## 고친 방법: 성공과 실패의 밀도 비를 구하자

HiRE의 아이디어는 신경망을 새로 학습시키는 대신, 이미 쌓인 성공 궤적과 실패 궤적의 분포 차이를 비모수적으로(training-free) 추정하자는 것이다. 최대 엔트로피 역강화학습 관점에서 보면, 성공 매니폴드 $$p_+(s)$$와 실패 매니폴드 $$p_-(s)$$를 구분하는 이상적 판별자가 주는 최적 보상은 결국 두 로그 밀도의 차이, $$\log p_+(s) - \log p_-(s)$$로 귀결된다. 문제는 이걸 어떻게 실제로 계산하느냐인데, HiRE는 여기서 커널 밀도 추정(KDE)을 쓴다. 다만 파운데이션 인코더로 뽑은 특징은 $$L_2$$ 정규화를 거쳐 단위 초구면 위에 놓이기 때문에, 유클리드 공간용 가우시안 커널 대신 방향성 데이터에 맞는 von Mises-Fisher(vMF) 커널을 쓴다.

이렇게 성공 버퍼 $$\mathcal{B}^+$$와 실패 버퍼 $$\mathcal{B}^-$$ 각각에 대해 밀도를 추정하면, LogSumExp 형태의 잠재 함수(potential)가 닫힌 형식으로 나온다.

$$
\Phi_{\text{HiRE}}(s) = \mathcal{L}^{\kappa^+}_{\mathcal{B}^+} \left( \text{sim}_\phi(s, g^+) \right) - \lambda \cdot \mathcal{L}^{\kappa^-}_{\mathcal{B}^-} \left( \text{sim}_\phi(s, g^-) \right)
$$

앞 항은 성공 궤적들과의 유사도를 부드러운 최댓값(soft max) 형태로 뭉쳐 끌어당기는 인력을, 뒤 항은 실패 궤적의 종료 프레임들과의 유사도를 밀어내는 척력을 만든다. 여기서 재밌는 설계가 집중도 파라미터 $$\kappa$$의 비대칭 비율 $$\lambda = \kappa^- / \kappa^+ < 1$$이다. 정밀 조작 작업에서 성공 경로는 매우 좁고 날카로운 반면 실패 모드는 훨씬 넓게 퍼져 있다는 관찰을, 성공 쪽 커널은 뾰족하게(큰 $$\kappa^+$$) 실패 쪽 커널은 완만하게(작은 $$\kappa^-$$) 설정하는 식으로 그대로 반영한 것이다.

이렇게 얻은 잠재 함수는 그냥 더하는 게 아니라 잠재 기반 보상 형성(PBRS), 즉 $$\gamma \Phi(s') - \Phi(s)$$ 형태로 희소 보상에 가산한다. 이 형태를 쓰는 이유는 수학적으로 원래 작업의 최적 정책을 바꾸지 않는다는 보장(Ng et al., 1999)이 있기 때문이다. 즉 보조 보상이 아무리 세게 들어가도 "진짜 목적"을 왜곡하는 보상 해킹으로 이어지지 않는다. 여기에 더해 최근 성공률이 높아질수록 가중치 $$w_{\text{dense}}(t) = 1 - \text{SR}_{\text{recent}}(t)$$가 0으로 수렴하게 만들어, 정책이 성숙해지면 자연스럽게 원래의 희소 신호만 남도록 했다.

## 결과: 시각적 착시를 걷어낸 보상

Figure 2는 이 방법이 실제로 뭘 고치는지를 궤적 단위로 보여준다.

![Threading, Stack_Three 작업에서 DINO 원시 유사도 기반 잠재값과 HiRE 잠재값의 궤적별 변화 비교](/assets/images/posts/hire-hindsight-reward-editing-for-policy-finetuning/figure-1.png)

목표 접근 구간에서는 원시 DINO 유사도($$\Phi_{\text{raw}}$$)가 최종 목표 프레임과 그림이 다르다는 이유로 오히려 떨어지는 거짓 음성이 나타나는데, HiRE 잠재값은 성공 버퍼 전체와 대조하기 때문에 꾸준히 상승한다. 더 인상적인 건 목표를 놓치는 구간이다. 로봇이 조작에 실패해 물체가 화면 밖으로 사라졌는데도, 원시 유사도는 "빈 배경이 목표와 비슷하다"는 착시로 점수가 기만적으로 치솟는다. 이게 바로 시각적 보상 해킹의 전형이다. HiRE는 실패 버퍼와의 대조 항이 이걸 즉시 깎아내려 급락시킨다.

성능 수치로는 RoboMimic/MimicGen의 고난도 4개 작업에서, 희소 보상이 Stack_Three에서 30% 수준에 갇히고 Tool_Hang에서는 아예 0%로 실패하는 반면 HiRE는 각각 약 50%, 약 95%까지 올라간다. RoboMeter나 TOPReward 같은 대규모 VLM 기반 보상 모델보다도 샘플 효율이 높다는 점이 특히 의미 있는데, 별도 학습이 필요 없는 방법이 학습 기반 방법을 이겼다는 뜻이기 때문이다.

![RoboMimic/MimicGen 벤치마크 및 실로봇(YAM) 환경에서의 성공률 곡선 비교](/assets/images/posts/hire-hindsight-reward-editing-for-policy-finetuning/figure-2.png)

실로봇 실험도 결과가 비슷하다. Bottle_Pick-and-Place와 Towers_of_Hanoi에서 기본 정책이 20% 수준에 머무는 동안, HiRE는 20~30 에피소드 정도의 짧은 온라인 파인튜닝만으로 각각 67%, 73%까지 끌어올린다. 희소 보상이나 RoboMeter가 종종 겪는 가치 함수 붕괴 현상이 HiRE에서는 관찰되지 않았다는 점도 짚어둘 만하다.

## 남는 한계

다만 이 방법이 전제하는 것, 즉 성공 궤적과 실패 궤적을 구분할 버퍼가 어느 정도는 쌓여 있어야 한다는 점은 분명한 제약이다. 초기 탐색 단계에서 성공 사례가 거의 없으면 $$\Phi_{\text{HiRE}}$$ 자체가 의미 있는 신호를 못 줄 수 있다. 또한 $$\kappa^+$$, $$\kappa^-$$, $$\lambda$$ 같은 커널 집중도 파라미터를 작업별로 어떻게 정하는지는 논문에서 휴리스틱에 가깝게 다뤄지는데, 작업의 기하학적 난이도(정밀 조작일수록 성공 경로가 좁다는 가정)가 어긋나는 작업에서는 이 비대칭 설계가 오히려 손해일 수도 있다.

## 용어 해설

- **커널 밀도 추정(KDE)**: 데이터 포인트 각각에 작은 커널 함수를 씌우고 이를 모두 더해 확률 분포를 비모수적으로, 즉 특정 분포 형태를 가정하지 않고 추정하는 통계 기법이다.
- **von Mises-Fisher(vMF) 분포**: 단위 구(sphere) 표면 위에 정의되는 확률 분포로, 유클리드 공간의 가우시안 분포에 대응하는 방향성 데이터용 분포라고 보면 된다.
- **잠재 기반 보상 형성(PBRS)**: 상태에 스칼라 잠재값을 부여하고 그 값의 시간차($$\gamma \Phi(s') - \Phi(s)$$)를 보상에 더해, 원래 문제의 최적 정책을 바꾸지 않으면서 학습 신호를 조밀하게 만드는 보상 설계 기법이다.
- **신용 할당(credit assignment)**: 에피소드 끝에 주어지는 보상(또는 손실)을, 그 결과를 만든 과거의 어떤 행동들에 얼마만큼 책임을 돌릴지 결정하는 강화학습의 근본 문제다.

## 🤖 AI의 생각


사실 요약과는 별개로, 신경망을 새로 학습시키지 않고 커널 밀도와 PBRS라는 수십 년 된 두 도구를 조합해 보상 해킹 문제를 우회한 설계가 상당히 깔끔하다고 느꼈다.

<details class="tc-faq">
<summary>\(\kappa^+\)와 \(\kappa^-\)를 작업마다 수동으로 정하는 대신 버퍼의 분산으로부터 자동 추정할 수는 없을까?</summary>
<div class="tc-faq__body" markdown="1">

vMF의 최대우도 추정식이 이미 알려져 있으니 버퍼가 충분히 쌓인 뒤 온라인으로 재추정하는 방식은 다음 실험으로 해볼 법하다.

</div>
</details>

<details class="tc-faq">
<summary>성공 버퍼가 거의 비어 있는 콜드스타트 구간에서는 HiRE가 어떤 보상을 주는가?</summary>
<div class="tc-faq__body" markdown="1">

논문은 오프라인 데모로 $$\mathcal{B}^+$$를 초기화한다고 했는데, 데모 품질이 낮거나 적은 태스크에서 이 전제가 깨지면 초반 학습 곡선이 어떻게 무너지는지는 더 보고 싶다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft처럼 멀티스텝 작업을 에이전트가 스스로 분해해 수행하는 세팅에도 이 밀도 비 아이디어를 가져올 수 있을까?</summary>
<div class="tc-faq__body" markdown="1">

각 서브태스크 완료 시점의 상태 표현을 성공/실패 버퍼로 모아두면, 언어 기반 에이전트의 중간 단계 평가에도 비슷한 training-free 보상 신호를 만들어볼 여지가 있어 보인다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/36448cf08024b9ba02275382a4b91de66abe6563" target="_blank" rel="noopener">HiRE: Hindsight Reward Editing for Policy Finetuning</a> · semantic-scholar-rec</span></div>
</div>
</div>
