---
title: "[이론] 11. MetaMorph — 형태를 트랜스포머의 토큰으로 넣는다"
date: 2026-08-27
categories: [Project]
tags: [taskcraft-theory, transformer, metamorph, morphology, reinforcement-learning, robotics]
excerpt: "로봇의 각 모듈을 트랜스포머 토큰으로 인코딩해 self-attention이 몸 구조 사이 관계를 스스로 배우게 만드는 MetaMorph의 구조를 원 논문 수식으로 짚고, NerveNet의 그래프 방식과 무엇이 다른지 정리합니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 11편, 이론 파트의 마지막 편입니다. 10편(NerveNet)에서 이어집니다.</em></p>

## 정의

10편(NerveNet)은 몸 구조를 그래프로 표현하고 메시지 패싱으로 정보를 전파했습니다. MetaMorph(Gupta, Fan, Ganguli, Fei-Fei, ICLR 2022)는 같은 문제(다른 몸 구조에 하나의 정책 일반화)를 다른 도구로 풉니다. 몸을 이루는 각 모듈(관절)을 트랜스포머의 토큰 하나로 취급하고, self-attention이 모듈 사이의 관계를 배우게 합니다. 저자들의 표현을 그대로 쓰면, 로봇 형태는 트랜스포머의 출력을 조건화할 수 있는 또 다른 모달리티일 뿐입니다.

<figure class="tp-fig">
<svg viewBox="0 0 780 260" role="img" aria-label="운동학 트리를 깊이 우선 순회해 얻은 모듈들의 관측을 선형 투영하고 학습된 위치 임베딩을 더해 토큰 시퀀스를 만든다. 이 시퀀스가 self-attention과 MLP로 이뤄진 트랜스포머 층을 L번 통과한 뒤, 외수용감각 정보와 결합해 모듈별 행동 분포를 낸다.">
  <defs>
    <marker id="r11Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <text x="30" y="20" font-size="11" style="fill:var(--text-muted)">Encode: 운동학 트리를 깊이 우선 순회 → 1D 토큰 시퀀스</text>

  <rect x="30" y="30" width="90" height="42" rx="7" fill="var(--bg)" stroke="currentColor" stroke-width="1.1"/>
  <text x="75" y="55" text-anchor="middle" font-size="10" fill="currentColor">module 1</text>
  <rect x="128" y="30" width="90" height="42" rx="7" fill="var(--bg)" stroke="currentColor" stroke-width="1.1"/>
  <text x="173" y="55" text-anchor="middle" font-size="10" fill="currentColor">module 2</text>
  <rect x="226" y="30" width="90" height="42" rx="7" fill="var(--bg)" stroke="currentColor" stroke-width="1.1"/>
  <text x="271" y="55" text-anchor="middle" font-size="10" fill="currentColor">module N</text>

  <g fill="none" stroke="currentColor" stroke-width="1.2" opacity="0.8">
    <line x1="75" y1="72" x2="75" y2="96" marker-end="url(#r11Arrow)"/>
    <line x1="173" y1="72" x2="173" y2="96" marker-end="url(#r11Arrow)"/>
    <line x1="271" y1="72" x2="271" y2="96" marker-end="url(#r11Arrow)"/>
  </g>

  <rect x="30" y="98" width="286" height="34" rx="7" fill="var(--bg)" stroke="currentColor" stroke-width="1.1"/>
  <text x="173" y="120" text-anchor="middle" font-size="10.5" fill="currentColor">Linear Projection + 학습된 위치 임베딩</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="318" y1="115" x2="354" y2="115" marker-end="url(#r11Arrow)"/>
  </g>

  <rect x="356" y="82" width="200" height="66" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="456" y="108" text-anchor="middle" font-size="11.5" fill="currentColor">Morphology-Aware</text>
  <text x="456" y="125" text-anchor="middle" font-size="11.5" fill="currentColor">Transformer (L층)</text>
  <text x="456" y="140" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">self-attention + MLP</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="558" y1="115" x2="594" y2="115" marker-end="url(#r11Arrow)"/>
  </g>

  <rect x="596" y="82" width="160" height="66" rx="8" fill="var(--accent-light)" stroke="var(--accent)" stroke-width="1.3"/>
  <text x="676" y="108" text-anchor="middle" font-size="11" style="fill:var(--accent)">Decode</text>
  <text x="676" y="126" text-anchor="middle" font-size="10" style="fill:var(--accent)">+ 외수용감각(카메라)</text>
  <text x="676" y="141" text-anchor="middle" font-size="10" style="fill:var(--accent)">→ 모듈별 행동 분포</text>

  <text x="173" y="185" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">몸 구조가 달라지면 토큰 개수·순서만 달라지고,</text>
  <text x="173" y="202" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">트랜스포머 파라미터는 그대로입니다.</text>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> 몸의 각 모듈이 토큰이 되어 트랜스포머를 통과하고, self-attention이 모듈 사이 관계를 학습으로 알아냅니다. NerveNet처럼 물리적 인접 구조를 미리 강제하지 않습니다.</figcaption>
</figure>

운동학 트리를 깊이 우선 순회(depth-first traversal)해서 모듈들을 일렬로 늘어놓고, 각 모듈의 관측(고유수용감각 + 그 모듈이 어떤 형태인지)을 선형 투영해 토큰 하나로 만듭니다. 몸 구조가 달라지면 토큰 개수와 순서가 달라질 뿐, 트랜스포머 자체의 파라미터는 그대로입니다. 이 토큰 시퀀스가 self-attention 층들을 통과하면서 모듈 사이 관계(어깨가 다리에 미치는 영향 등)를 배우고, 마지막에 카메라 같은 외수용감각 정보와 합쳐져 모듈별 행동 분포를 냅니다.

## 시리즈 지도

<figure class="tp-fig">
<div class="tp-roadmap">
  <div class="tp-roadmap__stage tp-roadmap__stage--prereq">
    <div class="tp-roadmap__num">선수</div>
    <div class="tp-roadmap__label">이미 아는 것</div>
    <div class="tp-roadmap__items">DATA403 RL 기초<br>(MDP·정책·가치함수)<br>VLA 개괄</div>
  </div>
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">00</div>
    <div class="tp-roadmap__label">예비</div>
    <div class="tp-roadmap__items">0. 인코더/디코더와 VAE</div>
  </div>
  <div class="tp-roadmap__stage">
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
  <div class="tp-roadmap__stage tp-roadmap__stage--current">
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
<figcaption><strong>이 그림이 보여주는 것.</strong> 1~13편이 정의 → 학습법 → 잠재 행동 추출 → 사람 표현 이식 → 실전/대안 → 종합·실행 순서로 이어지다가, 오른쪽 끝에서 taskcraft 실제 연구로 빠져나갑니다. 강조된 04단계가 이론 파트의 마지막인 지금 이 글입니다.</figcaption>
</figure>

## 수식/분석

모듈 임베딩은 관측을 선형 투영한 뒤 위치 임베딩을 더해서 만듭니다.

$$
\mathbf m_0 = [\phi(s^k_{l_1};\mathbf W_e); \cdots; \phi(s^k_{l_N};\mathbf W_e)] + \mathbf W_{\text{pos}}
$$

\\( \phi \\)는 임베딩 함수, \\( \mathbf W_e \\)는 그 가중치, \\( \mathbf W_{\text{pos}} \\)는 학습된 위치 임베딩입니다. 이후 \\( L \\)개 층에서 표준 트랜스포머 블록을 반복합니다.

$$
\mathbf m'_\ell = \text{MSA}(\text{LN}(\mathbf m_{\ell-1})) + \mathbf m_{\ell-1}, \qquad \mathbf m_\ell = \text{MLP}(\text{LN}(\mathbf m'_\ell)) + \mathbf m'_\ell, \qquad \ell=1\ldots L
$$

MSA는 multi-head self-attention, LN은 layer norm입니다. 여기까지는 표준 Vision Transformer 인코더와 거의 같은 구조입니다. 다른 건 입력 토큰이 이미지 패치가 아니라 몸의 모듈이라는 것뿐입니다.

행동은 인코더 출력과 외수용감각 정보를 결합해서 냅니다.

$$
\mathbf g = \gamma(s^k_g;\mathbf W_g), \qquad \mu(\mathbf s^k) = \phi(\mathbf m_{l_i}, \mathbf g; \mathbf W_d), \qquad \pi_\theta(a^k_t \mid s^k_t) = \mathcal N\big(\{\mu(s^k_i)\}_i^N, \Sigma\big)
$$

\\( \gamma \\)는 카메라 등 고차원 입력을 압축하는 별도 MLP이고, 이걸 각 모듈의 인코더 출력 \\( \mathbf m_{l_i} \\)과 이어붙여 그 모듈의 행동 평균 \\( \mu \\)를 냅니다. 공분산 \\( \Sigma \\)는 고정입니다. 가치 함수도 같은 방식으로, 모듈별 가치를 낸 뒤 평균 내서 전체 몸에 대한 값을 얻습니다.

여러 몸을 한 번에 학습시킬 때 생기는 문제도 다룹니다. 어떤 로봇은 넘어져도 잘 버티고, 어떤 로봇은 쉽게 넘어져서 에피소드가 짧게 끝납니다. 그냥 두면 안정적인 로봇이 더 많은 경험을 쌓아 더 빨리 학습되는 부익부 현상이 생깁니다. 이걸 막기 위해 로봇 \\( k \\)를 샘플링할 확률을 성능이 낮을수록 높이는 동적 리플레이 버퍼 균형을 씁니다.

$$
P_k = \frac{\mathcal E_k^\beta}{\sum_i \mathcal E_i^\beta}, \qquad \mathcal E_k^\tau = \alpha \mathcal E_k^\tau + (1-\alpha)\mathcal E_k^{(\tau-1)}
$$

\\( \mathcal E_k \\)는 에피소드 길이 같은 성능 지표를 최대 길이의 역수로 바꾼 값이라, 성능이 낮은(에피소드가 짧은) 로봇일수록 \\( P_k \\)가 커집니다.

## 이 개념이 풀고자 했던 문제

UNIMAL 설계 공간(Gupta et al. 2021)만 해도 로봇 형태가 \\( 10^{18} \\)가지가 넘습니다. 형태마다 정책을 따로 학습하는 건 불가능합니다. NerveNet의 GNN 접근이 이미 이 문제를 다뤘지만, MetaMorph는 비전·언어에서 트랜스포머 대규모 사전학습이 보인 성과(제한된 도메인 특화 구조 없이도 범용적으로 스케일링됨)를 로보틱스에 그대로 가져올 수 있는지 묻습니다.

## 핵심 아이디어

UNIMAL에서 뽑은 로봇 100종을 평평한 지형, 변화하는 지형, 장애물 세 환경에서 함께 학습시킨 결과, MetaMorph는 로봇별로 따로 학습한 MLP(사실상의 성능 상한선)에 근접하는 평균 보상을 냈고, 같은 조건에서 GNN(NerveNet을 이 실험 세팅에 맞게 재구현한 베이스라인)을 꾸준히 앞섰습니다. 동적 리플레이 버퍼 균형을 빼면 로봇의 10~15%가 MLP 상한선의 75%에도 못 미치는 성능을 보여, 이 균형 장치가 일부 몸만 잘 배우는 문제를 실제로 막는다는 것도 확인됩니다. 사전학습된 단일 정책은 학습 때 본 적 없는 동역학·형태·태스크 변화에도 zero-shot으로 어느 정도 일반화하고, 새로운 형태·태스크에는 적은 데이터로 빠르게 파인튜닝됩니다.

## taskcraft 아이디어와의 접목

> 10편에서 이미 정리했듯, NerveNet과 MetaMorph는 둘 다 닮은 집합(UNIMAL이라는 같은 설계 공간) 안에서의 일반화입니다. 이 편에서 더할 지점은 아키텍처 선택의 실질적 차이입니다. NerveNet은 그래프의 물리적 인접 구조(관절이 실제로 연결된 관계)를 메시지 패싱의 경로로 강제합니다. MetaMorph는 그 구조를 명시적으로 강제하지 않고, self-attention이 어떤 모듈과 어떤 모듈이 관련 있는지 학습으로 알아내게 둡니다. MetaMorph 논문이 학습된 attention mask를 분석해 운동 시너지(motor synergy)가 나타난다고 보고하는 것도, 이 관계를 사람이 미리 정해주지 않아도 트랜스포머가 스스로 찾아낼 수 있다는 근거입니다.
>
> 이건 공유 latent를 embodiment별로 어떻게 연결할 것인가에 두 가지 서로 다른 답(그래프 구조를 사람이 지정 vs 어텐션이 스스로 학습)을 제공합니다. taskcraft가 나중에(겉날개·보트를 넘어) 관절 기반 embodiment로 파일럿을 확장한다면, embodiment 구조를 인코더에 어떻게 알려줄 것인가를 결정할 때 이 두 축 사이에서 골라야 합니다. 다만 지금 파일럿(Minecraft 내 embodiment 교체)에는 이런 모듈형 구조 자체가 없어서, 이번 파일럿에는 직접 적용되지 않습니다.

## 한계 / 아직 안 풀린 문제

- UNIMAL 100종 전부 같은 설계 공간(구, 원기둥 모듈을 관절로 연결)에서 나옵니다. 완전히 다른 종류의 몸(바퀴, 날개, 무한궤도)이 이 토큰화 방식에 자연스럽게 들어가는지는 검증되지 않았습니다.
- 위치 임베딩이 깊이 우선 순회 순서에 의존합니다. 같은 몸이라도 순회 순서를 다르게 잡으면 다른 토큰 시퀀스가 되는데, 이 임의성이 학습에 미치는 영향은 명확히 다뤄지지 않았습니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| MetaMorph (Gupta, Fan, Ganguli, Fei-Fei, ICLR 2022) | 로봇 모듈을 트랜스포머 토큰으로 인코딩해 형태 간 일반화를 학습 |
| NerveNet (10편) | 같은 문제를 GNN 메시지 패싱으로 푸는 대안. MetaMorph 논문이 직접 베이스라인으로 재구현해 비교 |
| UNIMAL (Gupta et al., 2021) | MetaMorph가 쓰는 모듈형 로봇 설계 공간 |

## 다음 글

12편은 taskcraft의 가설 종합입니다. 0편(VAE)부터 여기까지 이어온 이론(압축·표현부터 시작해 world model, 잠재 행동, 지각·보상 이식, 영상 생성 기반 데이터 합성과 통합 플랫폼, 마지막으로 몸 구조 공유까지)이 각각 taskcraft의 어느 주장을 뒷받침하거나 반박하는지 한데 모아 정리합니다. 이 편까지가 이론 파트이고, 12편부터는 그 이론이 taskcraft라는 구체적 주장으로 어떻게 수렴하는지를 다룹니다.

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
