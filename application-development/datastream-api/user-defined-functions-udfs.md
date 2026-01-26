# User-Defined Functions

## 사용자 정의 함수(UDF) 지정 방식

### 인터페이스 구현

- 가장 기본적인 방법은 Flink가 제공하는 함수 인터페이스(MapFunction 등)를 구현해 전달하는 것
- 예: `MapFunction<String, Integer>`를 구현해 문자열을 정수로 변환하고 `data.map(new MyMapFunction())`로 사용

```
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
}
data.map(new MyMapFunction());
```

### 익명 클래스(Anonymous Class)

- 별도 클래스를 만들지 않고 익명 클래스로 함수를 바로 전달할 수 있다
- 간단한 로직을 빠르게 넣을 때 유용하다

```
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});

```

### Java 8 람다

- Java API에서 람다를 지원한다
- 예: `filter(s -> ...)`, `reduce((a,b) -> ...)` 형태로 간결하게 작성 가능

## Rich Function

### Rich 함수란

- 모든 UDF 기반 변환은 일반 함수 대신 **Rich*Function**(예: `RichMapFunction`)을 받을 수 있다
- Rich 함수는 `open()` / `close()` 같은 라이프사이클 훅과 `RuntimeContext` 접근을 제공해 확장성이 좋다

```java
class MyMapFunction implements MapFunction<String, Integer> {
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
}

class MyMapFunction extends RichMapFunction<String, Integer> {
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
}

```

### 사용 형태

- 클래스로 정의하거나, 익명 클래스로도 Rich 함수를 전달할 수 있다
- 기존 `MapFunction`을 `RichMapFunction`으로 바꿔도 `data.map(new MyMapFunction())`처럼 동일하게 사용한다

````
// 클래스 정의
data.map(new MyMapFunction());

// 익명 클래스
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});

````

## Accumulators & Counters

- Accumulator는 `add()`로 값을 누적하고, **잡 종료 후 최종 누적 결과**를 클라이언트에서 조회할 수 있는 메커니즘
- 디버깅, 데이터 분포/규모 파악 등 “잡 실행 결과에 대한 인사이트”를 빠르게 얻는 데 유용

### 내장 Accumulator 종류

- `IntCounter`, `LongCounter`, `DoubleCounter`: 카운터(증가/합산)
- `Histogram`: 구간(bin)별 빈도를 담는 히스토그램(내부적으로 `Integer -> Integer` 맵 형태)

### 사용 절차

1) UDF 내부에 accumulator 객체 생성(예: `IntCounter`)
2) 보통 Rich 함수의 `open()`에서 이름과 함께 등록
    - `getRuntimeContext().addAccumulator("num-lines", counter)`
3) 처리 로직(또는 `open()/close()` 포함) 어디서든 `add()`로 누적
    - `counter.add(1)`
4) 잡이 끝나면 `execute()`의 `JobExecutionResult`에서 결과 조회
    - `getAccumulatorResult("num-lines")`

````
// 1. UDF 내부에 accumulator 객체 생성
private IntCounter numLines = new IntCounter();

// 2. open()에서 등록
getRuntimeContext().addAccumulator("num-lines", this.numLines);

// 3. 처리 로직 어디서든 누적
this.numLines.add(1);

// 4. 잡 완료 후 결과 조회
myJobExecutionResult.getAccumulatorResult("num-lines");
````

- 주의: 결과 조회는 **잡 완료를 기다리는 실행**에서만 동작한다는 점이 언급됨

### 네임스페이스/병합 규칙

- Accumulator 이름은 **잡 단위로 단일 네임스페이스**를 공유한다
- 같은 이름을 여러 operator에서 쓰면 Flink가 내부적으로 **모든 부분 결과를 병합(merge)** 한다

### 커스텀 Accumulator

- `Accumulator`(또는 더 단순한 `SimpleAccumulator`) 인터페이스를 구현해 직접 만들 수 있다
- `Accumulator<V, R>`: 입력(누적 값) 타입 V와 최종 결과 타입 R을 분리해 가장 유연
- `SimpleAccumulator`: V와 R이 같은 경우(예: 카운터)에 적합
