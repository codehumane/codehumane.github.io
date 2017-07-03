# NewRelic Apdex 설정하기

[NewRelic](https://newrelic.com/)의 주요 수치 중 하나로 [Apdex](http://www.apdex.org/) 라는 것이 있다. 이 글에서는 이 값이 무엇이고, 왜 필요하며, 어떻게 설정하는지를 간단히 살펴본다.

## Apdex란?

Apdex(Application Performance Index)는 웹 어플리케이션의 응답 시간에 대한 사용자의 만족도를 나타낸다. 이를 좀 더 잘 이해하기 위해, 반응성<sup>responsiveness</sup>의 3가지 수준을 먼저 살펴보자.

![Apdex Responsiveness Zones](../images/NewRelic-Apdex/responsiveness-zones.png)

그림 1. Apdex 반응성 수준 (출처: http://apdex.org/overview.html)

각 수준에 대한 설명은 아래와 같다. (보다 자세한 내용은 [Apedx Overview](http://apdex.org/overview.html)를 참고)

| 응답 수준      | 구간            | 설명                              |
| ---------- | ------------- | ------------------------------- |
| Satisfied  | 0 ≥, T ≤      | 사용자가 응답 시간이 지연된다고 느끼지 않음.       |
| Tolerating | T >, F = 4T ≤ | 사용자가 응답 시간이 지연된다고 느끼지만 참을 만함.   |
| Frustrated | F >           | 사용자가 응답 시간을 용인하지 못하고 프로세스를 포기함. |

만약, T를 100ms라고 정의했다면, 응답시간이 100ms 이내인 경우는 `Satisfied`, 100~400ms 구간은 `Tolerating`, 400ms 이상이면 `Frustrated` 수준이 된다.

이제 이 기준값을 가지고 아래와 같이 Apdex Score를 산출할 수 있다.

![apdex-fomula](../images/NewRelic-Apdex/apdex-fomula.png)

그림 2. Apdex 공식 (출처: http://apdex.org/overview.html)

간단히 예를 들어보자.

- 서버가 1분 동안 총 80개의 요청을 처리함.
- T 값을 100ms로 지정함.
- 70개의 요청은 100ms 이내의 응답 시간을 가짐.
- 2개의 요청은 200ms의 응답 시간을 가짐.
- Apdex 점수 = (70 + (2 / 2)) / 80 = 0.89

참고로, NewRelic에서는 에러 응답도 `Frustrated` 수준에 포함시키고 있다. [Apdex: Measuring user satisfaction](https://docs.newrelic.com/docs/apm/new-relic-apm/apdex/apdex-measuring-user-satisfaction#error)에서 이를 확인할 수 있다.

## 왜 설정하는가?

Apdex가 무엇인지는 알았다. 그런데 NewRelic에서 이 수치를 설정하는 것이 어떤 의미가 있는가?

![apdex-fomula](../images/NewRelic-Apdex/newrelic-apdex.png)

그림 3. NewRelic의 Apdex 점수 모니터링 화면

Apdex 점수는 단순히 응답속도가 아니라, 여러가지 측정 수치를 0~1이라는 하나의 숫자로 제공한다.

단순히 응답시간이 좀 더 의미 있는 수치를 

- 여러가지 측정 수치를 0~1이라는 하나의 숫자로 확인할 수 있다.
- ​

단순한 응답속도가 아니라 사용자의 응답시간 만족도라고 할 수 있는, 좀 더 의미 있는 수치를 관리할 수 있게 된다. 예를 들어, 평균 응답 시간으로는 감지할 수 없는  `performance outliers` 등이 확인 가능하다.

