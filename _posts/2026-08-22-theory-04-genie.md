---
title: "[이론] 4. Genie"
date: 2026-07-30
categories: [Project]
tags: [taskcraft-theory, world-model, genie, reinforcement-learning, robotics]
excerpt: "보상도 가치도 행동 라벨도 없는 순수 영상만으로 압축을 강제하는 방법을 묻는 데서 출발해, codebook 크기 자체를 병목으로 쓰는 Genie의 latent action을 정리하고 이게 taskcraft 가설에 가장 가까운 선행 실증인 이유를 짚는다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 4편이다. 3편(MuZero)에서 이어진다.</em></p>

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
  <div class="tp-roadmap__stage">
    <div class="tp-roadmap__num">01</div>
    <div class="tp-roadmap__label">기초</div>
    <div class="tp-roadmap__items">1. World Model<br>2. Dreamer 계열<br>3. MuZero</div>
  </div>
  <div class="tp-roadmap__stage tp-roadmap__stage--current">
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

## 라벨도 보상도 없이 압축을 강제할 수 있는가

3편(MuZero) 끝에서 던진 질문이다. MuZero의 minimality는 정책·가치라는 명확한 태스크 신호가 있어야 성립했다. 바둑에는 승/패가 있고, MuZero는 그 신호에 맞춰서만 압축한다. 그런데 인터넷에 널린 게임 플레이 영상, 사람이 뭔가를 하는 영상에는 그런 신호가 아예 없다. 보상도, 가치도, 행동 로그도 없다. 2편(Dreamer)과 3편(MuZero)은 둘 다 행동 \\( a_t \\)가 라벨로 주어진다는 전제 위에 서 있었는데, Genie(Bruce et al., DeepMind 2024)는 이 전제 자체를 뺀다.

## 직관: 그릇을 작게 만들면 강제로 압축된다

신호가 없다면 어떻게 압축을 강제할까. Genie의 답은 손실함수에 패널티 항을 추가하는 게 아니라, **담을 수 있는 그릇 자체를 작게 만드는 것**이다. "이 프레임에서 저 프레임으로 넘어갈 때 뭔가 행동이 있었다"는 사실은 영상만 보고도 추론할 수 있다. 그 행동을 표현할 latent action \\( \hat a_t \\)를 극도로 작은 codebook(논문 기준 8개)에 강제로 욱여넣으면, 담을 수 있는 정보량 자체가 3비트로 막힌다. 아무리 많은 정보를 담고 싶어도 물리적으로 그럴 수 없다는 뜻이다. 1편이 "명시적으로 강제하려면 Information Bottleneck 스타일 패널티가 필요하다"고 했던 것을, Genie는 패널티 항이 아니라 **출력 공간 자체를 작게 만들어** 우회한다.

## 필요한 만큼만 수학

\\( \hat a_t \\)는 다음 프레임까지 보고(그래서 \\( t{+}1 \\)까지) 그 사이에 있었을 행동을 역으로 추정한다.

$$
\hat a_t \sim q_\phi(\hat a_t \mid o_{1:t+1})
$$

이 값이 미리 정해둔 codebook \\( \{c_1,\dots,c_K\} \\) 중 가장 가까운 항목으로 대체된다(벡터 양자화, VQ).

$$
z = c_k, \qquad k = \arg\min_j \lVert e - c_j\rVert
$$

\\( \arg\min \\)은 미분 불가능하다. 순전파는 \\( c_k \\)를 쓰고 역전파는 \\( e \\) 쪽으로 그래디언트를 그대로 흘려보내는 straight-through 트릭으로 우회한다(codebook 자체는 exponential moving average나 commitment loss로 갱신). 이 병목을 정보량으로 쓰면 다음 부등식이 성립한다.

$$
I(\hat a_t;\, o_{1:t+1}) \le \log_2 |\mathcal C|, \qquad |\mathcal C| = 8
$$

codebook 크기가 상호정보량의 하드 캡이 된다는 뜻이다.

## 논문에서 어떻게 확장되는가

Genie는 세 조각으로 구성된다. <span class="aside">(1) 위 Latent Action Model(LAM), (2) 각 프레임을 discrete token으로 압축하는 Video Tokenizer(VQ-VAE), (3) 과거 토큰들과 \\( \hat a_t \\)로 다음 프레임 토큰을 예측하는 Dynamics Model(MaskGIT 스타일 트랜스포머).</span> 학습 때는 LAM이 실제 \\( o_{t+1} \\)까지 보고 \\( \hat a_t \\)를 뽑아 Dynamics Model에 넘기고, 추론(플레이) 때는 사람이 8개 codebook 중 하나를 직접 골라 그 자리에 넣는다. 생성된 영상을 실제로 "조작"할 수 있다는 Genie 데모가 성립하는 이유다.

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
<figcaption><strong>이 그림이 보여주는 것.</strong> o_t와 o_t+1을 함께 본 Latent Action Model이 8개짜리 codebook 중 하나로 양자화된 latent action을 뽑는다(강조 상자). o_t는 별도로 Video Tokenizer를 거쳐 discrete token이 되고, Dynamics Model이 이 토큰과 latent action을 받아 다음 프레임 토큰을 예측한다.</figcaption>
</figure>

논문이 실제로 보인 건, 학습 데이터에 없던 새 이미지(손그림 스케치 등)를 넣어도 같은 codebook 인덱스가 일관된 효과(예: 코드 3 = 오른쪽으로 이동)를 낸다는 것이다. \\( \hat a_t \\)가 특정 게임의 픽셀 디테일이 아니라 **"그 변화의 종류" 자체**를 담고 있다는 실증이다.

## taskcraft와의 접목

이 편이 시리즈의 진짜 전환점이다. 1~3편은 전부 "이미 주어진 표현이 embodiment 정보를 얼마나 새는가"를 물었다. Genie는 처음으로 "embodiment에 종속되지 않는 행동 표현을 처음부터 어떻게 만들 것인가"에 구체적으로 답한다. 서로 다른 게임(서로 다른 embodiment에 가장 가까운 유비)에서도 같은 코드가 비슷한 효과를 낸다는 건, 같은 latent 좌표가 여러 embodiment에 걸쳐 재사용 가능한가라는 taskcraft 핵심 가설에 대한 첫 실증적 선례다.

다만 이식(transfer)까지 보인 건 아니다. Genie가 보인 건 "여러 2D 플랫포머 게임에 걸쳐 같은 codebook이 통한다"는 것이지, 이 codebook을 형태가 근본적으로 다른 embodiment(로봇 팔 vs 다리형 로봇)의 실제 행동 공간에 이식할 수 있다는 것까지는 아니다. Genie가 다루는 게임들은 시각 스타일도 조작 문법도 서로 닮았다. **닮은 집합 안의 일반화**와 **근본적으로 다른 형태 간 전이**는 다른 문제다.

## 한계 / 아직 안 풀린 문제

- 8개짜리 이산 codebook이 로봇의 연속 제어(관절 각도 등)로 확장 가능한지가 미지수다. discrete latent action을 continuous action으로 어떻게 매핑할지는 7편(DreamGen)이 정면으로 다루는 질문이다.
- codebook 크기 자체가 새 하이퍼파라미터다. 너무 작으면 의미 있는 행동 구분이 안 되고, 너무 크면 1편의 minimality 문제가 다시 돌아온다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| Genie (Bruce et al., DeepMind 2024) | LAM + Video Tokenizer + Dynamics Model, 완전 비지도 latent action |
| VQ-VAE (van den Oord et al., 2017) | 이산 codebook 양자화 기법. Video Tokenizer의 기반 |
| MaskGIT (Chang et al., 2022) | 병렬 마스크 예측 기반 이미지 토큰 생성. Dynamics Model 아키텍처 기반 |

## 다음 글로 넘어가기 전에

- Genie가 새로 보인 것: 아무 신호가 없어도 그릇(codebook)을 작게 만들면 압축이 강제된다는 것, 그리고 그렇게 얻은 행동 표현이 여러 게임에 걸쳐 재사용된다는 것.
- 그런데 이건 **행동** 쪽 얘기다. 지각(이 장면이 뭘 의미하는가) 쪽은 아직 안 건드렸다.

**행동 표현을 이렇게 비지도로 공유 가능하게 만들 수 있다면, 지각 표현도 같은 방식으로 공유할 수 있을까, 아니면 사람 영상이라는 더 강한 사전 지식을 지도 신호로 쓰는 게 나을까?** 5편(R3M)이 이 축으로 이어진다.

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
