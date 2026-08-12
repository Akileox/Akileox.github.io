---
title: "[이론] 1. World Model이란 무엇인가"
date: 2026-08-12
categories: [Project]
tags: [taskcraft-theory, world-model, reinforcement-learning, robotics]
excerpt: "관측을 압축해 다음 상태를 예측하는 world model의 정의를 Dyna의 model-based RL부터 Ha & Schmidhuber(2018)의 MDN-RNN, taskcraft 가설과의 정보이론적 연결까지 수식으로 짚습니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 1편입니다. DATA403(학부 강화학습) 수준의 배경(정책, 가치함수, MDP)은 전제하고, world model에서 새로 추가되는 개념만 다룹니다.</em></p>

## 정의

환경의 실제 상태 \\( s_t \\)를 에이전트는 직접 보지 못하고, 관측 \\( o_t \\)(카메라 프레임 등)만 받는다고 합시다(부분관측 MDP, POMDP). World model은 두 개의 학습된 함수로 이뤄집니다.

- **인코더**(근사 사후분포): \\( z_t \sim q_\phi(z_t \mid o_t) \\) — 관측을 잠재 상태로 압축
- **전이 모델**(사전분포): \\( \hat z_{t+1} \sim p_\theta(z_{t+1} \mid z_t, a_t) \\) — 실제 \\( o_{t+1} \\)을 보지 않고 행동만으로 다음 잠재 상태를 예측

디코더 \\( \hat o_t \sim p_\psi(o_t \mid z_t) \\)는 선택 사항입니다 — 있으면 인코더가 관측을 복원할 만큼 정보를 담았는지 확인하는 학습 신호로 씁니다.

## 시리즈 지도

이 시리즈는 다섯 단계로 taskcraft의 가설로 모입니다. 1편(이 글)에서 정의를 잡고, 2~3편이 그 정의를 학습 가능하게 만드는 두 갈래(Dreamer, MuZero)를 다루고, 4편(Genie)이 "라벨 없이 잠재 행동을 뽑는다"는 도약을 만듭니다. 거기서 사람 영상 표현(5~6편)과 실전 로봇 적용·대안(7~11편)이 갈라져 나왔다가, 12편에서 다시 taskcraft 가설로 합쳐집니다.

<figure class="tp-fig">
<div class="tp-roadmap">
  <div class="tp-roadmap__stage tp-roadmap__stage--current">
    <div class="tp-roadmap__num">01</div>
    <div class="tp-roadmap__label">기초</div>
    <div class="tp-roadmap__items">1. World Model<br>2. Dreamer 계열<br>3. MuZero</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">02</div>
    <div class="tp-roadmap__label">잠재 행동</div>
    <div class="tp-roadmap__items">4. Genie</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">03</div>
    <div class="tp-roadmap__label">사람 → 로봇 표현</div>
    <div class="tp-roadmap__items">5. R3M<br>6. VIP</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">04</div>
    <div class="tp-roadmap__label">실전 적용 / 대안</div>
    <div class="tp-roadmap__items">7. DreamGen<br>8. GR-1→GR-2<br>9. Genie Envisioner<br>10. NerveNet<br>11. MetaMorph</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">05</div>
    <div class="tp-roadmap__label">종합 / 실행</div>
    <div class="tp-roadmap__items">12. taskcraft 가설 종합<br>13. 파일럿 설계</div>
  </div>
</div>
<figcaption><strong>이 그림이 보여주는 것.</strong> 13편이 개별 논문 나열이 아니라 정의 → 학습법 → 잠재 행동 추출 → 사람 표현 이식 → 실전/대안 → 종합·실행이라는 하나의 논증 순서로 이어진다는 것. 강조된 1단계가 지금 이 글입니다.</figcaption>
</figure>

## 이 개념이 풀고자 했던 문제

Dyna(Sutton, 1990)가 원형입니다. 실제 전이로부터 \\( T(s'\mid s,a) \\), \\( R(s,a) \\)를 학습하고, 학습된 모델로 만든 가상 전이를 실제 전이와 섞어 같은 Q-learning 업데이트에 씁니다.

$$
Q(s,a) \leftarrow Q(s,a) + \alpha\left[r + \gamma \max_{a'} Q(s',a') - Q(s,a)\right]
$$

이 업데이트를 실제 경험과 모델이 만든 "상상된" 경험 둘 다에 적용합니다 — 학습량이 실제 환경 스텝 수에 묶이지 않게 됩니다. 다만 여기서 \\( s \\)는 압축되지 않은 raw 상태입니다. World Models(Ha & Schmidhuber, 2018)는 이 \\( s \\) 자리를 **학습된 압축 표현** \\( z \\)로 바꾸고, 전이 모델 \\( T \\)를 확률적·다봉분포로 만듭니다.

## 핵심 구조와 목적함수

인코더와 전이 모델을 함께 묶는 일반적인 변분 목적함수입니다(PlaNet/Dreamer 계열의 공통 골격 — 구체적 구현은 2편에서 다룹니다).

$$
\mathcal{L}(\phi,\theta,\psi) = \mathbb{E}_{q_\phi}\Big[\underbrace{\log p_\psi(o_t\mid z_t)}_{\text{복원}} \;-\; \beta\, \underbrace{D_{KL}\big(q_\phi(z_t\mid o_t)\,\Vert\,p_\theta(z_t\mid z_{t-1},a_{t-1})\big)}_{\text{예측 정확도}}\Big]
$$

첫 항(복원)은 \\( z_t \\)가 \\( o_t \\)를 복원할 만큼 정보를 담았는지 확인합니다. KL항은 실제 관측을 본 \\( z_t \\)(posterior)와 행동만으로 미리 예측한 \\( z_t \\)(prior = 전이 모델의 출력)가 얼마나 가까운지를 잽니다 — 이게 "전이 모델이 얼마나 잘 맞히는가"를 학습 신호로 바꾸는 방법입니다.

<figure class="tp-fig">
<svg viewBox="0 0 700 300" role="img" aria-label="관측이 인코더를 거쳐 잠재 상태로 압축되고, 전이 모델이 행동을 받아 다음 상태를 예측하는 구조. 잠재 상태에서 갈라져 나온 질문: 이 압축이 신체 정보까지 남기는지는 미해결이다.">
  <defs>
    <marker id="tpArrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="tpArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>
  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="100" y1="125" x2="148" y2="125" marker-end="url(#tpArrow)"/>
    <line x1="260" y1="125" x2="308" y2="125" marker-end="url(#tpArrow)"/>
    <line x1="400" y1="125" x2="448" y2="125" marker-end="url(#tpArrow)"/>
    <line x1="560" y1="125" x2="608" y2="125" marker-end="url(#tpArrow)"/>
    <line x1="505" y1="205" x2="505" y2="157" marker-end="url(#tpArrow)"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="345" y1="156" x2="345" y2="205" marker-end="url(#tpArrowAccent)"/>
  </g>
  <g>
    <rect x="10" y="95" width="90" height="60" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="55" y="121" text-anchor="middle" font-size="12.5" fill="currentColor">관측</text>
    <text x="55" y="139" text-anchor="middle" font-size="12.5" fill="currentColor">o_t</text>

    <rect x="150" y="95" width="110" height="60" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="205" y="130" text-anchor="middle" font-size="13" fill="currentColor">인코더 q_φ</text>

    <rect x="310" y="95" width="90" height="60" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.6"/>
    <text x="355" y="121" text-anchor="middle" font-size="12.5" fill="currentColor">잠재 상태</text>
    <text x="355" y="139" text-anchor="middle" font-size="12.5" fill="currentColor">z_t</text>

    <rect x="450" y="95" width="110" height="60" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="505" y="130" text-anchor="middle" font-size="13" fill="currentColor">전이 모델 p_θ</text>

    <rect x="610" y="95" width="80" height="60" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="650" y="121" text-anchor="middle" font-size="11.5" fill="currentColor">예측된</text>
    <text x="650" y="139" text-anchor="middle" font-size="11.5" fill="currentColor">z_t+1</text>

    <rect x="450" y="205" width="110" height="42" rx="8" fill="var(--bg-secondary)" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3"/>
    <text x="505" y="231" text-anchor="middle" font-size="12" style="fill:var(--text-muted)">행동 a_t</text>

    <rect x="235" y="205" width="220" height="78" rx="8" style="fill:var(--accent-light);stroke:var(--accent)" stroke-width="1.6"/>
    <text x="345" y="228" text-anchor="middle" font-size="12" style="fill:var(--accent)" font-weight="700">z_t가 "누가 이 변화를"도</text>
    <text x="345" y="245" text-anchor="middle" font-size="12" style="fill:var(--accent)" font-weight="700">같이 담고 있는가?</text>
    <text x="345" y="266" text-anchor="middle" font-size="11" style="fill:var(--accent)">미해결 — 12편에서 다시 다룸</text>
  </g>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> 관측이 인코더를 거쳐 \( z_t \)로 압축되고(윗줄), 전이 모델이 행동 \( a_t \)를 받아 다음 상태를 예측합니다. <strong>진짜 쟁점은 아랫줄의 초록 상자</strong>입니다 — \( z_t \)가 "나무가 잘리고 있다"는 태스크 신호만 남기는지, "사람 팔이 그렇게 만들었다"는 신체 정보까지 같이 남기는지는 이 구조만으로는 알 수 없습니다.</figcaption>
</figure>

Ha & Schmidhuber(2018)의 구체적 선택은 **MDN-RNN**(Mixture Density Network + RNN)입니다 — 다음 상태를 점 하나가 아니라 가우시안 혼합분포로 예측합니다.

$$
p_\theta(z_{t+1}\mid h_t) = \sum_{k=1}^K \pi_k(h_t)\,\mathcal N\big(z_{t+1};\,\mu_k(h_t),\,\sigma_k^2(h_t)\big), \qquad h_t=\mathrm{RNN}(h_{t-1},z_t,a_t)
$$

왜 혼합분포인가 — 미래가 다봉분포일 수 있기 때문입니다(갈림길에서 왼쪽으로 갈지 오른쪽으로 갈지). 벡터 하나로 하는 점 추정으로는 이런 다봉성을 표현할 수 없습니다.

## taskcraft 아이디어와의 접목

> 이 시리즈의 핵심 절입니다. 위 목적함수의 복원항은 \\( z_t \\)가 \\( o_{t+1} \\) 예측에 충분한 정보(sufficient statistic)를 담게만 강제합니다 — 대략 \\( I(z_t;o_{t+1}\mid a_t) \\)를 높이는 방향입니다. 하지만 \\( z_t \\)가 "필요한 만큼만" 담아야 한다는 **minimality** 제약은 이 손실함수 어디에도 없습니다. 신체를 특정하는 정보(팔 모양, 관절각)가 다음 프레임 예측에 도움이 된다면 — 대체로 도움이 됩니다, 픽셀 수준 재구성에선 "누가 움직였는가"가 실제로 유용한 신호이므로 — 그 정보는 그대로 남을 유인이 있습니다.
>
> $$
> \underbrace{I(z_t;o_{t+1}\mid a_t)}_{\text{최대화하도록 학습됨}} \qquad\text{vs.}\qquad \underbrace{I(z_t;\text{embodiment})}_{\text{최소화하도록는 학습되지 않음}}
> $$
>
> 명시적으로 강제하려면 Information Bottleneck(Tishby et al.) 스타일로 \\( I(z_t;o_t) \\) 자체에 패널티를 주는 항이 필요합니다(β-VAE의 \\( \beta \\)가 이 역할의 조잡한 근사입니다). 표준 world model 목적함수엔 "무엇을 embodiment 기준으로 버릴지" 명시하는 항이 없습니다 — position_paper.md 5절 첫 질문("world model latent가 정말 embodiment에 덜 종속적인지")이 이론적으로 자명하지 않은 정확한 이유입니다.

## 한계 / 아직 안 풀린 문제

- 위 정보이론적 논증은 "왜 걱정해야 하는가"에 대한 이유이지, "실제로 얼마나 새는가"에 대한 답은 아닙니다 — linear probe(추출된 \\( z_t \\) 위에 간단한 선형 분류기를 얹어 embodiment를 얼마나 잘 맞히는지 재는 방법)로 실증 측정이 필요합니다.
- MDN-RNN의 혼합 성분 수 \\( K \\), \\( \beta \\) 값 등은 전부 하이퍼파라미터라 "얼마나 압축하는가"의 정도가 논문마다 다릅니다 — 2편(Dreamer)에서 이 설계가 실제로 어떻게 달라지는지 다룹니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| World Models (Ha & Schmidhuber, 2018) | V(VAE)+M(MDN-RNN)+C(CMA-ES 선형 컨트롤러) 구조 — "world model" 용어의 기준점 |
| Dyna (Sutton, 1990) | model-based RL 고전 — raw state 기반 모델 + 상상 경험 mixing |
| Information Bottleneck (Tishby et al.) | \\( I(z_t;o_t) \\)에 명시적 패널티를 주는 프레이밍 — taskcraft 접목 절의 근거 |

## 다음 글

2편은 이 인코더-전이 모델 루프를 실제로 학습 가능하게 만든 Dreamer 계열(PlaNet → Dreamer/V2/V3)입니다 — RSSM(Recurrent State Space Model) 구조와 "잠재 공간 안에서 상상으로 정책을 학습한다"는 절차를 구체적 수식으로 다룹니다.

<style>
.post-series-note { color: var(--text-muted); font-size: 0.9rem; }
.tp-roadmap {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  margin: 0.5rem 0;
  overflow-x: auto;
}
.tp-roadmap__stage {
  border: 1px solid var(--border);
  padding: 0.9rem 0.85rem;
  min-width: 150px;
  background: var(--bg-secondary);
}
.tp-roadmap__stage + .tp-roadmap__stage { border-left: none; }
.tp-roadmap__stage--current { background: var(--accent-light); border-color: var(--accent); }
.tp-roadmap__num { font-family: "SFMono-Regular", Consolas, monospace; font-size: 0.7rem; color: var(--text-muted); margin-bottom: 0.4rem; }
.tp-roadmap__stage--current .tp-roadmap__num { color: var(--accent); font-weight: 700; }
.tp-roadmap__label { font-size: 0.8rem; font-weight: 600; line-height: 1.3; margin-bottom: 0.5rem; }
.tp-roadmap__items { font-size: 0.72rem; color: var(--text-muted); line-height: 1.7; }
.tp-fig svg { width: 100%; height: auto; display: block; background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 10px; }
.tp-fig figcaption { font-size: 0.82rem; color: var(--text-muted); margin-top: 0.6rem; line-height: 1.6; }
@media (max-width: 760px) {
  .tp-roadmap { grid-template-columns: 1fr; }
  .tp-roadmap__stage + .tp-roadmap__stage { border-left: 1px solid var(--border); border-top: none; }
}
</style>
