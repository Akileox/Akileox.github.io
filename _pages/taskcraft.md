---
layout: default
permalink: /taskcraft/
title: "taskcraft"
author_profile: false
classes: wide
---

<div class="tc-wrap">

  <div class="tc-hero">
    <p class="tc-hero__eyebrow">Research Note · In Progress</p>
    <h1 class="tc-hero__title">사람이 하는 걸 보고 배운 로봇은, 왜 다른 로봇에게 그걸 못 알려줄까</h1>
    <p class="tc-hero__dek">
      로봇이나 게임 에이전트가 사람 시연 영상에서 뽑은 표현을 자기 신체 구조(embodiment, 로봇마다 다른 관절·조작
      방식)에 바로 연결하면, 그 표현은 딱 그 로봇에게만 쓸모 있게 된다. Latent world model(관측을 압축해
      세계의 변화를 예측하는 표현)을 매개로 삼으면 이 종속을 끊을 수 있을지, Minecraft에서 값싸게 검증하는
      프로젝트다.
    </p>

    <div class="tc-stepper">
      <div class="tc-stepper__track">
        <div class="tc-stepper__step is-done">
          <div class="tc-stepper__line"></div>
          <div class="tc-stepper__dot">✓</div>
          <div class="tc-stepper__label">환경 구축</div>
        </div>
        <div class="tc-stepper__step is-current">
          <div class="tc-stepper__line"></div>
          <div class="tc-stepper__dot">2</div>
          <div class="tc-stepper__label">인코더 비교</div>
        </div>
        <div class="tc-stepper__step">
          <div class="tc-stepper__line"></div>
          <div class="tc-stepper__dot">3</div>
          <div class="tc-stepper__label">임베딩 검증</div>
        </div>
        <div class="tc-stepper__step">
          <div class="tc-stepper__line"></div>
          <div class="tc-stepper__dot">4</div>
          <div class="tc-stepper__label">정책 비교</div>
        </div>
        <div class="tc-stepper__step">
          <div class="tc-stepper__dot">5</div>
          <div class="tc-stepper__label">결론 정리</div>
        </div>
      </div>
    </div>
  </div>

  <div class="tc-section" id="q">
    <p class="section-label"><span class="section-label__num">01</span> 최종 질문</p>
    <p>
      사람 시연 영상으로 로봇을 학습시키는 방법은 대부분 하나의 구조를 따른다. 영상에서 뽑은 신호를 그
      에이전트 자신의 행동 공간, 즉 관절 각도나 키보드 입력에 곧바로 연결하는 것이다. 이렇게 학습된 표현은
      그 신체가 그 조작을 하는 방식에 묶인다. 형태가 다른 에이전트에는 재사용할 수 없다.
    </p>
    <p>
      질문은 하나다. Latent world model을 매개로 삼아 "세계가 어떻게 바뀌었는가"를 표현하면, 신체 구조에
      덜 종속된 task representation(과제 자체를 나타내는 표현)을 만들 수 있을까. Minecraft는 이 질문을
      값싸게 검증하기 위한 도구지 최종 목표가 아니다.
    </p>
  </div>

  <div class="tc-section" id="why">
    <p class="section-label"><span class="section-label__num">02</span> 왜 지금 이 문제인가</p>
    <p>
      로봇의 형태는 이미 로봇 팔 하나로 수렴하지 않는다. 4족 보행 로봇, 드론, 휴머노이드처럼 근본적으로 다른
      구조가 각자의 용도로 계속 늘어나고 있다. 새 형태가 나올 때마다 시연 데이터를 다시 모으고 파이프라인을
      처음부터 다시 짜야 한다면, 그 비용은 형태 수만큼 그대로 쌓인다.
    </p>
    <p>
      이 비용 문제는 두 갈래로 갈린다. 대량 생산되는 로봇은 teleoperation(사람이 원격으로 로봇을 조작해
      시연 데이터를 만드는 방식) 인력과 실물 대수를 투입해서 데이터 문제를 정면으로 밀어붙일 수 있다.
      하지만 재난구조 로봇처럼 애초에 대량 배치가 불가능한 형태는 돈으로도 해결이 안 된다. 실제 재난
      상황은 재현할 수 없고, 데이터 수집만을 위해 로봇을 위험 지역에 반복 투입하는 것 자체가 현실적이지
      않기 때문이다.
    </p>
    <p>
      Embodiment 무관 표현이 의미를 갖는 지점은 정확히 여기다. 자원을 투입해서 우회할 수 없는 형태에게,
      이미 풍부한 인간 시연 데이터로부터 뭔가를 물려줄 수 있는가.
    </p>
  </div>

  <div class="tc-section" id="diff">
    <p class="section-label"><span class="section-label__num">03</span> 기존 연구와 차이</p>
    <p>
      인간 영상에서 다른 embodiment로 표현을 옮기려는 시도는 이미 있다. R3M과 VIP는 인간 조작 영상에서
      로봇에 전이 가능한 시각 표현을 학습하고, DreamGen과 GR-1·GR-2, Genie 계열은 영상 생성 world model로
      로봇 행동 데이터를 만들어낸다. NerveNet과 MetaMorph는 로봇의 몸 구조를 그래프로 표현해서 하나의
      정책이 여러 형태에 대응하게 만든다.
    </p>
    <p>
      이 연구들이 다루는 형태는 로봇 팔과 그리퍼 계열이거나, 시뮬레이션 안에서 생성된 로봇 형태다. 서로
      어느 정도 관절 구조나 조작 방식을 공유하는 집합, 즉 cross-embodiment(서로 다른 신체 구조 간 전이)라고
      해도 닮은 신체들 사이의 이야기다.
    </p>
    <div class="tc-table-wrap">
      <table class="tc-table">
        <thead><tr><th></th><th>기존 cross-embodiment 연구</th><th>taskcraft</th></tr></thead>
        <tbody>
          <tr>
            <td>다루는 형태</td>
            <td>로봇 팔·그리퍼류, 혹은 시뮬레이션에서 생성된 로봇 형태. 서로 닮은 집합</td>
            <td class="hl">인간, 로봇, 게임 아바타처럼 근본적으로 다른 집합</td>
          </tr>
          <tr>
            <td>전제</td>
            <td>형태들이 어느 정도 공유하는 구조(관절 그래프, 기구학)가 있다고 가정</td>
            <td class="hl">공유 구조가 없어도 통하는 표현을 찾는다</td>
          </tr>
        </tbody>
      </table>
    </div>
    <p class="muted">
      이 차이가 질적으로 다른 문제인지, 닮은 집합을 넓히면 도달하는 정도의 문제인지는 아직 증거가 없다.
      다음 절이 그 증거를 찾는 계획이다.
    </p>
  </div>

  <div class="tc-section" id="plan">
    <p class="section-label"><span class="section-label__num">04</span> 검증 계획</p>
    <p>
      인간 영상만으로 학습된 공개 표현(R3M, VIP)을 얼린 채(추가 학습 없이 고정한 채) Minecraft에 그대로
      붙이고, Minecraft로 학습된 VPT 인코더와 성능을 비교한다. 여기에 embodiment 무관 학습을 목표로 하지
      않은 범용 인코더(CLIP)를 대조군으로 추가한다. R3M·VIP가 CLIP보다 낫다면 embodiment 무관 학습 자체가
      뭔가를 포착한다는 신호고, CLIP과 비슷하다면 이 방법들의 한계일 뿐 일반 명제에 대한 반증은 아니다.
    </p>
    <p>
      R3M·VIP는 로봇의 3인칭·손목 카메라 영상으로 학습됐고 Minecraft는 1인칭 시점이라, 실패해도 embodiment
      차이 때문인지 단순히 시각적 도메인이 달라서인지 완전히 구분되지는 않는다. 그래서 전체 행동 복제(BC)
      학습 전에 얼린 특징 위에 linear probe(추출된 특징 위에 간단한 선형 모델만 얹어 성능을 재는 방법)를
      먼저 붙여 태스크 진행도를 얼마나 예측하는지부터 본다. 더 저렴하고 더 직접적인 신호다.
    </p>
    <p class="muted">순서: VPT·R3M·VIP·CLIP 네 인코더 비교 → linear probe → 신호가 있으면 BC 비교.</p>
  </div>

  <div class="tc-section" id="open">
    <p class="section-label"><span class="section-label__num">05</span> 아직 답하지 못한 질문</p>
    <div class="tc-open">
      <span class="tc-open__tag">핵심 미해결 질문</span>
      <p>
        World model의 latent가 인간 embodiment 자체에 종속될 가능성이 있다. 인간 영상을 아무리 많이
        모아 학습시켜도, latent가 "세계에서 무엇이 변했는가"가 아니라 "인간 몸이 어떻게 그걸 일으켰는가"를
        같이 인코딩할 수 있다. 모델은 이 둘을 구분할 유인이 없다.
      </p>
      <p>
        검증 아이디어: 인간이 나무를 패는 영상과 Minecraft 아바타가 나무를 패는 영상을 같은 인코더에 통과시켜,
        latent가 태스크 기준으로 뭉치는지 embodiment 기준으로 뭉치는지 클러스터링으로 확인한다.
      </p>
    </div>
  </div>

  <div class="tc-section" id="faq">
    <p class="section-label"><span class="section-label__num">06</span> 예상되는 반론</p>

    <details class="tc-faq">
      <summary>이게 될 거였으면 빅테크가 이미 하고 있지 않을까</summary>
      <div class="tc-faq__body">
        <p>닮은 embodiment 집합에서는 이미 하고 있다(DreamGen, GR-1·GR-2). 형태가 근본적으로 다른 집합은
        상업적 규모가 작아서 투자 대비 수익이 낮을 수 있다. 과학적으로 흥미로워도 경제적으로는 덜 끌리는
        지점이다.</p>
        <p>빅테크 내부에서 비공개로 이미 진행 중일 가능성은 배제할 수 없다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>Minecraft는 장난감 도메인인데, 여기 결과가 실제 로봇에 뭘 말해줄 수 있나</summary>
      <div class="tc-faq__body">
        <p>말해주지 못한다. Minecraft는 방법론을 싸게 검증하는 파일럿이고, 연속 제어와 물리 노이즈가 있는
        실제 로봇으로의 일반화는 별도로 검증해야 할 문제다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>그 실험은 기존 방법을 테스트하는 것이지, 이 아이디어 자체를 검증하는 게 아니지 않나</summary>
      <div class="tc-faq__body">
        <p>맞다. 9주, 1인 프로젝트라는 스코프 안에서는 기존 방법이 어디서 깨지는지 확인하는 게 현실적인
        목표다. World model 매개 표현을 직접 구현하고 검증하는 작업은 이후 과제로 남겨둔다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>영상에는 촉각과 힘 정보가 없는데, 그것만으로 충분한가</summary>
      <div class="tc-faq__body">
        <p>충분하지 않을 수 있다. GelSight, DIGIT 같은 촉각 센서 데이터가 실제로 성능을 끌어올린다는
        결과도 있다. 다만 촉각 데이터는 전용 장비로 새로 찍어야만 존재해서, 이미 쌓여 있는 시각 데이터와
        규모가 다르다. 시각 전용 학습이라는 전제 자체의 한계로 남겨둔다.</p>
      </div>
    </details>
  </div>

  <div class="tc-section" id="status">
    <p class="section-label"><span class="section-label__num">07</span> 지금 상태</p>
    <div class="tc-milestones">
      <div class="tc-ms-row"><span>환경 구축 (Windows 네이티브 MineRL)</span><span class="status-badge status-badge--done">완료</span></div>
      <div class="tc-ms-row"><span>인코더 비교 (VPT·R3M·VIP·CLIP + linear probe)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>임베딩 검증 (embodiment 클러스터링 테스트)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>정책 비교 (BC·DAgger·PPO)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>결론 정리</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
    </div>
    <p style="margin-top:0.9rem;">
      <a href="/taskcraft/dashboard/">실험 진행 상황 대시보드 →</a>
    </p>
  </div>

  <div class="tc-section" id="refs">
    <p class="section-label"><span class="section-label__num">08</span> 참고문헌</p>
    <div class="tc-refs">
      <div class="tc-ref"><span class="tc-ref__group">Minecraft</span><span class="tc-ref__item">Baker et al., <a href="https://arxiv.org/abs/2206.11795" target="_blank">Video PreTraining (VPT)</a>, 2022</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2310.08235" target="_blank">GROOT: Learning to Follow Instructions by Watching Gameplay Videos</a>, 2023</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Lifshitz et al., <a href="https://arxiv.org/abs/2306.00937" target="_blank">STEVE-1</a>, 2023</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Hafner et al., <a href="https://arxiv.org/abs/2301.04104" target="_blank">DreamerV3, Mastering Diverse Domains through World Models</a>, 2023</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Wang et al., <a href="https://arxiv.org/abs/2305.16291" target="_blank">Voyager</a>, 2023</span></div>

      <div class="tc-ref"><span class="tc-ref__group">Cross-embodiment</span><span class="tc-ref__item">Nair et al., <a href="https://arxiv.org/abs/2203.12601" target="_blank">R3M</a>, 2022</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Ma et al., <a href="https://arxiv.org/abs/2210.00030" target="_blank">VIP</a>, 2022</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Bruce et al., <a href="https://arxiv.org/abs/2402.15391" target="_blank">Genie</a>, 2024</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2505.12705" target="_blank">DreamGen</a>, 2025</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2312.13139" target="_blank">GR-1</a>, 2023 · <a href="https://arxiv.org/abs/2410.06158" target="_blank">GR-2</a>, 2024</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2508.05635" target="_blank">Genie Envisioner</a>, 2025</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2310.08864" target="_blank">Open X-Embodiment / RT-X</a>, 2023</span></div>

      <div class="tc-ref"><span class="tc-ref__group">Morphology</span><span class="tc-ref__item">Wang et al., <a href="https://openreview.net/pdf?id=S1sqHMZCb" target="_blank">NerveNet</a>, ICLR 2018</span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item">Gupta et al., <a href="https://arxiv.org/abs/2203.11931" target="_blank">MetaMorph</a>, ICLR 2022</span></div>

      <div class="tc-ref"><span class="tc-ref__group">Robotics VLA</span><span class="tc-ref__item">NVIDIA, <a href="https://arxiv.org/abs/2503.14734" target="_blank">GR00T N1</a>, 2025</span></div>
    </div>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">09</span> 문의</p>
    <div class="tc-feedback">
      <p class="muted" style="margin:0;">미처 다루지 못한 선행연구나 의견이 있으시면 아래 이메일로 문의 부탁드립니다.</p>
      <a href="mailto:akileo@korea.ac.kr?subject=taskcraft%20feedback" class="tc-feedback__submit">이메일 문의</a>
    </div>
  </div>

</div>
