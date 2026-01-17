# Event-driven Applications

ProcessFunction은 **이벤트 처리 + 상태(state) + 타이머(timer)** 를 결합한 API로,  
Flink에서 **이벤트 중심(event-driven) 애플리케이션을 구현하는 핵심 구성 요소**다.  
RichFlatMapFunction과 유사하지만, **타이머 기능이 추가**되어 시간 기반 제어가 가능하다.

ProcessFunction은 단순 집계를 넘어, **복잡한 비즈니스 로직과 이벤트 반응형 처리**를 구현하는 기반이 된다.

### ProcessFunction 예제 개념

```java
// compute the sum of the tips per hour for each driver
DataStream<Tuple3<Long, Long, Float>> hourlyTips = fares
                .keyBy((TaxiFare fare) -> fare.driverId)
                .window(TumblingEventTimeWindows.of(Duration.ofHours(1)))
                .process(new AddTips());
````

기존에는 다음과 같이 **시간 윈도우 API**를 사용해 운전자별 시간당 팁 합계를 계산했다.

- Tumbling Event Time Window
- 윈도우 종료 시 결과 출력

같은 기능을 **KeyedProcessFunction**으로 직접 구현하면:

```java
// compute the sum of the tips per hour for each driver
DataStream<Tuple3<Long, Long, Float>> hourlyTips = fares
                .keyBy((TaxiFare fare) -> fare.driverId)
                .process(new PseudoWindow(Duration.ofHours(1)));
````

- 윈도우 개념을 코드로 명시
- 상태와 타이머를 직접 제어
- 더 높은 자유도 확보

### KeyedProcessFunction 구조

```java
// Compute the sum of the tips for each driver in hour-long windows.
// The keys are driverIds.
public static class PseudoWindow extends
        KeyedProcessFunction<Long, TaxiFare, Tuple3<Long, Long, Float>> {

    private final long durationMsec;

    public PseudoWindow(Time duration) {
        this.durationMsec = duration.toMilliseconds();
    }

    @Override
    // Called once during initialization.
    public void open(OpenContext ctx) {
        // ...
    }

    @Override
    // Called as each fare arrives to be processed.
    public void processElement(
            TaxiFare fare,
            Context ctx,
            Collector<Tuple3<Long, Long, Float>> out) throws Exception {

        // ...
    }

    @Override
    // Called when the current watermark indicates that a window is now complete.
    public void onTimer(long timestamp,
                        OnTimerContext context,
                        Collector<Tuple3<Long, Long, Float>> out) throws Exception {

        // ...
    }
}

```

KeyedProcessFunction은 다음 형태를 가진다.

- open(): 상태 초기화
- processElement(): 이벤트 처리
- onTimer(): 타이머 트리거 시 호출

특징:

- 키별(keyed) 상태 관리 가능
- Event Time / Processing Time 타이머 사용 가능
- Collector를 통해 결과 출력

### 상태 설계 (open 메서드)

```java
// Keyed, managed state, with an entry for each window, keyed by the window's end time.
// There is a separate MapState object for each driver.
private transient MapState<Long, Float> sumOfTips;

@Override
public void open(OpenContext ctx) {
    MapStateDescriptor<Long, Float> sumDesc =
            new MapStateDescriptor<>("sumOfTips", Long.class, Float.class);
    sumOfTips = getRuntimeContext().getMapState(sumDesc);
}
````

이 예제에서는 **MapState**를 사용한다.

- key: 윈도우 종료 시각
- value: 해당 윈도우의 팁 합계
- 운전자(driverId)별로 독립된 MapState 존재

이 방식은:

- 이벤트가 out-of-order로 도착해도
- 여러 윈도우를 동시에 열어 처리할 수 있게 해준다

### 이벤트 처리 로직 (processElement)

```java
public void processElement(
        TaxiFare fare,
        Context ctx,
        Collector<Tuple3<Long, Long, Float>> out) throws Exception {

    long eventTime = fare.getEventTime();
    TimerService timerService = ctx.timerService();

    if (eventTime <= timerService.currentWatermark()) {
        // This event is late; its window has already been triggered.
    } else {
        // Round up eventTime to the end of the window containing this event.
        long endOfWindow = (eventTime - (eventTime % durationMsec) + durationMsec - 1);

        // Schedule a callback for when the window has been completed.
        timerService.registerEventTimeTimer(endOfWindow);

        // Add this fare's tip to the running total for that window.
        Float sum = sumOfTips.get(endOfWindow);
        if (sum == null) {
            sum = 0.0F;
        }
        sum += fare.tip;
        sumOfTips.put(endOfWindow, sum);
    }
}
```

processElement의 핵심 흐름은 다음과 같다.

- 이벤트의 event time 추출
- 현재 watermark와 비교
    - watermark 이전 → late event
    - watermark 이후 → 정상 처리
- 이벤트가 속한 윈도우의 종료 시각 계산
- 해당 시각에 Event Time 타이머 등록
- MapState에 팁 금액 누적

특징:

- late event는 기본적으로 드롭
- 윈도우 종료 시각과 타이머 시각을 동일하게 사용
- 상태 조회 및 정리가 단순해짐

### 타이머 처리 (onTimer)

```java
public void onTimer(
        long timestamp,
        OnTimerContext context,
        Collector<Tuple3<Long, Long, Float>> out) throws Exception {

    long driverId = context.getCurrentKey();
    // Look up the result for the hour that just ended.
    Float sumOfTips = this.sumOfTips.get(timestamp);

    Tuple3<Long, Long, Float> result = Tuple3.of(driverId, timestamp, sumOfTips);
    out.collect(result);
    this.sumOfTips.remove(timestamp);
}
```

onTimer는 **watermark가 윈도우 종료 시각에 도달했을 때 호출**된다.

동작:

- 현재 key(driverId) 조회
- 해당 윈도우의 팁 합계 조회
- 결과 출력
- MapState에서 해당 윈도우 엔트리 제거

이 설계는:

- allowedLateness = 0 인 윈도우와 동일한 동작
- 타이머 실행 후에는 late event를 더 이상 수용하지 않음

### 성능 고려 사항

- RocksDB 상태 백엔드 사용 시
    - MapState, ListState는 매우 효율적
    - ValueState에 컬렉션을 직접 넣는 방식보다 권장
- MapState는 key/value 단위로 저장되어
    - 대규모 상태에서도 부분 접근이 효율적

### Side Outputs

Side Output은 **하나의 연산자에서 여러 출력 스트림을 생성**하는 메커니즘이다.

주요 활용 사례:

- 예외 이벤트
- 잘못된 데이터
- late events
- 운영 알림
- 스트림 분기(n-way split)

### Side Output 예제

```java
private static final OutputTag<TaxiFare> lateFares = new OutputTag<TaxiFare>("lateFares") {
};

```

late event를 버리지 않고 별도 스트림으로 분리할 수 있다.

```
if (eventTime <= timerService.currentWatermark()) {
    // This event is late; its window has already been triggered.
    ctx.output(lateFares, fare);
} else {
    // ...
}
```

```
// compute the sum of the tips per hour for each driver
SingleOutputStreamOperator hourlyTips = fares
        .keyBy((TaxiFare fare) -> fare.driverId)
        .process(new PseudoWindow(Duration.ofHours(1)));

hourlyTips.getSideOutput(lateFares).print();
```

- OutputTag<T> 로 side output 정의
- processElement에서 ctx.output(tag, value)로 전송
- 메인 스트림 결과에서 getSideOutput으로 접근

이를 통해:

- 정상 데이터와 late 데이터를 분리 처리 가능
- 운영 및 분석 로직 분리 가능

### 마무리 정리

- ProcessFunction은 윈도우 API를 **완전히 대체**할 수 있다
- 기본 윈도우로 해결 가능하면 윈도우 API 사용 권장
- 복잡한 시간·상태 제어가 필요하면 ProcessFunction이 적합
- ProcessFunction은 분석뿐 아니라:
    - 상태 만료
    - 이벤트 타임아웃 감지
    - 조인 누락 처리
      같은 이벤트 기반 로직에 매우 유용하다
