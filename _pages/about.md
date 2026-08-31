---
layout: default
permalink: /about/
author_profile: false
classes: wide
---

<style>
.about-prose {
  max-width: 760px;
  margin: 0 auto;
  padding: 3rem 1.5rem 5rem;
  font-size: 0.9rem;
  line-height: 1.85;
}
.about-name { font-size: 1.5rem; font-weight: 700; color: var(--text); margin: 0 0 0.2rem; letter-spacing: -0.01em; }
.about-role { font-size: 0.82rem; color: var(--text-muted); margin: 0 0 1.1rem; }

.about-links { display: flex; gap: 0.5rem; flex-wrap: wrap; margin-bottom: 2rem; }
.about-links a {
  font-size: 0.78rem; color: var(--text-muted); text-decoration: none;
  border: 1px solid var(--border); padding: 0.18rem 0.65rem; border-radius: 4px; transition: all 0.15s;
}
.about-links a:hover { color: var(--accent); border-color: var(--accent); }
.about-links .cv-btn {
  background: var(--accent); color: var(--accent-contrast) !important; border-color: var(--accent);
}
.about-links .cv-btn:hover { opacity: 0.85; }

.about-intro { margin-bottom: 2.5rem; text-align: justify; }

.about-divider { border: none; border-top: 1px solid var(--border); margin: 1.75rem 0; }
.about-section-label {
  font-size: 0.68rem; font-weight: 600; letter-spacing: 0.1em;
  text-transform: uppercase; color: var(--text-muted); margin: 0 0 0.6rem;
}
.about-placeholder {
  color: var(--text-muted); font-style: italic; font-size: 0.85rem;
  padding: 0.75rem 1rem; border: 1px dashed var(--border); border-radius: 6px; margin-bottom: 1rem;
}
.timeline-item {
  display: grid; grid-template-columns: 8.5rem 1fr;
  gap: 0.3rem 1rem; padding: 0.55rem 0; border-bottom: 1px solid var(--border);
  font-size: 0.84rem; align-items: start;
}
.timeline-item:last-child { border-bottom: none; }
.timeline-period { color: var(--text-muted); font-size: 0.75rem; padding-top: 0.15rem; white-space: nowrap; }
.timeline-content strong { display: block; color: var(--text); font-weight: 500; margin-bottom: 0.1rem; }
.timeline-content span { color: var(--text-muted); font-size: 0.78rem; }
.timeline-content a { color: var(--accent); text-decoration: none; font-size: 0.78rem; }
.timeline-content a:hover { text-decoration: underline; }

.academic-summary { display: flex; flex-direction: column; gap: 0.55rem; margin-bottom: 1rem; }
.academic-row { display: grid; grid-template-columns: 6.5rem 1fr; gap: 0.3rem 1rem; font-size: 0.84rem; }
.academic-row__label { color: var(--text-muted); font-size: 0.78rem; padding-top: 0.1rem; }
.academic-row__value { color: var(--text); }

.club-detail {
  border: 1px solid var(--border);
  border-radius: 6px;
  margin-bottom: 0.75rem;
  overflow: hidden;
}
.club-detail summary {
  cursor: pointer;
  padding: 0.75rem 1rem;
  font-size: 0.85rem;
  color: var(--text);
  list-style: none;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}
.club-detail summary::-webkit-details-marker { display: none; }
.club-detail summary::before {
  content: "▸";
  color: var(--text-muted);
  font-size: 0.7rem;
  transition: transform 0.15s;
  flex-shrink: 0;
}
.club-detail[open] summary::before { transform: rotate(90deg); }
.club-detail__sub { color: var(--text-muted); font-size: 0.78rem; }
.club-detail__body {
  padding: 0.75rem 1rem 1rem 2.2rem;
  border-top: 1px solid var(--border);
}
.club-detail__note { font-size: 0.78rem; color: var(--text-muted); margin: 0 0 0.5rem; }
.club-detail__body ul { margin: 0; padding-left: 1.1rem; font-size: 0.82rem; line-height: 1.8; color: var(--text); }
</style>

<div class="about-prose">

  <h1 class="about-name">Seungmin Lee (이승민)</h1>
  <p class="about-role">Undergraduate · Computer Science, Korea University · Akileo</p>

  <div class="about-links">
    <a href="https://github.com/Akileox" target="_blank">GitHub</a>
    <a href="https://instagram.com/s.mini.lee" target="_blank">Instagram</a>
    <a href="mailto:akileo@korea.ac.kr">Email</a>
    <!-- PDF 파일 준비 후: assets/cv/cv_seungmin_lee.pdf 에 업로드 -->
    <a href="/assets/cv/cv_seungmin_lee.pdf" class="cv-btn" download>CV →</a>
  </div>

  <p class="about-intro">
    My long-term research interest is a simple question: if you extract a task representation from human demonstration video that <strong>isn't tied to any particular body</strong>, can that representation be <strong>transplanted onto an agent with an entirely different form</strong>? I'm drawn to using a <strong>latent world model</strong> as the interface between watching a human act and reproducing that action in a differently-shaped agent, so imitation doesn't have to be relearned from scratch for every new embodiment.
  </p>

  <hr class="about-divider">
  <p class="about-section-label">Activities</p>

  <div class="timeline-item">
    <span class="timeline-period">2026.05 ~</span>
    <div class="timeline-content">
      <strong>TeamDJ</strong>
      <span>학원용 LMS 기획·개발 (AI 1차 응답, 자동 리포트/복습영상 기능 구현)</span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">2025</span>
    <div class="timeline-content">
      <strong>Google Gemini Ambassador</strong>
      <span>K-BioX · 1st Activity Team</span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">2025</span>
    <div class="timeline-content">
      <strong>OSSCA · Open Source Contribution Academy</strong>
      <span>PR Agent 리팩토링 ·
        <a href="https://github.com/The-PR-Agent/pr-agent/pull/1828" target="_blank">PR #1828</a>
      </span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">2022</span>
    <div class="timeline-content">
      <strong>경상북도 고등학생 해커톤 · 2위</strong>
      <span>버스 하차 자동 알림 시스템 기획</span>
    </div>
  </div>

  <hr class="about-divider">
  <p class="about-section-label">Clubs</p>

  <details class="club-detail">
    <summary>
      <span>🛡️</span>
      <strong>KUICS</strong>
      <span class="club-detail__sub">고려대학교 정보보호동아리</span>
    </summary>
    <div class="club-detail__body">
      <p class="club-detail__note">👉 Dreamhack.io 로드맵 기반 학습 내용</p>
      <ul>
        <li><strong>System Hacking</strong>: Assembly, GDB, Shellcode, Stack Buffer Overflow, ASLR 등</li>
        <li><strong>Web Hacking</strong>: Web Introduction, XSS, SQL Injection 등</li>
        <li><strong>Reverse Engineering</strong>: Binary Analysis, Tools (IDA Pro 등)</li>
      </ul>
    </div>
  </details>

  <hr class="about-divider">
  <p class="about-section-label">Education</p>

  <div class="timeline-item">
    <span class="timeline-period">2025 ~</span>
    <div class="timeline-content">
      <strong>Korea University</strong>
      <span>Computer Science and Engineering</span>
    </div>
  </div>

  <hr class="about-divider">
  <p class="about-section-label">Academic Summary</p>

  <div class="academic-summary">
    <div class="academic-row">
      <span class="academic-row__label">GPA</span>
      <span class="academic-row__value">4.45 / 4.5</span>
    </div>
    <div class="academic-row">
      <span class="academic-row__label">특이사항</span>
      <span class="academic-row__value">전공 평점 4.5 / 4.5 (전공 전 과목 A+)</span>
    </div>
  </div>

  <hr class="about-divider">
  <p class="about-section-label">Projects</p>

  <div class="timeline-item">
    <span class="timeline-period">2026.07 ~</span>
    <div class="timeline-content">
      <strong><a href="/taskcraft/">taskcraft</a></strong>
      <span>Morphology-Agnostic Imitation from Human Video via Latent World Models</span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">2026</span>
    <div class="timeline-content">
      <strong><a href="/project/ppo-dexterous-manipulation/">Dexterous Object Manipulation with PPO</a></strong>
      <span>Isaac Lab · Reward Shaping · Curriculum Learning</span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">In Progress</span>
    <div class="timeline-content">
      <strong>TeamDJ</strong>
      <span>학원용 LMS 웹사이트 · <a href="https://www.dongdongmath.com" target="_blank">dongdongmath.com</a></span>
    </div>
  </div>

</div>
