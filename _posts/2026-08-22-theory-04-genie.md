---
title: "[이론] 4. Genie — 라벨 없이 Latent Action을 뽑는다"
date: 2026-07-30
categories: [Project]
tags: [taskcraft-theory, world-model, genie, reinforcement-learning, robotics]
excerpt: "행동 라벨도 보상도 없는 영상만으로 8개짜리 codebook에 latent action을 강제 압축하는 Genie의 구조를 수식으로 짚고, 이게 taskcraft의 embodiment-agnostic 가설에 가장 가까운 선행 실증인 이유를 정리합니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 4편입니다. 3편(MuZero)에서 이어집니다.</em></p>

## 정의

2편(Dreamer)과 3편(MuZero)은 둘 다 행동 \\( a_t \\)가 라벨로 주어진다는 전제 위에 서 있습니다. Genie(Bruce et al., DeepMind 2024)는 이 전제를 뺍니다. 행동 라벨도, 보상도 전혀 없는 대량 인터넷 영상만으로, 프레임 사이에 있었을 법한 latent action \\( \hat a_t \\)를 완전 비지도로 학습합니다.

$$
\hat a_t \sim q_\phi(\hat a_t \mid o_{1:t+1})
$$

\\( q_\phi \\)는 다음 프레임까지 보고서(그래서 \\( t{+}1 \\)까지) 그 사이에 일어난 행동을 역으로 추정합니다. 이 \\( \hat a_t \\)가 극도로 작은 codebook(논문 기준 8개)으로 벡터 양자화(VQ, VQ-VAE)됩니다.

**벡터 양자화(VQ)란.** 인코더가 뱉은 연속 벡터 \\( e \\)를 그대로 쓰지 않고, 미리 정해둔 codebook \\( \{c_1,\dots,c_K\} \\) 중 가장 가까운 항목으로 대체합니다.

$$
z = c_k, \qquad k = \arg\min_j \lVert e - c_j\rVert
$$

이 대체 연산도 미분 불가능합니다(\\( \arg\min \\)). VQ-VAE는 2편에서 다룬 straight-through를 그대로 씁니다. 순전파는 \\( c_k \\)를 쓰고, 역전파는 \\( e \\) 쪽으로 그래디언트를 그대로 흘려보냅니다. codebook 자체는 exponential moving average 또는 별도 손실(commitment loss)로 갱신됩니다. Genie의 LAM도 이 메커니즘을 그대로 써서 \\( \hat a_t \\)를 8개 codebook 중 하나로 강제 이산화합니다.

<figure class="tp-fig">
<svg viewBox="0 0 760 320" role="img" aria-label="o_t와 o_t+1이 Latent Action Model을 거쳐 8개 codebook 중 하나로 양자화된다. o_t는 별도로 Video Tokenizer를 거쳐 discrete token이 되고, Dynamics Model이 이 토큰과 latent action으로 다음 토큰을 예측한다.">
  <defs>
    <marker id="g4Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="g4ArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="200" y1="55" x2="248" y2="70" marker-end="url(#g4Arrow)"/>
    <line x1="200" y1="120" x2="248" y2="90" marker-end="url(#g4Arrow)"/>
    <line x1="140" y1="112" x2="140" y2="185" marker-end="url(#g4Arrow)"/>
    <line x1="250" y1="205" x2="400" y2="205" marker-end="url(#g4Arrow)"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="460" y1="112" x2="460" y2="180" marker-end="url(#g4ArrowAccent)"/>
  </g>

  <rect x="40" y="30" width="150" height="45" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="115" y="57" text-anchor="middle" font-size="12" fill="currentColor">o_t</text>

  <rect x="40" y="95" width="150" height="45" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="115" y="122" text-anchor="middle" font-size="12" fill="currentColor">o_t+1</text>

  <rect x="250" y="45" width="200" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="350" y="72" text-anchor="middle" font-size="12" fill="currentColor">Latent Action Model</text>
  <text x="350" y="92" text-anchor="middle" font-size="11.5" style="fill:var(--text-muted)">q_φ</text>

  <rect x="40" y="185" width="150" height="55" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="115" y="207" text-anchor="middle" font-size="12" fill="currentColor">Video Tokenizer</text>
  <text x="115" y="225" text-anchor="middle" font-size="11" style="fill:var(--text-muted)">discrete token</text>

  <rect x="400" y="185" width="200" height="55" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="500" y="207" text-anchor="middle" font-size="12" fill="currentColor">Dynamics Model</text>
  <text x="500" y="225" text-anchor="middle" font-size="11" style="fill:var(--text-muted)">다음 토큰 예측</text>

  <rect x="470" y="30" width="70" height="55" rx="6" fill="var(--accent-light)" stroke="var(--accent)" stroke-width="1.3"/>
  <text x="505" y="52" text-anchor="middle" font-size="10" style="fill:var(--accent)">codebook</text>
  <text x="505" y="68" text-anchor="middle" font-size="11" style="fill:var(--accent)" font-weight="700">8개 중 1</text>

  <line x1="450" y1="60" x2="468" y2="58" style="stroke:var(--accent)" stroke-width="1.4" marker-end="url(#g4ArrowAccent)"/>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> o_t와 o_t+1을 함께 본 Latent Action Model이 8개짜리 codebook 중 하나로 양자화된 latent action을 뽑습니다(강조 상자). o_t는 별도로 Video Tokenizer를 거쳐 discrete token이 되고, Dynamics Model이 이 토큰과 latent action을 받아 다음 프레임 토큰을 예측합니다.</figcaption>
</figure>

Genie는 세 컴포넌트로 구성됩니다. (1) 위 Latent Action Model(LAM), (2) 각 프레임을 discrete token으로 압축하는 Video Tokenizer(VQ-VAE), (3) 과거 토큰들과 \\( \hat a_t \\)로 다음 프레임 토큰을 autoregressive하게 예측하는 Dynamics Model(MaskGIT 스타일 트랜스포머). 학습 시에는 LAM이 실제 \\( o_{t+1} \\)까지 보고 \\( \hat a_t \\)를 뽑아 Dynamics Model에 넘겨주고, 추론(플레이) 시에는 사용자가 8개 codebook 중 하나를 직접 골라 \\( \hat a_t \\) 자리에 넣습니다. 이게 생성된 영상을 실제로 조작할 수 있다는 Genie 데모가 성립하는 이유입니다.

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
  <div class="tp-roadmap__stage tp-roadmap__stage--current">
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
  <div class="tp-roadmap__stage tp-roadmap__stage--dest">
    <div class="tp-roadmap__num">→</div>
    <div class="tp-roadmap__label">taskcraft 연구</div>
    <div class="tp-roadmap__items">사람 시연 + VIP 진행도 신호<br>→ Minecraft 이종 embodiment 이식</div>
  </div>
</div>
<figcaption><strong>이 그림이 보여주는 것.</strong> 1~13편이 정의 → 학습법 → 잠재 행동 추출 → 사람 표현 이식 → 실전/대안 → 종합·실행 순서로 이어지다가, 오른쪽 끝에서 taskcraft 실제 연구로 빠져나갑니다. 강조된 02단계가 지금 이 글입니다.</figcaption>
</figure>

## 이 개념이 풀고자 했던 문제

3편(MuZero)의 minimality는 정책·가치라는 태스크 신호가 있어야 성립했습니다. Genie는 그런 신호가 아예 없습니다. 인터넷에 널린 게임 플레이 영상에는 행동 로그가 없습니다. 그런데도 "이 프레임에서 저 프레임으로 넘어갈 때 뭔가 행동이 있었다"는 사실 자체는 영상에서 추론할 수 있습니다. Genie는 이걸 codebook 크기라는 극단적 병목(bottleneck)으로 강제합니다. \\( \hat a_t \\)가 담을 수 있는 정보량 자체를 3비트로 제한해버리면, LAM이 아무리 많은 정보를 담고 싶어도 물리적으로 그럴 수 없습니다.

$$
I(\hat a_t;\, o_{1:t+1}) \le \log_2 |\mathcal C|, \qquad |\mathcal C| = 8
$$

\\( \mathcal C \\)는 codebook. 1편이 "명시적으로 강제하려면 Information Bottleneck 스타일 패널티가 필요하다"고 한 것을, Genie는 패널티 항이 아니라 출력 공간 자체를 작게 만들어 강제로 우회합니다.

논문이 실제로 보인 것은, 학습 데이터에 없던 새 이미지(손그림 스케치 등)를 넣어도 같은 codebook 인덱스가 일관된 효과(예: 코드 3 = 오른쪽으로 이동)를 낸다는 것입니다. \\( \hat a_t \\)가 특정 게임의 픽셀 디테일이 아니라 "그 변화의 종류" 자체를 담고 있다는 실증입니다.

## taskcraft 아이디어와의 접목

> position_paper.md는 Genie를 "행동 라벨 없이 영상에서 embodiment 무관한 행동 유사 표현을 뽑아내는 실제 사례라 이 프로젝트와 가장 가깝다"고 직접 명시합니다. 이 편이 이 시리즈의 진짜 전환점인 이유입니다. 1~3편은 전부 "이미 주어진 표현이 embodiment 정보를 얼마나 새는가"를 물었습니다. Genie는 처음으로 "embodiment에 종속되지 않는 행동 표현을 처음부터 어떻게 만들 것인가"에 대한 구체적 답을 냅니다. 서로 다른 게임(서로 다른 embodiment에 가장 가까운 유비)에서도 같은 코드가 비슷한 효과를 낸다는 것은, 같은 latent 좌표가 여러 embodiment에 걸쳐 재사용 가능한가라는 taskcraft 핵심 가설에 대한 첫 실증적 선례입니다.
>
> 다만 이식(transfer)까지 보인 건 아닙니다. Genie가 보인 건 "여러 2D 플랫포머 게임에 걸쳐 같은 codebook이 통한다"는 것이지, 이 codebook을 형태가 근본적으로 다른 embodiment(로봇 팔 vs 다리형 로봇)의 실제 행동 공간에 이식할 수 있다는 것까지는 아닙니다. 이 간극이 정확히 position_paper.md가 지적하는 "닮은 집합 안의 일반화"와 "근본적으로 다른 형태 간 전이"의 구분입니다. Genie가 다루는 게임들은 시각 스타일도 조작 문법도 서로 닮았습니다.

## 한계 / 아직 안 풀린 문제

- 8개짜리 이산 codebook이 로봇의 연속 제어(관절 각도 등)로 확장 가능한지가 미지수입니다. discrete latent action을 continuous action으로 어떻게 매핑할지는 7편(DreamGen)이 정면으로 다루는 질문입니다.
- codebook 크기 자체가 새 하이퍼파라미터입니다. 너무 작으면 의미 있는 행동 구분이 안 되고, 너무 크면 1편의 minimality 문제가 다시 돌아옵니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| Genie (Bruce et al., DeepMind 2024) | LAM + Video Tokenizer + Dynamics Model, 완전 비지도 latent action |
| VQ-VAE (van den Oord et al., 2017) | 이산 codebook 양자화 기법. Video Tokenizer의 기반 |
| MaskGIT (Chang et al., 2022) | 병렬 마스크 예측 기반 이미지 토큰 생성. Dynamics Model 아키텍처 기반 |

## 다음 글

5편은 R3M입니다. Genie는 게임 영상 안에서 비지도로 행동을 뽑았습니다. R3M은 처음부터 사람의 실제 조작 영상에서 로봇에 전이 가능한 시각 표현을 지도 학습으로 뽑습니다. 라벨 없이 뽑을 것인가, 사람 영상이라는 더 강한 사전 지식을 지도 신호로 쓸 것인가라는 축으로 이어집니다.

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
