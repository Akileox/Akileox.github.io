---
title: "[이론] 10. NerveNet"
date: 2026-08-23
categories: [Project]
tags: [taskcraft-theory, graph-neural-network, nervenet, morphology, robotics]
excerpt: "지금까지는 인코더가 영상에서 무엇을 뽑을지만 물었다. NerveNet은 축을 바꿔 몸 구조 자체를 그래프로 표현하고, 관절 종류별로 가중치를 공유하는 정책이 왜 관절 수가 다른 몸에도 zero-shot으로 적용되는지를 짚고 taskcraft의 디코더 설계 질문에 대안을 준다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 10편이다. 9편(Genie Envisioner)에서 이어진다.</em></p>

<p class="post-series-note" markdown="1"><em>이 편은 원 논문(Wang, Liao, Ba, Fidler, ICLR 2018) 원문을 근거로 정리했다. 세부 하이퍼파라미터 일부는 후속 연구(MetaMorph, 2022)의 재구현 서술도 함께 참고했다.</em></p>

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
</figure>

## 인코더 말고 디코더는 어떻게 공유하는가

9편 끝에서 던진 질문이다. 4~9편은 전부 "영상에서 무엇을 뽑아 공유할 것인가"였다. 인코더가 embodiment 정보를 덜 담게 만드는 방법은 계속 물었지만, **디코더**(그 표현을 받아 실제로 행동을 어떻게 다른 몸에 풀어내는가) 쪽은 R3M/VIP도 GE도 결국 "로봇마다 새로 학습한다"는 답으로 수렴했다.

NerveNet(2018)은 이 디코더 쪽 질문에 정면으로 다른 답을 낸다. 영상도, 사전학습도 없다. 대신 **에이전트의 몸 구조 자체를 그래프로 표현**하고, 그 그래프 위에서 동작하는 하나의 정책 네트워크(GNN)를 학습시켜서, 관절 수나 팔다리 개수가 다른 몸에도 같은 정책이 곧바로(zero-shot) 적용되게 만든다.

## 직관: 관절 종류가 가중치 공유의 단위다

표준 MLP 정책은 모든 관절의 관측을 한 벡터로 이어붙여 받는다. 관절 수가 바뀌면 입력·출력 차원 자체가 바뀌어서 네트워크를 다시 만들어야 한다. NerveNet은 각 관절을 노드로 두고, 그 관절의 지역 관측만 그 노드에 입력한다. 메시지 패싱을 몇 차례 돌려 이웃 관절의 정보를 전달받은 뒤, 각 노드가 자기 관절의 행동을 출력한다. **같은 종류의 관절(예: 다리 관절)은 전부 같은 가중치를 쓴다.** 다리가 넷인 몸이든 여섯인 몸이든, 다리 관절 각각에 같은 함수를 반복 적용하면 그만이다.

![Ant 로봇의 실제 몸(왼쪽)과 그걸 그대로 옮긴 운동학 그래프(오른쪽). Root-Hip-Ankle 노드가 색으로 대응된다](/assets/images/posts/theory-10-nervenet/figure-1.png)

논문 Figure 2다. 왼쪽은 MuJoCo의 Ant 로봇 몸이고, 오른쪽은 그 몸을 그대로 옮긴 운동학 그래프다. 몸통(Root, 빨강)에 다리 네 개가 붙어 있고, 각 다리는 Hip(보라)과 Ankle(초록) 두 관절로 이어진다. 색이 같은 노드는 정확히 같은 파라미터를 공유하는 관절 종류를 뜻한다. **핵심은 가중치 공유의 단위가 관절의 "종류"이지 그 몸 전체가 아니라는 것이다.** 메시지 함수, 업데이트 함수, 출력 함수는 관절 종류별로 하나씩만 존재하고, 그래프 안에 그 종류의 관절이 몇 개 있든 같은 함수를 반복해서 적용한다.

이건 합성곱 신경망이 이미지 위치와 무관하게 같은 필터를 공유하는 것과 같은 원리를, **공간이 아니라 그래프의 노드 타입 위에** 적용한 것이다. 몸 구조가 바뀌어도(다리 하나가 없어지거나 늘어나도) 함수 자체는 그대로이고, 그래프에 그 함수를 적용하는 횟수만 달라진다. 학습된 게 "이 몸을 위한 정책"이 아니라 "이런 종류의 관절이 주어졌을 때 어떻게 반응할지"이기 때문에, 그래프 구조만 바뀐 새 몸에도 같은 함수를 그대로 다시 배치할 수 있다.

## 필요한 만큼만 수학

표준 RL 정책(MLP)은 학습이 끝나면 그 몸 구조 하나에 고정된다. 로봇의 다리 하나가 고장 나거나, 팔 길이가 조금 바뀌거나, 관절이 하나 늘어나면 정책을 처음부터 다시 학습해야 한다. NerveNet은 몸 구조가 애초에 그래프(운동학 트리)라는 자연스러운 구조를 갖고 있다는 사실을 정책 네트워크 설계에 직접 반영해서, 이 재학습 비용을 줄이려 한다.

논문은 이를 두 가지로 나눠 검증했다. **size transfer**(같은 구조, 다른 크기, 예: 다리 길이나 개수 변화)와 **disability transfer**(관절 일부가 제거된 구조)다. MLP 기반 정책보다 훨씬 잘 전이되며, zero-shot 설정(전이 대상 몸으로 추가 학습을 전혀 하지 않은 상태)에서도 동작한다고 보고한다. 표준 MuJoCo 벤치마크에서의 기본 성능도 기존 방법과 비등한 수준을 유지한다.

## taskcraft와의 접목

NerveNet이 다루는 몸 구조 변형(다리 추가·제거, 크기 변화)은 전부 같은 설계 공간(다리 달린 로봇) 안에서의 변형이지, 다리 달린 로봇과 날개 달린 embodiment처럼 근본적으로 다른 집합 사이의 전이가 아니다.

그럼에도 이 편이 taskcraft에 주는 건 구체적이다. 공유 latent를 embodiment별로 어떻게 연결할 것인가에 대해, **가중치를 embodiment 자체가 아니라 embodiment를 구성하는 부품의 종류에 공유한다**는 아이디어를 NerveNet이 실제로 구현하고 있다. taskcraft가 언젠가 로봇 팔·다리 같은 관절 기반 embodiment로 파일럿을 확장한다면, 이 패턴이 직접적인 참고가 된다. 지금 파일럿(Minecraft 안 embodiment 교체)에는 관절 종류에 해당하는 구조가 없어서 당장 적용되지는 않지만, 공유 latent를 embodiment의 부분 구조 단위로 나눠 매칭한다는 발상 자체는 taskcraft의 핵심 질문(어떤 정보가 embodiment-specific이고 어떤 정보가 embodiment-invariant한가)과 같은 방향이다.

## 한계 / 아직 안 풀린 문제

- 노드 타입을 미리 정의해야 한다(어떤 관절이 같은 종류인지). 이 매핑 자체가 사람이 설계하는 부분이라, embodiment가 아예 다른 종류의 몸(다리 vs 날개)이면 애초에 "같은 종류의 노드"라는 개념부터 다시 정의해야 한다.
- 그래프 구조(운동학 트리)가 다른 embodiment로 전이하는 실험은 전부 같은 기본 구조(다리 달린 생물형) 변형 안에서 이뤄졌다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| NerveNet (Wang, Liao, Ba, Fidler, ICLR 2018) | 에이전트 몸을 운동학 그래프로 표현하고, 관절 종류별로 가중치를 공유하는 GNN 정책 |
| MetaMorph (11편, 다음 편) | NerveNet의 GNN 방식과 정면으로 비교되는, Transformer self-attention 기반 접근 |

## 다음 글로 넘어가기 전에

- NerveNet이 새로 보인 것: 디코더도 embodiment 전체가 아니라 **부품의 종류** 단위로 공유하면, 관절 수가 다른 몸에도 zero-shot 전이가 가능하다.
- 그런데 이 방식은 "어떤 관절이 같은 종류인가"를 사람이 미리 그래프로 정의해줘야 한다. 이 사전 지식이 없다면?

**몸 구조에 대한 사전 지식(그래프)을 사람이 미리 정해주지 않고, 모델이 데이터로부터 스스로 몸 구조 사이의 관계를 알아내게 만들 수는 없을까?** 11편(MetaMorph)이 이 질문에서 시작한다.

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
