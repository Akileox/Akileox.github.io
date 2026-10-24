---
title: "객체부터 픽셀까지 하나로 보는 VLM"
date: 2026-10-24
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "탐지, OCR, 세그멘테이션, 매팅을 하나의 비전-언어 모델로 묶은 WeVisOne-VL을 읽고 정리했습니다"
paper_title: "WeVisOne-VL: Unified Visual Perception from Objects to Pixels"
paper_summary: "탐지, OCR, 세그멘테이션, 매팅을 하나의 비전-언어 모델로 묶은 WeVisOne-VL을 읽고 정리했습니다"
paper_url: "https://huggingface.co/papers/2610.31329"
header:
  image: "/assets/images/posts/wevisone-vl-unified-visual-perception-from-objects-to-pixels/figure-1.png"
  teaser: "/assets/images/posts/wevisone-vl-unified-visual-perception-from-objects-to-pixels/figure-1.png"
---

이번 주 HF Daily에서 스코어 1.00으로 가장 눈에 띈 논문은 WeVisOne-VL이었습니다. 비전-언어 모델이 "이미지를 설명하는 것"은 이제 꽤 잘하지만 "이 객체가 정확히 어디 있고 경계가 어디까지인가"를 묻는 순간 흔들린다는 문제는 꽤 오래된 체증인데, 이 논문은 그 체증을 별도 모델을 붙이는 대신 토큰화 전략 하나로 풀려고 시도한다는 점에서 이번 주 노트로 골랐습니다.

문제의식부터 보면 단순하다. 바운딩 박스를 찾는 탐지, 개수를 세는 카운팅, 글자를 읽는 OCR 같은 객체 수준 태스크와, 픽셀 단위로 영역을 분리하는 세그멘테이션, 반투명 경계까지 다뤄야 하는 매팅 같은 픽셀 수준 태스크는 요구하는 출력 형식이 전혀 다르다. 전자는 숫자(좌표)로, 후자는 마스크(이미지)로 답해야 한다. 그래서 지금까지는 각 태스크마다 별도의 거대 모델을 붙이는 게 당연한 선택이었다. 탐지는 VLM이 하고, 정밀 분할은 SAM 같은 전용 세그멘테이션 모델에 넘기는 식이다.

왜 이게 안 좋은가 하면, 모델을 여러 개 운영해야 하는 비용 문제도 있지만 더 근본적인 건 인스턴스 정합성이다. VLM이 가리킨 "이 사람"과 세그멘테이션 모델이 분리한 마스크가 정말 같은 객체를 가리키는지 보장하기가 어렵다. 게다가 혼잡한 장면에서 비돌출 객체를 놓치거나, 아예 존재하지 않는 객체를 질의했을 때도 빈 박스를 내놓지 못하고 환각을 일으키는 문제가 겹친다. 여기에 고품질 세그멘테이션/매팅 어노테이션 자체가 구축 비용이 매우 비싸다는 데이터 병목까지 더해진다.

WeVisOne-VL이 고친 방법은 크게 세 갈래다. 첫째는 좌표를 $$<\text{0}>$$부터 $$<\text{1000}>$$까지의 정규화된 dedicated 토큰으로 바꿔서, 탐지 결과를 그냥 텍스트 생성처럼 Next-Token Prediction으로 뽑아낸다. 존재하지 않는 객체는 `None`으로 명시적으로 기각하게 만들어서 환각을 구조적으로 억제한다. 둘째는 픽셀 수준 태스크를 위해 바운딩 박스 뒤에 특수 토큰 $$<\text{OBJ}>$$를 하나 더 생성하게 만들고, 이 토큰의 잠재 표현을 9.6M 파라미터짜리 경량 픽셀 디코더에 넘겨서 마스크와 알파 매트를 복원한다. 거대한 세그멘테이션 파운데이션 모델을 통째로 붙이는 대신, 공유 ViT 피처를 정렬하는 어댑터와 박스 좌표로 쿼리를 유도하는 Deformable DETR 스타일 디코더, 그리고 coarse to fine 방식의 refinement만으로 충분하다는 게 이 논문의 베팅이다.

![WeVisOne-VL Data Engine 개요](/assets/images/posts/wevisone-vl-unified-visual-perception-from-objects-to-pixels/figure-1.png)

세 번째가 데이터 문제인데, 이게 제일 노동집약적인 부분이다. SAM3, 자체 OV 탐지기, Rex-Omni 같은 여러 전문가 모델의 후보를 Qwen3-VL로 교차 검증해서 1억 장 이상의 이미지와 36억 개 박스 어노테이션을 만들어내는 데이터 엔진(WDE)을 따로 구축했다. 매팅처럼 더 정밀한 작업은 BiRefNet, MatAnyone2 같은 전문가 모델과 SAM2 기반 어노테이터를 엮은 별도 파이프라인으로 뽑았다. 결국 모델 구조의 참신함 못지않게, 인스턴스 레벨에서 박스와 마스크와 알파 매트가 서로 어긋나지 않게 맞추는 이 데이터 정합 작업이 실제로는 가장 큰 비중을 차지했을 것 같다.

학습은 3단계 커리큘럼으로 간다. 1단계는 언어 모델만 건드려서 구조화된 출력 형식(좌표 토큰, $$<\text{OBJ}>$$ 토큰)에 익숙해지게 하고, 2단계에서 비전 인코더를 풀어 해상도와 한 이미지당 객체 수(최대 500개)를 늘리고, 3단계에서 VLM과 픽셀 디코더를 1대1 비율로 교차 학습시킨다. 여기에 더해 GRPO 기반 강화학습 단계를 추가하는데, 모든 샘플을 똑같이 학습시키지 않고 모델이 맞추기도 틀리기도 하는 중간 난이도 샘플을 골라내는 불확실성 기반 선별($$U(x) = 4\mu_x(1-\mu_x)\sigma_x$$)을 쓴다. 보상 함수도 헝가리안 매칭으로 일대일 대응시킨 뒤 IoU를 세제곱($$\text{IoU}^3$$)해서, 이미 꽤 정확한 박스를 더 정밀하게 다듬는 쪽에 지수적으로 큰 보상을 준다. 쉬운 샘플에 시간을 낭비하지 않고 애매한 경계에서 힘을 쓰게 만드는 설계다.

![경량 픽셀 디코더 구조](/assets/images/posts/wevisone-vl-unified-visual-perception-from-objects-to-pixels/figure-2.png)

결과적으로 이 논문이 보여주는 건 "전용 세그멘테이션 모델 없이도 9.6M짜리 디코더 하나로 박스부터 알파 매트까지 간다"는 주장이다. 토큰 예측 손실과 세그멘테이션 손실을 $$L_{\text{joint}} = L_{\text{NTP}} + 10 L_{\text{seg}}$$로 가중합해서, VLM의 언어 능력을 깎아먹지 않으면서 픽셀 디코더 성능을 같이 끌어올리는 식이다. 다만 이 통합이 정말 구조적 이점 때문인지, 아니면 36억 박스 규모의 데이터 엔진이 워낙 잘 만들어져서인지는 수치만으로는 분리해서 판단하기 어렵다는 생각이 든다.

## 용어 해설

- **헝가리안 매칭 (Hungarian Matching)**: 두 집합(예측 박스 집합과 정답 박스 집합) 사이에서 비용(혹은 보상)을 최소화(혹은 최대화)하는 일대일 대응을 다항 시간에 찾아내는 조합 최적화 알고리즘이다. 탐지 모델 학습에서 중복 예측 없이 각 정답에 예측 하나씩만 짝지을 때 흔히 쓰인다.
- **GRPO (Group Relative Policy Optimization)**: 같은 입력에 대해 여러 응답을 샘플링한 뒤 그룹 내 상대적 보상을 비교해서 정책을 업데이트하는 강화학습 알고리즘이다. 별도의 가치 함수(critic) 없이도 상대적 우위만으로 학습 신호를 만들 수 있다는 게 특징이다.
- **Focal Loss / Dice Loss**: Focal Loss는 쉬운 샘플의 손실 기여도를 줄이고 어려운 샘플에 집중하도록 설계된 분류 손실 함수로, 전경과 배경 픽셀 수가 크게 불균형한 세그멘테이션에서 자주 쓰인다. Dice Loss는 예측 영역과 정답 영역의 겹침 정도를 직접 최적화하는 손실로, 마스크 모양 자체의 일치도를 높이는 데 쓰인다.

## 🤖 AI의 생각


개인적으로는 이 논문이 "통합"이라는 단어를 상당히 설득력 있게 밀어붙인 사례라고 느껴진다. 거대 파운데이션 모델을 블랙박스로 이어붙이는 최근 흐름과 반대로, 작은 디코더와 토큰화 전략만으로 탐지부터 매팅까지 묶었다는 점이 신선하다. 다만 이게 구조 자체의 일반화 능력인지 36억 박스 규모의 데이터 엔진이 만든 결과인지는 분리해서 보기 어렵다는 의문은 남는다.

<details class="tc-faq">
<summary>\(<\text{OBJ}>\) 토큰의 잠재 표현만으로 픽셀 디코더에 충분한 정보가 전달되는지, 아니면 박스 좌표 토큰이 실질적으로 더 많은 일을 하고 있는 건 아닌지 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

논문의 ablation 없이는 판단하기 어렵지만, Box-guided Query Decoder라는 이름 자체가 박스 좌표에 상당히 의존하고 있음을 시사한다.

</div>
</details>

<details class="tc-faq">
<summary>이 정도로 큰 데이터 엔진(1억 이미지, 36억 박스) 없이도 비슷한 성능이 재현 가능한 수준인지 궁금하다.</summary>
<div class="tc-faq__body" markdown="1">

모델 구조는 9.6M으로 가볍지만 그 가벼움을 데이터 규모가 보완하는 구조라서, 소규모 재현 실험에서는 성능 저하가 꽤 클 것 같다는 추측이 든다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 영상에서 task-relevant entity를 뽑아내는 전처리가 필요해진다면 이런 통합 지각 모델을 가져다 쓸 수 있을까.</summary>
<div class="tc-faq__body" markdown="1">

world model이 다루는 상태를 구성하는 재료(객체, 경계, 마스크)를 만드는 프런트엔드로는 쓸 수 있겠지만, latent 전이 자체를 대체하는 역할은 아니라서 레이어를 분리해서 보는 게 맞을 것 같다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2610.31329" target="_blank" rel="noopener">WeVisOne-VL: Unified Visual Perception from Objects to Pixels</a> · hf-daily</span></div>
</div>
</div>
