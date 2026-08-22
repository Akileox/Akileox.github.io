---
title: "[이론] 2. Dreamer 계열 — Latent에서 상상으로 배운다"
date: 2026-08-15
categories: [Project]
tags: [taskcraft-theory, world-model, dreamer, reinforcement-learning, robotics]
excerpt: "PlaNet의 RSSM부터 DreamerV2의 이산 latent, DreamerV3의 symlog까지, 잠재 공간 안에서 상상으로 정책을 학습하는 구조를 수식으로 짚고 actor가 왜 embodiment-specific 디코더인지 taskcraft 가설과 연결합니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 2편입니다. 1편(World Model)에서 이어집니다.</em></p>

## 정의

1편의 전이 모델 \\( p_\theta(z_{t+1}\mid z_t,a_t) \\)을 실제로 학습 가능한 형태로 구체화한 것이 **RSSM**(Recurrent State-Space Model)입니다. 상태를 결정론적 부분 \\( h_t \\)와 확률적 부분 \\( z_t \\)로 나눕니다.

$$
h_t = f_\phi(h_{t-1}, z_{t-1}, a_{t-1}) \qquad \text{(결정론적 recurrent state)}
$$

$$
\hat z_t \sim p_\phi(\hat z_t \mid h_t) \qquad \text{(prior, 행동만으로 미리 예측)}
$$

$$
z_t \sim q_\phi(z_t \mid h_t, o_t) \qquad \text{(posterior, 실제 관측까지 본 뒤)}
$$

\\( h_t \\)가 장기 기억(RNN)을 맡고, \\( z_t \\)가 그 시점의 불확실성을 담습니다. 1편의 \\( z_t \\)는 사실 이 \\( (h_t, z_t) \\) 쌍을 뭉뚱그린 표기였습니다.

## 이 개념이 풀고자 했던 문제

World Models(1편)은 V(VAE)·M(MDN-RNN)·C(컨트롤러) 세 컴포넌트를 따로 학습했습니다. VAE를 먼저 오프라인으로 학습해 고정하고, 그 위에 MDN-RNN을 학습해 고정하고, 마지막에 컨트롤러를 진화 전략(CMA-ES)으로 학습합니다. 컨트롤러가 학습되는 동안 표현과 전이 모델은 전혀 갱신되지 않습니다.

PlaNet(Hafner et al., 2019)이 RSSM을 처음 도입했지만, 아직 명시적 정책망 없이 latent 위에서 CEM(cross-entropy method)으로 매 스텝 플래닝했습니다. Dreamer(Hafner et al., 2020)가 여기에 actor-critic을 얹어, 실제 환경을 더 돌리지 않고도 학습된 전이 모델 안에서 상상(imagination)으로 정책을 개선하는 절차를 완성합니다.

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
<figcaption><strong>이 그림이 보여주는 것.</strong> h_t-1, z_t-1, a_t-1이 f_φ를 거쳐 h_t가 되고, h_t는 prior와 posterior 두 갈래로 갈라집니다. posterior에만 실제 관측 o_t가 들어갑니다(점선 상자). <strong>진짜 쟁점은 아래 강조 상자</strong>입니다. actor가 h_t, z_t를 그대로 입력받아 행동을 출력하는 이상, z_t에 embodiment 정보가 남아있을 유인이 하나 더 늘어납니다.</figcaption>
</figure>

## 핵심 구조와 목적함수

학습 목적함수는 1편의 KL 항에 복원·보상 예측 항을 더한 것입니다.

$$
\mathcal L(\phi) = \mathbb E_q\Big[\sum_t \ln p_\phi(o_t\mid h_t,z_t) + \ln p_\phi(r_t\mid h_t,z_t) - \beta\, D_{KL}\big(q_\phi(z_t\mid h_t,o_t)\,\Vert\,p_\phi(z_t\mid h_t)\big)\Big]
$$

세 항 각각의 역할입니다. 첫 항(복원)은 \\( (h_t,z_t) \\)가 관측을 설명할 수 있게, 둘째 항(보상)은 태스크에 필요한 정보를 담게, 셋째 항(KL)은 prior가 posterior를 따라가게 만듭니다.

DreamerV2는 \\( z_t \\)를 연속 가우시안 대신 이산 범주형(보통 32개 범주 × 32개 클래스)으로 바꿉니다. 샘플링이 미분 불가능해지는 문제는 straight-through 추정기(순전파는 실제 샘플, 역전파는 연속 완화 분포의 그래디언트를 그대로 흘려보냄)로 우회합니다. 또한 posterior가 너무 빨리 collapse하는 것을 막기 위해 KL balancing을 씁니다.

$$
L_{KL} = \alpha\, D_{KL}\big(\mathrm{sg}[q]\,\Vert\,p\big) + (1-\alpha)\, D_{KL}\big(q\,\Vert\,\mathrm{sg}[p]\big), \qquad \alpha \approx 0.8
$$

\\( \mathrm{sg} \\)는 stop-gradient. prior 쪽 항의 가중치를 더 크게 줘서, prior가 posterior를 따라잡는 속도를 posterior가 prior에 맞춰 붕괴하는 속도보다 빠르게 만듭니다.

DreamerV3는 보상·가치 스케일이 도메인마다 천차만별인 문제를 symlog 변환으로 정규화합니다.

$$
\mathrm{symlog}(x) = \mathrm{sign}(x)\,\ln(1+|x|)
$$

예측은 항상 symlog 공간에서 하고, 역변환(symexp)으로 원래 스케일로 되돌립니다. 이 변환 하나로 하이퍼파라미터를 도메인마다 재튜닝하지 않고도(Atari, DMC, Minecraft 등) 같은 설정을 그대로 씁니다. Minecraft에서 사람 데이터 없이 다이아몬드를 캐낸 결과가 이 일반성의 대표 사례입니다.

정책은 상상된 궤적으로 학습합니다. 학습된 dynamics로 \\( h_1,\dots,h_H \\)를 실제 환경 스텝 없이 굴리고, \\( \lambda \\)-return으로 가치를 추정합니다.

$$
V_\lambda(s_\tau) = r_\tau + \gamma\Big[(1-\lambda)\,v_\psi(s_{\tau+1}) + \lambda\, V_\lambda(s_{\tau+1})\Big], \qquad V_\lambda(s_H) = v_\psi(s_H)
$$

actor \\( \pi_\phi(a_t\mid h_t,z_t) \\)는 이 \\( V_\lambda \\)를 최대화하도록 학습됩니다. dynamics가 미분 가능하므로 연속 행동에서는 그래디언트를 dynamics를 관통해 직접 흘려보낼 수 있고, 이산 행동(DreamerV2 이후)에서는 straight-through와 REINFORCE를 섞어 씁니다.

## taskcraft 아이디어와의 접목

> actor \\( \pi_\phi(a_t\mid h_t,z_t) \\)가 정확히 position_paper.md가 정리한 "공유 trunk + embodiment별 얕은 헤드" 패턴의 가장 단순한 사례입니다. \\( (h_t,z_t) \\)가 trunk, actor가 그 위에 얹힌 얕은 헤드입니다. 그런데 Dreamer는 이 trunk를 단일 환경, 단일 embodiment의 관측만으로 학습합니다. trunk가 여러 embodiment에 걸쳐 재사용 가능한가는 애초에 질문된 적이 없습니다.
>
> 여기서 1편의 minimality 문제가 더 구체적인 형태로 나타납니다. actor는 \\( (h_t,z_t) \\) 전체를 입력받습니다. \\( z_t \\)가 embodiment 정보를 담고 있어도 actor 입장에서는 버릴 이유가 없습니다. 오히려 그 정보가 있으면 "내 팔이 닿는 범위"처럼 자신의 embodiment에 특화된 더 정확한 정책을 만들 수 있어 유리합니다. 즉 actor의 존재 자체가 \\( z_t \\)에 embodiment 정보가 남아있을 유인을 하나 더 추가합니다.
>
> DreamerV3가 하나의 고정된 하이퍼파라미터 집합으로 Minecraft를 포함한 여러 도메인을 푼다는 것을 보인 건, taskcraft가 Minecraft를 통제 가능하고 비용이 싼 도구로 고른 선택이 world model 관점에서도 합리적이라는 방증입니다. 다만 DreamerV3의 "여러 도메인"은 여러 embodiment가 아니라 여러 환경입니다. 각 도메인 안에서는 여전히 단일 embodiment, 단일 관측 분포로 학습합니다.

## 한계 / 아직 안 풀린 문제

- Dreamer 계열은 \\( (h_t,z_t) \\)의 embodiment invariance를 검증한 적이 없습니다. 단일 환경 안에서 성능만 측정합니다.
- 이산 latent(DreamerV2/V3)의 codebook 크기가 "얼마나 압축하는가"를 결정하는 또 다른 하이퍼파라미터입니다. 4편(Genie)에서 이 압축 정도를 극단으로 밀어붙인 사례를 다룹니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| PlaNet (Hafner et al., 2019) | RSSM 최초 도입, CEM 플래닝. 명시적 정책망 없음 |
| Dreamer (Hafner et al., 2020) | RSSM + actor-critic latent imagination. 이 편의 핵심 대상 |
| DreamerV2 (Hafner et al., 2021) | 이산 categorical latent, straight-through, KL balancing |
| DreamerV3 (Hafner et al., 2023) | symlog, free bits, unimix — 도메인 무관 고정 하이퍼파라미터 |

## 다음 글

3편은 MuZero입니다. Dreamer는 픽셀 복원 손실을 그대로 유지하지만, MuZero는 복원을 아예 목적함수에서 뺍니다. 복원 압박이 없으면 embodiment 정보가 새는 유인도 줄어드는가가 3편의 질문입니다.

<style>
.post-series-note { color: var(--text-muted); font-size: 0.9rem; }
.tp-fig svg { width: 100%; height: auto; display: block; background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 10px; }
.tp-fig figcaption { font-size: 0.82rem; color: var(--text-muted); margin-top: 0.6rem; line-height: 1.6; }
</style>
