#### Original: [Railway Oriented Programming in Kotlin](https://www.greenbird.com/news/railway-oriented-programming-in-kotlin)


# Railway oriented programmig in kotlin
## What is that?
Railway Oriented Programming(ROP)는 함수가 순차 실행하도록하지만 꼭 동기화 될 필요는 없는 함수형 프로그래밍 기술입니다. 핵심 개념은 각 함수는 `Succes` 혹은 `Failure` 의 `Container`만 받을 수 있도록 하는 것입니다. `Failure`는 [Throwable](https://docs.oracle.com/javase/7/docs/api/java/lang/Throwable.html) 을 warpping하고 `Success` 는 `Any` 타입이 됩니다.

그러므로, 각 함수들은 `Success`로 부터 `Success` 혹은 `Failure` 시나리오고 갈 수 있고, `Failure` 시나리오는 `Failure` 시나리오로만 갈 수 있습니다.

당신은 `Failure`를 `Success`로 변환하고 싶을 때가 있을 것 입니다. 이 과정은 `rescue` 혹은 `recovery`라고 합니다; 이 표현은 언어마다 다릅니다.

당신이 각 상태를 다른 상태로 변환할 수 있다는 것은 중요한 요소입니다.

### 어떻게 실전에서 사용할 수 있을까요??
이 개념은 Kotlin 1.3이후의 버전부터 사용이 가능합니다. [Kotlin Keep](https://github.com/Kotlin/KEEP/blob/master/proposals/stdlib/result.md#encapsulate-successful-or-failed-function-execution) 에서는 이 기능의 변경이 발생할 수 있음을 명시하고 있습니다. 이 포스트를 작성하는 시점에 `Result<T>`는 `Container`-like 타입입니다. 이는 `Collection<T>`와 유사합니다.
```kotlin
fun capitalize(cities: List<String>): List<Result<String>>
```

동시에, `Result<T>`는 함수의 파라미터가 될 수 있습니다.

```kotlin
fun map(result: Result<String>): List<String>
```
- [왜 kotlin.Result는 반환형이 될 수 없을까요?](https://stackoverflow.com/questions/52631827/why-cant-kotlin-result-be-used-as-a-return-type)
- [Kotlin에서 Result를 어떻게 반환형으로 사용할 수 있을까요?](https://stackoverflow.com/questions/61223609/how-to-allow-resultt-to-be-return-type-in-kotlin)

`Result<T>` data-type의 [문서](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin/-result/)

### How to apply to existing code?
이 패러타임을 따르고 있는 함수를 예로 이 함수들의 어댑터를 작성해보겠습니다.
- Single Stack Function. 에러가 예외가 존재하지 않는 함수들 입니다.
  - [ComplexNumber](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/singletrackfunctions/ComplexNumber.kt#L8) 와 [ComplexNumberResult](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/singletrackfunctions/ComplexNumberResult.kt#L10) 가 그 예입니다.
- Dead-end Function. Error를 throw하는 함수입니다.
  - [OutofMemoryError](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html) 나 [StackOverflowError](https://docs.oracle.com/javase/8/docs/technotes/guides/troubleshoot/memleaks002.html) 입니다.
  - [HtmlParser](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/deadendfunctions/HtmlParser.kt#L3) 와 [HtlmParserResult](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/deadendfunctions/HtmlParserResult.kt) 가 그 예입니다.
- Exception을 throw하는 함수입니다. 이것은 IO에 유용합니다. [DownloadPage](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/throwexceptionsfunctions/DownloadPage.kt#L9) 가 그 예입니다.
- 다른 코드를 관리하는 함수들입니다. 로깅이나 다른 코드를 관리하는, 실제로 result를 반환하지 않는 함수들입니다.
  -[SentryLogin](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/supervisoryfunctions/SentryLogin.kt#L14) 과 [SentryLoginResult](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/service/supervisoryfunctions/SentryLoginResult.kt#L5) 가 그 예입니다.

각 클래스들은 관련된 테스트를 가지고 있습니다, 따라서 테스트에서 차이점을 알 수 있을 것 입니다. 어댑터는 낮은 레벨의 예일 수 있기 때문에, [application](https://github.com/ChameleonTartu/railway-oriented-programming-presentation/blob/master/src/main/kotlin/no/example/Application.kt#L7) 의 예시도 준비하였습니다. 이것은 URLs의 HTML page를 다운로드 하고 URLs가 유효하지 않을때, 이를 복구하는 케이스들입니다.
