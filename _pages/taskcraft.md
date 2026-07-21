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
    <h1 class="tc-hero__title">이 아이디어는 어디서 틀렸고,<br/>어떻게 고쳐졌나</h1>
    <p class="tc-hero__dek">
      사람 시연 영상에서 신체 구조(embodiment)에 종속되지 않는 task representation을
      뽑아내면, 형태가 다른 에이전트에도 이식할 수 있는가 — 이 질문 하나를
      Minecraft라는 통제된 파일럿 위에서 밀어붙이면서, 매번 어디가 허술했는지
      스스로 잡아내고 고친 기록.
    </p>
    <div class="tc-hero__status">
      <span class="status-badge status-badge--done">Milestone 1 — 환경 구축 완료</span>
      <span class="status-badge status-badge--upcoming">Milestone 2–5 진행 전</span>
    </div>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">00</span> 최종 질문</p>
    <p class="tc-section__lede">
      로봇이나 게임 에이전트에게 사람 시연 영상을 학습시키는 대부분의 방법은,
      영상에서 뽑아낸 신호를 그 에이전트 <strong>자신의</strong> 행동 공간(관절 각도,
      키보드 입력)에 직접 연결한다. 그 결과 표현은 특정 신체가 특정 조작을
      하는 방식에 묶여서, 형태가 다른 에이전트로는 재사용되지 않는다.
      Latent world model을 매개로 삼으면 이 종속을 끊을 수 있는가 — 이게
      taskcraft의 최종 질문이고, Minecraft는 이걸 값싸게 찔러보기 위한 도구일
      뿐 최종 타겟이 아니다.
    </p>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">01</span> 사고의 흐름</p>
    <p class="tc-section__lede">
      결론만 남기지 않고, 어디서 되물었고 어디서 틀렸는지를 그대로 둔다.
      순서는 실제로 생각이 진행된 순서다.
    </p>

    <div class="tc-timeline">

      <div class="tc-move">
        <div class="tc-move__tag">MOVE 1</div>
        <h3>가설을 세우다</h3>
        <p>
          인간 시연 영상 → embodiment 무관 task representation(latent world
          model이 매개) → 형태가 다른 agent로 이식. Minecraft(나무 → 작업대 →
          채굴)를 첫 testbed로 고른다.
        </p>
      </div>

      <div class="tc-move">
        <div class="tc-move__tag">MOVE 2</div>
        <h3>근데 왜 이게 중요한데?</h3>
        <p>처음 댄 근거는 "로봇 같은 target embodiment를 잘 모른다"였다.</p>
        <div class="tc-correction">
          <span class="was">target embodiment를 잘 모른다</span>
          <span class="now">강아지형·조류형 로봇, 드론처럼 형태가 근본적으로
          다른 구조가 이미 다양하게 존재한다. 문제는 무지가 아니라 이 다양성
          자체 — embodiment마다 teleoperation을 다시 하고, 파이프라인을
          처음부터 다시 짜야 한다.</span>
          <span class="who">정정 계기 — 피드백</span>
        </div>
        <p>
          가장 날카로웠던 지점은 재난구조 로봇. 실제 재난 상황은 재현이 안
          되고, 데이터 수집만을 위해 로봇을 위험 상황에 반복 투입하는 것 자체가
          불가능하다. 여기서 자원의 비대칭이 드러난다 — 데이터 수집 비용
          문제는 테슬라 같은 빅테크가 돈으로 밀어붙여 해결할 수 있지만,
          애초에 대량 배치가 불가능한 embodiment에는 돈을 아무리 써도 안
          통한다. Embodiment 무관 표현이 가장 크게 기여하는 지점은 정확히
          <strong>자원으로 못 미는 지점</strong>이라고 본다.
        </p>
      </div>

      <div class="tc-move">
        <div class="tc-move__tag">MOVE 3</div>
        <h3>이미 누가 하고 있지 않을까?</h3>
        <p>
          VPT·GROOT·STEVE-1·DreamerV3·Voyager 다섯 편으로 시작했는데, 전부
          "같은 embodiment(Minecraft 플레이어) 안에서" 문제를 푸는 연구였다.
          최종 주장과 직접 비교해야 할 진짜 경쟁자 — R3M·VIP·Genie·DreamGen·
          GR-1/2·RT-X(cross-embodiment 로봇 학습) — 가 원래 목록에서 빠져
          있었다.
        </p>
        <div class="tc-correction">
          <span class="was">"GROOT N1"이 dual-system 구조라 최종 비전의
          아이디어 원류라고 적어 놓았음</span>
          <span class="now">Minecraft 논문 "GROOT"와 NVIDIA 로봇 파운데이션
          모델 "GR00T N1"은 이름만 비슷한 별개 프로젝트. dual-system은 GR00T
          N1 쪽 특징이다.</span>
          <span class="who">정정 계기 — 질문</span>
        </div>
        <p>
          그리고 이 cross-embodiment 계열을 실제로 찾아보니, "world model +
          cross-embodiment"는 <strong>이미 붐비는 영역</strong>이었다 — NVIDIA(DreamGen),
          ByteDance(GR-1/GR-2), DeepMind(Genie, Genie Envisioner)가
          2024~2025년에 정확히 이 조합을 다루고 있다. "빠르게 초기 성과를 낼
          수 있는 빈 영역"이라는 기대는 오판이었다.
        </p>
      </div>

      <div class="tc-move">
        <div class="tc-move__tag">MOVE 4</div>
        <h3>그럼 진짜 차별점은 뭔데?</h3>
        <p>
          "world model을 명시적 매개로 쓴다"는 게 차별점이라고 썼는데,
          DreamGen이 이미 그걸 하고 있어서 무효가 됐다. 더 정확한 축을
          다시 찾아야 했다.
        </p>
        <div class="tc-table-wrap">
          <table class="tc-table">
            <thead>
              <tr><th></th><th>기존 cross-embodiment 연구</th><th>taskcraft 최종 비전</th></tr>
            </thead>
            <tbody>
              <tr>
                <td>다루는 embodiment 집합</td>
                <td>로봇 팔/그리퍼류(R3M, DreamGen, RT-X), 혹은 시뮬레이션 안
                무작위 생성 형태(NerveNet, MetaMorph) — 서로 닮은 집합</td>
                <td class="hl">인간·로봇·게임 아바타 등 형태가 근본적으로 다른 집합</td>
              </tr>
              <tr>
                <td>암묵적 가정</td>
                <td>embodiment들이 어느 정도 공유하는 구조(관절 그래프,
                팔+그리퍼 기구학)가 있다고 전제</td>
                <td class="hl">그런 공유 구조가 없어도 통하는 표현을 찾는다</td>
              </tr>
            </tbody>
          </table>
        </div>
        <p>
          다만 이 차이가 <strong>질적으로</strong> 다른 문제인지, 아니면 닮은
          집합을 확장하면 정도차이로 도달하는 문제인지는 아직 근거가 없다 —
          다음 단계로 넘어가는 이유.
        </p>
      </div>

      <div class="tc-move">
        <div class="tc-move__tag">MOVE 5</div>
        <h3>검증 가능한 형태로 좁히기</h3>
        <p>
          "질적 vs 정도" 차이를 실증하려면 뭘 재야 하나. 첫 안: 인간 영상으로만
          학습된 공개 표현(R3M, VIP)을 얼려서(frozen) Minecraft에 그대로 붙이고,
          VPT(Minecraft로 학습됨) 대비 성능을 비교한다.
        </p>
        <div class="tc-correction">
          <span class="was">R3M/VIP가 실패하면 "embodiment 무관 표현은
          근본적으로 다른 형태로 전이 안 된다"는 근거가 된다</span>
          <span class="now">R3M/VIP는 로봇 3인칭/손목 카메라 영상으로
          학습됐고 Minecraft는 1인칭 블록 그래픽이다 — 실패해도 embodiment
          gap 때문인지 그냥 시각적 도메인이 달라서인지 구분이 안 된다.
          CLIP처럼 embodiment 무관 학습을 목표로 하지 않은 범용 인코더를
          대조군으로 추가해야 한다.</span>
          <span class="who">정정 계기 — 문제제기</span>
        </div>
        <p>
          여기에 더해, full BC 학습보다 먼저 <strong>frozen feature 위에 linear
          probe</strong>를 붙여 태스크 진행도를 얼마나 잘 예측하는지부터 본다 —
          R3M/VIP 원 논문들이 자기 표현력을 검증할 때 쓰는 방식이라 더
          저렴하고 직접적이다. 순서: VPT / R3M / VIP / CLIP 네 인코더 →
          linear probe → (신호가 있으면) BC 비교.
        </p>
      </div>

      <div class="tc-move tc-move--current">
        <div class="tc-move__tag">MOVE 6 — 현재 가장 아픈 질문</div>
        <h3>world model의 latent 자체가 인간 embodiment에 종속되지 않나?</h3>
        <p>
          인간 영상을 아무리 잘 모아 학습시켜도, latent가 "세계에서 무엇이
          변했는가"가 아니라 "인간 몸이 어떻게 그걸 일으켰는가"를 은근히
          같이 인코딩할 수 있다. 모델은 이 둘을 구분할 유인이 애초에 없다.
        </p>
        <p>
          제안한 중간 검증: 인간이 나무를 패는 영상과 Minecraft 아바타가 나무를
          패는 영상을 같은 인코더에 통과시켜, latent가 <strong>태스크 기준</strong>으로
          뭉치는지 <strong>출처(embodiment) 기준</strong>으로 뭉치는지 클러스터링으로
          확인한다. 아직 답 없음 — 다음 갱신에서 이어간다.
        </p>
      </div>

    </div>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">02</span> 예상되는 반론</p>
    <p class="tc-section__lede">방어하지 않고, 어디까지 답할 수 있고 어디부터 모르는지 구분해 둔다.</p>

    <details class="tc-faq">
      <summary>이게 될 거였으면 빅테크가 이미 하고 있지 않겠냐</summary>
      <div class="tc-faq__body">
        <p>실제로 "닮은 embodiment 집합"에서는 이미 하고 있다(DreamGen,
        GR-1/2). 그런데 "형태가 근본적으로 다른 집합"(인간↔재난로봇↔드론)은
        상업적 볼륨이 작아서(대량 생산·배치가 안 되는 embodiment) 투자 대비
        수익이 낮을 수 있다 — 과학적으로 흥미로워도 경제적으로는 덜 끌리는
        지점.</p>
        <p>다만 빅테크 내부에서 비공개로 이미 하고 있을 가능성은 배제 못
        한다. 모른다는 것 자체를 인정하는 게 맞다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>Minecraft는 장난감 도메인인데, 여기 결과가 실제 로봇에 뭘 말해주나</summary>
      <div class="tc-faq__body">
        <p>말 못 한다. Minecraft는 방법론을 싸게 검증하는 파일럿일 뿐,
        연속 제어·물리 노이즈가 있는 실제 로봇으로의 일반화는 별도 검증이
        필요한 큰 질문으로 남겨 둔다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>world model latent이 결국 학습 데이터의 embodiment에 종속되는 거 아니냐</summary>
      <div class="tc-faq__body">
        <p>가장 아픈 질문이고(MOVE 6), 방어 논리가 없다. "아직 모른다,
        이렇게 테스트해보고 싶다"가 지금 낼 수 있는 가장 정직한 답이다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>그 실험(R3M/VIP/CLIP 비교)은 기존 방법 테스트지, 네 아이디어 자체를 검증하는 게 아니잖아</summary>
      <div class="tc-faq__body">
        <p>맞다. 9주·솔로 스코프에서는 "기존 방법이 어디서 깨지는지 확인"이
        현실적 목표이고, 아이디어 자체(world model 매개 표현)의 구현·검증은
        랩 합류 이후 과제로 명시해 둔다.</p>
      </div>
    </details>

    <details class="tc-faq">
      <summary>영상엔 촉각·힘 정보가 없는데, 그것만으로 충분한가</summary>
      <div class="tc-faq__body">
        <p>충분하지 않을 수 있다. GelSight·DIGIT·BioTac 같은 촉각 센서
        데이터가 실제로 성능을 끌어올린다는 결과도 있다(잡기 성공률
        82%→96% 등). 다만 촉각 데이터는 특수 장비로 새로 찍어야만 존재해서,
        유튜브처럼 이미 쌓여 있는 시각 데이터와 스케일이 근본적으로 다르다.
        시각 전용 학습이라는 전제 자체의 한계로 인정하고 간다.</p>
      </div>
    </details>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">03</span> 지금 상태</p>
    <div class="tc-milestones">
      <div class="tc-ms-row">
        <span>환경/도구 세팅 (Windows 네이티브 MineRL)</span>
        <span class="status-badge status-badge--done">완료</span>
      </div>
      <div class="tc-ms-row">
        <span>Observation pipeline (VPT/R3M/VIP/CLIP 비교 + linear probe)</span>
        <span class="status-badge status-badge--upcoming">진행 전</span>
      </div>
      <div class="tc-ms-row">
        <span>BC baseline</span>
        <span class="status-badge status-badge--upcoming">진행 전</span>
      </div>
      <div class="tc-ms-row">
        <span>DAgger 비교</span>
        <span class="status-badge status-badge--upcoming">진행 전</span>
      </div>
      <div class="tc-ms-row">
        <span>PPO 비교</span>
        <span class="status-badge status-badge--upcoming">진행 전</span>
      </div>
    </div>
  </div>

  <div class="tc-section">
    <p class="section-label"><span class="section-label__num">04</span> 피드백</p>
    <div class="tc-feedback">
      <p style="margin:0;font-size:0.9rem;color:var(--text-muted);">
        이 아이디어에서 놓친 선행연구, 반박, 질문 — 뭐든 좋다.
      </p>
      <form class="tc-feedback__form" id="tc-feedback-form">
        <textarea name="message" placeholder="의견을 남겨주세요" required></textarea>
        <input type="text" name="contact" placeholder="이메일/연락처 (선택)">
        <button type="submit" class="tc-feedback__submit">보내기</button>
        <p class="tc-feedback__status" id="tc-feedback-status"></p>
      </form>
    </div>
  </div>

  <p style="font-family:'SFMono-Regular',Consolas,monospace;font-size:0.78rem;color:var(--text-muted);margin-top:3rem;">
    2026-09 입대 전 10주 프로젝트 · 매주 갱신 ·
    <a href="https://github.com/Akileox/taskcraft">github.com/Akileox/taskcraft</a>
  </p>

</div>

<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2/dist/umd/supabase.min.js"></script>
<script>
  (function () {
    var SUPABASE_URL = "__SUPABASE_URL__";
    var SUPABASE_ANON_KEY = "__SUPABASE_ANON_KEY__";
    var form = document.getElementById("tc-feedback-form");
    var status = document.getElementById("tc-feedback-status");
    if (!form) return;

    var client = null;
    if (window.supabase && SUPABASE_URL.indexOf("__") !== 0) {
      client = window.supabase.createClient(SUPABASE_URL, SUPABASE_ANON_KEY);
    }

    form.addEventListener("submit", function (e) {
      e.preventDefault();
      if (!client) {
        status.textContent = "피드백 폼이 아직 연결되지 않았습니다.";
        status.className = "tc-feedback__status tc-feedback__status--err";
        return;
      }
      var message = form.message.value.trim();
      var contact = form.contact.value.trim();
      if (!message) return;

      var submitBtn = form.querySelector("button");
      submitBtn.disabled = true;
      status.textContent = "보내는 중...";
      status.className = "tc-feedback__status";

      client.from("feedback").insert({
        message: message,
        contact: contact || null,
        page: "/taskcraft/"
      }).then(function (res) {
        submitBtn.disabled = false;
        if (res.error) {
          status.textContent = "전송 실패: " + res.error.message;
          status.className = "tc-feedback__status tc-feedback__status--err";
        } else {
          status.textContent = "고마워요, 잘 받았습니다.";
          status.className = "tc-feedback__status tc-feedback__status--ok";
          form.reset();
        }
      });
    });
  })();
</script>
