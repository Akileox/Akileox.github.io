---
title: "[이론] 3. MuZero — 복원 없이 가치만 맞힌다"
date: 2026-08-18
categories: [Project]
tags: [taskcraft-theory, world-model, muzero, reinforcement-learning, robotics]
excerpt: "MuZero가 관측 복원 손실을 아예 목적함수에서 뺀 구조(representation/dynamics/prediction 세 함수)를 수식으로 짚고, 이게 1편의 minimality 문제에 대한 첫 반례 후보인 이유와 남은 한계를 taskcraft 가설과 연결합니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 3편입니다. 2편(Dreamer 계열)에서 이어집니다.</em></p>

## 정의

2편(Dreamer 계열)은 관측 복원 손실 \\( \ln p(o_t\mid h_t,z_t) \\)을 목적함수에 그대로 유지합니다. MuZero(Schrittwieser et al., 2020)는 이 항을 아예 뺍니다. Latent가 픽셀을 복원할 필요 없이, 다음 세 가지 예측에만 필요한 만큼 맞으면 됩니다.

$$
s_0 = h_\theta(o_1,\dots,o_t) \qquad \text{(representation function)}
$$

$$
r_k, s_k = g_\theta(s_{k-1}, a_k) \qquad \text{(dynamics function)}
$$

$$
p_k, v_k = f_\theta(s_k) \qquad \text{(prediction function: 정책, 가치)}
$$

\\( h_\theta \\)가 관측을 latent \\( s_0 \\)로 압축하고, \\( g_\theta \\)가 행동을 받아 다음 latent와 보상을 예측하고, \\( f_\theta \\)가 그 latent에서 정책 prior와 가치를 뽑습니다. 이 세 함수를 Monte Carlo Tree Search(MCTS)와 결합해 실제 플레이 시 매 스텝 탐색합니다.

<figure class="tp-fig">
<svg viewBox="0 0 760 300" role="img" aria-label="관측이 representation function을 거쳐 s0가 되고, dynamics function이 행동을 받아 s1과 보상을 예측하고, prediction function이 정책과 가치를 뽑는다. 아래 강조 상자: 복원 손실이 없다.">
  <defs>
    <marker id="m3Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="m3ArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="200" y1="75" x2="248" y2="75" marker-end="url(#m3Arrow)"/>
    <line x1="420" y1="75" x2="468" y2="75" marker-end="url(#m3Arrow)"/>
    <line x1="340" y1="150" x2="340" y2="112" marker-end="url(#m3Arrow)"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="140" y1="112" x2="140" y2="222" marker-end="url(#m3ArrowAccent)"/>
  </g>

  <rect x="40" y="40" width="160" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="120" y="70" text-anchor="middle" font-size="12" fill="currentColor">o_1..o_t</text>
  <text x="120" y="90" text-anchor="middle" font-size="12" style="fill:var(--text-muted)">h_θ (representation)</text>

  <rect x="250" y="40" width="170" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="335" y="70" text-anchor="middle" font-size="12" fill="currentColor">s_0</text>
  <text x="335" y="90" text-anchor="middle" font-size="12" style="fill:var(--text-muted)">g_θ (dynamics)</text>

  <rect x="290" y="150" width="140" height="45" rx="8" fill="var(--bg-secondary)" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3"/>
  <text x="360" y="177" text-anchor="middle" font-size="12" style="fill:var(--text-muted)">a_1</text>

  <rect x="470" y="40" width="170" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="555" y="65" text-anchor="middle" font-size="12" fill="currentColor">s_1, r_1</text>

  <rect x="30" y="222" width="700" height="65" rx="8" style="fill:var(--accent-light);stroke:var(--accent)" stroke-width="1.6"/>
  <text x="380" y="250" text-anchor="middle" font-size="12.5" style="fill:var(--accent)" font-weight="700">복원 손실 ln p(o_t | s_t) 이 목적함수에 없다</text>
  <text x="380" y="269" text-anchor="middle" font-size="11.5" style="fill:var(--accent)">→ "복원에 도움되니까 남기자"는 유인 자체가 성립하지 않는다</text>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> 관측이 representation function을 거쳐 s_0가 되고, dynamics function이 행동을 받아 다음 latent와 보상을 예측합니다. prediction function(그림에는 생략, s_1 이후 반복)이 정책과 가치를 뽑습니다. <strong>진짜 쟁점은 아래 강조 상자</strong>입니다. Dreamer(2편)와 달리 이 체인 어디에도 관측을 복원하는 손실이 없습니다.</figcaption>
</figure>

## 이 개념이 풀고자 했던 문제

Dreamer 계열은 왜 복원 손실을 유지할까요. 복원이 없으면 \\( s_t \\)가 애초에 무엇을 표현해야 하는지 알려주는 학습 신호가 약해질 수 있다는 우려 때문입니다. MuZero는 이 우려에 다른 답을 냅니다. 바둑·체스·쇼기·Atari처럼 "정확한 픽셀 재현"이 전혀 목표가 아닌 태스크에서는, 복원이 오히려 필요 없는 디테일까지 맞히도록 강제하는 낭비라는 것입니다. representation·dynamics·prediction 세 함수를 policy·value·reward 예측 손실만으로 end-to-end 학습합니다.

$$
\mathcal L_t(\theta) = \sum_{k=0}^{K} \Big[ l^p(\pi_{t+k}, p_t^k) + l^v(z_{t+k}, v_t^k) + l^r(u_{t+k}, r_t^k) \Big] + c\lVert\theta\rVert^2
$$

\\( \pi_{t+k} \\)는 MCTS가 실제로 탐색해서 얻은 방문 횟수 분포(정책의 개선된 타겟), \\( z_{t+k} \\)는 n-step 리턴 또는 MCTS 가치, \\( u_{t+k} \\)는 실제 관측된 보상입니다. 세 손실 다 복원이 아니라 결정에 필요한 양을 직접 겨냥합니다.

전작 AlphaZero는 시뮬레이터(체스 규칙 등)에 직접 접근해 실제 게임 상태로 트리를 탐색했습니다. MuZero는 시뮬레이터 없이, 학습된 \\( g_\theta \\)가 만드는 상상된 latent 궤적 위에서 트리를 탐색합니다. \\( g_\theta \\)가 "진짜 다음 상태"를 예측할 필요가 없고 \\( f_\theta \\)가 정확한 정책·가치를 뽑아낼 수 있는 무언가만 예측하면 되기 때문입니다. \\( s_t \\)가 실제 게임 상태와 대응될 필요조차 없습니다.

## taskcraft 아이디어와의 접목

> 이 편이 1편에서 제기한 minimality 문제의 첫 반례 후보입니다. 1편의 논증은 "복원에 도움이 되면 embodiment 정보도 남을 유인이 있다"였습니다. MuZero는 애초에 복원 손실 자체를 목적함수에서 뺐으므로, 복원에 도움되니까 남기자는 유인이 성립하지 않습니다.
>
> $$
> I(s_t; o_t)\text{를 직접 늘리는 항이 없다} \quad\Rightarrow\quad \text{복원 경로로 embodiment 정보가 새어 들어올 유인이 사라진다}
> $$
>
> 다만 이게 embodiment invariance를 보장하지는 않습니다. 가치·정책 예측에 신체 정보가 실제로 도움이 된다면(예: "내 팔이 닿는 범위" 자체가 가치 판단에 필요하다면) 여전히 \\( s_t \\)에 남을 이유가 있습니다. 즉 MuZero는 필요조건 하나(복원 압박 제거)를 만족시키지만, 충분조건(가치·정책 경로로도 embodiment 정보가 안 새는가)은 여전히 열려 있습니다. 이건 1편에서 예고한 "실제로 얼마나 새는가"를 linear probe로 측정해야 답이 나오는 질문입니다.
>
> 또 하나, MuZero의 세 함수 분리(h/g/f)는 다른 축의 참고 사례입니다. representation function이 여러 embodiment의 관측을 같은 \\( s_t \\) 공간으로 매핑하게 하려면, \\( h_\theta \\)를 embodiment마다 다르게 두고 \\( g_\theta,f_\theta \\)만 공유하는 설계가 필요합니다. 이건 "공유 trunk + embodiment별 얕은 헤드"를 정확히 뒤집은 배치(embodiment별 인코더 + 공유 trunk)라, taskcraft가 최종적으로 선택할 인코더/디코더 배치 설계에 참고할 만한 대안 패턴입니다.

## 한계 / 아직 안 풀린 문제

- MCTS가 구조적으로 필수입니다. 순수 정책망만으로 행동을 뽑는 구조가 아니라서, 로봇 연속 제어처럼 행동 공간이 크고 연속적인 경우 트리 탐색 자체가 비쌉니다.
- 바둑·체스·Atari 전부 단일 게임, 단일 embodiment 안에서만 검증됐습니다. 여러 게임에 걸친 \\( s_t \\) 공유는 시도된 적이 없습니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| MuZero (Schrittwieser et al., 2020) | representation/dynamics/prediction 세 함수, 복원 없는 MCTS 기반 model-based RL |
| AlphaZero (Silver et al., 2017) | MuZero의 전작. 시뮬레이터에 직접 접근하는 트리 탐색 |

## 다음 글

4편은 Genie입니다. MuZero는 가치·정책이라는 태스크 신호가 명확히 있어야 minimality가 성립합니다. Genie는 그런 신호(보상, 가치, 심지어 행동 라벨)가 아예 없는 순수 영상만으로 압축을 강제합니다. 신호 없이 어떻게 minimality를 얻는지가 4편의 질문입니다.

<style>
.post-series-note { color: var(--text-muted); font-size: 0.9rem; }
.tp-fig svg { width: 100%; height: auto; display: block; background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 10px; }
.tp-fig figcaption { font-size: 0.82rem; color: var(--text-muted); margin-top: 0.6rem; line-height: 1.6; }
</style>
