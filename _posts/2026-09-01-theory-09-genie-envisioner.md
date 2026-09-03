---
title: "[이론] 9. Genie Envisioner"
date: 2026-08-19
categories: [Project]
tags: [taskcraft-theory, world-model, genie-envisioner, flow-matching, robotics]
excerpt: "정책 학습뿐 아니라 시뮬레이션과 평가까지 하나의 영상 생성 world model로 통합할 수 있는가라는 질문에서 출발해, GE-Base/GE-Act/GE-Sim 세 컴포넌트로 이걸 실제로 구현한 Genie Envisioner를 정리하고 새 embodiment로 갈 때 무엇을 공유하고 무엇을 새로 배우는지 짚는다."
---

<p class="post-series-note" markdown="1"><em>taskcraft <a href="/taskcraft/">연구 노트</a>를 뒷받침하는 [이론] 시리즈 9편이다. 8편(GR-1 → GR-2)에서 이어진다.</em></p>

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

## 정책·시뮬레이션·평가를 하나로 묶을 수 있는가

8편 끝에서 던진 질문이다. 지금까지 나온 world model은 각자 한 가지 역할만 했다. R3M/VIP는 표현·보상, DreamGen은 데이터 합성, GR-1/GR-2는 정책 그 자체였다. 그런데 실제 로봇 연구에서 정책 학습, 시뮬레이션, 평가는 보통 서로 다른 도구다. 정책은 강화학습·모방학습 프레임워크로, 시뮬레이션은 물리 엔진(MuJoCo 등)으로, 평가는 실제 로봇이나 별도 벤치마크로 한다. 이 세 도구를 각각 만들고 맞추는 것 자체가 비용이다.

Genie Envisioner(GE, 2025)는 셋 다 **"다음 영상이 어떻게 이어지는가"를 예측하는 하나의 world model**에서 파생될 수 있다고 본다.

## 직관: 하나의 world model에서 두 갈래로 갈라진다

GE는 이 역할들을 GE-Base(world model), GE-Act(action decoder), GE-Sim(시뮬레이터) 세 컴포넌트로 나누되, 셋 다 같은 영상 생성 world model 하나에서 파생시킨다. GE-Base가 AgiBot-World-Beta(약 100만 에피소드)로 "다음 영상 청크가 어떻게 이어질지"를 배우고 나면, 두 갈래로 쓴다. GE-Act는 이 world model의 잠재 표현을 실제 관절 궤적으로 바꾸고, GE-Sim은 반대로 행동을 조건으로 받아 다음 관측 영상을 생성해서 정책을 오프라인으로 평가하는 폐루프 시뮬레이터 역할을 한다.

<figure class="tp-fig">
<svg viewBox="0 0 780 340" role="img" aria-label="AgiBot G1 데이터로 학습한 GE-Base가 두 갈래로 쓰인다. GE-Act는 이 표현을 실제 관절 궤적으로 바꾸고, GE-Sim은 행동을 조건으로 받아 다음 관측 영상을 생성해 폐루프 시뮬레이션을 만든다. 새 embodiment로 옮길 때는 GE-Base만 적응시키고 GE-Act는 그 로봇 전용으로 처음부터 새로 학습한다.">
  <defs>
    <marker id="r9Arrow" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" fill="currentColor"/>
    </marker>
    <marker id="r9ArrowAccent" viewBox="0 0 10 10" refX="8" refY="5" markerWidth="7" markerHeight="7" orient="auto-start-reverse">
      <path d="M0,0 L10,5 L0,10 z" style="fill:var(--accent)"/>
    </marker>
  </defs>

  <rect x="30" y="20" width="220" height="46" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="140" y="48" text-anchor="middle" font-size="11.5" fill="currentColor">AgiBot-World-Beta (약 100만 에피소드)</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="252" y1="43" x2="288" y2="43" marker-end="url(#r9Arrow)"/>
  </g>

  <rect x="290" y="16" width="200" height="54" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="390" y="38" text-anchor="middle" font-size="12" fill="currentColor">GE-Base</text>
  <text x="390" y="55" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">video diffusion transformer</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <line x1="390" y1="72" x2="390" y2="108" marker-end="url(#r9Arrow)"/>
  </g>
  <g fill="none" style="stroke:var(--accent)" stroke-width="1.4">
    <line x1="410" y1="70" x2="580" y2="200" marker-end="url(#r9ArrowAccent)"/>
  </g>

  <rect x="270" y="110" width="240" height="54" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="390" y="132" text-anchor="middle" font-size="11.5" fill="currentColor">GE-Act (160M, autoregressive)</text>
  <text x="390" y="149" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">flow matching → action trajectory</text>

  <rect x="560" y="150" width="190" height="100" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="655" y="176" text-anchor="middle" font-size="11.5" fill="currentColor">GE-Sim</text>
  <text x="655" y="194" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">action-conditioned</text>
  <text x="655" y="210" text-anchor="middle" font-size="10.5" style="fill:var(--text-muted)">비디오 롤아웃</text>
  <text x="655" y="230" text-anchor="middle" font-size="10" style="fill:var(--text-muted)">Pose2Image / Motion Vector</text>

  <g fill="none" stroke="currentColor" stroke-width="1.3" opacity="0.85">
    <path d="M 512 155 C 540 130, 545 260, 512 240" fill="none" marker-end="url(#r9Arrow)"/>
  </g>
  <text x="518" y="140" font-size="10" style="fill:var(--text-muted)">액션 출력</text>
  <text x="518" y="270" font-size="10" style="fill:var(--text-muted)">생성 관측 재입력</text>

  <line x1="15" y1="280" x2="745" y2="280" style="stroke:var(--accent)" stroke-width="1.2" stroke-dasharray="6 4"/>
  <text x="380" y="273" text-anchor="middle" font-size="10.5" style="fill:var(--accent)">새 embodiment로 확장</text>

  <rect x="30" y="292" width="200" height="40" rx="8" fill="var(--accent-light)" stroke="var(--accent)" stroke-width="1.3"/>
  <text x="130" y="317" text-anchor="middle" font-size="11" style="fill:var(--accent)">GE-Base만 적응(finetune)</text>

  <rect x="260" y="292" width="230" height="40" rx="8" fill="var(--bg)" stroke="currentColor" stroke-width="1.2"/>
  <text x="375" y="317" text-anchor="middle" font-size="11" fill="currentColor">GE-Act는 그 로봇 전용 재학습</text>

  <text x="520" y="317" font-size="11" style="fill:var(--text-muted)">→ Agilex, Dual Franka</text>
</svg>
<figcaption><strong>이 그림이 보여주는 것.</strong> GE-Base 하나에서 GE-Act(행동 생성)와 GE-Sim(시뮬레이션)이 파생된다. 새 로봇으로 옮길 때는 GE-Base만 적응시키고(강조 상자), GE-Act는 그 로봇 전용으로 처음부터 새로 학습한다.</figcaption>
</figure>

다른 로봇(Agilex, Dual Franka)으로 옮길 때는 GE-Base(비디오 생성 부분)만 적응시키고, GE-Act(행동 헤드)는 그 로봇에 맞춰 처음부터 새로 학습한다.

## 필요한 만큼만 수학

GE-Act가 행동을 생성하는 방식은 diffusion이 아니라 **flow matching**이다. Diffusion은 노이즈에서 데이터로 가는 확률적 역과정을 여러 스텝에 걸쳐 반복하며 매 스텝 노이즈를 제거한다. Flow matching은 노이즈 분포와 데이터 분포를 잇는 연속적인 확률 흐름(probability flow)의 속도장을 직접 회귀로 학습해서, 노이즈에서 데이터로 가는 경로를 상대적으로 직선에 가깝게 만든다. 그 결과 같은 품질을 내는 데 필요한 역과정 스텝 수가 diffusion보다 적을 수 있다. GE-Act는 이 성질을 이용해 5단계 디노이징만으로 54스텝 토크 궤적을 200ms 안에 생성한다. <span class="aside">(실시간 로봇 제어에서 지연시간이 병목이라는 걸 고려하면, 왜 diffusion이 아니라 flow matching인가에 대한 실용적 답이 여기 있다.)</span>

GE-Sim의 행동 조건화는 두 계층이다. Pose2Image는 7차원 엔드이펙터 포즈(위치·회전·그리퍼)를 화면 픽셀 좌표에 투영해서 팔마다 다른 색으로 표시하고, Motion Vector는 연속된 포즈 사이의 변화량(\\( \Delta p, \Delta r \\))을 모션 토큰으로 인코딩한다. 두 표현 다 "world model이 이 행동을 하면 다음 장면이 이렇게 바뀐다"를 예측할 때 넣는 조건이다.

GE-Base가 다양한 프레임율(3~30Hz)과 저주파(5Hz) 두 단계로 사전학습되는 이유가 여기서 갈린다. 고주파 학습은 시간적 불변성(다양한 재생 속도에서도 일관된 예측)을, 저주파 학습은 제어에 필요한 만큼의 추상화를 목표로 한다. <span class="aside">(3편 MuZero가 강조했던 "필요한 만큼만 맞추기"와 같은 방향이다.)</span>

## taskcraft와의 접목

이 편이 지금까지 다룬 world model 논문 중 taskcraft의 최종 비전에 가장 가까운 증거를 준다. GE가 AgiBot G1에서 Agilex Cobot Magic, Dual Franka로 확장할 때 실제로 한 일을 보면, 공유 latent를 embodiment별로 어떻게 연결하는가라는 질문에 대한 2025년 최전선의 실제 답이 드러난다. **GE-Base(세계가 어떻게 변하는가)는 새 embodiment로 finetuning되지만, GE-Act(이 변화를 위해 무엇을 해야 하는가)는 재사용되지 않고 그 로봇 전용으로 처음부터 새로 학습된다.** 논문 스스로도 embodiment 간 기본 차이로 인해 사전학습된 액션 디코더를 재사용하지 않고, 새 로봇의 제어 역학에 맞춘 헤드를 학습한다고 밝힌다.

이건 6편(VIP)에서 R3M을 두고 정리했던 구도, **지각은 공유, 행동은 로컬**과 정확히 같은 분업이다. 2025년 시점 가장 통합된 world model 플랫폼조차 행동 생성 부분은 embodiment마다 새로 학습해야 한다는 걸, GE가 다시 한번 확인해준다. 다만 GE가 다루는 세 embodiment(AgiBot G1, Agilex, Dual Franka) 역시 전부 팔+그리퍼 계열이라, 닮은 집합 분류를 벗어나지 않는다. taskcraft가 겨냥하는 형태가 근본적으로 다른 집합에서 이 지각-행동 분업이 그대로 성립할지는 여전히 열린 질문이다.

## 한계 / 아직 안 풀린 문제

- 세 embodiment 모두 로봇 팔 계열이다. GE-Base가 정말 embodiment에 덜 종속적인 표현을 배웠는지, 아니면 팔+그리퍼라는 공유 구조에 암묵적으로 의존하는지 이 논문만으로는 구분할 수 없다.
- GE-Sim이 생성하는 롤아웃의 물리적 타당성이 실제 물리 시뮬레이터만큼 신뢰할 수 있는지는 별도 검증이 필요하다. 영상이 그럴듯해 보이는 것과 실제 동역학이 맞는 것은 다르다(7편 DreamGen의 한계와 같은 문제).

## 관련 개념 / 관련 논문

| 논문 | 기여 |
|---|---|
| Genie Envisioner (AgiBot 외, 2025) | GE-Base(world model)/GE-Act(flow matching 행동 디코더)/GE-Sim(액션 조건부 시뮬레이터)로 정책 학습·시뮬레이션·평가를 통합 |
| GR-1 → GR-2 (8편) | 영상 예측과 행동 예측을 한 트랜스포머로 묶는 접근의 연장선. GE는 여기에 시뮬레이션 역할까지 더함 |
| Flow matching (Lipman et al., 2023) | diffusion의 확률적 역과정 대신 결정론적 속도장을 직접 회귀하는 생성 모델링 방식 |

## 다음 글로 넘어가기 전에

- GE가 새로 보인 것: 2025년 가장 통합된 플랫폼조차 "지각은 공유, 행동은 로컬"이라는 분업을 벗어나지 못한다.
- 4~9편은 전부 영상에서 무엇을 뽑아 공유할 것인가라는 축이었다. 인코더가 어떻게 embodiment 정보를 덜 담을지는 계속 물었지만, **디코더(행동을 어떻게 embodiment별로 다시 풀어내는가)** 쪽은 거의 다루지 않았다.

**축을 바꿔서, embodiment의 몸 구조 자체를 어떻게 표현하면 정책이 다른 몸 구조로 옮겨가는가?** 10편(NerveNet)이 이 질문에서 시작한다.

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
