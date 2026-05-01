---
layout: default
permalink: /about/
author_profile: false
classes: wide
---

<style>
.about-prose {
  max-width: 660px;
  margin: 0 auto;
  padding: 3rem 1.5rem 5rem;
  font-size: 0.9rem;
  line-height: 1.85;
}
.about-name { font-size: 1.5rem; font-weight: 700; color: var(--text); margin: 0 0 0.2rem; letter-spacing: -0.01em; }
.about-role { font-size: 0.82rem; color: var(--text-muted); margin: 0 0 1.1rem; }

.about-links { display: flex; gap: 0.5rem; flex-wrap: wrap; margin-bottom: 2.5rem; }
.about-links a {
  font-size: 0.78rem; color: var(--text-muted); text-decoration: none;
  border: 1px solid var(--border); padding: 0.18rem 0.65rem; border-radius: 4px; transition: all 0.15s;
}
.about-links a:hover { color: var(--accent); border-color: var(--accent); }
.about-links .cv-btn {
  background: var(--accent); color: #fff !important; border-color: var(--accent);
}
.about-links .cv-btn:hover { opacity: 0.85; }

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

.skill-row { display: flex; gap: 0.3rem; flex-wrap: wrap; margin-bottom: 0.4rem; align-items: center; }
.skill-row-label { font-size: 0.72rem; color: var(--text-muted); width: 5rem; flex-shrink: 0; }
.skill-tag {
  background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 3px;
  padding: 0.1rem 0.4rem; font-size: 0.72rem; color: var(--text);
  font-family: "SFMono-Regular", Consolas, monospace;
}
</style>

<div class="about-prose">

  <h1 class="about-name">SeungMin Lee</h1>
  <p class="about-role">Undergraduate · Computer Science, Korea University · Akileo</p>

  <div class="about-links">
    <a href="https://github.com/Akileox" target="_blank">GitHub</a>
    <a href="https://instagram.com/s.mini.lee" target="_blank">Instagram</a>
    <a href="mailto:akileo@korea.ac.kr">Email</a>
    <a href="/cv/" class="cv-btn">CV →</a>
  </div>

  <!-- 자기소개: /about/ 페이지 상단 소개 문단. 직접 채워주세요. -->
  <div class="about-placeholder">[내용 입력] — 자기소개 (about 페이지 소개 문단)</div>

  <hr class="about-divider">
  <p class="about-section-label">Activities</p>

  <div class="timeline-item">
    <span class="timeline-period">2026.05 —</span>
    <div class="timeline-content">
      <strong>Classified</strong>
      <span>프로그램 기획 및 개발 진행 중</span>
    </div>
  </div>
  <div class="timeline-item">
    <span class="timeline-period">2025</span>
    <div class="timeline-content">
      <strong>Google Gemini Ambassador</strong>
      <span>K-BioX · Bio-MBTI web app · 1st Activity Team</span>
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
  <p class="about-section-label">Education</p>

  <div class="timeline-item">
    <span class="timeline-period">2025 —</span>
    <div class="timeline-content">
      <strong>Korea University</strong>
      <span>Computer Science and Engineering</span>
    </div>
  </div>

  <hr class="about-divider">
  <p class="about-section-label">Skills</p>

  <div class="skill-row">
    <span class="skill-row-label">Languages</span>
    <span class="skill-tag">Python</span>
    <span class="skill-tag">C</span>
    <span class="skill-tag">C++</span>
    <span class="skill-tag">Scala</span>
    <span class="skill-tag">Assembler</span>
    <span class="skill-tag">SQL</span>
    <span class="skill-tag">JavaScript</span>
  </div>
  <div class="skill-row">
    <span class="skill-row-label">Frameworks</span>
    <span class="skill-tag">PyTorch</span>
  </div>
  <div class="skill-row">
    <span class="skill-row-label">Tools</span>
    <span class="skill-tag">Git</span>
    <span class="skill-tag">Linux</span>
  </div>

</div>
