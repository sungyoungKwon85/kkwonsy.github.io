# kotlin coroutines keep dive 

코틀린 코루틴 책을 정리한다.   

<br><br>
---

## 차례 
1. 코틀린 코루틴 이해하기: 코틀린 코루틴을 배워야 하는 이유 
2. 코틀린 코루틴 이해하기: 시퀀스 빌더 
3. 코틀린 코루틴 이해하기: 중단은 어떻게 작동할까 
4. 코틀린 코루틴 이해하기: 코루틴의 실제 구현
5. 코틀린 코루틴 이해하기: 언어 차원에서의 지원 vs 라이브러리 
6. 코틀린 코루틴 라이브러리: 코루틴 빌더 
7. 코틀린 코루틴 라이브러리: 코루틴 컨텍스트 
8. 코틀린 코루틴 라이브러리: Job과 자식 코루틴 기다리기 
9. 코틀린 코루틴 라이브러리: 취소 
10. 코틀린 코루틴 라이브러리: 예외 처리 
11. 코틀린 코루틴 라이브러리: 코루틴 스코프 함수 
12. 코틀린 코루틴 라이브러리: 디스패처 
13. 코틀린 코루틴 라이브러리: 코루틴 스코프 만들기 
14. 코틀린 코루틴 라이브러리: 공유 상태로 인한 문제
15. 코틀린 코루틴 라이브러리: 코루틴 테스트하기 
16. 채널과 플로우: 채널
17. 채널과 플로우: 셀렉트
18. 채널과 플로우: 핫데이터 소스와 콜드 데이터 소스
19. 채널과 플로우: 플로우
20. 채널과 플로우: 플로우의 실제 구현
21. 채널과 플로우: 플로우 만들기
22. 채널과 플로우: 플로우 생명주기 함수
23. 채널과 플로우: 플로우 처리
24. 채널과 플로우: 공유플로우와 상태플로우
25. 채널과 플로우: 플로우 테스트
26. 적용하기: 일반적인 사용 예제
27. 적용하기: 활용 비법
28. 적용하기: 다른언어에서의 사용법
29. 적용하기: 코루틴을 시작하는 것과 중단 함수 중 어떤 것이 나을까?
30. 적용하기: 모범사례 


<br><br>
---
코틀린 코루틴 라이브러리에 대해 살펴보기 전에 몇 가지 기본적인 개념부터 살펴본다.  


## 1장. 코들린 코루틴을 배워야 하는 이유 
자바는 언어자체적으로 멀티스레드를 지원하고 있음. but 스레드는 비싸다.  
RxJava, Reactor와 같은 JVM 계열 라이브러리가 이미 널리 사용되고 있음. but 고쳐야 할 게 많다.(subScribeOn, objserveOn etc..)  
<br>
코틀린은 멀티 플랫폼에서 작동시킬 수가 있음. (JVM, JS, iOS etc)  
코틀린 코루틴을 도입한다고해서 **기존 코드 구조를 광범위하게 뜯어 고칠 필요가 없다.(suspend 키워드 정도)**  
**코틀린 코루틴의 핵심 기능은 코루틴을 특정 지점에서 멈추고 이후에 재개할 수 있다는 것.**  
**코루틴을 도입하면 동시성을 쉽게 구현할 수 있고, 쉽게 테스트할 수 있고, 쉽게 코루틴을 취소할 수 있음.**  
스레드로 데이터베이스나 다른 서비스로부터 응답을 기다릴 때마다 블로킹한다면 비용이 엄청남.  


<br><br>
---

## 2장. 시퀀스 빌더 
코틀린의 시퀀스는 List나 Set과 같은 컬렉션과 비슷한 개념.  
필요할 때 마다 값을 하나씩 계산하는 lazy 처리를 함.  
`sequence` 라는 함수를 이용해 정의한다.  
`yield` 함수를 호출해서 다음 값을 생성한다.  
`DSL(Domain Specific Language)`  
`suspend SequenceScope<T>.() -> Unit` // 인자는 수신 객체 지정 람다 함수임. 
```
val seq = sequence {
  print("first")
  yield(1)
  print("second")
  yield(2)
  print("third")
  yield(3)
  print("done")
}

fun main() {
  for (num in seq) {
    print("$num") // 숫자가 미리 생성되는게 아니라 필요할 때 생성된다. 이전에 yield를 호출했던 지점에서 다시 실행된다. 
  } 
  // first
  // 1                  
  // second
  // 2
  // third
  // 3
  // done
}
```
멈췄던 `yield` 로 가서 다시 실행되는게.. 코루틴을 이용해서 중단된 지점을 관리하는 것이다.  
<br><br><br>

### 실제 사용 예
```
// 난수를 만들 때 사용
fun randomNumbers(
  seed: Long = System.currentTimeMillis()
): Sequence<Int> = sequence {
  val random = Random(seed)
  while (true) {
    yield(random.nextInt())
  }
}


// 임의의 문자열을 만들 떄 사용 
fun randomUniqueStrings(
  length: Int, 
  seed: Long = System.currentTimeMillis()
): Sequence<String> = sequence {
  val random = Random(seed)
  val cahrPool = ('a'..'z') + ('A'..'Z') + ('0'..'9')
  while (true) {
    val randomString = (1..length)
      .map { i -> random.nextInt(charPool.size) }
      .map(charPool::get)
      .joinToString("");
    yield(randomString)
  }
}.distinct()
```

시퀀스 빌더는 `yield`(반환)이 아닌 중단 함수를 사용하면 안됨.  
중단이 필요하다면 나중에 배울 플로우를 사용하는게 나음.  
플로우 빌더는 시퀀스 빌더와 비슷하지만, 플로우는 여러가지 코루틴 기능을 지원함.  


<br><br>
---

## 3장. 중단은 어떻게 작동할까?
중단 함수는 코틀린 코루틴의 핵심임.  
코루틴은 중단되었을 때 `Continuation 객체`를 반환한다. 이를 이용하면 멈췄던 곳에서 다시 코루틴을 실행할 수 있다.  
다른 스레드에서 시작할 수 있고, 직렬화/역직렬화가 가능해 다시 실행될 수 있다.  
<br><br>

#### 재개
작업을 재개하려면 코루틴이 필요함.  
코루틴은 이후에 소개될 `runBlocking`, `launch` 같은 코루틴 빌더를 통해 만들 수 있음.  
```
suspend fun main() {
  println("Before")

  // 중단함수임. 
  suspendCoroutine<Unit> {} 
  // After는 출력되지 않고 중단된다. 

  println("After")
}
```
재개는 어떻게 하나? 앞서 말한 `Continuation`은 어디에? -> `suspendCoroutine`을 보면 {}(람다표현식) 으로 끝난게 보임.  
인자로 들어간 람다 함수는 중단되기 전에 실행되고, `Continuation` 객체를 인자로 받는다.  
```
// 람다 함수는 continuation 객체를 저장한 뒤 코루틴을 다시 실행할 시점을 결정하기 위해 사용된다. 
suspendCoroutine<Unit> { continuation -> 
  continuation.resume(Unit)  // 코루틴을 중단한 후 바로 실행함 
}
// Before
// After
```
```
suspendCoroutine<Unit> { continuation -> 
  // 잠깐 정지 된 뒤 재개되는 다른 스레드를 실행할 수 있음. 
  // 허나 1초 뒤 사라지는 스레드.. 비용 측면에서 별로임 
  Thread {
    println("Suspended")
    Thread.sleep(1000)
    continuation.resume(Unit)  
    println("Resumed")
  }
}
// Before
// Suspended
// 1초 후 
// After
// Resumed
```
```
// 위 로직을 개선해서 정해진 시간이 지나면 호출하도록 스케쥴러를 만들어서 쓰자. 
// 전용 스레드를 만든 것임. 
private val executor = Executors.newSingleThreadScheduledExecutor {
  Thread(it, "scheduler").apply { isDaemon = true }
}

suspend fun delay(timeMillis: Long): Unit = suspendCoroutine { 
  cont -> executor.schecdule({
    cont.resume(Unit)
  }, timeMillis, TimeUnit.MILLISECONDS)
}

suspend fun main() {
  println("Before")

  delay(1000)

  println("After")
}
```
코틀린 코루틴 라이브러리에서 delay가 구현된 방식이랑 일치함.  
<br><br>

#### 값으로 재개하기 
```
val i: Int = suspendCoroutine<Int> { cont -> cont.resume(43) } // 지정된 타입과 같은 타입을 리턴. 
```

#### 예외로 재개하기 
```
suspendCoroutine<Unit> { cont -> cont.resumeWithException(MyException())}
```

**강조하고 싶은 것은 함수가 아닌 코루틴을 중단시킨다는 것임.**   



<br><br>
---

## 4장.  코루틴의 실제 구현 
- 중단 함수는 함수가 시작할 때와 중단 함수가 호출 되었을 때 상태를 가진다. (state machine)
- `continuation` 객체는 상태를 나타내는 숫자와 로컬 데이터를 가진다
- 함수의 `continuation` 객체가 이 함수를 부르는 다른 함수의 `continuation` 객체를 decorate 합니다. 그래서 모든 `continuation` 객체는 실행을 재개하거나 재개된 함수를 완료할 때 사용되는 콜 스택으로 사용된다.  

코루틴의 간략화된 ㅐㄴ부 구현에 대해 관심있다면 이 장을 볼 것.  

<br><br>
---

## 5장. 언어차원에서의 지원 vs 라이브러리 
코틀린 언어에서 자체적으로 지원하는 부분(컴파일러의 지원과 코틀린 기본 라이브러리의 요소)과 코틀린 코루틴 라이브러리(`kotlinx.coroutines`) 로 구성되어 있음.  
`kotlinx.coroutines`를 사용하려면 의존성을 추가해야 한다.  
언어차원에서 지원하는 것 보다 사용하기 훨씬 쉬우며 동시성을 명확하게 구현할 수 있게 해준다.  
`launce`, `async`, `Deferred` 등을 지원  



<br><br>
<br><br>
<br><br>
---
코루틴 언어 자체적으로 지원하는 기능이 어떤지 배웠으니 이제 `kotlinx.coroutines` 라이브러리를 살펴본다.  

## 6장. 코루틴 빌더 
중단 함수는 `continuation` 객체를 다른 중단 함수로 전달해야 하므로 일반함수가 중단함수를 호출할 수가 없다.  
중단 함수를 호출하는 시작 지점이 반드시 있다. `코루틴 빌더`가 그 역할을 한다.  
- `launch`
- `runBlocking`
- `async`

<br><br>

#### launch 빌더 
thread 함수를 호출해서 새로운 스레드를 시작하는 것과 비슷함  
마치 불꽃놀이를 할 때 불꽃을 각각 하늘로 퍼지게 하는 것처럼 별개로 실행됨  
`launch` 함수는 `CoroutineScope` 인터페이스의 확장 함수임.  
`CoroutineScope` 인터페이스는 부모 코루틴과 자식 코루틴 사이의 관계를 정립하기 위해 사용되는 **구조화된 동시성**의 핵심이다.  
```
fun main() {
  GlobalScope.launch {
    delay(1000L)
    println("World")
  }
  GlobalScope.launch {
    delay(1000L)
    println("World")
  }
  GlobalScope.launch {
    delay(1000L)
    println("World")
  }
  println("Hello")
  Thread.sleep(2000L)
}
// Hello
// 1초 후 
// World x 3
```
예제에서 `GlobalScope` 객체에 `launch` 를 호출하는 방식을 사용했으나 실제로는 지양해야 한다  
그리고 main 함수 끝에 `Thread.sleep` 이 있는데, 스레드를 잠들게 하지 않으면 메인 함수가 코루틴을 실행하자마자 끝나버리기 때문에 넣은 코드이다.  
나중에 배울 구조화된 동시성을 사용하면 Thread.sleep은 필요없어진다.  
<br><br>

#### runBlocking 빌더 
코루틴이 스레드를 블로킹 하지 않고 작업을 중단시키기만 하는 것이 일반적이나, 블로킹이 필요한 경우가 있음.  
메인 함수인 경우라면 프로그램을 너무 빨리 끝내지 않기 위해 스레드를 블로킹 해야 함. 이때 `runBlocking`을 쓴다.   
**시작한 스레드를 중단시킨다.**  
프로그램이 끝나는걸 방지하기 위해서 또는 유닛 테스트시 쓸만하다.(사실 `runTest` 를 주로 쓰고있음)    
(main함수에도 `suspend`를 붙이면 됨)  
```
fun main() = runBlocking {
  ...
}

@Test
fun `a test`() = runBlocking {
  ...
}
```
<br><br>

#### async 빌더 
`launch` 와 비슷하지만 값을 생성하도록 설계되어 있고 람다 표현식에 의해 반환된다(`Deferred<T>`)  
`launch` 와 비슷하게 호출되자마자 코루틴을 즉시 시작한다  
`await`에서 값이 반환되는 즉시 사용할 수 있다, 값이 생성되기전에 호출해도 나올 떄 까지 기다린다  
값이 필요 없을 때는 `launch`를 쓰는게 맞다  
```
fun main() = runBlocking {
  val res1 = GlobalScope.async {
    delay(1000L)
    "Text 1"
  }
  val res2 = GlobalScope.async {
    delay(1000L)
    "Text 2"
  }
  val res3 = GlobalScope.async {
    delay(1000L)
    "Text 3"
  }
  println(res1.await())
  println(res2.await())
  println(res3.await())
}
```
여러가지 다른 곳에서 데이터를 얻어와 합치는 경우처럼, 병렬로 실행할 때 주로 사용된다.  
<br><br>

#### 구조화된 동시성 
`GlobalScope` 에서 코루틴이 시작되었다면 프로그램은 해당 코루틴을 기다리지 않음  
코루틴은 어떤 스레드도 블록하지 않으므로 프로그램이 끝나는걸 막을 수 없음  


부모는 자식들을 위한 Scope을 제공하고 자식들을 해당 Scope 내에서 호축하는데 이를 통해 **구조화된 동시성**이라는 관계가 성립한다.  
- 자식은 부모로부터 컨텍스트를 상속받는다. (자식이 재정의할 수도 있다)
- 부모는 모든 자식이 작업을 마치길 기다린다 
- 부모 코루틴이 취소되면 자식도 취소된다 
- 자식 코루틴에서 에러가 발생하면 부모에도 전파되어 소멸한다 

다른 코루틴 빌더와 달리, `runBlocking`은 `CoroutineScope`의 학장 함수가 아님.  
`runBlocking`은 자식이 될 수 없고 루트 코루틴으로만 사용가능하다.  
<br><br>

#### 현업에서의 코루틴 사용 
중단함수는 코루틴 빌더로 시작되어야 한다  
중단 함수는 다른 함수로부터 호출되어야 한다  
`runBlocking`을 제외한 모든 코루틴 빌더는 `CoroutineScope`에서 시작되어야 한다  
큰 애플리케이션에서는 Scope을 직접 만들거나(*13장) 프레임워크에서 제공하는 Scope을 사용한다(eg: `Ktor`)  
첫번째 빌더가 Scope에서 시작되면 다른 빌더가 첫 번째 빌더의 Scope에서 시작될 수 있다  
```
@Controller 
class UserController(
  private val tokenService: TokenService,
  private val userService: UserService,
) {
  @GetMapping("/me")
  suspend fun findUser(
    @PathVariable userId: String, 
    @RequestHeader("Authorization") authorization: String
  ): UserJson {
    val userId = tokenService.readUserId(authorization)
    val user = userService.findUserBy(userId)
    return user.toJson()
  }
} 
```
중단함수에는 스코프를 어떻게 처리할까? 함수 내에 스코프가 없다.  
코루틴 빌더가 사용할 스코프를 만들어주는 `coroutineScope` 함수를 사용하면 된다.  

```
suspend fun getArticlesForUser(
  userToken: String?,
): Linst<ArticleJson> = coroutineScope { // !!
  val articles = async { articleRepository.getArticles() }
  val user = userService.getUser(userToken)
  articles.await()
    .filter { canSeeOnList(user, it) }
    .map { toArticleJson(it) }
}
```
`async`를 호출하려면 스코프가 필요한데 스코프를 함수에 남기고 싶지 않으면 중단 함수 밖에서 스코프를 만들면 된다. by `coroutineScope`  
`coroutineScope` 는 중단 함수 내에서 스코프가 필요할 떄 일반적으로 사용하는 함수이다.  
원리를 이해하려면 컨텍스트, 취소, 예외 처리 같은 것을 공부해야 한다  
메인함수와 runBlocking을 사용하기보단 얘를 쓰자.  






<br><br>
---

## 7장. 코루틴 컨텍스트 
`launch`의 정의를 보자  
```
public fun CoroutineScope.launch(
  context: CorouinteContext = EmptyCoroutineContext, 
  start: CoroutineStart = CoroutineStart.DEFAULT, 
  block: suspend CoroutneScope.() -> Unit
): Job {...}
```
첫번째 파라미터가 `CoroutineContext`이다  
마지막 인차는 `CoroutineScope` 타입이다  
```
public interface CoroutineScope {
  public val coroutineContext: CoroutineContext
}
```
`CoroutineScope`은 `CoroutineContext`를 감싸는 Wrapper처럼 보인다  
```
public interface Cotinuation<in T> {
  public val context: CoroutineContext
  public fun resumeWith(result: Result<T>)
}
```
`Continuation` 또한 `CoroutineContext`를 포함하고 있다  
중요한 요소인 걸 알 수 있다  
<br><br>

#### CoroutineContext 인터페이스 
원소나 원소들의 집합을 나태나는 인터페이스이다.  
`Job`, `CoroutineName`, `CoroutineDispatcher`와 같은 Element 객체들이 인덱싱된 집합. Map, Set과도 비슷하다.  
특이한 점은 각 Element 또한 `CoroutineContext` 라는 점이다. 그 자체만으로도 컬렉션이라고 할 수 있겠다.  
모든 원소는 식별할 수 있는 유일한 키를 가지고 있고, 키는 주소로 비교가 된다.  
```
fun main() {
  val name: CoroutineName = CoroutineName("A name")
  val element: CoroutineContext.Element = name
  val context: CoroutineContext = element 

  val job: Job = Job()
  val jobElement: CoroutineContext.Element = job 
  val jobContext: CoroutineContext = jobElement 
}
```
<br><br>

#### CoroutineContext 원소 찾기 
컬렉션과 비슷하므로 get을 이용하면 된다  
원소가 없으면 null이 반환된다  
```
fun main() {
  val ctx: CoroutineContext = coroutineName("A name")

  val coroutineName: CoroutineName? = ctx[CoroutineName]
  val job: Job? = ctx[Job]
}
```
<br><br>

#### 두 개의 컨텍스트를 합쳐 하나로 만들 수 있음 
```
  val ctx1: CoroutineContext = coroutineName("name1")
  println(ctx1[CoroutineName]?.name) // name1
  println(ctx1[Job]?.isActive) // null

  val ctx2: CoroutineContext = Job()
  println(ctx2[CoroutineName]?.name) // null
  println(ctx2[Job]?.isActive) // true (기본적으로 Job의 상태가 있음)

  val ctx3: CoroutineContext = ctx1 + ctx2
  println(ctx3[CoroutineName]?.name) // name1
  println(ctx3[Job]?.isActive) // true
```
<br><br>

#### 비어있는 코루틴 컨텍스트 
```
  val emtpy: CoroutineContext = EmptyCoroutineContext
  println(empty[CoroutineName]) // null
  println(empty[Job]) // null

  val ctxName = empty + coroutineName("name1") + empty
  println(ctxName[CoroutineName]) // CoroutineName(name1)
```
빈 컨텍스트를 더해도 아무런 변화가 없음.  
<br><br>

#### 제거
```
  val ctx1: CoroutineContext = coroutineName("name1")
  val ctx2: CoroutineContext = Job()

  val ctx3 = (ctx1 + CoroutineName("name2")).minusKey(CouroutineName)
  println(ctx3[CoroutineName]?.name) // null <-- 제거 되어 버림
  println(ctx3[Job]?.isActive) // true
```
<br><br>

#### 코루틴 컨텍스트와 빌더 
`CoroutineContext`는 코루틴의 데이터를 저장하고 전단하는 방법.  
부모-자식 관계에서 부모는 기본적으로 자식에게 컨텍스트를 전달한다.  
```
fun CoroutineScope.log(msg: String) {
  val name = coroutineContext[CoroutineName]?.name 
  println("[$name] $msg")
}

fun main() = runBlocking(CoroutineName("main")) { // <----- 부모
  log("Started") // [main] Started
  val v1 = async {
    delay(500)
    log("Running async") // [main] Running async
    42
  }
  launch {
    delay(1000)
    log("Running launch") // [main] Running launch
  }
  log("The answer is ${v1.await()}) // [main] The answer is 42
}
```
공식: `defaultContext + parentContext + childContext`  
디폴트로 설정되는 원소는 `ContinuationInterceptor` 가 설정되어 있지 않을때는 `Dispatchers.Default` 이다.  
```
suspend fun printName() {
  println(coroutineContext[CoroutineName]?.name)
}

suspend fun main() = withContext(CoroutineName("Outer")) {
  printName() // Outer
  launch(CoroutineName("Inner")) { // <-- 자식이 새로 적용
    printName() // Inner
  }
  delay(10)
  printName() // Outer
}
```


<br><br>
---

## 8장. 잡과 자식 코루틴 기다리기 
6장의 구조화된 동시성 절에서 특성을 말함  
1. 자식은 부모로부터 컨텍스트를 받음
2. 부모는 모든 자식을 기다림
3. 부모 코루틴이 취소되면 자식도 취소됨 
4. 자식이 에러나면 부모도 소멸됨 

2-4번은 `Job` 컨텍스트와 관련이 있음  





<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

<br><br>
---

## 1장. 

