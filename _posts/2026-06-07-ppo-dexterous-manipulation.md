---
title: "Dexterous Object Manipulation with PPO: Training Notes"
date: 2026-06-07
categories: [Project]
tags: [reinforcement-learning, ppo, isaac-lab, robotics, reward-shaping, curriculum-learning]
excerpt: "Training a Shadow Hand in Isaac Lab to imitate human grasping from video: why tracking reward alone produces a hand that fakes contact, and the reward/curriculum design that fixes it."
header:
  teaser: /assets/images/projects/ppo-manipulation/main-sequence-poster.jpg
---

<div class="lang-toggle" role="group" aria-label="Language">
  <button type="button" class="lang-toggle__btn is-active" data-lang="en">EN</button>
  <button type="button" class="lang-toggle__btn" data-lang="ko">KO</button>
</div>

<div class="lang-content" data-lang="en" markdown="1">

For my Reinforcement Learning course's final project, I trained a PPO policy in Isaac Lab so a simulated Shadow Hand, driven by human hand motion captured from real demonstration video, learns to grasp and manipulate an object. The input is per-frame human hand keypoints, object position, and object rotation, reconstructed from multi-view video (HO-Cap-based data). The core algorithm was given as skeleton code. The actual project was designing the observation and reward function so that imitation turns into real, physically plausible manipulation.

That distinction matters more than it sounds. The reference trajectory is reconstructed from video, so it isn't guaranteed to be physically consistent: hands can visually interpenetrate objects, or an object can appear to move without a real grasp. A policy that just copies the reference will inherit those artifacts. It has to reproduce the *outcome* (the object following its reference trajectory) using real contact and friction in the simulator, not by matching keypoints alone.

## Action and Observation Setup

Before writing any reward, I worked out what the policy's actions actually do physically. Actions split into three groups: palm position (converted to a force on the palm root), palm rotation (a 6D rotation representation converted to torque), and finger joints (converted to per-joint target angles, applied via position control). Position control is more stable than direct torque here, since finger contact is sensitive to small errors.

<img src="/assets/images/projects/ppo-manipulation/apply-action.png" alt="apply_action structure: pos_offset, rot_offset, finger_actions">

_Action space: `9 + #DoF`, split into palm translation, palm rotation, and finger joint control._

Observations follow the same logic: not just current state, but current, reference, and the delta between them, for hand keypoints, palm, object pose, and object-relative fingertip position. Feeding the delta directly gives the policy a much more explicit tracking signal than making it infer the gap between current and target itself.

<img src="/assets/images/projects/ppo-manipulation/hand-keypoints.png" alt="Human hand keypoints (MANO) mapped to robot hand keypoints">

_21-keypoint MANO hand mapped onto the robot hand. Fingertip keypoints get a contact-patch offset; other keypoints map directly to body position._

## The Shortcut I Didn't Expect

The first reward was a straightforward tracking cost: minimize the distance between current and reference hand keypoints, object pose, and object-relative fingertip position. It trained fine, and position error dropped steadily. But watching rollouts revealed the policy wasn't actually grasping anything. It had found a shortcut: keep the palm or a couple of fingers near the object so the *object* tracking cost stays low, without forming a real grasp with the thumb and other fingers on opposite sides.

This makes sense in hindsight. Nothing in a pure tracking reward distinguishes "object is near the reference pose" from "object is stably held." Adding fingertip contact sensors (binary + soft contact force) helped, but only partially: touching an object isn't the same as gripping it, so contact reward alone still couldn't rule out one-sided pushing.

## Finger-Group Reward: Making "Grasp" Explicit

The fix was to stop treating each fingertip independently and instead score the *geometric relationship* between the thumb and the other fingers, in stages:

1. **Opposition**: reward is higher when the thumb-to-object direction and the other-fingers-to-object direction point opposite ways, favoring a pinch over a one-sided push.
2. **Object-between**: the object's center should fall inside the aperture formed by the thumb and the two other fingers with the strongest contact. This pair is selected dynamically per step rather than fixed, since different objects end up gripped by different finger subsets.
3. **Closure & hold**: `inside_bonus` (object between fingers) only counts for much when `balanced_soft_grasp` (thumb and other-finger contact both present) is also true. A `hold_bonus` then rewards low relative velocity between object and palm, favoring a grip that's maintained rather than just touched once.

This is the difference between contact observation ("is a fingertip touching something") and this reward ("do these contacts form a grasp"). It's what actually closed the CASE1 shortcut.

## Reference Reset Curriculum (Mixed RSI)

Even with the right reward, early training rarely reached a grasped state. Most episodes failed during the approach phase, before the grasp reward ever produced a useful gradient. The reward could describe a good grasp; it couldn't make the policy experience one often enough to learn from it.

The fix borrows the idea behind Prioritized Experience Replay, applied to episode resets instead of a replay buffer. Some episodes still start from frame 0, so the policy doesn't forget how to approach the object, while others start from a reference frame selected for having grasp-like geometry: thumb and fingers both near the object, object inside the aperture, object actually in motion in the reference. This is "Mixed RSI," mixing frame-0 resets with reference-state resets so the policy spends more time practicing the hard part.

A few details keep this from destabilizing physics. Object velocity is zeroed on reset, since using the reference velocity mid-grasp caused the object to fly off before the hand had a real hold. Frames requiring too large a palm displacement are also excluded from sampling, to avoid spawning the hand inside the object.

## Early Termination and Friction Curriculum

Two more pieces closed out the main design.

**Early termination**: once contact was expected and the palm had reached the object, episodes with a large and *sustained* tracking failure are cut short, with a penalty so the policy can't game early exit. It's pruning: stop spending training time on trajectories that have already failed, without punishing the normal slips and recoveries that come with any contact-rich task.

**Friction curriculum**: Optional Sequence 2 (lifting a book lying flat on a table) exposed a purely geometric problem. The Shadow Hand's fingertips are too thick to slide under a book flush with the table, no matter how the reward is shaped. Raising static/dynamic friction sharply early in training let the policy drag the book up along the palm using surface friction instead of needing to wedge fingers underneath. Friction is then annealed back down to the real evaluation value over training. Dropping it in one step instead of gradually broke the learned strategy, so the schedule matters as much as the endpoint:

<img src="/assets/images/projects/ppo-manipulation/friction-schedule.png" alt="Friction curriculum schedule table">

## Results

**Main Sequence** converged cleanly. Reward, grasp contact, and object tracking cost all move together, with grasp forming first and object cost dropping afterward.

<img src="/assets/images/projects/ppo-manipulation/main-reward.png" alt="Main sequence reward curve">

<img src="/assets/images/projects/ppo-manipulation/main-grasp-contact.png" alt="Main sequence grasp contact">

<video controls poster="/assets/images/projects/ppo-manipulation/main-sequence-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/main-sequence.mp4" type="video/mp4">
</video>

_Main sequence rollout after training._

**Optional Sequence 1** had noisier contact dynamics. Different object pose and size meant not every finger contributed equally, which is exactly what motivated switching from averaging all four fingers to a top-2 finger selection. Grasp contact and success bonus still trended up together, and object cost dropped to about 0.23 by the end.

<video controls poster="/assets/images/projects/ppo-manipulation/optional-sequence-1-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/optional-sequence-1.mp4" type="video/mp4">
</video>

_Optional sequence 1 rollout._

**Optional Sequence 2** (the book) is where friction curriculum earned its keep. Reward recovered from deeply negative early episodes to 300+ once the policy learned to drag the book up via friction. It's also the one sequence I didn't fully solve: lift and initial grasp became reliable, but holding the book stably while rotating it in-hand stayed noisy, since a firmer grip (good for holding) and rotation (which needs some slip) pull in opposite directions.

<img src="/assets/images/projects/ppo-manipulation/opt2-reward.png" alt="Optional sequence 2 reward curve">

<video controls poster="/assets/images/projects/ppo-manipulation/optional-sequence-2-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/optional-sequence-2.mp4" type="video/mp4">
</video>

_Optional sequence 2 rollout. Lift succeeds; in-hand rotation still unstable._

## What I'd Do Differently

The single biggest lesson wasn't a reward term. Most of the actual debugging happened by watching rollouts, not by staring at reward curves. Several reward terms that looked reasonable on paper barely moved training; a couple of simple gates (contact-based early termination, finger grouping) mattered far more than their complexity suggested.

If I extend this, the reward structure should probably split explicitly into pregrasp, lift, and in-hand-manipulation phases with their own curricula, instead of one framework covering all three. Optional Sequence 2's unresolved rotation-versus-hold conflict is exactly the kind of thing a single global reward struggles to balance.

</div>

<div class="lang-content" data-lang="ko" markdown="1" hidden>

강화학습 수업 기말 프로젝트로, Isaac Lab에서 로봇 손(Shadow Hand)이 실제 시연 영상에서 뽑아낸 사람 손 동작을 모방해 물체를 조작하도록 PPO 정책을 학습시켰다. Input은 프레임별 사람 손 keypoint, 물체 위치, 물체 회전(다중 시점 영상 기반 HO-Cap 데이터)이다. 알고리즘 자체는 스켈레톤 코드로 주어졌고, 실제 작업은 "모방"이 진짜 물리적으로 가능한 "조작"으로 이어지도록 observation과 reward를 설계하는 것이었다.

이 구분이 별거 아닌 것 같아도 중요하다. Reference trajectory는 영상에서 재구성된 것이라 물리적으로 항상 맞지는 않는다. 손과 물체가 시각적으로 겹쳐 보이거나, 실제 grasp 없이 물체가 움직인 것처럼 보이는 경우가 있다. Reference를 그대로 베끼는 정책은 이런 오류까지 그대로 물려받는다. 정책은 keypoint를 맞추는 게 아니라, 시뮬레이터 안의 실제 접촉과 마찰로 같은 결과(물체가 reference를 따라가는 것)를 만들어내야 한다.

## Action / Observation 설계

Reward를 쓰기 전에 먼저 action이 시뮬레이터에서 실제로 어떤 물리적 의미를 갖는지부터 정리했다. Action은 손바닥 위치(손바닥 root에 가해지는 force로 변환), 손바닥 회전(6D rotation → torque로 변환), 손가락 관절(직접 torque가 아니라 관절 목표 angle로 변환해 position control로 적용) 세 그룹으로 나뉜다. 손가락 접촉은 작은 오차에도 민감해서, position control이 torque 직접 제어보다 더 안정적이다.

<img src="/assets/images/projects/ppo-manipulation/apply-action.png" alt="apply_action 구조: pos_offset, rot_offset, finger_actions">

_Action space는 `9 + 관절 수(DoF)`. 손바닥 위치·회전·손가락 관절 제어로 구성._

Observation도 같은 논리를 따른다. 현재 상태만이 아니라 현재/reference/그 차이(delta)를 hand keypoint, palm, object pose, object-relative fingertip 위치에 대해 모두 넣었다. Delta를 직접 넣어주면 정책이 목표와의 차이를 스스로 추론하는 것보다 훨씬 명확한 tracking 신호를 얻는다.

<img src="/assets/images/projects/ppo-manipulation/hand-keypoints.png" alt="사람 손(MANO) keypoint를 로봇 손 keypoint로 매핑">

_21개 keypoint의 MANO 손 모델을 로봇 손에 매핑. Fingertip keypoint는 contact patch 근처로 offset을 주고, 나머지는 body 위치를 그대로 사용._

## 예상 못 한 Shortcut

가장 먼저 만든 reward는 단순한 tracking cost였다. 현재와 reference 사이의 hand keypoint, object pose, object-relative fingertip 거리를 줄이는 것이다. 학습은 잘 됐고 position error도 꾸준히 줄었다. 그런데 rollout을 직접 보니 정책이 실제로는 아무것도 잡고 있지 않았다. 손바닥이나 손가락 한두 개만 물체 근처에 두어 *object* tracking cost만 낮추는 shortcut을 찾은 것이다. 엄지와 다른 손가락이 물체를 사이에 두고 지지하는 진짜 grasp는 없이 말이다.

지금 보면 당연하다. 순수 tracking reward에는 "물체가 reference 위치 근처에 있다"와 "물체가 안정적으로 잡혀 있다"를 구분할 근거가 없다. Fingertip contact sensor(binary + soft contact force)를 추가하니 어느 정도는 도움이 됐지만 부분적이었다. 닿는 것과 쥐는 것은 다르기 때문에, contact reward만으로는 한쪽 손가락으로 미는 것도 여전히 걸러내지 못했다.

## Finger-Group Reward: "Grasp"를 명시적으로 정의하기

해결책은 fingertip을 각각 독립적으로 보는 대신, 엄지와 다른 손가락 사이의 *기하학적 관계*를 단계적으로 평가하는 것이었다.

1. **Opposition**: 엄지에서 물체로 향하는 방향과 다른 손가락에서 물체로 향하는 방향이 반대일수록 보상이 커진다. 한쪽에서 미는 접촉보다 양쪽에서 집는 접촉을 선호하게 된다.
2. **Object-between**: 물체 중심이 엄지와, contact가 가장 강한 다른 손가락 두 개 사이의 aperture 안에 들어와야 한다. 이 둘은 고정된 세트가 아니라 매 step 동적으로 선택하는데, 물체마다 실제로 잡는 손가락 조합이 다르기 때문이다.
3. **Closure & hold**: `inside_bonus`(물체가 손가락 사이에 있음)는 `balanced_soft_grasp`(엄지와 다른 손가락 contact가 동시에 존재)가 참일 때만 의미가 커진다. 이어서 `hold_bonus`는 물체와 손바닥 사이의 상대속도가 작을수록 커지는데, 한 번 닿는 게 아니라 계속 쥐고 있는 상태를 보상하기 위해서다.

이게 contact observation("손가락이 뭔가에 닿았는가")과 이 reward("그 접촉들이 grasp를 이루는가")의 차이다. 이게 실제로 CASE1 shortcut을 막은 지점이다.

## Reference Reset Curriculum (Mixed RSI)

Reward를 제대로 만든 뒤에도 학습 초반에는 grasp 상태에 거의 도달하지 못했다. 대부분의 episode가 grasp reward가 유의미한 gradient를 주기도 전인 접근 단계에서 실패했다. Reward는 좋은 grasp가 뭔지 설명할 수는 있었지만, 정책이 그 상태를 충분히 자주 경험하게 만들지는 못했다.

해결책은 Prioritized Experience Replay의 아이디어를 replay buffer가 아니라 episode reset에 적용한 것이다. 일부 episode는 여전히 frame 0에서 시작하고(그래야 정책이 물체에 접근하는 법을 잊지 않는다), 일부는 grasp-like geometry를 가진 reference frame(엄지와 손가락이 모두 물체 근처, 물체가 aperture 안, reference상 물체가 실제로 움직이는 중)에서 시작한다. 이게 "Mixed RSI"다. Frame-0 reset과 reference-state reset을 섞어서 어려운 구간을 더 자주 연습하게 만든다.

물리적으로 불안정해지지 않도록 몇 가지를 신경 썼다. Object velocity는 reset 시 0으로 두는데, 중간 상태의 reference velocity를 그대로 넣으면 손이 완전히 잡기도 전에 물체가 튕겨나가기 때문이다. Palm 이동량이 너무 큰 frame도 sampling 후보에서 제외했다. 손이 물체 안에 겹쳐서 생성되는 것을 막기 위해서다.

## Early Termination과 Friction Curriculum

두 가지를 더 추가해 설계를 마무리했다.

**Early termination**: contact가 필요한 구간이고 손이 물체 근처까지 도달한 뒤, tracking 실패가 크고 *지속적으로* 유지되는 episode는 조기 종료한다(단, penalty를 줘서 정책이 일부러 빨리 끝내는 걸 막는다). 일종의 pruning이다. 이미 실패한 trajectory에 학습 자원을 쓰지 않으면서도, 접촉이 많은 task에서 자연스러운 미끄러짐·회복까지 실패로 처리하지는 않는다.

**Friction curriculum**: Optional Sequence 2(테이블에 평평하게 놓인 책 들어올리기)는 순수하게 기하학적인 문제를 드러냈다. Shadow Hand의 fingertip은 책과 테이블 사이의 틈으로 들어가기엔 너무 두껍다. Reward를 아무리 조정해도 소용없었다. 학습 초반에 마찰을 크게 높이면, 정책이 손가락을 아래로 끼워 넣는 대신 손바닥 방향 마찰만으로 책을 끌어올릴 수 있었고, 이후 마찰을 실제 평가 조건까지 점진적으로 낮췄다. 한 번에 낮추면 학습된 전략이 무너져서, 최종 값만큼이나 낮추는 속도(schedule)가 중요했다.

<img src="/assets/images/projects/ppo-manipulation/friction-schedule.png" alt="Friction curriculum 스케줄 표">

## 결과

**Main Sequence**는 깔끔하게 수렴했다. Reward, grasp contact, object tracking cost가 함께 움직였고, grasp가 먼저 형성된 뒤 object cost가 낮아지는 순서로 진행됐다.

<img src="/assets/images/projects/ppo-manipulation/main-reward.png" alt="Main sequence reward 그래프">

<img src="/assets/images/projects/ppo-manipulation/main-grasp-contact.png" alt="Main sequence grasp contact 그래프">

<video controls poster="/assets/images/projects/ppo-manipulation/main-sequence-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/main-sequence.mp4" type="video/mp4">
</video>

_학습 완료 후 Main sequence rollout._

**Optional Sequence 1**은 contact dynamics가 더 불안정했다. 물체 자세와 크기가 달라지니 모든 손가락이 똑같이 기여하지 않았고, 이게 바로 네 손가락 평균 대신 top-2 finger 선택으로 바꾼 이유다. Grasp contact와 success bonus는 함께 증가했고, object cost는 최종적으로 약 0.23까지 떨어졌다.

<video controls poster="/assets/images/projects/ppo-manipulation/optional-sequence-1-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/optional-sequence-1.mp4" type="video/mp4">
</video>

_Optional sequence 1 rollout._

**Optional Sequence 2**(책)는 friction curriculum이 진짜 값을 한 부분이다. 초반의 깊은 음수 reward가 friction 덕분에 책을 끌어올리는 법을 익히면서 300 이상까지 회복됐다. 동시에 완전히 풀지 못한 유일한 sequence이기도 하다. Lift와 초기 grasp는 안정적으로 됐지만, 공중에서 책을 쥔 채로 회전시키는 건 계속 불안정했다. 단단히 쥐는 것(hold에 유리)과 회전(어느 정도 미끄러짐이 필요)이 서로 반대 방향으로 당기기 때문이다.

<img src="/assets/images/projects/ppo-manipulation/opt2-reward.png" alt="Optional sequence 2 reward 그래프">

<video controls poster="/assets/images/projects/ppo-manipulation/optional-sequence-2-poster.jpg" style="width:100%;border-radius:8px;display:block;margin:1rem 0;">
  <source src="/assets/videos/projects/ppo-manipulation/optional-sequence-2.mp4" type="video/mp4">
</video>

_Optional sequence 2 rollout. Lift는 성공했지만, 공중에서의 회전은 여전히 불안정._

## 다르게 했다면

가장 큰 교훈은 어떤 reward 항 하나가 아니었다. 실제 디버깅의 대부분은 reward curve를 들여다보는 게 아니라 rollout을 직접 보는 데서 나왔다. 이론적으로 타당해 보였던 여러 reward 항은 학습에 거의 영향을 주지 않았고, 오히려 단순한 장치(contact 기반 early termination, finger grouping) 몇 개가 훨씬 크게 작동했다.

이어서 확장한다면, 하나의 reward 프레임워크로 전부 다루는 대신 pregrasp / lift / in-hand manipulation 단계를 명시적으로 나누고 각자 커리큘럼을 두는 게 나을 것 같다. Optional Sequence 2에서 풀지 못한 회전-대-hold 충돌이 바로 하나의 전역 reward로는 균형을 맞추기 어려운 지점이다.

</div>

<script>
  document.querySelectorAll('.lang-toggle__btn').forEach(function (btn) {
    btn.addEventListener('click', function () {
      var lang = btn.dataset.lang;
      document.querySelectorAll('.lang-toggle__btn').forEach(function (b) {
        b.classList.toggle('is-active', b === btn);
      });
      document.querySelectorAll('.lang-content').forEach(function (el) {
        el.hidden = el.dataset.lang !== lang;
      });
    });
  });
</script>
