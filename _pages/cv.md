---
layout: default
permalink: /cv/
author_profile: false
classes: wide
---

<style>
.cv-prose {
  max-width: 660px;
  margin: 0 auto;
  padding: 3rem 1.5rem 5rem;
  font-size: 0.875rem;
  line-height: 1.8;
}
.cv-header { margin-bottom: 2.5rem; }
.cv-header h1 { font-size: 1.4rem; font-weight: 700; letter-spacing: -0.01em; margin: 0 0 0.2rem; }
.cv-header p { color: var(--text-muted); font-size: 0.82rem; margin: 0 0 0.9rem; }

.cv-links { display: flex; gap: 0.5rem; flex-wrap: wrap; align-items: center; }
.cv-links a {
  font-size: 0.75rem; color: var(--text-muted); text-decoration: none;
  border: 1px solid var(--border); padding: 0.15rem 0.6rem; border-radius: 4px; transition: all 0.15s;
}
.cv-links a:hover { color: var(--accent); border-color: var(--accent); }
.cv-download {
  display: inline-flex; align-items: center; gap: 0.35rem;
  background: var(--accent); color: #fff !important; font-size: 0.75rem; font-weight: 500;
  padding: 0.3rem 0.85rem; border-radius: 999px; text-decoration: none !important;
  transition: opacity 0.15s;
}
.cv-download:hover { opacity: 0.82; }

.cv-divider { border: none; border-top: 1px solid var(--border); margin: 1.5rem 0; }
.cv-section-label {
  font-size: 0.68rem; font-weight: 600; letter-spacing: 0.1em;
  text-transform: uppercase; color: var(--text-muted); margin: 0 0 0.6rem;
}
.cv-item {
  display: grid; grid-template-columns: 8.5rem 1fr;
  gap: 0.3rem 1rem; padding: 0.55rem 0; border-bottom: 1px solid var(--border);
  font-size: 0.84rem; align-items: start;
}
.cv-item:last-child { border-bottom: none; }
.cv-period { color: var(--text-muted); font-size: 0.75rem; padding-top: 0.15rem; white-space: nowrap; }
.cv-content strong { display: block; color: var(--text); font-weight: 500; margin-bottom: 0.1rem; }
.cv-content span { color: var(--text-muted); font-size: 0.78rem; }
.cv-content a { color: var(--accent); text-decoration: none; font-size: 0.78rem; }
.cv-content a:hover { text-decoration: underline; }

.skill-row { display: flex; gap: 0.3rem; flex-wrap: wrap; margin-bottom: 0.4rem; align-items: center; }
.skill-row-label { font-size: 0.72rem; color: var(--text-muted); width: 5rem; flex-shrink: 0; }
.skill-tag {
  background: var(--bg-secondary); border: 1px solid var(--border); border-radius: 3px;
  padding: 0.1rem 0.4rem; font-size: 0.72rem; color: var(--text);
  font-family: "SFMono-Regular", Consolas, monospace;
}

.cv-inquiry {
  margin-top: 3rem; padding: 1.5rem; border-radius: var(--radius-sm);
  background: var(--bg-secondary); border: 1px solid var(--border);
}
.cv-inquiry h3 { font-size: 0.85rem; font-weight: 600; color: var(--text); margin: 0 0 0.4rem; }
.cv-inquiry p { font-size: 0.82rem; color: var(--text-muted); margin: 0 0 0.75rem; line-height: 1.7; }
.cv-inquiry a { color: var(--accent); text-decoration: none; font-weight: 500; }
.cv-inquiry a:hover { text-decoration: underline; }
</style>

<div class="cv-prose">

  <div class="cv-header">
    <h1>SeungMin Lee</h1>
    <p>Computer Science · Korea University · akileo@korea.ac.kr</p>
    <div class="cv-links">
      <!-- PDF 파일 준비 후: assets/cv/cv_seungmin_lee.pdf 에 업로드하고 아래 주석 해제 -->
      <!-- <a href="/assets/cv/cv_seungmin_lee.pdf" class="cv-download" download>
        <svg xmlns="http://www.w3.org/2000/svg" width="12" height="12" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2.5" stroke-linecap="round" stroke-linejoin="round"><path d="M21 15v4a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2v-4"/><polyline points="7 10 12 15 17 10"/><line x1="12" y1="15" x2="12" y2="3"/></svg>
        Download PDF
      </a> -->
      <a href="https://github.com/Akileox" target="_blank">GitHub</a>
      <a href="https://instagram.com/s.mini.lee" target="_blank">Instagram</a>
      <a href="mailto:akileo@korea.ac.kr">Email</a>
    </div>
  </div>

  <hr class="cv-divider">
  <p class="cv-section-label">Education</p>
  <div class="cv-item">
    <span class="cv-period">2025 —</span>
    <div class="cv-content">
      <strong>Korea University</strong>
      <span>Undergraduate, Computer Science and Engineering</span>
    </div>
  </div>

  <hr class="cv-divider">
  <p class="cv-section-label">Activities</p>

  <div class="cv-item">
    <span class="cv-period">2025</span>
    <div class="cv-content">
      <strong>Google Gemini Ambassador</strong>
      <span>K-BioX · Bio-MBTI web app (Gemini API) · 1st Activity Team</span>
    </div>
  </div>
  <div class="cv-item">
    <span class="cv-period">2025</span>
    <div class="cv-content">
      <strong>OSSCA · Open Source Contribution Academy</strong>
      <span>PR Agent refactoring ·
        <a href="https://github.com/The-PR-Agent/pr-agent/pull/1828" target="_blank">PR #1828</a>
      </span>
    </div>
  </div>
  <div class="cv-item">
    <span class="cv-period">2022</span>
    <div class="cv-content">
      <strong>경상북도 고등학생 해커톤 · 2nd Place</strong>
      <span>Bus stop auto-alert system design</span>
    </div>
  </div>

  <hr class="cv-divider">
  <p class="cv-section-label">Projects</p>

  <div class="cv-item">
    <span class="cv-period">In Progress</span>
    <div class="cv-content">
      <strong>Robot Arm Manipulation via Monocular Vision</strong>
      <span>Nvidia Isaac Sim · PPO · SAC</span>
    </div>
  </div>
  <div class="cv-item">
    <span class="cv-period">In Progress</span>
    <div class="cv-content">
      <strong>Program Making -Classified-</strong>
      <span>Full-stack · Client project</span>
    </div>
  </div>
  <div class="cv-item">
    <span class="cv-period">2026</span>
    <div class="cv-content">
      <strong>LunarLander · Policy Gradient</strong>
      <span>Python · REINFORCE</span>
    </div>
  </div>
  <div class="cv-item">
    <span class="cv-period">2026</span>
    <div class="cv-content">
      <strong>CartPole · DQN</strong>
      <span>Python · Deep Q-Network</span>
    </div>
  </div>

  <hr class="cv-divider">
  <p class="cv-section-label">Skills</p>

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

  <!-- Job / Internship Inquiry -->
  <div class="cv-inquiry">
    <h3>Job & Internship Inquiries</h3>
    <p>
      Research internship, industry internship, or collaboration 문의는 이메일로 주세요.<br>
      For job or internship offers, please reach out via email.
    </p>
    <a href="mailto:akileo@korea.ac.kr">akileo@korea.ac.kr →</a>
  </div>

</div>
