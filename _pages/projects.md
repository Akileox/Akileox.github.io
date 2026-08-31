---
layout: default
permalink: /projects/
author_profile: false
classes: wide
---

<div class="projects-wrap">

  <h1 style="font-family:var(--font-serif);font-size:2rem;font-weight:700;color:var(--text);margin:0 0 0.5rem;letter-spacing:-0.01em;">Projects</h1>
  <p style="color:var(--text-muted);font-size:0.95rem;margin:0 0 2.5rem;">Coursework, personal projects, and activities I've been involved in.</p>

  <div>

    <!-- taskcraft (current) -->
    <div class="project-row">
      <div class="project-row__status"><span class="status-badge status-badge--progress">In Progress</span></div>
      <div>
        <h3 class="project-row__title">taskcraft: Embodiment-Agnostic Task Representation</h3>
        <p class="project-row__desc">
          사람 시연 영상에서 신체 구조에 종속되지 않는 task representation을 latent world model로 추출해, 형태가 다른 에이전트에 이식할 수 있는가에 대한 연구. Minecraft(MineRL) 안에서 서로 다른 embodiment 간 전이를 검증하는 파일럿을 설계 중이다.
        </p>
        <div class="project-row__tags">
          <span class="project-row__tag">World Model</span>
          <span class="project-row__tag">Imitation Learning</span>
          <span class="project-row__tag">MineRL</span>
          <span class="project-row__tag">Cross-Embodiment</span>
        </div>
        <div class="project-row__links">
          <a href="/taskcraft/">연구 노트 →</a>
          <a href="https://github.com/Akileox/taskcraft" target="_blank">GitHub</a>
        </div>
      </div>
    </div>

    <!-- TeamDJ LMS -->
    <div class="project-row">
      <div class="project-row__status"><span class="status-badge status-badge--progress">In Progress</span></div>
      <div>
        <h3 class="project-row__title">TeamDJ</h3>
        <p class="project-row__desc">
          학원용 LMS(학습관리시스템) 웹사이트 제작.
        </p>
        <div class="project-row__tags">
          <span class="project-row__tag">Web Development</span>
          <span class="project-row__tag">LMS</span>
        </div>
        <div class="project-row__links">
          <a href="https://teamdj.vercel.app/" target="_blank">Live →</a>
        </div>
      </div>
    </div>

    <!-- Isaac Lab dexterous manipulation -->
    <div class="project-row">
      <div class="project-row__status"><span class="status-badge status-badge--done">Completed</span></div>
      <div>
        <h3 class="project-row__title">Dexterous Object Manipulation with PPO</h3>
        <p class="project-row__desc">
          Isaac Lab에서 로봇 손(Shadow Hand)이 사람 손 동작과 물체 궤적을 모방해 조작하도록 PPO 정책을 학습시킨 프로젝트. Tracking reward만으로는 안정적인 grasp가 형성되지 않는 문제를 발견하고, contact·finger-group reward와 reset curriculum 설계로 해결했다.
        </p>
        <div class="project-row__tags">
          <span class="project-row__tag">Isaac Lab</span>
          <span class="project-row__tag">PPO</span>
          <span class="project-row__tag">Dexterous Manipulation</span>
          <span class="project-row__tag">Reward Shaping</span>
          <span class="project-row__tag">Curriculum Learning</span>
        </div>
        <div class="project-row__links">
          <a href="/project/ppo-dexterous-manipulation/">연구 노트 →</a>
        </div>
      </div>
    </div>

    <!-- Computer Architecture -->
    <div class="project-row">
      <div class="project-row__status"><span class="status-badge status-badge--done">Completed</span></div>
      <div>
        <h3 class="project-row__title">Multi-Cycle RISC-V Processor Implementation</h3>
        <p class="project-row__desc">
          기존 single-cycle RISC-V processor를 FSM 기반 multi-cycle architecture로 변환. Instruction type별로 필요한 stage만 거치도록 FSM과 intermediate register(rs1/rs2_data_reg, aluout_reg), ALU/Result MUX를 설계하고 Quartus functional simulation으로 검증했다.
        </p>
        <div class="project-row__tags">
          <span class="project-row__tag">Computer Architecture</span>
          <span class="project-row__tag">RISC-V</span>
          <span class="project-row__tag">Verilog</span>
          <span class="project-row__tag">FSM</span>
        </div>
      </div>
    </div>

    <!-- Hackathon -->
    <div class="project-row">
      <div class="project-row__status"><span class="status-badge status-badge--done">Completed · 🏆 2nd Place</span></div>
      <div>
        <h3 class="project-row__title">Bus Stop Auto-Alert System</h3>
        <p class="project-row__desc">
          Designed a system to automatically notify passengers when their bus stop is approaching, reducing missed stops. Presented at the 경상북도 고등학생 해커톤 and won 2nd place.
        </p>
        <div class="project-row__tags">
          <span class="project-row__tag">Hackathon</span>
          <span class="project-row__tag">IoT Concept</span>
          <span class="project-row__tag">Problem Design</span>
        </div>
      </div>
    </div>

  </div>
</div>
