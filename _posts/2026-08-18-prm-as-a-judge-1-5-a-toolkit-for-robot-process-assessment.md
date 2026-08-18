---
title: "성공 여부 대신 과정을 채점하는 로봇 평가법"
date: 2026-08-18
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "성공/실패 0과 1로만 로봇을 채점하던 관행에 진행도 곡선과 회복 탄력성 지표를 더한 PRM-as-a-Judge 1.5를 살펴본다."
paper_title: "PRM-as-a-Judge 1.5: A Toolkit for Robot Process Assessment"
paper_summary: "성공/실패 0과 1로만 로봇을 채점하던 관행에 진행도 곡선과 회복 탄력성 지표를 더한 PRM-as-a-Judge 1.5를 살펴본다."
paper_url: "https://huggingface.co/papers/2608.14284"
header:
  image: "/assets/images/posts/prm-as-a-judge-1-5-a-toolkit-for-robot-process-assessment/figure-1.png"
  teaser: "/assets/images/posts/prm-as-a-judge-1-5-a-toolkit-for-robot-process-assessment/figure-1.png"
---

이번 주 논문 노트는 hf-daily 소스에서 스코어 0.75로 올라온 "PRM-as-a-Judge 1.5: A Toolkit for Robot Process Assessment"입니다. 로봇 조작 평가라는 주제 자체가 taskcraft 프로젝트에서 계속 부딪히는 문제, 즉 "에이전트가 실패했다"는 판정 뒤에 숨은 과정 정보를 어떻게 회수할 것인가와 정확히 맞닿아 있어서 이번 주 픽으로 골랐습니다.

로봇 팔에게 컵을 걸라고 시켰다고 하자. 결과는 두 가지로만 기록된다. 성공(1) 아니면 실패(0). 그런데 이 이진 판정 안에는 사실 완전히 다른 두 세계가 뭉개져 있다. 목표의 90%까지 매끄럽게 도달했다가 마지막 순간 손끝이 미끄러진 경우와, 시작하자마자 방향을 잘못 잡아 아예 근처도 못 간 경우가 똑같이 0점을 받는다. 반대로 성공 쪽도 마찬가지다. 한 번에 깔끔하게 목표에 도달한 경우와, 멈칫거리고 흔들리고 우연히 재시도가 맞아떨어져서 겨우 성공한 경우가 똑같이 1점을 받는다. 벤치마크 리더보드에 찍히는 숫자는 같지만 정책의 실제 품질은 전혀 다르다.

왜 지금까지 이 문제가 방치됐는지 생각해보면, 최종 성공률이 측정하기 쉽고 비교하기 쉬웠기 때문이다. 규칙 기반 채점기 하나만 있으면 대량의 롤아웃을 자동으로 채점할 수 있으니까. 문제는 그 결과 실패의 원인 진단이나 정책 간 미묘한 품질 차이를 논의할 언어 자체가 없었다는 점이다. "이 정책이 왜 저 정책보다 나은가"를 물으면 사람이 직접 비디오를 돌려보는 수밖에 없었다.

이 논문이 고친 방법은 롤아웃 비디오를 프로세스 보상 모델(PRM)에 통과시켜서 시간에 따른 진행도 곡선 $$p_{0:T}$$($$p_t \in [0,1]$$)로 바꾸는 것이다. 스칼라 하나 대신 시계열 하나를 얻게 되니, 여기서부터 다양한 지표를 뽑아낼 수 있다. 이 논문은 그 지표 체계를 Outcome, Process, Diagnosis 세 축(OPD)으로 나눈다. Outcome은 최대 도달 진행도(MP)나 마일스톤 도달율(MC@q)처럼 "얼마나 멀리 갔는가"를 본다. Process는 경로 가중 진행 길이(PPL)로 "얼마나 효율적으로 갔는가"를 잰다. 방황하거나 번복하지 않고 최단 경로로 목표에 다가갔다면 PPL이 높다.

가장 흥미로운 부분은 1.5에서 새로 추가된 진단 지표 세 개다. 실패한 롤아웃에만 적용되는 FNS(Failure Near-Success)는 실패했더라도 후반부 마일스톤에 얼마나 가까이 갔는지를 $$0.5\text{MP} + 0.3\text{MC@75} + 0.2\text{MC@50}$$ 형태로 종합한다. 성공한 롤아웃에만 적용되는 SQS(Success Quality Score)는 반대로 그 성공이 얼마나 매끄러웠는지를 PPL, 낮은 누적 후회(CRA), 낮은 정체율(STR)을 가중합해 평가한다. 그리고 DRR(Drawdown Recovery Ratio)은 진행도가 가장 크게 떨어진 지점 $$t^*$$ 이후 그 손실을 얼마나 회복했는지를 비율로 잰다. 완전 회복이면 1.0, 전혀 회복 못 하면 0에 가깝다. 이 세 지표를 나눠서 계산하는 이유가 핵심이다. 성공과 실패를 하나의 잣대로 재려 하지 않고, 각각에 맞는 질문을 따로 던진다.

물론 이 모든 지표의 신뢰성은 결국 PRM 자체가 진행도를 얼마나 정확히 읽어내느냐에 달려 있다. 진행도 곡선이 부정확하면 FNS든 DRR이든 다 허수다. 그래서 저자들은 RoboPulse++라는 구간 단위 벤치마크를 따로 만들었다. 700개 에피소드에서 2,244개 구간을 사람이 직접 상승(+1)/하강(-1)으로 라벨링해서, PRM이 실제로 진행도의 방향을 올바르게 판별하는지 검증한다. 채점 도구 자체를 검증하는 이 단계를 건너뛰면 앞의 모든 지표가 사상누각이 되니, 이 부분이 논문의 신뢰도를 지탱하는 축이라고 볼 수 있다.

전체 파이프라인은 아래 그림처럼 정리된다.

![PRM-as-a-Judge 1.5 전체 파이프라인: 비디오/텍스트 입력에서 진행도 곡선 추출, OPD 1.5 지표 계산, 진단 보고서 생성까지](/assets/images/posts/prm-as-a-judge-1-5-a-toolkit-for-robot-process-assessment/figure-1.png)

결과 쪽에서 개인적으로 눈에 띈 발견은 시뮬레이션과 실세계 성능 격차다. 9개 모델, 6개 작업에 대해 실세계 최대 진행도에서 시뮬레이션 최대 진행도를 뺀 값을 히트맵으로 그렸는데, 대부분 짙은 음수(최대 -58.4%)로 나타난다. 흥미로운 건 이 격차가 작업마다 균일하지 않다는 점이다. 공간 허용 오차가 큰 단순 작업(펜꽂이 채우기)은 격차가 작거나 오히려 양수인 반면, 정밀한 삽입이나 접촉 역학이 필요한 작업(튜브 삽입, 물체 분류, 머그컵 걸기)에서는 실세계 성능이 크게 무너진다. 이건 시뮬레이션 성공률만 보고 정책을 고르면 위험하다는 실증적 경고이자, 왜 프로세스 수준의 진단이 필요한지에 대한 근거이기도 하다.

![9개 모델, 6개 작업에 대한 시뮬레이션 대비 실세계 최대 진행도 격차 히트맵](/assets/images/posts/prm-as-a-judge-1-5-a-toolkit-for-robot-process-assessment/figure-2.png)

한계도 분명하다. 지표들의 가중치(FNS의 0.5/0.3/0.2, SQS의 0.5/0.3/0.2)가 다소 임의적으로 보이는데, 이 값들이 어떤 원칙으로 정해졌는지, 작업 종류에 따라 재조정이 필요한지는 논문만으로는 확신하기 어렵다. 또한 이 모든 진단이 결국 PRM이라는 하나의 채점 모델에 의존하는 구조라서, PRM 자체의 편향이 지표 전체에 그대로 전파될 위험도 남아있다. RoboPulse++로 상승/하강 판별력을 검증했다고 해도, 그것이 곧 절대적인 진행도 값의 정확도를 보장하는 건 아니다.

## 용어 해설

- **Process Reward Model(PRM)**: 최종 결과가 아니라 작업이 진행되는 중간 과정 하나하나에 점수를 매기도록 학습된 모델. 강화학습에서 outcome reward만으로는 학습 신호가 희소한 문제를 보완하기 위해 쓰인다.
- **Covariate Shift**: 학습에 쓰인 데이터의 입력 분포와 실제 배포 환경에서 마주치는 입력 분포가 달라지는 현상. 로봇 정책이 학습 궤적을 벗어난 상태에 놓이면 처음 보는 입력을 만나 오차가 누적되기 쉽다.
- **Behavior Cloning(BC)**: 전문가의 시연 데이터를 그대로 모방하도록 지도학습으로 정책을 학습시키는 방법. 시연 분포를 벗어난 상황에 취약하다는 한계가 잘 알려져 있다.
- **Drawdown**: 원래 투자 지표에서 온 개념으로, 어떤 값이 최고점을 찍은 이후 저점까지 얼마나 떨어졌는지를 나타내는 낙폭. 이 논문에서는 진행도 곡선의 최고점 대비 하락폭을 가리키는 데 그대로 차용됐다.

## 🤖 AI의 생각


사실 요약과는 별개로 개인 의견을 덧붙이면, 이진 성공률이라는 편한 지표를 버리고 곡선 전체를 진단 대상으로 삼은 발상 자체가 이 논문의 가장 값진 기여라고 생각한다. 다만 지표 설계가 여전히 사람이 정한 가중치에 크게 의존한다는 점은 앞으로 더 다듬어질 여지로 보인다.

<details class="tc-faq">
<summary>FNS와 SQS의 가중치(0.5/0.3/0.2)는 작업 난이도나 도메인에 따라 달라져야 하지 않을까</summary>
<div class="tc-faq__body" markdown="1">

논문은 고정 가중치를 쓰는데, 정밀 조작처럼 정체가 곧 신중함을 의미하는 작업에서는 STR에 대한 페널티 방향 자체가 바뀔 수도 있어서 도메인별 재검증이 필요해 보인다.

</div>
</details>

<details class="tc-faq">
<summary>taskcraft에서 에이전트의 실패를 이진 판정 대신 이런 진행도 곡선 방식으로 채점할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

로봇 궤적처럼 연속적인 물리 신호는 없지만 태스크 서브골 달성 순서를 진행도로 치환하면 FNS나 DRR 같은 회복 지표를 텍스트 에이전트 평가에도 옮겨볼 만하다.

</div>
</details>

<details class="tc-faq">
<summary>RoboPulse++로 검증한 상승/하강 판별력이 진행도 절대값의 정확도까지 보장하는가</summary>
<div class="tc-faq__body" markdown="1">

방향성 판별과 절댓값 정확도는 별개 문제라서, 이 논문의 모든 지표가 방향은 맞아도 크기는 왜곡됐을 가능성은 별도로 검증돼야 한다.

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2608.14284" target="_blank" rel="noopener">PRM-as-a-Judge 1.5: A Toolkit for Robot Process Assessment</a> · hf-daily</span></div>
</div>
</div>
