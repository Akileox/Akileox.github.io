---
title: "이미지와 텍스트를 잇는 CLIP의 발상"
date: 2026-09-03
categories: [AI]
tags: [core-paper, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "4억 개 웹 이미지-텍스트 쌍으로 배운 CLIP이 어떻게 재학습 없이 새 분류 문제를 푸는지 살펴본다"
paper_title: "Learning Transferable Visual Models From Natural Language Supervision"
paper_summary: "4억 개 웹 이미지-텍스트 쌍으로 배운 CLIP이 어떻게 재학습 없이 새 분류 문제를 푸는지 살펴본다"
paper_url: "https://arxiv.org/abs/2103.00020v1"
header:
  image: "/assets/images/posts/learning-transferable-visual-models-from-natural-language-supervision/figure-1.png"
  teaser: "/assets/images/posts/learning-transferable-visual-models-from-natural-language-supervision/figure-1.png"
---

이번 주 소개할 논문은 OpenAI의 "Learning Transferable Visual Models From Natural Language Supervision", 흔히 CLIP이라 부르는 그 논문입니다. 스코어링 점수가 1.00으로 최고점을 받았는데, 이유는 명확합니다. VLA(Vision-Language-Action) 계열 모델들이 이미지 인코더로 CLIP을 가져다 쓰는 경우가 워낙 많고, "언어로 비전을 감독한다"는 이 논문의 아이디어 자체가 이후 등장하는 거의 모든 멀티모달 연구의 출발점이기 때문입니다. Imitation Learning이나 RL은 수업에서 배웠어도 VLA 이후 흐름은 낯설 독자분들께, CLIP은 그 흐름을 이해하는 데 반드시 거쳐야 할 관문이라 이번 주 첫 논문으로 골랐습니다.

## 지도학습 비전 모델이 부딪힌 벽

2020년 전후까지 컴퓨터 비전에서 이미지 분류 모델을 만드는 표준적인 방법은 이랬다. ImageNet처럼 사람이 미리 정해둔 1000개짜리 클래스 목록을 놓고, 모델이 각 이미지에 대해 그 1000개 중 하나를 맞히도록 지도 학습시킨다. 이 방식은 벤치마크 성능을 끌어올리는 데는 훌륭하게 작동했지만, 구조적인 한계가 하나 있다. 모델의 출력이 학습 시점에 고정해둔 클래스 집합에 완전히 못박혀 있다는 것이다. 새로운 개념을 하나 추가하고 싶으면 그 개념에 대한 라벨 데이터를 다시 모으고, 분류기의 출력 헤드를 다시 설계하고, 재학습을 돌려야 한다.

반면 같은 시기 자연어처리 쪽에서는 이미 다른 길이 열려 있었다. GPT 계열처럼 인터넷의 방대한 텍스트를 그냥 다음 단어 예측이라는 단일 목적으로 사전 훈련시킨 모델이, 별도의 미세조정 없이도 번역이나 요약 같은 온갖 다운스트림 태스크로 제로샷 전이되는 걸 보여줬다. 비전에서도 이런 게 가능하지 않을까, 라는 질문이 CLIP의 출발점이다.

## 왜 캡션을 그대로 베끼는 방식은 안 됐나

사실 "이미지와 텍스트를 같이 학습시킨다"는 아이디어 자체는 CLIP 이전에도 있었다. VirTex 같은 선행 연구는 이미지를 보고 그 이미지의 캡션을 한 단어씩 생성하도록 학습시키는 방식을 썼다. 언뜻 보면 합리적이다. 캡션을 정확히 생성할 수 있으려면 이미지 내용을 제대로 이해해야 할 테니까.

문제는 계산 비용이었다. 캡션에 등장하는 모든 단어를 정확한 순서로 예측해야 하는 목적식은 지나치게 어려운 문제다. 이미지 하나에 대해 캡션의 정확한 워딩까지 맞히려다 보니 학습이 느리고, 웹 스케일의 데이터(수억 장)로 확장하기엔 계산량이 감당이 안 됐다. CLIP 저자들은 이 지점에서 발상을 바꾼다. "정확한 텍스트를 생성할 필요는 없다. 이 이미지와 이 텍스트가 서로 짝이 맞는지만 판별하면 된다." 이렇게 목적을 낮추자 학습 효율이 4배 이상 뛰었다고 논문은 보고한다.

## 대조 학습으로 이미지와 텍스트를 한 공간에 묶기

이 아이디어를 구현하는 방법이 대조 학습(contrastive learning)이다. 배치 안에 $$N$$개의 (이미지, 텍스트) 쌍이 있다고 하자. 이미지 인코더가 각 이미지를 벡터로, 텍스트 인코더가 각 텍스트를 벡터로 변환한다. 이제 이 $$N$$개 이미지 벡터와 $$N$$개 텍스트 벡터 사이의 모든 조합, 즉 $$N \times N$$개의 유사도를 계산할 수 있다. 이 중 실제로 짝이 맞는 대각선 위 $$N$$개 쌍(이미지 $$i$$와 그 이미지의 진짜 캡션 $$i$$)의 유사도는 최대한 키우고, 나머지 $$N^2 - N$$개의 짝이 안 맞는 쌍은 유사도를 최대한 낮추도록 학습시킨다.

수식으로 보면 이렇다. 이미지 임베딩 $$\mathbf{I}_i$$와 텍스트 임베딩 $$\mathbf{T}_j$$의 코사인 유사도를 온도 파라미터 $$\tau$$로 스케일링한 로짓은

$$S_{i, j} = \frac{\mathbf{I}_i \cdot \mathbf{T}_j}{\lVert \mathbf{I}_i \rVert_2 \lVert \mathbf{T}_j \rVert_2} \cdot \exp(\tau)$$

이렇게 정의되고, 이미지 관점에서의 손실과 텍스트 관점에서의 손실을 각각 크로스 엔트로피로 구한 뒤 평균을 낸다.

$$L_I = -\frac{1}{N} \sum_{i=1}^{N} \log \frac{\exp(S_{i, i})}{\sum_{j=1}^{N} \exp(S_{i, j})}, \quad L_T = -\frac{1}{N} \sum_{j=1}^{N} \log \frac{\exp(S_{j, j})}{\sum_{i=1}^{N} \exp(S_{i, j})}$$

$$L = \frac{1}{2} (L_I + L_T)$$

여기서 $$L_I$$는 "이미지 $$i$$에 맞는 텍스트를 배치 안 $$N$$개 후보 중에서 골라내라"는 문제이고, $$L_T$$는 그 반대 방향이다. 이 둘을 같이 최적화하면서 이미지 인코더와 텍스트 인코더가 서로 다른 modality임에도 하나의 공유된 임베딩 공간 위에서 정렬된다. 이미지 인코더로는 ResNet 계열과 Vision Transformer(ViT) 계열을 둘 다 실험했고, 텍스트 인코더로는 12계층짜리 Transformer를 썼다. 이 학습을 4억 개의 (이미지, 텍스트) 쌍으로 이루어진 WIT 데이터셋 위에서 돌렸다.

## 프롬프트 하나로 새 태스크에 던져 넣기

학습이 끝난 CLIP을 실제로 어떻게 쓰는지가 이 논문에서 가장 인상적인 부분이다.

![CLIP의 대조 사전 훈련과 제로샷 예측 파이프라인](/assets/images/posts/learning-transferable-visual-models-from-natural-language-supervision/figure-1.png)

그림에서 보듯 절차는 세 단계다. 먼저 대조 학습으로 이미지 인코더와 텍스트 인코더를 정렬시킨다(1단계). 그다음 새로운 분류 문제가 주어지면, 그 문제의 클래스 이름들(plane, car, dog 등)을 텍스트 인코더에 통과시켜 임베딩 벡터들을 만든다(2단계). 이 임베딩들이 곧 분류기의 가중치 역할을 한다. 마지막으로 분류하고 싶은 이미지를 이미지 인코더에 넣어 임베딩을 얻고, 이 이미지 임베딩과 가장 코사인 유사도가 높은 클래스 임베딩을 찾으면 그게 예측 결과다(3단계).

여기서 중요한 건 재학습이 전혀 없다는 점이다. 새 분류 문제가 생기면 그냥 클래스 이름을 텍스트로 바꿔서 인코더에 넣어주기만 하면 된다. 다만 저자들은 단순히 "dog" 같은 단어 하나만 넣으면 성능이 떨어진다는 걸 발견했다. 단어 하나는 문맥이 없어 다의성 문제(가령 "crane"이 학인지 기중기인지 애매한 경우)가 생기기 때문이다. 그래서 "A photo of a {label}." 같은 문장 형태의 프롬프트를 쓰고, 이런 프롬프트를 80여 개 템플릿으로 만들어 앙상블하는 프롬프트 엔지니어링을 적용했다. 이것만으로 ImageNet 제로샷 정확도가 최대 5%p 가까이 올랐다.

## 분포가 바뀌어도 흔들리지 않는 이유

성능 자체도 인상적이지만, 이 논문에서 개인적으로 더 눈에 띈 결과는 분포 변화(distribution shift)에 대한 견고성이다.

![ImageNet 정확도가 같아도 분포 변화 데이터셋에서 CLIP과 ResNet의 성능 격차가 크게 벌어지는 그래프](/assets/images/posts/learning-transferable-visual-models-from-natural-language-supervision/figure-2.png)

표준 ImageNet 검증셋에서 똑같이 76.2%를 찍는 지도학습 ResNet-101과 제로샷 CLIP(ViT-L/14@336px)을 스케치나 적대적 예제, 렌더링 이미지 등으로 구성된 분포 변화 데이터셋에 그대로 던져봤다. ResNet-101은 ImageNet-A에서 2.7%까지 떨어지는 반면, CLIP은 같은 데이터셋에서 77.1%를 유지했다. 두 모델이 원래 데이터셋에서는 성능이 똑같았다는 걸 감안하면 이 격차는 놀랍다. 지도학습 모델이 ImageNet이라는 특정 데이터셋의 촬영 방식이나 배경 같은 스퓨리어스 상관관계에 과적합돼 있었던 반면, CLIP은 애초에 다양한 출처의 웹 이미지로 학습됐기 때문에 특정 데이터셋의 편향에 덜 얽매인다는 게 저자들의 해석이다.

## 이 논문이 taskcraft와 닿는 지점

CLIP은 로봇이나 embodiment를 다루는 논문이 아니지만, 발상 자체는 taskcraft가 기대는 가설과 구조적으로 비슷하다. CLIP은 이미지와 텍스트라는 서로 다른 modality를 각자의 인코더로 처리한 뒤, 공유된 임베딩 공간에서 코사인 유사도로 정렬시킨다. 이 정렬된 공간이 있으면 클래스 이름을 텍스트로 바꿔주는 것만으로 재학습 없이 새 태스크에 전이된다.

taskcraft가 그리는 latent world model도 비슷한 그림이다. 사람 시연 영상, 로봇 관절 데이터, 게임 에이전트처럼 서로 다른 embodiment를 각각 다른 인코더로 연결하되, "행동이 세계 상태를 어떻게 바꾸는가"라는 공유된 latent 표현을 매개로 삼으면, 그 표현 자체는 특정 embodiment에 덜 종속적일 수 있다는 가설이다. 다만 CLIP의 정렬은 대조 학습으로 "같은 쌍인지"만 판별하는 정적인 정렬인 반면, taskcraft가 필요로 하는 정렬은 시간에 따른 상태 전이까지 포함해야 한다는 점에서 훨씬 어려운 문제다.

## 용어 해설

- **대조 학습(Contrastive Learning)**: 정답 쌍끼리는 표현 공간에서 가깝게, 오답 쌍끼리는 멀게 만들도록 학습시키는 방법론. 정확한 값을 직접 생성하는 대신 "이게 맞는 쌍인가"만 판별하면 되므로 목적식이 단순해진다.
- **제로샷 전이(Zero-Shot Transfer)**: 특정 다운스트림 태스크에 대한 추가 학습 데이터나 미세조정 없이, 사전 훈련된 모델을 그 태스크에 곧바로 적용하는 것.
- **임베딩(Embedding)**: 이미지, 텍스트, 단어 같은 원본 데이터를 고정된 길이의 실수 벡터로 변환한 것. 벡터 사이의 거리나 유사도로 원본 데이터 간의 의미적 관계를 나타낼 수 있다.
- **온도 파라미터(Temperature)**: 소프트맥스 함수에 곱해져 확률 분포를 얼마나 뾰족하거나 완만하게 만들지 조절하는 스칼라 값. 값이 클수록 가장 큰 로짓에 확률이 더 집중된다.

## 🤖 AI의 생각

사실 요약과는 별개로 개인적인 소감을 붙이자면, CLIP이 증명한 건 결국 "충분히 크고 다양한 페어 데이터에서 대조 학습을 하면, 태스크별로 헤드를 새로 설계하지 않아도 하나의 공유 표현이 여러 다운스트림 태스크를 커버한다"는 사실이고, 이게 taskcraft가 바라는 embodiment-agnostic latent와 정신적으로 사촌 관계라고 본다.

<details class="tc-faq">
<summary>CLIP의 프롬프트 앙상블처럼 텍스트 문맥을 바꿔주는 것만으로 성능이 오른 현상은, 결국 텍스트 인코더가 얼마나 다양한 문장 패턴을 사전 훈련 중에 봤는지에 달린 것 아닐까</summary>
<div class="tc-faq__body" markdown="1">

그럴 가능성이 크고, 그렇다면 웹 텍스트의 다양성 자체가 제로샷 성능의 상한을 정하는 요인이라는 뜻이라 프롬프트 엔지니어링이 만능은 아니라고 본다.

</div>
</details>

<details class="tc-faq">
<summary>이미지-텍스트 쌍은 인터넷에 4억 개나 있었는데, 서로 다른 embodiment가 같은 태스크를 수행하는 쌍 데이터는 애초에 존재하지 않는데 taskcraft는 이걸 어떻게 메울 것인가</summary>
<div class="tc-faq__body" markdown="1">

직접 쌍 데이터 없이도 world model을 매개로 dynamics를 통해 간접적으로 정렬을 유도하는 방식이 필요해 보이고, 이 부분이 CLIP보다 근본적으로 어려운 지점이라고 생각한다.

</div>
</details>

<details class="tc-faq">
<summary>CLIP의 분포 변화 견고성은 데이터 다양성 덕분이라는데, 로봇 데이터는 애초에 웹 텍스트만큼 다양한 소스를 모으기 어려운데 이 견고성이 taskcraft에도 재현될 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

회의적인 편이고, 로봇 쪽은 데이터 수집 비용 자체가 병목이라 다양성보다는 world model의 구조적 일반화 능력에 더 기대야 할 것 같다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://arxiv.org/abs/2103.00020v1" target="_blank" rel="noopener">Learning Transferable Visual Models From Natural Language Supervision</a> · arxiv</span></div>
</div>
</div>
