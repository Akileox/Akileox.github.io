---
title: "[이론] 2. Dreamer 계열"
date: 2026-07-22
categories: [Project]
tags: [taskcraft-theory, world-model, dreamer, reinforcement-learning, robotics]
excerpt: "1편의 KL 목적함수 하나만으로는 안정적으로 학습이 안 된다는 문제에서 출발해, RSSM 구조와 actor-critic objective로 잠재 공간 안에서 상상으로 정책을 학습하는 Dreamer 계열을 정리하고 actor가 왜 embodiment-specific 디코더인지 taskcraft 가설과 연결한다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 2편이다. 1편(World Model)에서 이어진다.</em></p>

## 시리즈 지도

<figure class="tp-fig">
<div class="tp-roadmap">
  <div class="tp-roadmap__stage tp-roadmap__stage--prereq">
    <div class="tp-roadmap__num">선수</div>
    <div class="tp-roadmap__label">이미 아는 것</div>
    <div class="tp-roadmap__items">RL 기초~PPO<br>IL(BC~GAIL)<br>VLA 기초, Diffusion Policy</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">00</div>
    <div class="tp-roadmap__label">예비</div>
    <div class="tp-roadmap__items">0. VAE와 생성 모델</div>
  </div>
  <div class="tp-roadmap__stage tp-roadmap__stage--current">
    <div class="tp-roadmap__num">01</div>
    <div class="tp-roadmap__label">기초</div>
    <div class="tp-roadmap__items">1. World Model<br>2. Dreamer 계열<br>3. MuZero</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">02</div>
    <div class="tp-roadmap__label">Latent Action</div>
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
  <div class="tp-roadmap__stage tp-roadmap__stage--dest">
    <div class="tp-roadmap__num">→</div>
    <div class="tp-roadmap__label">taskcraft 연구</div>
    <div class="tp-roadmap__items">사람 시연 + VIP 진행도 신호<br>→ Minecraft 이종 embodiment 이식</div>
  </div>
</div>
</figure>

## 1편의 목적함수만으로는 부족하다

1편은 world model을 "\\( z_t \\) 위에서 정의된 전이 모델"로 정의하고, KL 목적함수 하나를 제시했다.

$$
\mathcal L(\theta) = \mathbb E\Big[D_{KL}\big(q(z_t\mid o_t)\,\Vert\,p_\theta(z_t\mid z_{t-1},a_{t-1})\big)\Big]
$$

그런데 이 식만으로는 실제로 무엇을 학습해야 하는지가 불완전하다. \\( z_t \\)를 어떤 구조로 표현할지(벡터 하나? 여러 조각?), 정책은 이 latent를 어떻게 써야 하는지, 그리고 world model을 학습한 뒤 정책은 실제로 어떻게 개선하는지가 빠져 있다. World Models(1편)의 원래 방법은 이 세 조각(V·M·C)을 따로 학습했다. VAE를 먼저 오프라인으로 학습해 고정하고, 그 위에 MDN-RNN을 학습해 고정하고, 마지막에 컨트롤러를 진화 전략(CMA-ES)으로 학습한다. 컨트롤러가 학습되는 동안 표현과 전이 모델은 전혀 갱신되지 않는다.

## 핵심 아이디어: 잠재 공간 안에서 상상으로 정책을 학습한다

PlaNet(Hafner et al., 2019)이 다음 구조(RSSM, Recurrent State-Space Model)를 처음 도입했지만, 아직 명시적 정책망 없이 latent 위에서 CEM(cross-entropy method)으로 매 스텝 플래닝했다. Dreamer(Hafner et al., 2020)가 여기에 actor-critic을 얹어, 실제 환경을 더 돌리지 않고도 학습된 전이 모델 안에서 **상상(imagination)**으로 정책을 개선하는 절차를 완성한다.

RSSM은 상태를 결정론적 부분 \\( h_t \\)와 확률적 부분 \\( z_t \\)로 나눈다.

$$
h_t = f_\phi(h_{t-1}, z_{t-1}, a_{t-1}) \qquad \text{(결정론적 recurrent state)}
$$

$$
\hat z_t \sim p_\phi(\hat z_t \mid h_t) \qquad \text{(prior, 행동만으로 미리 예측)}
$$

$$
z_t \sim q_\phi(z_t \mid h_t, o_t) \qquad \text{(posterior, 실제 관측까지 본 뒤)}
$$

\\( h_t \\)가 장기 기억(RNN)을 맡고, \\( z_t \\)가 그 시점의 불확실성을 담는다. 1편의 \\( z_t \\)는 사실 이 \\( (h_t,z_t) \\) 쌍을 뭉뚱그린 표기였다.

<figure class="tp-fig">
<svg viewBox="0 0 760 320" role="img" aria-label="h_t-1, z_t-1, a_t-1이 f_phi를 거쳐 h_t가 되고, h_t가 prior와 posterior 두 갈래로 갈라진다. o_t는 posterior에만 입력된다. 아래 강조 상자: actor가 h_t, z_t를 그대로 받아 행동을 출력한다.">
  <defs>
    <marker id="d2Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="d2ArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="220" y1="82" x2="288" y2="82" marker-end="url(#d2Arrow)"/>
    <line x1="460" y1="68" x2="538" y2="40" marker-end="url(#d2Arrow)"/>
    <line x1="460" y1="98" x2="538" y2="122" marker-end="url(#d2Arrow)"/>
    <line x1="635" y1="150" x2="635" y2="163" marker-end="url(#d2Arrow)"/>
  </g>
  <g fill="none" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3" opacity="0.7">
    <line x1="670" y1="42" x2="670" y2="93"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="375" y1="123" x2="375" y2="215" marker-end="url(#d2ArrowAccent)"/>
  </g>

  <rect x="30" y="45" width="190" height="75" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="125" y="76" text-anchor="middle" font-size="12" fill="currentColor">h_t-1, z_t-1,</text>
  <text x="125" y="96" text-anchor="middle" font-size="13" fill="currentColor">a_t-1</text>

  <rect x="290" y="45" width="170" height="75" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="375" y="76" text-anchor="middle" font-size="12" fill="currentColor">f_φ</text>
  <text x="375" y="96" text-anchor="middle" font-size="13" fill="currentColor">→ h_t</text>

  <rect x="540" y="15" width="190" height="55" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="635" y="36" text-anchor="middle" font-size="11.5" fill="currentColor">prior</text>
  <text x="635" y="54" text-anchor="middle" font-size="12.5" fill="currentColor">p(z_t | h_t)</text>

  <rect x="540" y="95" width="190" height="55" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="635" y="116" text-anchor="middle" font-size="11.5" fill="currentColor">posterior</text>
  <text x="635" y="134" text-anchor="middle" font-size="12" fill="currentColor">q(z_t | h_t, o_t)</text>

  <rect x="560" y="163" width="150" height="42" rx="8" fill="var(--bg-secondary)" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3"/>
  <text x="635" y="189" text-anchor="middle" font-size="12" style="fill:var(--text-muted)">o_t</text>

  <text x="655" y="70" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">KL</text>

  <rect x="30" y="215" width="700" height="80" rx="8" style="fill:var(--accent-light);stroke:var(--accent)" stroke-width="1.6"/>
  <text x="380" y="243" text-anchor="middle" font-size="12.5" style="fill:var(--accent)" font-weight="700">actor π_φ(a_t | h_t, z_t)도 embodiment-specific 디코더</text>
  <text x="380" y="263" text-anchor="middle" font-size="12" style="fill:var(--accent)">z_t가 신체 정보를 담고 있어도, actor는 그걸 버릴 이유가 없다</text>
  <text x="380" y="281" text-anchor="middle" font-size="11" style="fill:var(--accent)">오히려 있으면 더 정확한 정책을 만들 수 있어 유리하다</text>
</svg>
<figcaption>h_t-1, z_t-1, a_t-1이 f_φ를 거쳐 h_t가 되고, h_t는 prior와 posterior 두 갈래로 갈라진다. posterior에만 실제 관측 o_t가 들어간다(점선 상자). 진짜 쟁점은 아래 강조 상자다. actor가 h_t, z_t를 그대로 입력받아 행동을 출력하는 이상, z_t에 embodiment 정보가 남아있을 유인이 하나 더 늘어난다.</figcaption>
</figure>

## 필요한 만큼만 수학: 학습 목적함수

학습 목적함수는 1편의 KL 항에 복원·보상 예측 항을 더한 것이다.

$$
\mathcal L(\phi) = \mathbb E_q\Big[\sum_t \ln p_\phi(o_t\mid h_t,z_t) + \ln p_\phi(r_t\mid h_t,z_t) - \beta\, D_{KL}\big(q_\phi(z_t\mid h_t,o_t)\,\Vert\,p_\phi(z_t\mid h_t)\big)\Big]
$$

세 항 각각의 역할이다. 첫 항(복원)은 \\( (h_t,z_t) \\)가 관측을 설명할 수 있게, 둘째 항(보상)은 태스크에 필요한 정보를 담게, 셋째 항(KL)은 prior가 posterior를 따라가게 만든다. 정책은 상상된 궤적으로 학습한다. 학습된 dynamics로 \\( h_1,\dots,h_H \\)를 실제 환경 스텝 없이 굴리고, \\( \lambda \\)-return으로 가치를 추정한다.

$$
V_\lambda(s_\tau) = r_\tau + \gamma\Big[(1-\lambda)\,v_\psi(s_{\tau+1}) + \lambda\, V_\lambda(s_{\tau+1})\Big], \qquad V_\lambda(s_H) = v_\psi(s_H)
$$

actor \\( \pi_\phi(a_t\mid h_t,z_t) \\)는 이 \\( V_\lambda \\)를 최대화하도록 학습된다. dynamics가 미분 가능하므로 연속 행동에서는 그래디언트를 dynamics를 관통해 직접 흘려보낼 수 있고, 이산 행동(DreamerV2 이후)에서는 straight-through와 REINFORCE를 섞어 쓴다.

## 논문에서는: DreamerV2, V3로의 확장

DreamerV2는 \\( z_t \\)를 연속 가우시안 대신 이산 범주형(보통 32개 범주 × 32개 클래스)으로 바꾼다. 이산 변수는 샘플링 연산 자체가 미분 불가능해서, 0편의 재파라미터화 트릭이 그대로는 안 통한다. <span class="aside">(카테고리 하나를 뽑는 연산에는 그래디언트가 흐를 경로가 없다.)</span> **straight-through 추정기**로 우회한다. 순전파 때는 실제 이산 샘플(one-hot)을 그대로 쓰고, 역전파 때는 마치 연속 완화를 미분한 것처럼 그래디언트를 통과시킨다. 편향(bias)이 있는 근사지만 실전에서 잘 작동한다.

또한 posterior가 너무 빨리 collapse하는 것을 막는 **KL balancing**을 쓴다. KL 항을 그대로 두면 posterior가 자기 자신을 prior 쪽으로 붕괴시켜(정보를 버려서) KL을 손쉽게 줄여버리는 지름길이 생긴다. prior 쪽 항의 가중치를 posterior 쪽보다 크게 줘서(보통 0.8:0.2), prior가 posterior를 따라가는 속도를 posterior가 붕괴하는 속도보다 빠르게 만든다.

DreamerV3는 보상·가치 스케일이 도메인마다 천차만별인 문제를 \\( \mathrm{symlog}(x) = \mathrm{sign}(x)\,\ln(1+|x|) \\) 변환 하나로 정규화해서, 하이퍼파라미터를 도메인마다 재튜닝하지 않고도(Atari, DMC, Minecraft 등) 같은 설정을 그대로 쓴다. Minecraft에서 사람 데이터 없이 다이아몬드를 캐낸 결과가 이 일반성의 대표 사례다.

## taskcraft와의 접목

actor \\( \pi_\phi(a_t\mid h_t,z_t) \\)가 정확히 "공유 trunk + embodiment별 얕은 헤드" 패턴의 가장 단순한 사례다. \\( (h_t,z_t) \\)가 trunk, actor가 그 위에 얹힌 얕은 헤드다. 그런데 Dreamer는 이 trunk를 단일 환경, 단일 embodiment의 관측만으로 학습한다. trunk가 여러 embodiment에 걸쳐 재사용 가능한가는 애초에 질문된 적이 없다.

여기서 1편의 minimality 문제가 더 구체적인 형태로 나타난다. actor는 \\( (h_t,z_t) \\) 전체를 입력받는다. \\( z_t \\)가 embodiment 정보를 담고 있어도 actor 입장에서는 버릴 이유가 없다. 오히려 그 정보가 있으면 "내 팔이 닿는 범위"처럼 자신의 embodiment에 특화된 더 정확한 정책을 만들 수 있어 유리하다. 즉 actor의 존재 자체가 \\( z_t \\)에 embodiment 정보가 남아있을 유인을 하나 더 추가한다.

DreamerV3가 하나의 고정된 하이퍼파라미터 집합으로 Minecraft를 포함한 여러 도메인을 푼다는 것을 보인 건, taskcraft가 Minecraft를 통제 가능하고 비용이 싼 도구로 고른 선택이 world model 관점에서도 합리적이라는 방증이다. 다만 DreamerV3의 "여러 도메인"은 여러 embodiment가 아니라 여러 환경이다. 각 도메인 안에서는 여전히 단일 embodiment, 단일 관측 분포로 학습한다.

## 한계 / 아직 안 풀린 문제

- Dreamer 계열은 \\( (h_t,z_t) \\)의 embodiment invariance를 검증한 적이 없다. 단일 환경 안에서 성능만 측정한다.
- 이산 latent(DreamerV2/V3)의 codebook 크기가 "얼마나 압축하는가"를 결정하는 또 다른 하이퍼파라미터다. 3편(MuZero)과 4편(Genie)에서 이 압축 정도를 다루는 다른 방식을 본다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| PlaNet (Hafner et al., 2019) | RSSM 최초 도입, CEM 플래닝. 명시적 정책망 없음 |
| Dreamer (Hafner et al., 2020) | RSSM + actor-critic latent imagination. 이 편의 핵심 대상 |
| DreamerV2 (Hafner et al., 2021) | 이산 categorical latent, straight-through, KL balancing |
| DreamerV3 (Hafner et al., 2023) | symlog, free bits, unimix. 도메인 무관 고정 하이퍼파라미터 |

## 다음 글로 넘어가기 전에

- Dreamer가 1편의 문제를 어떻게 풀었나: RSSM(h_t, z_t 분리)으로 구조를 구체화하고, actor-critic으로 실제 환경 스텝 없이 상상 속에서 정책을 개선하는 절차를 완성했다.
- 그런데 여전히 안 풀린 것: 복원 손실(observation reconstruction)이 목적함수에 그대로 남아있다. 픽셀을 복원하려는 압박이 embodiment 정보를 z에 남기는 유인 중 하나라는 게 1편의 논증이었다.

**복원 손실 자체를 목적함수에서 빼버리면 이 유인이 사라질까?** 3편(MuZero)이 정확히 이 질문에서 시작한다.

<style>
.post-series-note { color: var(--text-muted); font-size: 0.9rem; }
.tp-fig svg { width: 100%; height: auto; display: block; background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 10px; }
.tp-fig figcaption { font-size: 0.82rem; color: var(--text-muted); margin-top: 0.6rem; line-height: 1.6; }
.tp-roadmap {
  display: grid;
  grid-template-columns: repeat(8, 1fr);
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
.tp-roadmap__stage--prereq { border-style: dashed; opacity: 0.75; }
.tp-roadmap__stage--dest { border-style: dashed; border-color: var(--accent); }
.tp-roadmap__stage--dest .tp-roadmap__num, .tp-roadmap__stage--dest .tp-roadmap__label { color: var(--accent); }
.tp-roadmap__num { font-family: "SFMono-Regular", Consolas, monospace; font-size: 0.7rem; color: var(--text-muted); margin-bottom: 0.4rem; }
.tp-roadmap__stage--current .tp-roadmap__num { color: var(--accent); font-weight: 700; }
.tp-roadmap__label { font-size: 0.8rem; font-weight: 600; line-height: 1.3; margin-bottom: 0.5rem; }
.tp-roadmap__items { font-size: 0.72rem; color: var(--text-muted); line-height: 1.7; }
@media (max-width: 760px) {
  .tp-roadmap { grid-template-columns: 1fr; }
  .tp-roadmap__stage + .tp-roadmap__stage { border-left: 1px solid var(--border); border-top: none; }
}
</style>
