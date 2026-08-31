---
layout: default
permalink: /taskcraft/dashboard/
title: "taskcraft: Dashboard"
author_profile: false
classes: wide
---

<div class="tc-wrap tc-wrap--with-toc">
<div class="tc-main">

  <div class="tc-hero">
    <p class="tc-hero__eyebrow">Research Note · Dashboard</p>
    <h1 class="tc-hero__title" style="font-size:clamp(1.6rem, 3vw, 2.1rem);">taskcraft 진행 대시보드</h1>
    <p class="tc-hero__dek">
      실험이 실제로 어디까지 진행됐는지 확인하는 페이지다. 아이디어와 논증은
      <a href="/taskcraft/">연구 노트</a>에 있다.
    </p>
  </div>

  <div class="tc-dash-grid">
    <div class="tc-dash-card">
      <p class="tc-dash-card__label">마일스톤</p>
      <p class="tc-dash-card__value">1 / 5</p>
    </div>
    <div class="tc-dash-card">
      <p class="tc-dash-card__label">완료된 실험</p>
      <p class="tc-dash-card__value">0</p>
    </div>
    <div class="tc-dash-card">
      <p class="tc-dash-card__label">마지막 갱신</p>
      <p class="tc-dash-card__value" style="font-size:1.1rem;">2026-07-21</p>
    </div>
  </div>

  <div class="tc-section" id="milestones">
    <p class="section-label"><span class="section-label__num">01</span> 마일스톤</p>
    <div class="tc-milestones">
      <div class="tc-ms-row"><span>환경 구축 (Windows 네이티브 MineRL)</span><span class="status-badge status-badge--done">완료</span></div>
      <div class="tc-ms-row"><span>인코더 비교 (VPT·R3M·VIP·CLIP + linear probe)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>임베딩 검증 (embodiment 클러스터링 테스트)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>정책 비교 (BC·DAgger·PPO)</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
      <div class="tc-ms-row"><span>결론 정리</span><span class="status-badge status-badge--upcoming">진행 전</span></div>
    </div>
  </div>

  <div class="tc-section" id="log">
    <p class="section-label"><span class="section-label__num">02</span> 실험 아카이브</p>
    <p class="muted" style="font-size:0.9rem;color:var(--text-muted);margin-bottom:1rem;">
      실험이 끝날 때마다 여기에 결과와 링크가 하나씩 쌓인다. 지금은 마일스톤 1(환경 구축)만 끝난
      상태라 비어 있다.
    </p>
    <div class="tc-dash-empty">
      아직 기록된 실험이 없습니다. 첫 실험(인코더 비교)이 끝나면 이 자리에 결과 요약과
      <a href="https://github.com/Akileox/taskcraft/blob/main/docs/experiment_log.md" style="color:var(--accent);">experiment_log.md</a>
      링크가 추가됩니다.
    </div>
  </div>

  <div class="tc-section" id="links">
    <p class="section-label"><span class="section-label__num">03</span> 원본 자료</p>
    <div class="tc-refs">
      <div class="tc-ref"><span class="tc-ref__group">문서</span><span class="tc-ref__item"><a href="https://github.com/Akileox/taskcraft/blob/main/docs/position_paper.md" target="_blank">포지셔닝 문서 (전체 사고 흐름)</a></span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://github.com/Akileox/taskcraft/blob/main/docs/roadmap.md" target="_blank">로드맵</a></span></div>
      <div class="tc-ref"><span class="tc-ref__group"></span><span class="tc-ref__item"><a href="https://github.com/Akileox/taskcraft/tree/main/docs/research_notes" target="_blank">연구 노트 (논문별 정리)</a></span></div>
      <div class="tc-ref"><span class="tc-ref__group">코드</span><span class="tc-ref__item"><a href="https://github.com/Akileox/taskcraft" target="_blank">github.com/Akileox/taskcraft</a></span></div>
    </div>
  </div>

</div><!-- /.tc-main -->

<aside class="tc-sidebar">
  <nav class="post-single__toc post-single__toc--sidebar">
    <p class="post-single__toc-label">Contents</p>
    <ul class="post-single__toc-list">
      <li><a href="#milestones">01. 마일스톤</a></li>
      <li><a href="#log">02. 실험 아카이브</a></li>
      <li><a href="#links">03. 원본 자료</a></li>
    </ul>
  </nav>
</aside>

</div><!-- /.tc-wrap -->
