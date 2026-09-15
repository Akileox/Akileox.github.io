---
title: "고정된 프롬프트를 버린 이상 탐지 모델"
date: 2026-09-15
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "제로샷 이상 탐지에서 변분 프롬프트와 웨이블릿 주파수 분해로 미세 이상 단서를 잡아내는 VFAD를 정리했다"
paper_title: "VFAD: Variational Semantic Prompting Meets Frequency-Adaptive Representation Learning for Zero-Shot Anomaly Detection"
paper_summary: "제로샷 이상 탐지에서 변분 프롬프트와 웨이블릿 주파수 분해로 미세 이상 단서를 잡아내는 VFAD를 정리했다"
paper_url: "https://www.semanticscholar.org/paper/3c226d81586a5a0c5495cdaf3188029f563580c8"
header:
  image: "/assets/images/posts/vfad-variational-semantic-prompting-meets-frequency-adaptive-representation-lear/figure-1.png"
  teaser: "/assets/images/posts/vfad-variational-semantic-prompting-meets-frequency-adaptive-representation-lear/figure-1.png"
---

이번 주는 semantic-scholar의 추천 알고리즘이 0.85라는 꽤 높은 점수를 매긴 논문을 골랐습니다. taskcraft가 다루는 로봇 학습 도메인과는 거리가 있지만, "고정된 표현으로 다양한 이상을 설명할 수 있는가"라는 문제의식 자체가 범용적이어서 읽어볼 가치가 있다고 판단했습니다.

## 문제 제기: 이상은 정의할 수 없는데 프롬프트는 하나뿐이다

Zero-shot anomaly detection(ZSAD)은 타깃 카테고리에 대한 학습 데이터 없이 처음 보는 카테고리에서 이상을 찾고 위치까지 짚어내야 하는 과제다. CLIP 같은 비전-언어 모델을 갖다 쓰는 방법들(WinCLIP, AnomalyCLIP, Bayes-PFL, MoE-CLIP 등)이 이 문제에 꾸준히 도전해왔는데, 이 논문은 그 흐름 전체가 공유하는 두 가지 허점을 짚는다.

첫째는 프롬프트 자체의 표현력 문제다. "정상"과 "이상"이라는 텍스트를 수작업 템플릿이든 학습 가능한 벡터든 결국 고정된 임베딩 하나로 표현하는데, 실제 산업/의료 데이터에서 이상은 카테고리마다 형태도 다르고 애매한 경계도 많다. 하나의 결정론적 벡터로 이 다양성과 불확실성을 다 담으려는 시도 자체가 무리라는 것이다.

둘째는 시각 표현의 해상도 문제다. CLIP은 원래 이미지 전체와 텍스트를 맞추는 전역 정렬에 최적화된 모델이라, 객체 전체의 지배적인 의미는 잘 잡아내지만 표면의 미세한 균열이나 국소적 텍스처 변화 같은 subtle anomaly cue는 흐려지기 쉽다.

## 왜 안 됐는지: 프롬프트를 보강해도, 패치를 다뤄도 부족했다

이 한계를 메우려는 시도가 없었던 건 아니다. 전역 CLS 특징으로 프롬프트를 보강하는 방식(VCP-CLIP류)이 대표적인데, 문제는 CLS 토큰에 배경이나 객체 자체의 불필요한 정보까지 같이 섞여 들어온다는 점이다. 프롬프트를 풍부하게 만들려다 오히려 노이즈를 더 얹는 셈이다.

패치 단위로 파고든 방법들도 한계가 있다. 이미지를 균일한 패치로 잘라 똑같은 방식으로 변환하는데, 실제로는 영역마다 이상이 드러나는 방식이 다르고(어떤 곳은 구조가 깨지고 어떤 곳은 텍스처만 미묘하게 바뀐다), 구조 정보와 텍스처 정보가 서로 보완적으로 작동해야 하는데 그 상호작용을 반영할 틀이 없었다. Figure 1의 (a), (b)가 이 두 흐름을 나란히 보여준다.

![기존 ZSAD 방법과 VFAD의 접근 방식 비교](/assets/images/posts/vfad-variational-semantic-prompting-meets-frequency-adaptive-representation-lear/figure-1.png)

## 고친 방법: 프롬프트는 분포로, 특징은 주파수로 쪼갠다

VFAD는 이 두 문제를 각각 겨냥한 모듈 두 개를 CLIP 구조 위에 얹는다.

첫 번째는 Variational Semantic Prompt Extractor(VSPE)다. 학습 가능한 쿼리 앵커가 이미지 패치 토큰에 크로스 어텐션을 걸어 이상과 관련된 지역 정보만 골라 모은다. 여기까지는 기존 방법과 비슷한데, VFAD는 이렇게 모은 표현을 결정론적 벡터로 바로 쓰지 않고 잠재 가우시안 분포의 평균과 분산으로 사영한 뒤 재매개변수화 트릭으로 샘플링한다. 즉 "이상의 시맨틱은 이런 값이다"가 아니라 "이런 분포에서 나올 법하다"로 바꿔 표현하는 것이다. KL 발산으로 이 분포를 표준정규분포 쪽으로 눌러주면 특정 카테고리 패턴에 과적합되는 것도 어느 정도 억제된다. 이렇게 샘플링된 프롬프트는 텍스트 인코더의 여러 층에 계속 주입되어, 텍스트 표현이 시각적 지역 정보를 점진적으로 반영하게 만든다.

두 번째는 Frequency-Adaptive Representation Aggregation(FARA)이다. 패치 특징을 다시 공간 특징 맵으로 펼친 뒤 이산 웨이블릿 변환으로 저주파(전역 구조)와 고주파 세 갈래(지역 텍스처와 디테일)로 쪼갠다. 저주파와 고주파 각각에 독립적인 Mixture-of-Experts 브랜치를 두고, 라우터가 상황에 맞는 전문가만 top-k로 골라 활성화한다. 강화된 결과는 역웨이블릿 변환으로 다시 공간 도메인에 합쳐지고, 잔차 연결로 원래 표현과 더해져 과적합을 눌러준다. 구조 정보와 텍스처 정보를 억지로 한 파이프라인에 밀어넣지 않고 애초에 나눠서 특화 처리한다는 발상이다.

![VFAD 전체 아키텍처: VSPE의 변분 샘플링 구조와 FARA의 주파수 분해 및 MoE 라우팅 구조](/assets/images/posts/vfad-variational-semantic-prompting-meets-frequency-adaptive-representation-lear/figure-2.png)

전체 파이프라인은 6, 12, 18, 24번째 층의 패치 특징을 FARA로 강화해 픽셀 수준 이상 맵을 만들고, 전역 CLS 토큰과 풀링된 지역 표현을 합쳐 이미지 수준 점수를 뽑는다. 학습 손실은 분할용 Dice와 Focal, 분류용 BCE, 그리고 변분 정규화용 KL을 합친 형태다.

## 결과와 남는 의문

논문은 산업(MVTec 등)과 의료 도메인을 합쳐 13개 벤치마크에서 기존 CLIP 기반 ZSAD 방법들을 능가했다고 보고한다. 다만 노트에 정리된 정보만으로는 각 벤치마크별 구체적인 수치나, VSPE와 FARA 두 모듈을 각각 뺐을 때 성능이 얼마나 떨어지는지에 대한 ablation 수치까지는 확인하지 못했다. 변분적 접근과 주파수 분리라는 두 축이 실제로 어느 쪽이 더 크게 기여하는지, 그리고 두 모듈이 서로 상호작용하는지(가령 FARA가 강화한 표현이 VSPE의 크로스 어텐션 품질에도 영향을 주는지) 는 추가로 파고들 만한 지점이다.

## 용어 해설

- **KL 발산(Kullback-Leibler Divergence)**: 두 확률 분포가 얼마나 다른지를 재는 척도로, 값이 0이면 두 분포가 완전히 같다는 뜻이다. 딥러닝에서는 학습된 분포를 특정 목표 분포(주로 표준정규분포)에 가깝게 유도하는 정규화 항으로 자주 쓰인다.
- **크로스 어텐션(Cross-Attention)**: 어텐션 메커니즘의 한 형태로, 쿼리와 키/값이 서로 다른 두 집합(예: 학습 가능한 쿼리와 이미지 패치)에서 나올 때 한쪽이 다른 쪽의 어느 부분에 집중할지를 학습하는 구조다.
- **Mixture-of-Experts(MoE)**: 여러 개의 작은 신경망(전문가)을 두고, 입력마다 라우터가 그중 일부만 선택적으로 활성화해 처리하도록 하는 구조다. 전체 파라미터는 크지만 실제 연산량은 활성화된 전문가만큼만 늘어난다.

## 🤖 AI의 생각


이 논문은 산업/의료 이상 탐지라는, taskcraft가 다루는 로봇 학습 도메인과는 표면적으로 완전히 동떨어진 주제라서 직접 인용할 일은 없을 것 같다. 다만 개인적으로 흥미롭게 본 지점은 VSPE 모듈의 발상이다. 정상/이상 프롬프트를 결정론적 벡터 하나로 고정하지 않고 변분적 정보 병목을 통해 분포로 표현해서 카테고리마다 다른 이상의 다양성과 불확실성을 흡수하려 한다는 아이디어는, taskcraft가 고민하는 embodiment 무관 표현 문제와 구조적으로 닮은 데가 있다고 생각한다. taskcraft에서도 latent world model이 뽑아낸 task representation이 특정 embodiment의 관절 구조에 과적합되지 않게 하려면, 결정론적 임베딩보다는 분포적(변분적) 표현이 category(또는 embodiment) 특이적 패턴에 덜 묶이게 만드는 데 유용할 수 있지 않을까 하는 가설이 떠오른다. 물론 이건 방법론적 유비일 뿐이고, 도메인이 워낙 다르니 실제로 적용 가능할지는 전혀 검증된 바 없는 순전히 주관적인 연상이다. 그리고 FARA의 주파수 분리(저주파는 전역 구조, 고주파는 지역 텍스처) 발상도, 만약 로봇 조작 영상에서 "무엇을 하는가(구조/저주파적 정보)"와 "어떻게 세밀하게 하는가(고주파적 디테일)"를 분리해서 다룰 수 있다면, embodiment에 덜 종속적인 정보(저주파, 전역 전이 구조)와 embodiment 특이적 디테일(고주파, 구체적 관절 움직임)을 나누는 데 참고가 될 법한 은유로 읽히기도 한다. 다만 이는 어디까지나 억지로 끌어온 해석이라는 점을 밝혀둔다.

<details class="tc-faq">
<summary>VSPE와 FARA를 각각 뺐을 때 성능 저하 폭이 어느 정도인지, ablation 수치를 직접 확인해보면 어느 모듈이 실제로 더 결정적인지 알 수 있지 않을까</summary>
<div class="tc-faq__body" markdown="1">

논문 노트에는 최종 성능 우위만 언급되어 있어 이 부분은 원문의 ablation 표를 따로 찾아봐야 답이 나올 것 같다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft의 latent task representation도 embodiment마다 고정된 결정론적 벡터 대신 분포로 표현하면 특정 관절 구조에 덜 과적합되지 않을까</summary>
<div class="tc-faq__body" markdown="1">

방법론적 유비일 뿐 실제로 검증된 적은 없어서 순전히 가설 수준의 연상이다

</div>
</details>

<details class="tc-faq">
<summary>FARA처럼 저주파(구조)와 고주파(텍스처)를 나눠 특화 처리하는 방식을 로봇 조작 영상에 적용하면 무엇을 하는가와 어떻게 세밀하게 하는가를 분리해서 다룰 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

은유로는 그럴듯하지만 웨이블릿 분해가 영상 신호에 대해 갖는 의미와 로봇 동작의 저수준/고수준 정보 분리가 같은 종류의 분리인지는 따져봐야 한다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://www.semanticscholar.org/paper/3c226d81586a5a0c5495cdaf3188029f563580c8" target="_blank" rel="noopener">VFAD: Variational Semantic Prompting Meets Frequency-Adaptive Representation Learning for Zero-Shot Anomaly Detection</a> · semantic-scholar-rec</span></div>
</div>
</div>
