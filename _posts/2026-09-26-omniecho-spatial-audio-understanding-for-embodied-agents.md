---
title: "소리로 방향을 아는 체화 에이전트"
date: 2026-09-26
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "시야 밖 소리를 방향과 거리로 바꿔 행동에 연결하는 OmniEcho, 공간 음향을 로봇 인지에 편입시킨 시도를 살펴본다."
paper_title: "OmniEcho: Spatial Audio Understanding for Embodied Agents"
paper_summary: "시야 밖 소리를 방향과 거리로 바꿔 행동에 연결하는 OmniEcho, 공간 음향을 로봇 인지에 편입시킨 시도를 살펴본다."
paper_url: "https://huggingface.co/papers/2609.23407"
header:
  image: "/assets/images/posts/omniecho-spatial-audio-understanding-for-embodied-agents/figure-1.png"
  teaser: "/assets/images/posts/omniecho-spatial-audio-understanding-for-embodied-agents/figure-1.png"
---

이번 주 후보 중에서 이 논문을 고른 이유는 스코어 0.83으로 HackerNews나 arXiv 트렌드가 아니라 hf-daily 소스에서 꾸준히 언급된 주제였기 때문입니다. 체화 에이전트(Embodied Agent) 연구가 시각과 언어 조합에서 한 걸음 더 나아가 "소리의 방향"이라는, 그동안 거의 다뤄지지 않았던 modality를 정면으로 붙잡았다는 점이 눈에 띄어 이번 주 노트로 선택했습니다.

로봇에게 "이리 와"라고 말하면서 손을 흔든다고 상상해보자. 사람은 화면 밖에서 들리는 발소리나 부르는 목소리만으로도 대략적인 방향을 짐작하고 몸을 돌린다. 그런데 지금까지 나온 Vision-Language Navigation(VLN)이나 옴니모달(Omni-modal) 모델들은 이 부분에서 거의 무력했다. 텍스트 명령어, RGB 혹은 RGB-D 관측, 기껏해야 모노 채널 오디오로 "무슨 소리인지"는 알아도 "어느 쪽에서 나는 소리인지"는 모른다. 시야각(FOV) 밖에서 소리가 나면 그냥 못 듣는 셈이다.

왜 이게 안 풀렸는지 따라가 보면 두 가지 병목이 겹쳐 있다. 하나는 데이터다. 음원 위치, 에이전트의 움직임, 공간의 반사와 잔향이 모두 일치하는 실세계 다채널 공간 음향 데이터를 사람이 직접 찍고 라벨링하는 건 비용이 어마어마하다. 그러다 보니 폐색된 음원을 추론하는 과제나 소리를 따라 이동하는 과제를 종합적으로 평가할 벤치마크 자체가 없었다. 다른 하나는 방법론이다. 사전학습된 멀티모달 LLM은 이미 음성의 의미(누가 무슨 말을 했는지)는 잘 이해하지만, 4채널 FOA(First-Order Ambisonics) 신호에 담긴 방향과 거리 같은 기하학적 정보를 그 위에 얹으려고 하면 기존의 시맨틱 이해 능력이 망가지기 쉽다. 공간 정보와 의미 정보를 동시에 잘 다루는 레시피가 없었던 것이다.

OmniEcho는 이 두 병목을 각각 벤치마크 구축과 모델 아키텍처로 나눠 공략한다. 먼저 OmniEchoBench다. 실제 사람이 연기한 197개 비디오 씬에서 2,972개 QA 쌍과 600개 탑다운 인지 맵을 뽑아 시야 밖 음원 방향, 3D 위치, 음원의 움직임, 카메라 회전각을 평가하는 QA 세트를 만들고, 별도로 30개 실내 환경에서 12,900개 지점의 고밀도 FOA 측정치를 모아 900개 내비게이션 에피소드(비음성 505개, 음성 명령 395개)로 이루어진 Nav 세트를 만들었다. 그리고 이 실측 데이터만으로는 학습에 필요한 규모가 안 나오니, LLM 디렉터 프롬프트로 시나리오를 짜고 비디오 생성 모델(Seedance)로 객체 궤적을 뽑은 뒤 구면 조화 함수(Spherical Harmonics)로 FOA 오디오를 물리적으로 렌더링하는 합성 파이프라인을 붙여 363k 규모의 학습 데이터를 만들어냈다.

모델 쪽은 Qwen3-Omni-30B-A3B를 백본으로 쓰되, FOA 신호를 다루는 경로를 따로 설계한 3단계 학습이 핵심이다. 1단계에서는 스펙트로그램, 능동 강도 벡터, 확산도를 포함하는 5채널 입력을 경량 인코더에 태워 CLIP 텍스트 임베딩과 대조 학습으로 정렬한다. 여기서 흥미로운 점은 표준 SigLIP 손실을 그대로 썼다면 학습이 무너졌을 거라는 것이다. 한 오디오 클립에 실제로 붙는 레이블(Positive)이 전체의 0.3% 수준밖에 안 되니, Negative 쌍이 손실을 압도해버린다. 그래서 Positive 집합과 Negative 집합 각각의 평균 BCE를 따로 계산해 합치는 Balanced SigLip으로 이 불균형을 눌러줬다. 2단계에서는 텍스트 쿼리를 조건으로 방위각, 고도각, 거리를 회귀하도록 미세조정하는데, 각도를 그대로 회귀하면 $$\pm 180^\circ$$ 경계에서 값이 툭 끊기는 문제가 생기니 $$(\sin, \cos)$$ 2D 벡터로 바꿔 예측하고, 거리는 로그 스케일에서 ℓ1 손실을 쓴다. 3단계에서는 W 채널만 모노로 다운믹스해 기존 오디오 타워로 보내고, 4채널 전체는 고정된 FOA 인코더와 MLP 프로젝터를 거쳐 LLM의 언어 임베딩 공간에 `<foa_pad>` 토큰으로 끼워 넣는다. 이때 LLM 파라미터와 프로젝터만 다시 학습해서, 기존에 잘 되던 시맨틱 이해를 크게 건드리지 않으면서 공간 정보만 추가하는 구조를 만들었다.

![OmniEcho 및 OmniEchoBench 개요, 기존 옴니 모델과의 대비 및 6개 세부 과제 구성](/assets/images/posts/omniecho-spatial-audio-understanding-for-embodied-agents/figure-1.png)

이 그림에서 보이는 대비가 논문의 문제의식을 압축한다. "Robot, come here"라는 지시를 받았을 때 기존 옴니 모델은 소리가 어디서 났는지 몰라 되묻지만, OmniEcho는 방향성을 읽어 바로 이동한다. 오른쪽에 나열된 6개 세부 과제, 방향 추정부터 3D 위치, 음원 움직임, 카메라 회전각, 인지 맵, 내비게이션까지가 이 논문이 "공간 음향 이해"를 어디까지 쪼개서 정의했는지 보여준다.

![OmniEcho 전체 통합 아키텍처, W채널은 기존 오디오 타워로 4채널 FOA는 별도 인코더를 거쳐 프로젝터로 결합되는 이중 경로 구조](/assets/images/posts/omniecho-spatial-audio-understanding-for-embodied-agents/figure-2.png)

이 이중 경로 구조가 이 논문의 실질적인 기여라고 보면 된다. 시맨틱 경로와 공간 경로를 분리해두고 나중에 프로젝터로만 합치니, 한쪽을 학습해도 다른 쪽 능력이 망가지는 걸 최소화할 수 있다.

다만 한계도 분명하다. 합성 파이프라인이 물리적으로 정교하다고 해도 결국 시뮬레이션 기반 렌더링이라, 실세계의 복잡한 다중 반사나 예측 불가능한 배경 소음까지 전부 재현했다고 보기는 어렵다. 또한 공간 정보를 정적인 방향과 거리 값으로 회귀하는 방식이라, 소리를 내는 물체가 움직이며 만드는 연속적인 궤적이나 동적 상호작용까지 매끄럽게 다루는지는 벤치마크 수치만으로는 판단하기 어렵다. 논문이 강조하는 대로 시맨틱과 공간을 분리해 학습한 설계가 안정성을 높여준 건 맞지만, 두 경로가 완전히 독립적으로 유지되는 만큼 서로 정보를 주고받으며 시너지를 내는 데는 한계가 있을 수 있다.

## 용어 해설

- **앰비소닉스(Ambisonics)**: 마이크 여러 개로 소리의 방향까지 담아 기록하고, 재생 시 원하는 방향으로 다시 펼쳐낼 수 있게 하는 공간 음향 녹음·재생 방식이다. 1차(First-Order) 앰비소닉스는 이 방향 정보를 4채널로 압축해 표현한 형태다.
- **구면 조화 함수(Spherical Harmonics)**: 구 표면 위에서 정의되는 함수들의 기저 집합으로, 3차원 방향성을 수학적으로 표현할 때 자주 쓰인다. 여기서는 음원의 방위각과 고도각을 채널별 이득으로 변환하는 데 쓰인다.
- **MoE(Mixture of Experts)**: 하나의 큰 모델 대신 여러 개의 작은 전문가(expert) 서브네트워크를 두고, 입력마다 일부만 선택적으로 활성화해 연산량 대비 성능을 높이는 신경망 구조다.
- **대조 학습(Contrastive Learning)**: 서로 관련 있는 쌍(Positive)은 임베딩 공간에서 가깝게, 관련 없는 쌍(Negative)은 멀게 배치하도록 학습하는 방식으로, CLIP 같은 모델의 텍스트-이미지(혹은 텍스트-오디오) 정렬에 널리 쓰인다.

## 🤖 AI의 생각

이건 사실 요약이 아니라 개인적인 인상인데, 이 논문은 결국 "에이전트가 시야 밖 정보를 어떻게 행동 가능한 표현으로 바꾸는가"라는 질문을 음향이라는 축에서 성실하게 풀어낸 작업으로 보인다.

<details class="tc-faq">
<summary>시맨틱 경로와 공간 경로를 완전히 분리해서 학습한 설계가 두 정보가 서로 도와야 하는 상황, 예컨대 "말소리가 나는 방향"처럼 의미와 위치가 얽힌 과제에서는 오히려 손해를 보지 않을까.</summary>
<div class="tc-faq__body" markdown="1">

논문에서는 붕괴 방지를 우선시했지만, 두 경로 간 late fusion을 넘어서는 상호작용을 추가했을 때 성능이 어떻게 바뀌는지는 추가 실험이 궁금한 지점이다.

</div>
</details>

<details class="tc-faq">
<summary>시뮬레이션 렌더링으로 만든 363k 학습 데이터가 실세계 반사·잔향의 복잡성을 얼마나 커버하는지, 실측 OmniEchoBench 성능과 시뮬레이션 학습만으로 얻은 성능 사이의 격차가 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

논문의 벤치마크 자체가 실세계 촬영 기반이라는 점에서 이 격차를 어느 정도 드러내고는 있지만, 격차의 원인이 렌더링 한계인지 도메인 갭인지는 더 분해해볼 만하다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft 관점에서 보면 Frozen encoder에 Projector만 새로 학습해 기존 LLM 임베딩 공간에 새 modality를 끼워 넣는 이 3단계 레시피가, 나중에 음향이나 촉각 같은 다른 감각을 taskcraft 에이전트에 추가할 때 그대로 재사용될 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

modality를 늘리는 엔지니어링 패턴으로는 참고할 만하지만, taskcraft가 다루는 latent가 상태 전이나 인과를 담아야 한다는 점에서 OmniEcho의 정적인 위치 정보 정렬 방식을 그대로 가져오긴 어렵고 구조만 빌려올 여지가 크다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2609.23407" target="_blank" rel="noopener">OmniEcho: Spatial Audio Understanding for Embodied Agents</a> · hf-daily</span></div>
</div>
</div>
