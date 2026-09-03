---
title: "[이론] 11. MetaMorph"
date: 2026-08-27
categories: [Project]
tags: [taskcraft-theory, transformer, metamorph, morphology, reinforcement-learning, robotics]
excerpt: "몸 구조에 대한 사전 지식을 사람이 미리 그래프로 정해주지 않고 모델이 스스로 알아내게 할 수 있을까라는 질문에서 출발해, 로봇 모듈을 트랜스포머 토큰으로 넣어 self-attention이 몸 구조 관계를 학습하게 만드는 MetaMorph를 NerveNet과 대비해 정리한다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 11편, 이론 파트의 마지막 편이다. 10편(NerveNet)에서 이어진다.</em></p>

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

## 몸 구조에 대한 사전 지식 없이도 공유할 수 있는가

10편 끝에서 던진 질문이다. NerveNet은 관절 종류별 가중치 공유로 zero-shot 전이를 얻어냈지만, 대가가 있었다. "어떤 관절이 같은 종류인가"라는 그래프를 사람이 미리 정의해줘야 한다는 것이다. embodiment가 정말 다양해지면(다리 개수도, 팔 길이도, 심지어 팔인지 다리인지도 제각각이면) 이 매핑 자체를 사람이 손으로 계속 관리하기 어려워진다.

MetaMorph(Gupta, Fan, Ganguli, Fei-Fei, ICLR 2022)는 같은 문제(다른 몸 구조에 하나의 정책 일반화)를 다른 도구로 푼다. 몸을 이루는 각 모듈(관절)을 **트랜스포머의 토큰 하나**로 취급하고, self-attention이 모듈 사이의 관계를 스스로 배우게 한다. 저자들의 표현을 그대로 쓰면, 로봇 형태는 트랜스포머의 출력을 조건화할 수 있는 또 다른 모달리티일 뿐이다.

![모듈형 로봇 설계 공간에서 공동 사전학습으로 MetaMorph를 학습시켜, 처음 보는 dynamics·kinematics·task 변화에 zero-shot으로 일반화시키는 전체 구조](/assets/images/posts/theory-11-metamorph/figure-1.png)

논문 Figure 1이다. 왼쪽 UNIMAL 설계 공간(다양한 모듈형 로봇들)으로 MetaMorph를 공동 사전학습(joint pretraining)시키면, 오른쪽처럼 학습 때 본 적 없는 dynamics 변화(관절 감쇠, 무게), kinematics 변화(자유도, 새 형태), 새로운 태스크로도 zero-shot 일반화가 된다는 게 이 논문 전체의 주장이다. **하나의 정책이 서로 다른 로봇 100종을 동시에 배우고, 그 배운 것이 처음 보는 101번째 몸에도 옮겨간다.**

## 직관: 몸을 토큰 시퀀스로 펼친다

운동학 트리를 깊이 우선 순회(depth-first traversal)해서 모듈들을 일렬로 늘어놓고, 각 모듈의 관측(고유수용감각 + 그 모듈이 어떤 형태인지)을 선형 투영해 토큰 하나로 만든다. 몸 구조가 달라지면 토큰 개수와 순서가 달라질 뿐, **트랜스포머 자체의 파라미터는 그대로**다. 이 토큰 시퀀스가 self-attention 층들을 통과하면서 모듈 사이 관계(어깨가 다리에 미치는 영향 등)를 배우고, 마지막에 카메라 같은 외수용감각 정보와 합쳐져 모듈별 행동 분포를 낸다. 여기까지는 표준 Vision Transformer 인코더와 거의 같은 구조다. 다른 건 입력 토큰이 이미지 패치가 아니라 몸의 모듈이라는 것뿐이다.

## 필요한 만큼만 수학

모듈 임베딩은 관측을 선형 투영한 뒤 위치 임베딩을 더해서 만든다.

$$
\mathbf m_0 = [\phi(s^k_{l_1};\mathbf W_e); \cdots; \phi(s^k_{l_N};\mathbf W_e)] + \mathbf W_{\text{pos}}
$$

\\( \phi \\)는 임베딩 함수, \\( \mathbf W_e \\)는 그 가중치, \\( \mathbf W_{\text{pos}} \\)는 학습된 위치 임베딩이다. 이후 \\( L \\)개 층에서 표준 트랜스포머 블록을 반복한다.

$$
\mathbf m'_\ell = \text{MSA}(\text{LN}(\mathbf m_{\ell-1})) + \mathbf m_{\ell-1}, \qquad \mathbf m_\ell = \text{MLP}(\text{LN}(\mathbf m'_\ell)) + \mathbf m'_\ell, \qquad \ell=1\ldots L
$$

MSA는 multi-head self-attention, LN은 layer norm이다. 행동은 인코더 출력과 외수용감각 정보를 결합해서 낸다.

$$
\mathbf g = \gamma(s^k_g;\mathbf W_g), \qquad \mu(\mathbf s^k) = \phi(\mathbf m_{l_i}, \mathbf g; \mathbf W_d), \qquad \pi_\theta(a^k_t \mid s^k_t) = \mathcal N\big(\{\mu(s^k_i)\}_i^N, \Sigma\big)
$$

\\( \gamma \\)는 카메라 등 고차원 입력을 압축하는 별도 MLP이고, 이걸 각 모듈의 인코더 출력 \\( \mathbf m_{l_i} \\)과 이어붙여 그 모듈의 행동 평균 \\( \mu \\)를 낸다. 공분산 \\( \Sigma \\)는 고정이다. 가치 함수도 같은 방식으로, 모듈별 가치를 낸 뒤 평균 내서 전체 몸에 대한 값을 얻는다.

<span class="aside">(여러 몸을 한 번에 학습시키면 안정적인 로봇이 더 많은 경험을 쌓아 더 빨리 학습되는 부익부 현상이 생긴다. 이걸 막기 위해 로봇 k를 샘플링할 확률을 성능이 낮을수록 높이는 동적 리플레이 버퍼 균형을 쓴다: \\( P_k = \mathcal E_k^\beta / \sum_i \mathcal E_i^\beta \\), 여기서 \\( \mathcal E_k \\)는 에피소드 길이가 짧을수록(성능이 낮을수록) 커지는 지표다.)</span>

## 논문에서 어떻게 확장되는가

UNIMAL 설계 공간(Gupta et al. 2021)만 해도 로봇 형태가 \\( 10^{18} \\)가지가 넘는다. 형태마다 정책을 따로 학습하는 건 불가능하다. UNIMAL에서 뽑은 로봇 100종을 평평한 지형, 변화하는 지형, 장애물 세 환경에서 함께 학습시킨 결과, MetaMorph는 로봇별로 따로 학습한 MLP(사실상의 성능 상한선)에 근접하는 평균 보상을 냈고, 같은 조건에서 GNN(NerveNet을 이 실험 세팅에 맞게 재구현한 베이스라인)을 꾸준히 앞섰다. 동적 리플레이 버퍼 균형을 빼면 로봇의 10~15%가 MLP 상한선의 75%에도 못 미치는 성능을 보여, 이 균형 장치가 일부 몸만 잘 배우는 문제를 실제로 막는다는 것도 확인된다. 사전학습된 단일 정책은 학습 때 본 적 없는 동역학·형태·태스크 변화에도 zero-shot으로 어느 정도 일반화하고, 새로운 형태·태스크에는 적은 데이터로 빠르게 파인튜닝된다.

## taskcraft와의 접목

10편에서 이미 짚었듯, NerveNet과 MetaMorph는 둘 다 닮은 집합(UNIMAL이라는 같은 설계 공간) 안에서의 일반화다. 이 편에서 더할 지점은 **아키텍처 선택의 실질적 차이**다. NerveNet은 그래프의 물리적 인접 구조(관절이 실제로 연결된 관계)를 메시지 패싱의 경로로 강제한다. MetaMorph는 그 구조를 명시적으로 강제하지 않고, self-attention이 어떤 모듈과 어떤 모듈이 관련 있는지 **학습으로 알아내게** 둔다. MetaMorph 논문이 학습된 attention mask를 분석해 운동 시너지(motor synergy)가 나타난다고 보고하는 것도, 이 관계를 사람이 미리 정해주지 않아도 트랜스포머가 스스로 찾아낼 수 있다는 근거다.

이건 공유 latent를 embodiment별로 어떻게 연결할 것인가에 두 가지 서로 다른 답(**그래프 구조를 사람이 지정 vs 어텐션이 스스로 학습**)을 제공한다. taskcraft가 나중에(겉날개·보트를 넘어) 관절 기반 embodiment로 파일럿을 확장한다면, embodiment 구조를 인코더에 어떻게 알려줄 것인가를 결정할 때 이 두 축 사이에서 골라야 한다. 다만 지금 파일럿(Minecraft 내 embodiment 교체)에는 이런 모듈형 구조 자체가 없어서, 이번 파일럿에는 직접 적용되지 않는다.

## 한계 / 아직 안 풀린 문제

- UNIMAL 100종 전부 같은 설계 공간(구, 원기둥 모듈을 관절로 연결)에서 나온다. 완전히 다른 종류의 몸(바퀴, 날개, 무한궤도)이 이 토큰화 방식에 자연스럽게 들어가는지는 검증되지 않았다.
- 위치 임베딩이 깊이 우선 순회 순서에 의존한다. 같은 몸이라도 순회 순서를 다르게 잡으면 다른 토큰 시퀀스가 되는데, 이 임의성이 학습에 미치는 영향은 명확히 다뤄지지 않았다.

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| MetaMorph (Gupta, Fan, Ganguli, Fei-Fei, ICLR 2022) | 로봇 모듈을 트랜스포머 토큰으로 인코딩해 형태 간 일반화를 학습 |
| NerveNet (10편) | 같은 문제를 GNN 메시지 패싱으로 푸는 대안. MetaMorph 논문이 직접 베이스라인으로 재구현해 비교 |
| UNIMAL (Gupta et al., 2021) | MetaMorph가 쓰는 모듈형 로봇 설계 공간 |

## 다음 글로 넘어가기 전에

- 10, 11편이 함께 보여준 것: embodiment 구조를 공유하는 방법은 "사람이 명시적으로 지정"(NerveNet)과 "모델이 데이터로 알아냄"(MetaMorph) 둘 다 가능하고, 실제로는 후자가 GNN 베이스라인을 앞섰다.
- 0편(VAE)부터 여기까지, 이 시리즈는 크게 두 갈래를 훑었다. 4~9편은 인코더(무엇을 지각하고 무엇을 보상으로 삼을지), 10~11편은 디코더(그걸 받아 어떻게 몸마다 다르게 풀어낼지). 그런데 taskcraft의 실제 파일럿(Minecraft, 겉날개 embodiment)은 이 두 갈래 어디에도 정확히 들어맞지 않는다. 관절 그래프도 없고, 로봇 팔도 아니다.

**지금까지 훑은 이론(압축, world model, 잠재 행동, 사람 표현 이식, 데이터/플랫폼 통합, 몸 구조 공유) 중 정확히 무엇이 taskcraft의 실제 가설을 뒷받침하고, 무엇이 아직 검증되지 않은 채 남는가?** 12편이 이 질문에 답하며 이론 파트를 가설로 수렴시킨다.

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
