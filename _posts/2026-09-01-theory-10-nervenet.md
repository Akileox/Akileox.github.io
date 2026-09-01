---
title: "[이론] 10. NerveNet — 몸 구조를 그래프로 공유한다"
date: 2026-08-23
categories: [Project]
tags: [taskcraft-theory, graph-neural-network, nervenet, morphology, robotics]
excerpt: "에이전트의 몸을 운동학 그래프로 표현하고, 관절 종류별로 가중치를 공유하는 GNN 정책이 왜 관절 수가 다른 몸에도 zero-shot으로 적용되는지를 정리하고, taskcraft의 디코더 설계 질문에 어떤 대안을 주는지 짚습니다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 10편입니다. 9편(Genie Envisioner)에서 이어집니다.</em></p>

<p class="post-series-note" markdown="1"><em>이 편은 원 논문(Wang, Liao, Ba, Fidler, ICLR 2018) 원문 대신, 후속 연구(MetaMorph, 2022)의 재구현 서술과 널리 알려진 공개 자료를 근거로 정리했습니다. 세부 하이퍼파라미터는 원문으로 다시 확인이 필요할 수 있습니다.</em></p>

## 정의

4~9편은 전부 영상에서 무엇을 뽑아 공유할 것인가였습니다. NerveNet(2018)은 완전히 다른 질문입니다. 영상도, 사전학습도 없습니다. 대신 에이전트의 몸 구조 자체를 그래프로 표현하고, 그 그래프 위에서 동작하는 하나의 정책 네트워크(GNN)를 학습시켜서, 관절 수나 팔다리 개수가 다른 몸에도 같은 정책이 곧바로(zero-shot) 적용되게 만듭니다.

<figure class="tp-fig">
<svg viewBox="0 0 780 260" role="img" aria-label="에이전트의 몸을 운동학 트리로 표현하면 노드는 관절, 엣지는 물리적 연결이 된다. 같은 종류의 관절끼리 가중치를 공유하는 메시지 패싱을 몇 차례 돌린 뒤 관절별로 행동을 출력한다. 이 가중치는 관절 수가 달라져도 재사용할 수 있어 새 몸 구조에 zero-shot으로 적용된다.">
  <defs>
    <marker id="r10Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
  </defs>

  <text x="30" y="22" font-size="11" style="fill:var(--text-muted)">에이전트를 그래프로: 노드 = 관절, 엣지 = 물리적 연결(운동학 트리)</text>

  <circle cx="110" cy="70" r="26" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="110" y="74" text-anchor="middle" font-size="10.5" fill="currentColor">root</text>

  <circle cx="60" cy="140" r="24" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="60" y="144" text-anchor="middle" font-size="10" fill="currentColor">관절1</text>
  <circle cx="160" cy="140" r="24" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="160" y="144" text-anchor="middle" font-size="10" fill="currentColor">관절2</text>
  <circle cx="30" cy="205" r="22" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="30" y="209" text-anchor="middle" font-size="9.5" fill="currentColor">1-1</text>
  <circle cx="190" cy="205" r="22" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="190" y="209" text-anchor="middle" font-size="9.5" fill="currentColor">2-1</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.75">
    <line x1="94" y1="90" x2="70" y2="120"/>
    <line x1="126" y1="90" x2="150" y2="120"/>
    <line x1="52" y1="162" x2="35" y2="187"/>
    <line x1="168" y1="162" x2="185" y2="187"/>
  </g>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="240" y1="140" x2="290" y2="140" marker-end="url(#r10Arrow)"/>
  </g>

  <rect x="292" y="108" width="200" height="64" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="392" y="132" text-anchor="middle" font-size="11.5" fill="currentColor">메시지 패싱</text>
  <text x="392" y="150" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">같은 종류 관절끼리</text>
  <text x="392" y="165" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">가중치 공유</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="494" y1="140" x2="530" y2="140" marker-end="url(#r10Arrow)"/>
  </g>

  <rect x="532" y="108" width="220" height="64" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="642" y="132" text-anchor="middle" font-size="11.5" fill="currentColor">관절별 행동 출력</text>
  <text x="642" y="150" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">(같은 출력 함수를</text>
  <text x="642" y="165" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">관절마다 적용)</text>

  <line x1="15" y1="200" x2="765" y2="200" style="stroke:var(--accent)" stroke-width="1.2" stroke-dasharray="6 4"/>
  <text x="642" y="222" text-anchor="middle" font-size="10.5" style="fill:var(--accent)">관절 수가 달라져도 같은 가중치 재사용 → 새 몸 구조에 zero-shot 적용</text>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> 운동학 트리의 각 관절이 노드가 되고, 같은 종류의 관절은 메시지 패싱·출력 함수 가중치를 공유합니다. 다리가 하나 늘거나 줄어도 같은 함수를 그만큼 더/덜 적용하면 됩니다.</figcaption>
</figure>

표준 MLP 정책은 모든 관절의 관측을 한 벡터로 이어붙여 받고, 관절 수가 바뀌면 입력·출력 차원 자체가 바뀌어서 네트워크를 다시 만들어야 합니다. NerveNet은 각 관절을 노드로 두고, 그 관절의 지역 관측만 그 노드에 입력합니다. 메시지 패싱을 몇 차례 돌려 이웃 관절의 정보를 전달받은 뒤, 각 노드가 자기 관절의 행동을 출력합니다. 같은 종류의 관절(예: 다리 관절)은 전부 같은 가중치를 씁니다. 다리가 넷인 몸이든 여섯인 몸이든, 다리 관절 각각에 같은 함수를 반복 적용하면 그만입니다.

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

## 핵심 아이디어

핵심은 가중치 공유의 단위가 관절의 종류이지 그 몸 전체가 아니라는 점입니다. 메시지 함수, 업데이트 함수, 출력 함수는 관절 종류별로 하나씩만 존재하고, 그래프 안에 그 종류의 관절이 몇 개 있든 같은 함수를 반복해서 적용합니다. 이건 합성곱 신경망이 이미지 위치와 무관하게 같은 필터를 공유하는 것과 같은 원리를, 공간이 아니라 그래프의 노드 타입 위에 적용한 것입니다. 몸 구조가 바뀌어도(다리 하나가 없어지거나 늘어나도) 함수 자체는 그대로이고, 그래프에 그 함수를 적용하는 횟수만 달라집니다. 이게 왜 zero-shot 구조 전이가 가능한지의 핵심 이유입니다. 학습된 게 이 몸을 위한 정책이 아니라 이런 종류의 관절이 주어졌을 때 어떻게 반응할지이기 때문입니다.

표준 RL 정책(MLP)은 학습이 끝나면 그 몸 구조 하나에 고정됩니다. 로봇의 다리 하나가 고장 나거나, 팔 길이가 조금 바뀌거나, 관절이 하나 늘어나면 정책을 처음부터 다시 학습해야 합니다. NerveNet은 몸 구조가 애초에 그래프(운동학 트리)라는 자연스러운 구조를 갖고 있다는 사실을 정책 네트워크 설계에 직접 반영해서, 이 재학습 비용을 줄이려 합니다. 논문은 이를 size transfer(같은 구조, 다른 크기)와 disability transfer(관절 일부가 제거된 구조)로 나눠 검증했고, MLP 기반 정책보다 훨씬 잘 전이되며 zero-shot 설정에서도 동작한다고 보고합니다. 표준 MuJoCo 벤치마크에서의 기본 성능도 기존 방법과 비등한 수준을 유지합니다.

## taskcraft 아이디어와의 접목

> NerveNet이 다루는 몸 구조 변형(다리 추가·제거, 크기 변화)은 전부 같은 설계 공간(다리 달린 로봇) 안에서의 변형이지, 다리 달린 로봇과 날개 달린 embodiment처럼 근본적으로 다른 집합 사이의 전이가 아닙니다.
>
> 그럼에도 이 편이 taskcraft에 주는 건 구체적입니다. 공유 latent를 embodiment별로 어떻게 연결할 것인가에 대해, 가중치를 embodiment 자체가 아니라 embodiment를 구성하는 부품의 종류에 공유한다는 아이디어를 NerveNet이 실제로 구현하고 있습니다. taskcraft가 언젠가 로봇 팔·다리 같은 관절 기반 embodiment로 파일럿을 확장한다면, 이 패턴이 직접적인 참고가 됩니다. 지금 파일럿(Minecraft 안 embodiment 교체)에는 관절 종류에 해당하는 구조가 없어서 당장 적용되지는 않지만, 공유 latent를 embodiment의 부분 구조 단위로 나눠 매칭한다는 발상 자체는 taskcraft의 핵심 질문(어떤 정보가 embodiment-specific이고 어떤 정보가 embodiment-invariant한가)과 같은 방향입니다.

## 한계 / 아직 안 풀린 문제

- 노드 타입을 미리 정의해야 합니다(어떤 관절이 같은 종류인지). 이 매핑 자체가 사람이 설계하는 부분이라, embodiment가 아예 다른 종류의 몸(다리 vs 날개)이면 애초에 같은 종류의 노드라는 개념부터 다시 정의해야 합니다.
- 그래프 구조(운동학 트리)가 다른 embodiment로 전이하는 실험은 전부 같은 기본 구조(다리 달린 생물형) 변형 안에서 이뤄졌습니다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| NerveNet (Wang, Liao, Ba, Fidler, ICLR 2018) | 에이전트 몸을 운동학 그래프로 표현하고, 관절 종류별로 가중치를 공유하는 GNN 정책 |
| MetaMorph (11편, 다음 편) | NerveNet의 GNN 방식과 정면으로 비교되는, Transformer self-attention 기반 접근 |

## 다음 글

11편은 MetaMorph입니다. NerveNet이 그래프와 메시지 패싱으로 구조를 공유했다면, MetaMorph는 같은 문제를 Transformer의 self-attention으로 풉니다.

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
