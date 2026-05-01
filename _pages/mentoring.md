---
layout: default
permalink: /mentoring/
author_profile: false
classes: wide
---

<style>
.mentoring-wrap {
  max-width: 720px;
  margin: 0 auto;
  padding: 3rem 1.5rem 5rem;
}
.mentoring-wrap h1 { font-size: 1.5rem; font-weight: 700; margin: 0 0 0.3rem; letter-spacing: -0.01em; }
.mentoring-wrap .intro { font-size: 0.875rem; color: var(--text-muted); margin: 0 0 2.5rem; line-height: 1.7; }

.session-list { display: flex; flex-direction: column; gap: 1rem; margin-top: 1.5rem; }

.session-row {
  display: flex;
  align-items: center;
  gap: 1.25rem;
  padding: 1.1rem 1.25rem;
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  background: var(--bg);
  text-decoration: none;
  color: inherit;
  transition: border-color 0.15s, box-shadow 0.15s, transform 0.15s;
}
.session-row:hover {
  border-color: var(--accent);
  box-shadow: var(--shadow-sm);
  transform: translateX(3px);
  color: inherit;
  text-decoration: none;
}

.session-row__period {
  font-size: 0.75rem;
  color: var(--text-muted);
  white-space: nowrap;
  min-width: 9rem;
  font-family: "SFMono-Regular", Consolas, monospace;
}

.session-row__info { flex: 1; }
.session-row__title { font-size: 0.9rem; font-weight: 600; color: var(--text); margin: 0 0 0.15rem; }
.session-row__desc { font-size: 0.78rem; color: var(--text-muted); margin: 0; }

.session-row__badge {
  font-size: 0.68rem; font-weight: 600; padding: 0.18rem 0.6rem;
  border-radius: 999px; white-space: nowrap; flex-shrink: 0;
}
.badge--active { background: #d1fae5; color: #065f46; }
.badge--past { background: var(--bg-secondary); color: var(--text-muted); border: 1px solid var(--border); }

.session-row__arrow { color: var(--text-muted); font-size: 0.9rem; flex-shrink: 0; }
</style>

<div class="mentoring-wrap">

  <h1>Mentoring</h1>
  <!-- /mentoring/ 페이지 소개 문구. 직접 채워주세요. -->
  <p class="intro">얻은 지식을 더 많은 사람들과 공유하는 방법</p>

  <p class="section-label">Sessions</p>

  <div class="session-list">

    <a href="/mentoring/gshs-tr-2026/" class="session-row">
      <span class="session-row__period">2026.07 — present</span>
      <div class="session-row__info">
        <p class="session-row__title">Team Research</p>
        <p class="session-row__desc">GSHS</p>
      </div>
      <span class="session-row__badge badge--active">● Upcoming</span>
      <span class="session-row__arrow">→</span>
    </a>

    <a href="/mentoring/team-dj-2025w/" class="session-row">
      <span class="session-row__period">2024.12 — 2025.02<br>2025.12 — 2026.02</span>
      <div class="session-row__info">
        <p class="session-row__title">Team DJ</p>
        <p class="session-row__desc">고등학교 수학 첨삭 조교</p>
      </div>
      <span class="session-row__badge badge--past">Completed</span>
      <span class="session-row__arrow">→</span>
    </a>

  </div>
</div>
