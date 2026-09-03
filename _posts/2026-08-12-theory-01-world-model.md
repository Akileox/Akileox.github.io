---
title: "[이론] 1. World Model"
date: 2026-07-18
categories: [Project]
tags: [taskcraft-theory, world-model, reinforcement-learning, robotics]
excerpt: "실제 환경과 매번 부딪히지 않고 미래를 예측하는 world model이 왜 필요한지부터, 압축된 상태 z_t 위에서 정의된 전이 모델이 실제로 어떻게 학습되는지, 그리고 그 z_t에 무엇이 담기는지를 taskcraft 질문과 연결해 짚는다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 1편이다. 0편에서 latent space와 task space가 다르다는 걸 확인했다. 이 편은 그 latent가 "미래 예측"이라는 구체적인 목적에 쓰일 때 어떤 모양이 되는지를 본다.</em></p>

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

## 왜 world model이 필요한가

RL 에이전트가 정책을 개선하려면 결국 "이 행동을 하면 다음에 뭐가 일어나는가"를 알아야 한다. 가장 단순한 방법은 실제 환경에서 직접 부딪혀보는 것이다. 그런데 environment interaction 자체에 비용이 크다. 로봇이면 마모·안전 문제, 시뮬레이터라도 스텝 수가 늘어날수록 학습 시간이 늘어난다. **시행착오만으로 학습하는 것의 한계**는 결국 "실제로 겪어본 만큼만 안다"는 것이다.

여기서 나오는 질문은 단순하다. 환경을 직접 경험하는 대신, 머릿속(내부 모델)에서 미래를 미리 예측하고 그 위에서 학습할 수는 없을까. 그 내부 모델이 world model이다.

Dyna(Sutton, 1990)가 이 아이디어의 원형이다. 실제 전이로부터 \\( T(s'\mid s,a) \\), \\( R(s,a) \\)를 학습하고, 학습된 모델로 만든 가상 전이를 실제 전이와 섞어 같은 Q-learning 업데이트에 쓴다.

$$
Q(s,a) \leftarrow Q(s,a) + \alpha\left[r + \gamma \max_{a'} Q(s',a') - Q(s,a)\right]
$$

이 업데이트를 실제 경험과 모델이 만든 "상상된" 경험 둘 다에 적용하면, 학습량이 실제 환경 스텝 수에 더 이상 묶이지 않는다. 다만 여기서 \\( s \\)는 압축되지 않은 raw 상태다. Dyna의 핵심은 "환경의 dynamics를 모델링하고 그 모델로 상상된 경험을 만든다"는 것이고, World Models(Ha & Schmidhuber, 2018)는 이 아이디어를 고차원 시각 관측에서도 학습 가능하도록, \\( s \\) 자리를 0편에서 다룬 압축 표현 \\( z \\)로 바꾼 것으로 볼 수 있다.

## 핵심 아이디어: Observation → Latent → Dynamics

World model은 **환경의 관측과 시간적 변화를 내부 표현으로 모델링해서, 실제 환경 대신 그 모델 안에서 미래를 예측하고 행동을 평가할 수 있게 하는 내부 환경 모델**이다. 가장 압축된 형태로 쓰면 두 화살표로 요약된다.

$$
o_t \to z_t, \qquad z_t, a_t \to z_{t+1}
$$

관측을 잠재 상태로 압축하고(0편에서 이미 다룬 인코더 \\( q_\phi(z_t\mid o_t) \\)), 그 잠재 상태와 행동만으로 다음 잠재 상태를 예측한다. 실제 관측 \\( o_{t+1} \\)을 보지 않고 예측한다는 게 핵심이다. 왜 raw observation이 아니라 latent 위에서 이걸 하는가. 픽셀 단위로 미래를 직접 예측하려면 불필요한 디테일(배경 텍스처, 조명 등)까지 다 맞혀야 해서 훨씬 비싸고 어렵다. 압축된 \\( z_t \\) 위에서 하면 "다음에 무슨 일이 일어나는가"라는 본질만 다루면 된다.

여기서 바로 0편의 질문이 다시 등장한다. 압축은 이미 0편에서 "그 자체로는 좋은 표현을 보장하지 않는다"는 걸 확인했다. World model에서는 압축의 기준이 하나 더 생긴다. **다음 상태 예측에 필요한 정보를 담고 있는가**다. 이 기준이 어떤 정보를 남기고 어떤 정보를 버리는지가 이 편의 핵심 질문이다.

## 논문에서는: World Models (2018)

Ha & Schmidhuber(2018)는 이 아이디어를 **V(Vision) + M(Memory) + C(Controller)** 세 모듈로 구현했다. 중요한 건 이 셋을 동시에 학습시키지 않는다는 점이다. 순서대로, 따로 학습한다.

**V: VAE.** 0편에서 다룬 그 VAE다. 실제 게임 화면들을 모아 인코더 \\( q_\phi(z\mid o) \\)와 디코더를 먼저 학습시키고, 학습이 끝나면 이 VAE는 고정한다. 이후 단계에서는 파라미터가 더 이상 안 바뀐다.

**M: MDN-RNN.** 고정된 VAE로 실제 플레이 데이터의 모든 프레임을 미리 \\( z_t \\)로 인코딩해둔다. 그 다음, 실제로 관측된 \\( z_t, a_t \\)를 입력으로 실제로 관측된 다음 \\( z_{t+1} \\)을 맞히도록 지도학습(supervised)시킨다.

$$
\mathcal L(\theta) = -\mathbb E\big[\log p_\theta(z_{t+1}\mid h_t)\big]
$$

$$
p_\theta(z_{t+1}\mid h_t) = \sum_{k=1}^K \pi_k(h_t)\,\mathcal N\big(z_{t+1};\,\mu_k(h_t),\,\sigma_k^2(h_t)\big), \qquad h_t=\mathrm{RNN}(h_{t-1},z_t,a_t)
$$

<span class="aside">(다음 상태를 점 하나가 아니라 K개 가우시안을 섞은 혼합분포로 예측한다는 뜻이다. 왜 혼합분포인가. 미래가 다봉분포일 수 있기 때문이다. 갈림길에서 왼쪽으로 갈지 오른쪽으로 갈지, 벡터 하나로 하는 점 추정으로는 표현할 수 없다. 0편의 "생성은 함수가 아니라 분포를 학습한다"가 여기서는 "미래는 함수가 아니라 분포로 예측한다"는 형태로 반복된다.)</span>

여기서 z는 모두 1단계에서 고정된 VAE가 실제 프레임으로부터 계산해둔 값이다. 즉 이 손실은 "관측을 본 추정치"와 "관측 없이 예측한 추정치"를 맞추는 구조가 아니라, 그냥 실제 다음 latent를 맞히는 표준적인 최대우도 회귀에 가깝다. <span class="aside">(이 구별이 중요한 이유는 다음 글로 넘어가기 전에에서 다시 짚는다.)</span>

**C: Controller.** M까지 학습시킨 뒤에는, 이 MDN-RNN을 "꿈속 환경"처럼 써서 실제 게임을 켜지 않고도 롤아웃을 만들 수 있다. 이 상상된 롤아웃 위에서 아주 작은 선형 컨트롤러를 CMA-ES(진화 전략)로 학습시킨다. 실제로 논문은 이 컨트롤러를 순수하게 "꿈속"에서만 학습시켜도 실제 환경에서 통한다는 걸 보여준다. **환경 dynamics를 latent space에서 학습해두면, 그 안에서 상상만으로 정책까지 학습할 수 있다**는 게 이 논문이 보여준 핵심이다.

<figure class="tp-fig">
<svg viewBox="0 0 700 330" role="img" aria-label="z_t가 전이 모델을 거쳐 다음 상태를 예측하는 구조. 행동 a_t는 전이 모델에만 입력된다. z_t에서 아래로 뻗어나온 전체 폭의 질문 상자: 이 압축이 신체 정보까지 남기는지는 미해결이다.">
  <defs>
    <marker id="tpArrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="tpArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="230" y1="75" x2="268" y2="75" marker-end="url(#tpArrow)"/>
    <line x1="430" y1="75" x2="468" y2="75" marker-end="url(#tpArrow)"/>
    <line x1="350" y1="150" x2="350" y2="112" marker-end="url(#tpArrow)"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="150" y1="112" x2="150" y2="238" marker-end="url(#tpArrowAccent)"/>
  </g>

  <g>
    <rect x="70" y="40" width="160" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="150" y="69" text-anchor="middle" font-size="12" fill="currentColor">잠재 상태</text>
    <text x="150" y="90" text-anchor="middle" font-size="15" fill="currentColor">z_t</text>

    <rect x="270" y="40" width="160" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="350" y="69" text-anchor="middle" font-size="12" fill="currentColor">전이 모델</text>
    <text x="350" y="90" text-anchor="middle" font-size="14" fill="currentColor">p_θ</text>

    <rect x="470" y="40" width="160" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
    <text x="550" y="69" text-anchor="middle" font-size="12" fill="currentColor">예측된</text>
    <text x="550" y="90" text-anchor="middle" font-size="14" fill="currentColor">z_t+1</text>

    <rect x="280" y="150" width="140" height="50" rx="8" fill="var(--bg-secondary)" stroke="currentColor" stroke-width="1" stroke-dasharray="3 3"/>
    <text x="350" y="180" text-anchor="middle" font-size="12.5" style="fill:var(--text-muted)">행동 a_t</text>

    <rect x="70" y="240" width="560" height="70" rx="8" style="fill:var(--accent-light);stroke:var(--accent)" stroke-width="1.6"/>
    <text x="350" y="264" text-anchor="middle" font-size="12.5" style="fill:var(--accent)" font-weight="700">z_t가 "누가 이 변화를 일으켰는가"도</text>
    <text x="350" y="283" text-anchor="middle" font-size="12.5" style="fill:var(--accent)" font-weight="700">같이 담고 있는가?</text>
    <text x="350" y="301" text-anchor="middle" font-size="11.5" style="fill:var(--accent)">미해결. 다음 절에서 다룸</text>
  </g>
</svg>
<figcaption>z_t가 전이 모델을 거쳐 다음 상태를 예측한다(윗줄). 행동 a_t는 전이 모델에만 입력된다. 진짜 쟁점은 아래 상자다. z_t가 "나무가 잘리고 있다"는 태스크 신호만 남기는지, "사람 팔이 그렇게 만들었다"는 신체 정보까지 같이 남기는지는 이 구조만으로는 알 수 없다.</figcaption>
</figure>

## taskcraft와의 접목: z_t에는 무엇이 들어가는가

0편에서 이미 latent가 "복원에 도움되면 뭐든 담는다"는 걸 봤다. World model에서는 복원 대신 **다음 상태 예측**이 기준이라, 질문이 이렇게 바뀐다.

> World model이 미래를 예측하기 위해 필요한 정보와, task transfer에 필요한 정보는 동일한가?

직관적으로 보면, \\( z_t \\)는 다음 상태 예측에 필요한 정보를 보존하도록 학습된다. <span class="aside">(MDN-RNN의 NLL을 낮추려면 결국 다음 프레임을 잘 맞혀야 하고, 그러려면 그 예측에 쓸모 있는 정보가 z_t에 남아 있어야 하기 때문이다. 다만 이건 직관적 서술이지, 이 손실이 특정 정보량을 엄밀하게 최대화한다는 증명은 아니다.)</span> 그런데 \\( z_t \\)가 "필요한 만큼만" 담아야 한다는 **minimality** 제약은 손실함수 어디에도 없다. 신체를 특정하는 정보(팔 모양, 관절각)가 다음 상태 예측에 도움이 된다면 대체로 도움이 된다. 다음 상태가 정확히 어떻게 바뀌는지는 결국 "누가, 어떻게 움직였는가"에 좌우되기 때문이다. 그 정보는 \\( z_t \\) 안에 그대로 남을 유인이 있다.

**그렇다면 예측에 필요한 정보만 남기고 나머지는 버리도록 강제하는 별도의 objective가 있어야 하지 않을까?** 이 질문에 대한 근사적 시도들(Information Bottleneck, β-VAE 계열)이 있긴 하지만, 그중 무엇도 "task에 필요한 정보"와 "embodiment 정보"를 정확히 갈라주지는 못한다. 이 구분은 이 시리즈 전체가 계속 따라가는 질문이라 여기서 답을 내리지 않고 열어둔다.

## 한계 / 아직 안 풀린 문제

- 위 논증은 "왜 걱정해야 하는가"에 대한 이유이지, "실제로 얼마나 새는가"에 대한 답은 아니다. linear probe(추출된 \\( z_t \\) 위에 간단한 선형 분류기를 얹어 embodiment를 얼마나 잘 맞히는지 재는 방법)로 실증 측정이 필요하다.
- MDN-RNN은 완전히 순차적으로 학습된다. VAE가 먼저 고정되고, 그 위에서 MDN-RNN이 학습되고, 그 위에서 Controller가 학습된다. 세 모듈이 서로의 학습에 맞춰 같이 조정되지 않는다는 뜻이다. 그러다 보니 상상된 롤아웃을 길게 이어갈수록 예측 오차가 누적되고, 그 오차를 실제 관측으로 다시 바로잡을 방법이 이 구조 안에는 없다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| World Models (Ha & Schmidhuber, 2018) | V(VAE)+M(MDN-RNN)+C(CMA-ES 선형 컨트롤러), 세 모듈을 순차적으로 분리 학습. "world model" 용어의 기준점 |
| Dyna (Sutton, 1990) | model-based RL 고전. raw state 기반 모델과 상상 경험을 함께 사용 |
| Information Bottleneck (Tishby et al.) | \\( I(z_t;o_t) \\)에 명시적 패널티를 주는 프레이밍. 아직 이 편에서 답을 내리지 않은 질문과 관련된 참고 개념 |

## 다음 글로 넘어가기 전에

- world model이 왜 필요한가: 실제 환경과 매번 부딪히는 비용을 줄이려고. 내부에서 미래를 예측하고 그 안에서 학습한다.
- World Models(2018)는 무엇을 보여줬는가: VAE, MDN-RNN, Controller를 순서대로 따로 학습시켜도, 순수하게 "꿈속"에서 학습한 정책이 실제 환경에서 통한다는 것.
- 왜 이게 taskcraft 질문으로 이어지는가: "예측에 도움되는 정보"와 "embodiment에 무관한 정보"가 겹치지 않을 수 있어서다. 정책이 필요한 건 후자에 가깝다.

그런데 이 구조에는 근본적인 약점이 있다. VAE와 MDN-RNN이 완전히 분리되어 있어서, 상상된 롤아웃이 길어질수록 오차가 쌓여도 이를 실제 관측과 다시 맞춰볼 방법이 없다. **관측을 계속 참고하면서도 그 관측 없이 미래를 상상할 수 있게 만들려면, 즉 이 둘을 하나의 학습 가능한 구조로 합치려면 어떻게 해야 하는가?** 2편(Dreamer 계열)이 posterior(관측을 본 추정)와 prior(관측 없이 예측한 추정)를 명시적으로 나누고 이 둘을 KL로 정렬시키는 RSSM으로 이 질문에 답한다.

<style>
.post-series-note { color: var(--text-muted); font-size: 0.9rem; }
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
.tp-fig svg { width: 100%; height: auto; display: block; background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 10px; }
.tp-fig figcaption { font-size: 0.82rem; color: var(--text-muted); margin-top: 0.6rem; line-height: 1.6; }
@media (max-width: 760px) {
  .tp-roadmap { grid-template-columns: 1fr; }
  .tp-roadmap__stage + .tp-roadmap__stage { border-left: 1px solid var(--border); border-top: none; }
}
</style>
