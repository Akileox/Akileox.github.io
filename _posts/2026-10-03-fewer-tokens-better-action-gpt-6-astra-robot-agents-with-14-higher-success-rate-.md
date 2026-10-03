---
title: "토큰 줄이니 로봇 성공률이 올랐다"
date: 2026-10-03
categories: [AI]
tags: [weekly-trend, ai, computer-vision, robotics, paper-review]
comments: true
toc: true
ai_generated: true
ai_model: "claude-sonnet-5"
ai_extract_model: "gemini-flash-latest"
excerpt: "로봇 VLM 에이전트가 도구 호출 대신 코드를 짜게 했더니 토큰은 65% 줄고 성공률은 더 올랐다는 연구를 읽었습니다."
paper_title: "Fewer Tokens, Better Action: GPT-6 Astra Robot Agents with 14% Higher Success Rate but 65% Fewer Tokens"
paper_summary: "로봇 VLM 에이전트가 도구 호출 대신 코드를 짜게 했더니 토큰은 65% 줄고 성공률은 더 올랐다는 연구를 읽었습니다."
paper_url: "https://huggingface.co/papers/2610.01939"
header:
  image: "/assets/images/posts/fewer-tokens-better-action-gpt-6-astra-robot-agents-with-14-higher-success-rate-/figure-1.png"
  teaser: "/assets/images/posts/fewer-tokens-better-action-gpt-6-astra-robot-agents-with-14-higher-success-rate-/figure-1.png"
---

이번 주 HuggingFace Daily Papers에서 스코어 1.00으로 가장 눈에 띈 논문입니다. VLM 에이전트가 로봇을 제어할 때 흔히 겪는 "매 동작마다 LLM을 다시 불러야 하는" 비효율 문제를 정면으로 다루면서, 성공률과 비용을 동시에 개선했다는 결과가 보기 드물어 이번 주 노트로 골랐습니다.

## 문제 제기: 로봇 에이전트는 왜 이렇게 수다스러운가

VLM이 로봇 팔을 제어하는 방식은 보통 도구 호출(tool-calling) 구조를 따른다. 에이전트가 "그릇 쪽으로 이동해"라는 도구를 호출하면 로봇이 움직이고, 그 결과(카메라 이미지, 성공 여부)가 다시 LLM에 돌아오고, LLM은 다음 행동을 결정한다. 문제는 이 루프 하나하나가 전부 별도의 LLM 호출이라는 점이다. 로봇 조작 작업은 localization, 접근, 파지, 검사, 복구(recovery)처럼 짧은 동작이 반복되는 경우가 많은데, 각 단계마다 새 대화 턴이 생기고 그 안에 카메라 이미지가 누적되면서 컨텍스트가 빠르게 불어난다. 개별 프리미티브 자체는 충분히 믿을 만한데도, 전체 작업을 끝내기까지 드는 추론 비용은 작업이 길어질수록 기하급수적으로 커진다.

## 왜 기존 방식으로는 안 됐나

핵심 원인은 두 가지로 요약된다. 첫째, 도구 호출 인터페이스에서는 "이전 동작 결과를 보고 다음 동작을 조건부로 결정한다"는 아주 단순한 로직조차 LLM 호출 없이는 구현할 수 없다. 둘째, 기존 시스템(RPent)은 프리미티브가 실행될 때마다 자동으로 여러 장의 카메라 이미지를 반환해서 컨텍스트에 쌓는다. 즉 에이전트가 이미지를 보고 싶든 아니든 무조건 받게 되는 구조다. 이 두 제약이 합쳐지면, 반복적인 재시도나 다단계 작업일수록 LLM 호출 횟수와 토큰 사용량이 함께 폭증할 수밖에 없다.

## 고친 방법: 코드로 짜고, 보고 싶을 때만 본다

저자들이 제안한 PyRUA-Lean은 로봇 스택과 프리미티브 구현 자체는 RPent에서 그대로 가져오되, 인터페이스만 도구 호출에서 인터랙티브 파이썬 코드 실행으로 바꾼다. 로봇은 `robo`라는 파이썬 객체로 노출되고, 에이전트는 하나의 코드 셀 안에서 여러 프리미티브를 순서대로 호출하면서 그 결과를 조건문으로 검사해 재시도, 분기, 중단을 스스로 처리한다. 이 전체 과정이 LLM 호출 한 번 없이 셀 내부 실행만으로 끝난다. 예외가 나면 트레이스백만 LLM에 전달되고, 변수들은 영속적인 네임스페이스에 남아 다음 셀에서도 재사용된다. 개별 프리미티브가 제공하지 않는 기하 계산이나 헬퍼 함수도 에이전트가 직접 짜서 쓸 수 있다.

두 번째 축은 선택적 관찰이다. 프리미티브 실행 결과는 런타임에만 존재하고, 에이전트가 `print`나 `robo.show()`로 명시적으로 요청한 정보만 셀이 끝나는 시점에 LLM에게 전달된다.

![PyRUA-Lean과 tool-calling 인터페이스의 구조 비교 및 성공률·토큰 사용량 결과](/assets/images/posts/fewer-tokens-better-action-gpt-6-astra-robot-agents-with-14-higher-success-rate-/figure-2.png)

Figure 2의 배치(placement) 예시가 이 차이를 잘 보여준다. 기존 tool-calling 방식은 목표 위치 계산, 하강, 검사, 해제를 각각 별도의 LLM 턴으로 처리해서 4번의 호출이 필요했다. PyRUA-Lean에서는 같은 작업을 기하 계산과 6개의 프리미티브 호출이 들어간 코드 셀 하나로 압축하고, 작업이 성공하면 이미지조차 요청하지 않는다. 결국 4턴이 1턴으로 줄어드는 셈이다.

## 결과: 더 적게 말했는데 더 잘했다

비교는 동일한 GPT-6 Astra 플래너, 동일한 동결된 VLA 정책(π0.5, LingBot-VLA, RLDX-1), 동일한 프리미티브, 동일한 LLM 호출 예산 조건에서 LIBERO-PRO, RoboTwin 2.0, RoboCasa365의 700개 작업 인스턴스를 대상으로 진행됐다. 전체 성공률은 63.1%에서 71.7%로 올랐고, 입력 토큰은 65% 줄었다.

![벤치마크별 성공률 변화와 토큰·비용 절감을 보여주는 레이더 차트 및 막대그래프](/assets/images/posts/fewer-tokens-better-action-gpt-6-astra-robot-agents-with-14-higher-success-rate-/figure-1.png)

Figure 1을 보면 개선 폭이 균일하지 않다는 점이 흥미롭다. "Long long-horizon" 작업에서 +36.0pp, "Stack & order multi-step" 작업에서 +40.0pp로 복잡하고 긴 호라이즌 작업일수록 개선이 크게 나타났다. 반면 토큰 절감은 벤치마크마다 차이는 있지만 전반적으로 일관됐고, 특히 LIBERO-PRO에서는 4.5배(1.23M에서 273k로) 줄어 가장 두드러졌다. 저자들은 추가 분석을 통해 토큰 절감의 주된 원인이 이미지 반환 자체를 줄인 것이 아니라 LLM 호출 횟수 자체가 줄어든 데 있다는 점을 확인했다. 실제로 베이스라인 tool-calling 방식에 "이미지는 요청 시에만 반환"하는 수정을 단순히 얹어본 비교 실험에서는 PyRUA-Lean과 같은 효과가 재현되지 않았다. 즉 핵심은 관찰을 아끼는 것 자체보다, 여러 동작을 하나의 호출 안에서 조건부로 엮어낼 수 있는 구조 자체에 있다는 뜻이다.

다만 논문이 다루는 범위는 명확히 한정적이다. 비교는 전부 동결된 VLA 정책과 고정된 프리미티브 집합 위에서 이루어졌고, 플래너가 새로운 저수준 동작을 발명하거나 프리미티브 자체의 품질을 개선하는 문제는 다루지 않는다. 또한 수식이나 이론적 분석보다는 시스템 설계와 실증 비교에 초점을 맞춘 논문이라, 코드 기반 조합이 특히 긴 호라이즌 작업에서 더 유리한지에 대한 설명은 사례 중심에 머문다.

## 용어 해설

- **도구 호출(tool-calling)**: LLM이 외부 함수나 API를 호출할 수 있도록 미리 정의된 인터페이스를 통해 입출력을 주고받는 방식. 한 번의 호출마다 결과가 다시 LLM 컨텍스트로 돌아와야 다음 판단이 가능하다.
- **VLA(Vision-Language-Action) 정책**: 카메라 이미지와 언어 지시를 입력받아 로봇의 저수준 동작(관절 각도, 그리퍼 제어 등)을 직접 출력하도록 학습된 모델.
- **호라이즌(horizon)**: 작업을 완수하기까지 필요한 의사결정 또는 동작 단계의 길이. 호라이즌이 길다는 것은 목표 달성까지 거쳐야 할 중간 단계가 많다는 뜻이다.
- **네임스페이스(namespace)**: 프로그램에서 변수나 함수 이름이 유지되고 참조되는 영역. 영속적 네임스페이스라는 것은 한 번 정의된 변수가 다음 실행에서도 그대로 남아 재사용 가능하다는 뜻이다.

## 🤖 AI의 생각


사실관계 정리와는 별개로, 개인적으로는 이 논문이 "에이전트를 더 똑똑하게 만드는 것"이 아니라 "에이전트가 대화하는 방식 자체를 바꾸는 것"만으로 성능과 비용을 동시에 개선했다는 점이 인상적입니다.

<details class="tc-faq">
<summary>코드 실행 기반 조합이 긴 호라이즌 작업에서 유독 크게 개선된 이유를, 단순히 호출 횟수 감소를 넘어 더 구조적으로 설명할 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

논문이 사례 중심 설명에 그친 부분이라, 조건부 재시도 로직이 복잡한 작업일수록 LLM이 아니라 파이썬 런타임이 처리하는 결정의 비율이 높아진다는 가설을 별도로 검증해볼 만하다

</div>
</details>

<details class="tc-faq">
<summary>동결된 VLA 정책과 고정 프리미티브 집합이라는 전제를 걷어내고, 플래너가 직접 새로운 프리미티브를 코드로 합성하게 하면 어떻게 될까</summary>
<div class="tc-faq__body" markdown="1">

지금은 안전한 저수준 함수 호출로 제한돼 있지만 taskcraft처럼 더 상위의 표현을 학습시킨다면 플래너 스스로 중간 수준의 재사용 가능한 스킬을 만들어낼 여지가 있다

</div>
</details>

<details class="tc-faq">
<summary>taskcraft가 latent world model로 얻으려는 embodiment 무관성을, 이 논문처럼 인터페이스 설계만으로 상당 부분 흉내 낼 수 있을까</summary>
<div class="tc-faq__body" markdown="1">

이 논문의 상위 플래너는 코드 호출 수준에서 이미 embodiment에 무관하게 작동하므로, 표현 학습 없이도 인터페이스 추상화만으로 어디까지 갈 수 있는지 가늠해볼 좋은 대조군이 된다

</div>
</details>

<div class="post-references">
<p class="section-label">참고문헌</p>
<div class="tc-refs">
<div class="tc-ref"><span class="tc-ref__group">원문</span><span class="tc-ref__item"><a href="https://huggingface.co/papers/2610.01939" target="_blank" rel="noopener">Fewer Tokens, Better Action: GPT-6 Astra Robot Agents with 14% Higher Success Rate but 65% Fewer Tokens</a> · hf-daily</span></div>
</div>
</div>
