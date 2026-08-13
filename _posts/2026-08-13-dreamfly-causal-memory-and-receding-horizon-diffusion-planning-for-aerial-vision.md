---
title: "드론 항법에 기억과 계획을 더하다"
date: 2026-08-13
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "드론이 과거 시야를 잊지 않고 몇 수 앞을 내다보며 걷되, 멈춰야 할 때는 따로 판단하게 만든 항법 프레임워크를 살펴본다"
paper_title: "DreamFly: Causal Memory and Receding-Horizon Diffusion Planning for Aerial Vision-Language Navigation"
paper_summary: "드론이 과거 시야를 잊지 않고 몇 수 앞을 내다보며 걷되, 멈춰야 할 때는 따로 판단하게 만든 항법 프레임워크를 살펴본다"
paper_url: "https://arxiv.org/abs/2608.12308v1"
header:
  image: "/assets/images/posts/dreamfly-causal-memory-and-receding-horizon-diffusion-planning-for-aerial-vision/figure-1.png"
  teaser: "/assets/images/posts/dreamfly-causal-memory-and-receding-horizon-diffusion-planning-for-aerial-vision/figure-1.png"
---

이번 주는 arXiv에서 스코어링된 논문 중 DreamFly를 골랐습니다. 점수 자체는 0.17로 아주 높은 편은 아니지만, VLA(Vision-Language-Action) 이후 로봇 정책 설계가 어떤 고민을 하고 있는지 보여주는 사례로는 손색이 없다고 판단했습니다. Imitation Learning이나 RL의 고전적인 틀은 수업에서 다들 배웠지만, 그 위에 diffusion 같은 생성 모델을 얹고 메모리와 종료 판단까지 따로 설계하는 요즘 흐름은 처음 보면 낯설 수 있습니다. 이번 글에서는 드론이 지시문을 듣고 3D 공간을 항법하는 문제를 통해, 그 낯선 조각들을 하나씩 풀어보겠습니다.

## 드론에게 길을 알려준다는 문제

Vision-Language Navigation(VLN)은 "왼쪽으로 돌아서 빨간 지붕 건물을 지나 멈춰라" 같은 자연어 지시문을 받고, 카메라로 본 것만으로 그 지시를 수행하는 문제다. 지상 로봇에서는 이미 여러 해 연구된 주제인데, 드론으로 옮기면 문제가 하나 더 붙는다. 고도가 계속 바뀌면서 같은 랜드마크가 화면에서 커졌다 작아졌다 하고, 건물이나 나무에 가려 방금 본 목표물이 갑자기 시야에서 사라진다. 사람도 비행하면서 방향을 잡을 때 "아까 저기서 봤던 그 건물"을 기억해두는데, 드론 에이전트도 마찬가지로 과거에 본 것을 기억해야 하고, 동시에 매 순간 다음 몇 걸음을 계획해야 하고, 마지막으로 목표에 도착했을 때 정확히 멈춰야 한다. DreamFly는 이 세 가지를 각각 별도의 모듈로 풀어낸 논문이다.

![드론 항법 문제 상황과 DreamFly의 전체 파이프라인](/assets/images/posts/dreamfly-causal-memory-and-receding-horizon-diffusion-planning-for-aerial-vision/figure-1.png)

## 기억이 새는 문제

가장 먼저 짚을 부분은 과거 시각 정보를 어떻게 기억하느냐다. 직관적으로는 "지금까지 본 프레임을 다 저장해두고 필요할 때 꺼내 쓰면 되지 않나" 싶지만, 실제로 이렇게 구현하면 두 가지 문제가 생긴다. 하나는 저장 공간이 계속 늘어난다는 것이고, 더 심각한 건 학습 데이터를 만드는 과정에서 미래 정보가 메모리에 섞여 들어가는 누수(leakage)가 생기기 쉽다는 점이다. 예를 들어 사람이 만든 시연 궤적 전체를 놓고 메모리를 구성하면, $$t$$ 시점의 결정을 내릴 때 실제로는 $$t$$ 이후에 일어난 일까지 참고하게 되는 경우가 생긴다. 학습할 때는 성능이 잘 나오는 것처럼 보이지만, 실제 배포 환경에서는 미래를 미리 알 수 없으니 이 이점이 그대로 사라지고, 오히려 잘못된 습관이 든 정책이 나온다.

DreamFly는 이걸 '읽기 후 쓰기(read-before-write)'라는 단순하지만 엄격한 규칙으로 해결한다. $$t$$ 시점에 의사결정을 내릴 때는 오직 $$t$$ 이전 관측으로만 구성된 메모리 $$M_{<t}$$를 조회하고, 현재 관측 $$O_t$$는 결정이 끝난 다음에야 메모리에 추가된다. 메모리 자체도 CLIPSeg와 OWLv2 같은 open-vocabulary 검출 모델로 지시문과 관련된 시각 후보만 골라 16개 슬롯짜리 고정 크기로 유지한다. 이렇게 걸러낸 과거 정보를 현재 피처에 합칠 때는 다음과 같은 게이트 잔차 연결을 쓴다.

$$\tilde{Z}_t = Z_t + M_{\text{img}} \odot G_t \odot (C_t W_O)$$

여기서 $$C_t$$는 현재 피처 $$Z_t$$가 쿼리가 되어 과거 메모리 슬롯을 교차 주의집중(cross-attention)으로 조회한 결과이고, $$G_t = 1 + \tanh(Z_t W_G + b_G)$$는 그 결과를 얼마나 반영할지 결정하는 게이트다. 이 게이트 덕분에 과거 메모리가 유의미하지 않은 순간에는 자연스럽게 영향력이 줄어들고, 강제로 항상 과거를 참고하게 만드는 부작용을 피한다. 정리하면 메모리는 무조건 쌓는 게 아니라 시간 순서를 지키면서 골라 담고, 필요할 때만 꺼내 쓰는 구조다.

![DreamFly의 인과적 메모리, 후퇴 지평선 계획, LiteStop 구조를 보여주는 전체 개요도](/assets/images/posts/dreamfly-causal-memory-and-receding-horizon-diffusion-planning-for-aerial-vision/figure-2.png)

## 한 번에 볼 것인가, 한 걸음씩 갈 것인가

두 번째 축은 행동을 어떻게 계획하느냐다. 최근 로봇 정책 학습에서 많이 쓰이는 방식 중 하나가 Diffusion Policy인데, 간단히 말하면 노이즈에서 시작해서 점점 노이즈를 제거해가며 행동 시퀀스를 만들어내는 생성 모델을 정책으로 쓰는 접근이다. 한 스텝짜리 행동만 예측하는 방식보다 장점은, 디퓨전 모델이 여러 스텝의 행동을 한꺼번에 하나의 청크(chunk)로 만들어낼 수 있어서 몇 수 앞을 내다보는 조망 능력이 생긴다는 점이다.

문제는 이 청크를 실제로 어떻게 실행하느냐다. 만든 $$K$$개의 행동을 그대로 순서대로 다 실행해버리면(open-loop), 그 사이에 들어오는 새로운 시각 정보를 전혀 반영하지 못한다. 드론이 예상과 다르게 살짝 어긋난 방향으로 날아가고 있어도, 청크를 다 소진할 때까지는 고칠 방법이 없다. 반대로 매 스텝 한 개의 행동만 예측하면 반응성은 좋아지지만 앞서 말한 조망 능력을 잃는다. DreamFly는 이 둘을 절충해서 'Plan-K, Execute-One' 전략을 쓴다. 이산 디퓨전 모델(Dream-VLA)로 $$K$$개 스텝의 행동을 한꺼번에 예측하되, 실제로는 그중 첫 번째 행동만 실행하고, 다음 스텝에서 새로 들어온 관측으로 다시 $$K$$개를 통째로 재계획한다. 후퇴 지평선(receding horizon)이라는 이름 그대로, 계획하는 창은 항상 앞으로 $$K$$스텝을 보지만 실행은 매번 한 스텝씩만 하는 것이다.

학습할 때도 이 우선순위를 손실함수에 반영한다.

$$\mathcal{L}_{\text{act}} = \frac{\sum_{t \in \mathcal{B}} \sum_{h=0}^{K-1} v_{t,h} c_{t,h} \gamma^h \text{CE}_{\mathcal{V}}(z_{t,h}, \chi(\bar{a}_{t,h}^*))}{\sum_{t \in \mathcal{B}} \sum_{h=0}^{K-1} v_{t,h} c_{t,h} \gamma^h}$$

여기서 $$\gamma^h$$는 지평선 감쇠 가중치로, $$h$$가 커질수록(즉 먼 미래의 행동일수록) 손실에 반영되는 비중이 줄어든다. 어차피 실제로 실행되는 건 첫 행동뿐이니, 모델이 그 첫 행동을 정확히 맞추는 데 더 집중하도록 훈련 신호를 설계한 것이다. 먼 미래 예측은 참고 자료 정도로만 쓰고, 당장 실행할 한 걸음에 학습 노력을 몰아준다는 발상이 꽤 실용적이다.

## 멈추는 것은 다른 문제다

세 번째 축은 언제 멈출 것인가다. 이 부분이 개인적으로 이 논문에서 가장 눈에 들어온 지점인데, 저자들은 이동 오류와 정지 오류가 근본적으로 다른 위험을 가진다는 점을 지적한다. 방향을 살짝 잘못 잡은 이동 오류는 다음 스텝에서 재계획하며 고칠 수 있지만, 목표에 도착하기 전에 잘못 멈추거나 도착했는데도 멈추지 않으면 그 순간 에피소드 전체가 실패로 끝난다. 그런데 일반적인 방식대로 정지 행동을 다른 이동 행동들과 똑같은 하나의 생성 과정 안에서 다루면, 학습 신호가 이 비대칭성을 구분하지 못하고 뭉뚱그려진다.

DreamFly는 그래서 정지 판단을 아예 별도 모듈인 LiteStop으로 분리한다. 흥미로운 점은 이 모듈이 디퓨전의 전체 디노이징 과정이 끝난 결과물을 보는 것이 아니라, 맨 처음 모든 행동 토큰이 마스킹된 상태에서 나오는 초기 로짓 그리드 $$H_t^{(0)}$$만 보고 판단한다는 것이다.

$$\ell_t^{\text{stop}} = g_{\text{stop}}(H_t^{(0)}) = W_2 \, \text{SiLU}(W_1 \, \text{LN}(\text{vec}(H_t^{(0)})) + b_1) + b_2$$

이 가벼운 MLP 헤드는 본체 정책의 파라미터를 모두 동결한 채로 따로 학습된다. 즉 정지 여부를 판단하는 데 굳이 디퓨전 전체 단계를 다 기다릴 필요가 없고, 계획의 첫 반응만 보고도 빠르게 결정을 내릴 수 있다는 뜻이다. 정지라는 하나의 행동을 위해 별도의 학습 목표와 별도의 네트워크를 준비한 셈인데, 이는 모든 행동을 하나의 모방 학습 손실로 뭉뚱그리던 기존 관행에서 한 걸음 벗어난 설계다.

## 한계와 남는 질문

논문 노트에는 정량적 비교표까지는 정리되어 있지 않아서, 이 글에서 구체적인 성공률 수치를 인용하기는 조심스럽다. 구조적으로 봐도 이 프레임워크는 처음부터 끝까지 드론이라는 하나의 embodiment와 그 행동 공간(이산 액션 토큰, stop 여부)에 맞춰 설계되어 있어서, 지상 로봇이나 매니퓰레이터처럼 다른 형태의 에이전트로 그대로 옮겨 쓸 수 있는 구조는 아니다. 다만 과거 관측을 인과적으로 정렬해 별도 메모리로 쌓아두는 방식이나, 행동을 디코딩하기 전에 먼저 세계 상태에 대한 표현을 뽑고 그로부터 정지 여부 같은 부가 판단을 내리는 LiteStop의 구조는, 지각·기억 표현과 행동 디코더를 어느 정도 떼어놓으려는 태도로 읽을 수 있다.

## 용어 해설

- **Cross-Attention (교차 주의집중)**: 두 개의 서로 다른 시퀀스(예를 들어 현재 관측과 과거 메모리) 중 하나를 쿼리로, 다른 하나를 키와 값으로 써서 서로 다른 출처의 정보를 결합하는 attention 메커니즘이다.
- **이산 Diffusion (Discrete Diffusion)**: 이미지 픽셀처럼 연속값이 아니라 토큰처럼 이산적인 값을 대상으로, 노이즈를 점진적으로 씌우고 벗기는 과정을 마스킹과 복원으로 대체해 학습하는 diffusion 모델의 한 변형이다.
- **Open-Vocabulary Detection**: 학습 시 정해둔 고정된 클래스 목록에 없는 대상도, 자연어로 주어진 임의의 이름이나 설명만으로 이미지 속에서 찾아내는 객체 검출 방식이다.

## 🤖 AI의 생각


개인적인 의견을 덧붙이자면, 이 논문에서 가장 인상 깊었던 결정은 정지라는 단 하나의 행동을 위해 별도의 네트워크와 학습 목표를 따로 마련한 부분이다. 아래는 더 파고들면 흥미로울 지점들을 정리해봤다.

<details class="tc-faq">
<summary>LiteStop이 초기 마스크 상태의 로짓만 보고 정지를 판단하는데, 이게 디퓨전 전체 단계를 다 거친 뒤의 판단보다 정말 더 안정적일까</summary>
<div class="tc-faq__body" markdown="1">

논문 노트에는 이 설계 선택의 이유(속도)만 나와 있고 전체 단계 사용과의 정량 비교는 없어서, 초기 단계만으로 충분한지는 별도 검증이 필요해 보인다

</div>
</details>

<details class="tc-faq">
<summary>지평선 감쇠 가중치 $$\gamma$$ 값 하나로 먼 미래 예측의 중요도를 낮추는데, 이 값이 너무 작으면 애초에 K스텝을 계획하는 의미가 줄어들지 않을까</summary>
<div class="tc-faq__body" markdown="1">

감쇠가 강할수록 사실상 1-스텝 예측에 가까워지므로, 조망 능력과 학습 안정성 사이에서 $$\gamma$$가 실질적인 트레이드오프 손잡이 역할을 할 것 같다

</div>
</details>

<details class="tc-faq">
<summary>지각 표현과 행동 디코더를 분리하는 이 논문의 구조가 taskcraft가 지향하는 embodiment 무관 표현과 얼마나 통할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

여기서 분리된 표현은 여전히 드론의 이산 행동 공간에 강하게 묶여 있어서, 그대로 재사용하기보다는 "지각과 행동을 나눈다"는 설계 원칙만 참고할 수 있을 것 같다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2608.12308v1" target="_blank" rel="noopener">DreamFly: Causal Memory and Receding-Horizon Diffusion Planning for Aerial Vision-Language Navigation</a> · arxiv</span></div>
</div>
</div>
