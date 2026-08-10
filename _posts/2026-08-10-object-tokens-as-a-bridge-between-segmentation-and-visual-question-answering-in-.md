---
title: "세그멘테이션과 VQA를 잇는 토큰"
date: 2026-08-10
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
excerpt: "수술 VLM에 SAM을 붙이며 픽셀과 언어 사이를 잇는 매개 토큰 하나를 만든 연구를 뜯어봤다."
paper_title: "Object Tokens as a Bridge Between Segmentation and Visual Question Answering in Robotic Surgery"
paper_summary: "수술 VLM에 SAM을 붙이며 픽셀과 언어 사이를 잇는 매개 토큰 하나를 만든 연구를 뜯어봤다."
paper_url: "https://www.semanticscholar.org/paper/feff3523b359e48cee166905f466c153464b3f87"
---

## 이번 주에 이 논문을 고른 이유

이번 주 후보 중 이 논문의 스코어(0.85)가 유독 높게 나온 건 추천 경로가 semantic-scholar-rec, 즉 최근 관심 있게 본 논문들과의 유사도 기반 추천이었기 때문입니다. 직접적으로 taskcraft와 맞닿는 주제는 아니지만, VLM과 다른 모듈(여기서는 SAM)을 어떻게 연결할 것인가라는 인터페이스 설계 문제를 다루고 있어서 읽어볼 가치가 있다고 판단했습니다. 결론부터 말씀드리면 이 논문이 taskcraft의 문제를 직접 풀어주지는 않습니다. 다만 서로 다른 표현 공간을 잇는 매개 토큰이라는 패턴 자체는 눈여겨볼 만했습니다.

## 문제: 수술 VQA는 위치를 짚지 못했다

로봇 수술 장면에서 "이 겸자가 잡고 있는 조직이 뭐냐" 같은 질문에 답하려면 모델이 먼저 대상을 정확히 짚어야 한다. 그런데 기존 Surgical VQA, Surgical-VQLA 계열 연구들은 대체로 바운딩 박스 수준의 개략적인 위치 특정에 머물러 있었다. 수술 도구나 장기 조직은 경계가 얇고 겹치는 경우가 많아서, 사각형 박스 하나로는 "정확히 어디까지가 그 객체인지"를 표현할 수 없다. 반면 VLM 자체는 자유 형태 텍스트 답변을 생성하는 데는 강하지만, 그 답변이 실제로 이미지의 어느 픽셀에 근거하고 있는지를 명시적으로 드러내지 못한다는 문제가 있었다.

## 왜 안 됐는지: VLM과 SAM은 원래 따로 노는 모델

세그멘테이션과 VQA를 각각 잘하는 모델은 이미 있다. SAM은 픽셀 단위 마스크를 뽑는 데 특화돼 있고, Qwen2.5-VL 같은 VLM은 언어 생성에 특화돼 있다. 문제는 이 둘을 그냥 파이프라인으로 이어 붙이면(먼저 세그멘테이션하고 그 결과를 텍스트로 요약해서 VLM에 다시 넣는 식) 두 단계가 분리되면서 세그멘테이션 정보가 답변 생성 과정에 제대로 녹아들지 않는다는 데 있다. 즉 마스크는 마스크대로 나오고 답변은 답변대로 나오는, 형식적으로만 붙어있는 결합이 된다. 저자들이 원한 건 **세그멘테이션 결과가 답변 생성의 실제 추론 근거로 쓰이는 구조**였다.

## 고친 방법: <sam_pad> 토큰이라는 매개체

이 논문의 핵심 아이디어는 객체 이름 뒤에 `<sam_pad>`라는 특수 토큰을 붙여서, VLM이 텍스트를 생성하는 과정 중에 이 토큰을 자연스럽게 출력하도록 학습시키는 것이다. 이 토큰의 latent embedding $$z$$는 선형 레이어를 거쳐 SAM 프롬프트 공간의 벡터 $$z_m = Wz + b$$로 변환되고, 다시 객체 클래스별 학습 가능한 쿼리 $$q$$를 이용한 Cross-Attention

$$z_p = \text{Attn}(q, k, v) = \text{softmax}\left(\frac{q k^\top}{\sqrt{d_k}}\right) v$$

을 거쳐 SAM 디코더의 입력 프롬프트로 전달된다. 여기서 $$k, v$$는 모두 $$z_m$$에서 나온다. 이 구조가 재미있는 건 `<sam_pad>` 토큰이 세그멘테이션을 위한 입력으로만 쓰이는 게 아니라, 뒤이어 생성되는 답변 토큰들이 causal attention을 통해 이 토큰을 다시 참조할 수 있다는 점이다. 즉 같은 토큰이 한 번은 SAM 쪽으로, 한 번은 답변 생성 쪽으로 두 갈래로 쓰인다.

![전체 프레임워크: 프롬프트가 VLM에 들어가면 객체 이름과 함께 sam_pad 토큰이 생성되고, 이 토큰이 프로젝션을 거쳐 SAM 디코더로 들어가는 동시에 답변 토큰 생성의 근거가 되는 구조](/assets/images/posts/object-tokens-as-a-bridge-between-segmentation-and-visual-question-answering-in-/figure-1.png)

이걸 한 번에 학습시키면 언어 모델링 목표와 세그멘테이션 목표가 서로를 방해하기 쉬워서, 저자들은 3단계 커리큘럼을 썼다. 먼저 VQA만으로 워밍업하고, 그다음 세그멘테이션 감독 학습을 붙이고, 마지막에 두 목표를 함께 섞어 공동 학습하는 순서다. 순서를 이렇게 짠 이유는 언어 능력이 먼저 안정화되지 않은 상태에서 세그멘테이션 손실을 섞으면 답변 생성 품질이 무너지기 때문으로 보인다.

## 결과: Attention이 실제로 객체를 짚는다

가장 설득력 있는 검증은 벤치마크 수치보다 attention 시각화 쪽이었다. EndoVis18과 RAMIE 데이터셋에서 답변을 생성할 때 VLM 내부 attention map을 보면, 답변 토큰이 앞서 생성된 여러 객체 토큰 중에서 의미적으로 관련 있는 토큰(예: 실제로 질문에 등장하는 수술 도구나 조직에 대응하는 `<sam_pad>` 토큰)에 뚜렷하게 높은 가중치를 준다.

![답변 토큰이 이전에 생성된 객체 토큰들 중 의미적으로 관련된 토큰에 높은 attention 가중치를 부여하는 것을 보여주는 시각화](/assets/images/posts/object-tokens-as-a-bridge-between-segmentation-and-visual-question-answering-in-/figure-2.png)

이건 단순히 "세그멘테이션도 되고 VQA도 된다"는 병렬 성공이 아니라, **답변이 실제로 공간적으로 grounded된 근거를 참조해서 나온다**는 걸 보여준다는 점에서 이 논문의 주장을 가장 직접적으로 뒷받침하는 증거다.

## 한계와 남는 질문

노트를 정리하면서 아쉬웠던 건 이 구조가 검증된 도메인이 수술 로봇이라는 단일 embodiment, 단일 카메라 뷰에 국한돼 있다는 점이다. 객체 클래스 수만큼 학습 가능한 쿼리를 미리 정의해야 하는 구조도, 클래스 집합이 고정된 수술 환경에서는 문제가 없지만 개방형 환경으로 확장하려면 다시 손봐야 할 부분으로 보인다. 그리고 커리큘럼의 3단계 구성이 실제로 얼마나 민감한지, 예를 들어 2단계를 생략하거나 순서를 바꾸면 어느 정도 성능이 떨어지는지에 대한 ablation이 노트에 없어서 궁금한 지점으로 남았다.

## 🤖 AI의 생각

이 부분은 사실 요약이 아니라 제 의견입니다. 이 논문에서 정말 흥미로운 부분은 `<sam_pad>`라는 토큰 하나가 SAM 디코더와, 자기 자신보다 뒤에 오는 답변 토큰이라는 두 개의 서로 다른 소비자를 동시에 가진다는 점입니다. 같은 표현이 파이프라인 방향과 자기회귀 방향 양쪽에서 재사용된다는 게, 단순히 두 모델을 이어 붙이는 것보다 훨씬 깊은 결합이라고 느껴집니다.

<details class="tc-faq">
<summary>이 토큰이 정말 공간 정보를 담고 있는지, 아니면 그저 클래스 이름을 다시 인코딩한 것에 가까운지 어떻게 구분할 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

같은 이름의 객체가 두 개 이상 있을 때(왼쪽 겸자, 오른쪽 겸자) 토큰이 실제로 서로 다른 인스턴스를 구분해내는지를 보면, 이 매개 토큰이 진짜 픽셀 근거를 담고 있는지 아니면 언어적 지름길인지가 드러날 것 같다.

</div>
</details>

<details class="tc-faq">
<summary>이질적인 두 표현 공간을 잇는 학습된 중간 토큰이라는 설계 패턴을 taskcraft의 latent world model에도 적용할 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

수술 로봇이라는 단일 embodiment 안에서 세그멘테이션과 언어를 잇는 문제와, 사람 시연에서 뽑아낸 task representation을 신체 구조가 다른 에이전트로 옮기는 문제는 풀어야 할 것 자체가 다르다. 다만 taskcraft에서 latent world model이 embodiment마다 다른 행동 공간을 잇는 공유 인터페이스로 작동하길 바란다면, 중간 표현이 단방향 전달용 벡터가 아니라 양방향으로 참조 가능한 토큰이 되도록 설계하는 것도 하나의 선택지가 될 수 있다. 다만 이건 어디까지나 패턴 차원의 유비이고, 실제로 가져오려면 embodiment 간 전이 가능성부터 별도로 검증해야 한다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/feff3523b359e48cee166905f466c153464b3f87" target="_blank" rel="noopener">Object Tokens as a Bridge Between Segmentation and Visual Question Answering in Robotic Surgery</a> · semantic-scholar-rec</span></div>
</div>
</div>
