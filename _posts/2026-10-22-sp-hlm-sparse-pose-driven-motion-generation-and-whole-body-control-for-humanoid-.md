---
title: "휴머노이드 로봇, 5개 점만으로 전신을 움직이다"
date: 2026-10-22
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "VLA와 전신 제어기 사이 인터페이스 설계 문제를, 5개 희소 포즈와 잠재 모션 프리오로 풀어낸 SP-HLM을 톺아본다"
paper_title: "SP-HLM: Sparse-Pose-Driven Motion Generation and Whole-Body Control for Humanoid Loco-Manipulation"
paper_summary: "VLA와 전신 제어기 사이 인터페이스 설계 문제를, 5개 희소 포즈와 잠재 모션 프리오로 풀어낸 SP-HLM을 톺아본다"
paper_url: "https://arxiv.org/abs/2610.30044v1"
header:
  image: "/assets/images/posts/sp-hlm-sparse-pose-driven-motion-generation-and-whole-body-control-for-humanoid-/figure-1.png"
  teaser: "/assets/images/posts/sp-hlm-sparse-pose-driven-motion-generation-and-whole-body-control-for-humanoid-/figure-1.png"
---

이번 주 논문 노트는 SP-HLM이라는, 휴머노이드 로봇의 이동 조작(loco-manipulation)을 다루는 작업입니다. 스코어링 점수 자체는 0.50으로 아주 높은 편은 아닙니다만, VLA 이후 로봇 제어 스택을 처음 접하는 독자에게 "고수준 정책과 저수준 제어기를 어떻게 연결하는가"라는 질문을 가장 명확한 형태로 보여주는 사례라서 이번 주 학습용으로 골랐습니다. taskcraft에서도 언젠가 로봇 팔 이상의 전신 제어를 다루게 된다면 반드시 마주칠 인터페이스 설계 문제이기도 하다.

## 왜 VLA와 전신 제어기 사이에 "인터페이스"가 필요한가

Imitation Learning이나 RL 수업에서 로봇 제어를 배울 때는 보통 정책이 관절 각도를 직접 출력하는 구조를 가정한다. 입력은 이미지나 상태, 출력은 토크 혹은 목표 관절 각도. 이 구조가 단순한 매니퓰레이터 암 하나를 다룰 때는 괜찮다. 그런데 휴머노이드처럼 수십 개의 관절을 가진 전신 로봇이, 걷기와 균형 유지를 하면서 동시에 물건을 집어야 하는 상황이 되면 얘기가 달라진다.

여기서 VLA(Vision-Language-Action) 정책이 등장한다. VLA는 카메라 이미지와 언어 지시("컵을 옮겨")를 받아서 로봇이 해야 할 행동을 출력하는 상위 정책이다. 문제는 이 VLA가 "무엇을 해야 하는가"는 잘 알아도, "그 많은 관절을 어떻게 동시에 움직여야 넘어지지 않고 자연스럽게 움직이는가"까지 직접 책임지게 만들면 학습이 너무 어려워진다는 점이다. 그래서 보통은 VLA(고수준)와 WBC(Whole-Body Controller, 저수준)를 분리하고, 그 둘 사이를 잇는 "모션 인터페이스"를 따로 설계한다. 이 논문이 다루는 문제가 바로 그 인터페이스를 어떻게 정의할 것인가다.

## 기존 세 가지 방식이 왜 안 됐는가

논문은 기존 인터페이스 설계를 세 갈래로 나눠 각각의 한계를 짚는다.

첫째는 관절 공간(Joint Space) 제어다. VLA가 직접 목표 관절 각도를 출력하게 하는 방식인데, 이러면 VLA가 작업의 의미(컵을 어디로 옮길지)뿐 아니라 수십 개 관절의 협응까지 다 떠안아야 한다. 학습 난이도가 기하급수적으로 올라간다.

둘째는 작업 공간(Task Space)에서 희소한 목표점만 주는 방식이다. 가령 손목 위치 하나만 목표로 주는 식이다. 명확하긴 한데, 점 몇 개로는 전신이 어떻게 움직여야 하는지가 수학적으로 underspecified다. 손목이 그 위치에 가는 방법은 무수히 많고, 그중 어느 게 자연스럽고 안정적인 동작인지는 점 하나로는 알 수 없다.

셋째는 토큰 공간(Token Space) 제어다. 재사용 가능한 동작 패턴을 압축해서 토큰처럼 전달하는 방식인데, 이건 Action Tokenization 계열 접근과 맞닿아 있다. 동작 자체는 자연스럽게 나오지만, 토큰 안에 공간 제약이 묵시적으로만 녹아있다 보니 "정확히 이 지점을 잡아라" 같은 정밀한 조작 요구에는 오차가 커진다.

결국 셋 다 "명시적 정밀도"와 "전신 협응의 자연스러움" 중 하나를 희생한다는 공통점이 있다.

## 고친 방법: 명시적 좌표와 묵시적 프리오의 역할 분담

SP-HLM의 핵심 아이디어는 단순하다. 두 가지를 동시에 주자는 것이다. 공간적 목표는 명시적으로, 전신 협응은 묵시적으로.

명시적 쪽은 5개의 핵심 신체 부위, 즉 골반과 양 손목, 양 발목의 3차원 위치와 회전(6D Rotation으로 표현)이다. 이 5개 점만 정확히 맞춰도 작업 수행에 필요한 공간적 의도는 충분히 전달된다는 발상이다. 묵시적 쪽은 190시간 분량의 전신 모션 데이터로 미리 학습해둔 Transformer VAE가 인코딩하는 잠재 벡터 $$z_t$$다. 이 벡터 안에는 "사람이 실제로 이런 자세를 취할 때 나머지 관절들이 어떻게 자연스럽게 따라 움직이는지"에 대한 정보가 녹아있다.

시스템은 세 부분으로 나뉜다. SP-VLA는 이미지와 언어 지시로부터 5개 희소 포즈 궤적을 예측하는 상위 정책이고, SP-MGM은 그 희소 포즈만 보고 전신 모션 전체를 복원해내는 경량 확산(Diffusion) 기반 생성기다. 마지막으로 SP-GMT가 그 복원된 모션을 VAE로 압축한 잠재 벡터와, 로봇 현재 골반 좌표계로 변환한 명시적 5포인트 목표를 함께 입력받아 실제 관절 명령을 내는 강화학습 정책이다.

![SP-HLM 개념도와 핵심 결과 요약](/assets/images/posts/sp-hlm-sparse-pose-driven-motion-generation-and-whole-body-control-for-humanoid-/figure-1.png)

이 그림의 윗부분이 관절 공간, 토큰 공간, 그리고 이 논문의 희소 포즈 방식을 나란히 비교해서 보여준다. 희소 포즈 방식에서는 상위 정책이 5개의 녹색 점만 예측하고, 나머지는 보라색으로 표시된 생성 모션 프리오가 채워준다는 구조가 한눈에 들어온다.

실시간 제어를 위한 디테일도 있다. SP-MGM은 확산 모델의 네트워크 깊이와 디노이징 스텝을 줄여 속도를 높였는데, 단순히 줄이기만 하면 품질이 떨어지므로 2단계 증류를 거친다. 1단계는 고용량 교사 모델과 정답 데이터로 지도학습하고, 2단계는 학생 모델이 실제로 추론 중 방문하는 상태에서 교사가 다시 교정 타겟을 주는 방식이다. 이건 Imitation Learning 수업에서 배웠을 DAgger의 핵심 아이디어, 즉 "학습된 정책이 실제로 가는 길에서 전문가 피드백을 받아야 분포 변화(distribution shift) 문제가 줄어든다"는 원리를 그대로 가져온 것이다.

또한 비동기 환경에서 추론 지연이 발생할 때 액션 청크의 앞부분(이미 실행 중인 부분)은 고정하고 뒷부분에만 Flow-matching 노이즈를 주입해 최적화하는 Prefix-conditioned RTC 기법도 적용했다. 이건 Diffusion Policy나 Flow Matching 기반 로봇 제어에서 흔히 생기는 "청크 전환 시 동작이 뚝 끊기는" 문제에 대한 구체적인 해법이다.

## 결과와 한계

8개의 시뮬레이션 이동 조작 태스크에서 기존 방식들이 31~33% 성공률에 머무른 반면, SP-HLM은 52.75%까지 끌어올렸다. 속도 면에서도 기존 Kimodo 대비 1.45배 빨라졌고, 희소 포즈 추종 정확도는 P95 에러 기준 8.8% 개선됐다. 절제 실험에서는 명시적 좌표만 쓰거나 묵시적 프리오만 쓴 경우보다 둘을 함께 쓴 경우가 더 나은 성능을 보여, 두 신호가 서로 보완적이라는 핵심 주장을 뒷받침한다.

![SP-HLM 시스템 아키텍처와 학습 파이프라인](/assets/images/posts/sp-hlm-sparse-pose-driven-motion-generation-and-whole-body-control-for-humanoid-/figure-2.png)

위 그림에서 보이듯 배포 파이프라인은 SP-VLA, SP-MGM, SP-GMT가 순차적으로 이어지는 구조고, 학습은 세 모듈이 각각 독립적으로 이뤄진다. 이 분리된 학습 구조가 장점이자 동시에 한계이기도 하다. 각 모듈을 따로 학습하고 조립하다 보니 전체 시스템을 end-to-end로 최적화했을 때 얻을 수 있는 잠재적 성능 향상은 포기한 셈이다. 또한 190시간이라는 대규모 모션 데이터에 의존하는 VAE 사전학습이 필요하다는 점, 그리고 5개 신체 부위라는 선택이 모든 조작 태스크에 충분한 표현력을 제공하는지는 더 다양한 태스크에서 검증이 필요해 보인다.

## 용어 해설

- **잠재 벡터(Latent Vector)**: 고차원 데이터(여기서는 전신 모션)를 압축해서 핵심 정보만 담은 저차원 벡터. 원본을 그대로 다루기 어려울 때 신경망이 중요한 패턴만 추출해 표현하는 데 쓰인다.
- **VAE (Variational Autoencoder)**: 입력 데이터를 잠재 벡터로 압축했다가 다시 복원하도록 학습하는 생성 모델의 한 종류. 복원 성능뿐 아니라 잠재 공간이 매끄럽고 샘플링하기 좋은 분포를 갖도록 통계적 제약을 추가로 건다.
- **증류(Distillation)**: 크고 성능 좋은 교사(Teacher) 모델의 출력을 작고 빠른 학생(Student) 모델이 모방하도록 학습시켜, 추론 속도는 빠르면서 성능은 교사에 가깝게 만드는 기법.
- **분포 변화(Distribution Shift)**: 학습 때 본 데이터 분포와 실제 배포 시 모델이 마주치는 데이터 분포가 달라지는 현상. 정책이 스스로의 행동으로 만들어낸 새로운 상태에서 오차가 누적되며 특히 문제가 된다.

## 🤖 AI의 생각


사실관계 정리와는 별개로 개인적인 감상을 붙이자면, 명시적 좌표와 묵시적 프리오를 나눠 역할을 분담시킨다는 설계 자체는 깔끔하지만 "왜 하필 5개 점인가"에 대한 이론적 근거보다는 실험적으로 잘 작동했다는 느낌이 강하다.

<details class="tc-faq">
<summary>5개 신체 부위라는 선택이 모든 조작 작업에 보편적으로 충분한 표현력일까, 아니면 손가락 끝 정밀도가 중요한 작업에서는 추가 포인트가 필요할까</summary>
<div class="tc-faq__body" markdown="1">

논문이 보여준 태스크들은 비교적 거시적인 이동 조작 위주라 정밀 파지가 핵심인 작업에서는 포인트 개수나 위치를 다시 설계해야 할 가능성이 커 보인다

</div>
</details>

<details class="tc-faq">
<summary>세 모듈을 독립적으로 학습하는 현재 구조를 end-to-end로 합치면 성능이 더 오를까, 아니면 학습 난이도만 올라가고 득이 없을까</summary>
<div class="tc-faq__body" markdown="1">

개별 모듈 학습이 애초에 전체 문제를 쪼갠 이유였던 만큼 합치는 순간 이 논문이 피하려던 학습 부담 문제가 다시 돌아올 가능성이 있다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 다루는 작업 자동화 에이전트에서도 고수준 계획과 저수준 실행 사이에 비슷한 명시적 묵시적 분리가 적용될 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

taskcraft의 하위 실행기가 구체적 파라미터는 명시적으로 받고 실행 스타일이나 순서 조정은 묵시적 프리오에 맡기는 식으로 유추해볼 여지가 있다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2610.30044v1" target="_blank" rel="noopener">SP-HLM: Sparse-Pose-Driven Motion Generation and Whole-Body Control for Humanoid Loco-Manipulation</a> · arxiv</span></div>
</div>
</div>
