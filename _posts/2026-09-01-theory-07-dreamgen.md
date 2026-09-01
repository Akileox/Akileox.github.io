---
title: "[이론] 7. DreamGen — 영상 생성이 데이터를 대신 만든다"
date: 2026-08-11
categories: [Project]
tags: [taskcraft-theory, world-model, dreamgen, video-generation, robotics]
excerpt: "영상 생성 모델을 target embodiment에 LoRA로 적응시켜 새 행동·새 환경의 합성 영상을 만들고, 거기서 IDM으로 pseudo-action을 추출하는 DreamGen의 파이프라인을 정리하고, taskcraft가 이 규모를 왜 포기했는지를 짚습니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 7편입니다. 6편(VIP)에서 이어집니다.</em></p>

## 정의

5, 6편은 사람 영상에서 표현(R3M)이나 보상(VIP)을 뽑아 로봇에 이식했습니다. 둘 다 로봇 쪽 학습 데이터는 여전히 사람이 실제로 모은 시연(teleoperation) 몇십 개였습니다. DreamGen(NVIDIA 외, 2025)은 다른 질문을 던집니다. 그 몇십 개 시연조차 영상 생성 모델로 늘릴 수 있는가. 실제로 로봇을 움직이지 않고, 이미 있는 시연을 씨앗 삼아 이 로봇이 새로운 행동을, 새로운 환경에서 하는 영상을 합성하고, 그 합성 영상에서 가짜 행동 라벨(neural trajectory)을 뽑아 정책을 학습시킵니다.

<figure class="tp-fig">
<svg viewBox="0 0 780 320" role="img" aria-label="단일 환경 단일 태스크의 실제 로봇 시연 영상을 씨앗으로, 영상 생성 모델을 LoRA로 target embodiment에 적응시키고, 새 행동·새 환경 프롬프트로 합성 영상을 생성한다. IDM 또는 LAPA로 이 합성 영상에서 pseudo-action을 추출해 neural trajectory 데이터셋을 만들고, 그걸로 정책을 학습한다.">
  <defs>
    <marker id="r7Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="r7ArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <text x="30" y="22" font-size="11" style="fill:var(--text-muted)">씨앗: 단일 환경·단일 태스크 teleoperation (수천 개)</text>

  <rect x="30" y="34" width="150" height="46" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="105" y="61" text-anchor="middle" font-size="11.5" fill="currentColor">실제 시연 영상</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="182" y1="57" x2="218" y2="57" marker-end="url(#r7Arrow)"/>
  </g>

  <rect x="220" y="34" width="220" height="46" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="330" y="52" text-anchor="middle" font-size="11" fill="currentColor">영상 생성 모델(WAN2.1)</text>
  <text x="330" y="68" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">LoRA로 target embodiment 적응</text>

  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="442" y1="57" x2="478" y2="57" marker-end="url(#r7ArrowAccent)"/>
  </g>

  <rect x="480" y="34" width="270" height="46" rx="8" fill="var(--accent-light)" stroke="var(--accent)" stroke-width="1.3"/>
  <text x="615" y="52" text-anchor="middle" font-size="11" style="fill:var(--accent)">새 행동·새 환경 프롬프트</text>
  <text x="615" y="68" text-anchor="middle" font-size="10.5" style="fill:var(--accent)">→ 합성 영상 생성</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="615" y1="82" x2="615" y2="118" marker-end="url(#r7Arrow)"/>
  </g>

  <rect x="480" y="120" width="270" height="70" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="615" y="140" text-anchor="middle" font-size="11.5" fill="currentColor">pseudo-action 추출</text>
  <text x="615" y="158" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">IDM(채택): diffusion transformer</text>
  <text x="615" y="174" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">+ SigLIP-2, flow matching</text>

  <text x="440" y="205" font-size="10.5" style="fill:var(--text-muted)">(LAPA: VQ-VAE latent action, 행동 데이터 불필요 — 실험상 IDM과 유사, 미채택)</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="615" y1="192" x2="615" y2="228" marker-end="url(#r7Arrow)"/>
  </g>

  <rect x="480" y="230" width="270" height="46" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="615" y="248" text-anchor="middle" font-size="11" fill="currentColor">neural trajectory 데이터셋</text>
  <text x="615" y="264" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">(실제 수집량의 최대 333배)</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="478" y1="253" x2="442" y2="253" marker-end="url(#r7Arrow)"/>
  </g>

  <rect x="230" y="230" width="200" height="46" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="330" y="258" text-anchor="middle" font-size="12" fill="currentColor">정책 학습 (BC 등)</text>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> 씨앗 시연으로 영상 생성 모델을 target embodiment에 적응시킨 뒤(강조 상자), 새 행동·환경 프롬프트로 합성 영상을 만들고 IDM으로 행동을 역추출해 정책을 학습합니다.</figcaption>
</figure>

씨앗은 pick-and-place 같은 단일 태스크, 단일 환경 시연 몇 천 개뿐입니다. 이걸로 영상 생성 모델을 그 로봇 전용으로 적응시킨 뒤, 전자레인지를 연다처럼 씨앗에는 없던 행동이나 처음 보는 환경을 프롬프트로 주고 그럴듯한 영상을 만들어냅니다. 문제는 영상은 픽셀일 뿐 행동이 아니라는 겁니다. 그래서 그 영상에서 다시 행동을 역추출하는 단계(IDM 또는 LAPA)가 필요합니다.

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
<figcaption><strong>이 그림이 보여주는 것.</strong> 1~13편이 정의 → 학습법 → 잠재 행동 추출 → 사람 표현 이식 → 실전/대안 → 종합·실행 순서로 이어지다가, 오른쪽 끝에서 taskcraft 실제 연구로 빠져나갑니다. 강조된 04단계가 지금 이 글입니다.</figcaption>
</figure>

## 수식/분석

Pseudo-action을 뽑는 두 방식은 4편(Genie)과 5·6편(R3M/VIP)에서 이미 나온 재료의 재조합입니다.

IDM(Inverse Dynamics Model, 채택된 방식)은 SigLIP-2 시각 인코더를 얹은 diffusion transformer가 연속된 두 프레임을 보고 그 사이의 행동 청크를 flow matching 목적함수로 예측합니다. 실제 teleoperation 데이터로 미리 학습해두는 지도학습 모델입니다.

LAPA(Latent Action Pretraining)는 4편 Genie와 같은 계열입니다. 현재 프레임과 1초 후 미래 프레임을 인코더-디코더에 넣고, 두 프레임 사이의 시각적 변화(visual delta)를 담도록 VQ-VAE 목적함수로 잠재 행동을 이산 코드북에 학습시킵니다. 행동 라벨이 전혀 필요 없습니다.

논문은 IDM을 기본값으로 채택했습니다. 이유는 성능 우위가 아니라 실용성입니다. IDM 행동은 neural trajectory만으로 학습과 평가를 전부 할 수 있게 해주고, 이 논문의 모든 실험 세팅에서 강한 IDM을 학습시킬 teleoperation 데이터는 이미 갖고 있었습니다. RoboCasa 실험에서 두 방식 성능은 비슷했고(신경궤적만으로 20.6% 성공률), 어차피 teleoperation 데이터가 있으니 라벨 없는 방식(LAPA)을 고집할 이유가 없었다는 겁니다. 이건 4편에서 본 행동 라벨 없이도 배울 수 있다는 Genie의 주장이, 라벨이 있으면 굳이 안 쓸 이유는 없다는 실용적 판단 앞에서 더 이상 절대적 우위가 아니게 되는 지점을 보여줍니다.

영상 생성 모델 자체(WAN2.1)는 처음부터 다시 학습하지 않습니다. LoRA(Low-Rank Adaptation)로 target embodiment의 시연 데이터에만 가볍게 적응시킵니다. 여러 카메라 뷰는 2×2 그리드로 이어붙여 한 장처럼 처리합니다.

## 이 개념이 풀고자 했던 문제

로봇 정책이 새로운 행동이나 새로운 환경으로 일반화하려면 그만큼 다양한 시연 데이터가 필요한데, teleoperation은 embodiment 하나, 환경 하나, 행동 하나를 모을 때마다 사람 손이 들어가는 선형 비용입니다. DreamGen은 이 비용을 실제 로봇 대신 영상 생성 모델에 떠넘길 수 있는지 묻습니다.

## 핵심 아이디어

결과는 뚜렷합니다. 씨앗은 pick-and-place 시연 2,884개, 단일 환경뿐인데도, 합성 영상으로 학습한 정책이 처음 보는 14개 행동에서 평균 43.2% 성공률(기준선 11.2%)을, 처음 보는 10개 환경에서 28.5% 성공률(기준선 0%)을 냈습니다. 실제 로봇을 더 움직이지 않고도 행동 다양성과 환경 다양성 둘 다 늘릴 수 있다는 것을 실측으로 보인 겁니다. 다만 저자들이 스스로 구분하듯, 이건 정책 학습·역동역학·순동역학을 한데 묶어 쓰던 이전 접근과 달리, 이 세 요소를 의도적으로 분리해서 각각 최신 영상 생성 모델의 성능을 최대한 활용하려는 설계이기도 합니다.

## taskcraft 아이디어와의 접목

> 이 편의 역할은 taskcraft의 실제 파일럿 설계와 정확히 대비를 만드는 것입니다. DreamGen이 하는 일 전체(영상 생성 모델을 target embodiment에 맞춰 LoRA로 적응시키고, 그걸로 새 영상을 합성하고, 거기서 다시 행동을 추출하는 파이프라인)는 taskcraft가 Genie 수준의 실제 학습은 7주 1인 프로젝트로는 불가능하다고 판단하며 명시적으로 포기한 그 규모입니다. taskcraft의 현재 파일럿은 이 전체 파이프라인 대신 R3M/VIP처럼 이미 공개된 frozen encoder만 얼린 채 갖다 쓰는 축소판입니다. 영상을 생성하지도, 어떤 모델도 finetune하지 않습니다.
>
> 이 대비가 왜 중요한가 하면, DreamGen이 실제로 검증하는 것도 다른 embodiment로의 전이가 아니기 때문입니다. LoRA 적응은 target embodiment 자기 자신의 데이터로 이뤄지고, 합성되는 것도 그 로봇 자신의 새 행동·새 환경입니다. 즉 DreamGen은 cross-embodiment 계열로 분류되곤 하지만, 실제 기여는 한 embodiment 안에서 데이터를 어떻게 늘릴 것인가에 가깝습니다. taskcraft가 겨냥하는 형태가 근본적으로 다른 embodiment 간 전이와는 질문 자체가 다릅니다.

## 한계 / 아직 안 풀린 문제

- 영상 생성 모델의 그럴듯함이 실제 물리적 타당성(plausibility)을 보장하지 않습니다. 생성된 상호작용이 그 embodiment의 물리 제약에 맞는지 검증하는 절차가 여전히 필요합니다.
- LoRA 적응과 IDM 학습 둘 다 target embodiment의 실제 teleoperation 데이터를 전제합니다. 데이터가 원천적으로 희귀한 embodiment(재난구조 로봇 등)에는 이 파이프라인의 출발점 자체가 없습니다.
- IDM을 기본값으로 채택한 이유가 이미 teleoperation 데이터가 있어서라는 점은, 정말 데이터가 전혀 없는 상황에서는 LAPA 계열(4편 Genie와 같은 축)로 되돌아가야 함을 뜻합니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| DreamGen (NVIDIA 외, 2025) | 영상 생성 모델을 target embodiment에 적응시켜 합성 궤적(neural trajectory)을 만들고, IDM 또는 LAPA로 pseudo-action을 추출 |
| Genie (4편) | LAPA의 잠재 행동 추출 방식이 계승한 원류(라벨 없는 비지도 잠재 행동) |
| R3M / VIP (5, 6편) | DreamGen과 같은 인간·로봇 영상 기반 계열로 묶이지만, 표현·보상만 넘기는 R3M/VIP와 달리 DreamGen은 데이터 자체(합성 궤적)를 넘긴다는 점이 다름 |

## 다음 글

8편은 GR-1 → GR-2입니다. DreamGen이 기존 정책 학습 파이프라인은 그대로 두고 데이터만 합성으로 늘린다였다면, GR-1/GR-2는 반대로 대규모 영상 사전학습과 로봇 파인튜닝을 하나의 모델 안에 통째로 묶습니다.

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
