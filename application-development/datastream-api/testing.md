# Testing

Apache Flink는 streaming/batch 애플리케이션을 **테스트 피라미드** 여러 레벨(단위 테스트 → operator/UDF 테스트 → 잡(통합) 테스트)에서 검증할 수 있는 툴링을 제공한다.
일반적으로 Flink 런타임 자체는 올바르게 동작한다고 가정하고, **비즈니스 로직이 들어있는 User-Defined Function(UDF)** 를 중심으로 테스트하는 것을 권장한다.

## Testing User-Defined Functions

### Unit Testing Stateless, Timeless UDFs

상태/시간에 의존하지 않는 UDF(MapFunction 등)는 일반적인 단위 테스트로 가장 쉽게 검증 가능하다.

```java
public class IncrementMapFunction implements MapFunction<Long, Long> {
    @Override
    public Long map(Long record) {
        return record + 1;
    }
}

public class IncrementMapFunctionTest {
    @Test
    public void testIncrement() throws Exception {
        IncrementMapFunction f = new IncrementMapFunction();
        assertEquals(3L, f.map(2L));
    }
}
```

### Collector를 사용하는 UDF 테스트

FlatMapFunction/ProcessFunction처럼 `Collector` 로 결과를 내보내는 함수는 **mock Collector** 를 주입해 호출 여부/값을 검증할 수 있다.

```java
public class IncrementFlatMapFunctionTest {
    @Test
    public void testIncrement() throws Exception {
        IncrementFlatMapFunction f = new IncrementFlatMapFunction();

        Collector<Long> collector = mock(Collector.class);
        f.flatMap(2L, collector);

        Mockito.verify(collector, times(1)).collect(3L);
    }
}
```

## Unit Testing Stateful or Timely UDFs & Custom Operators

managed state 또는 timer(event time/processing time)를 쓰는 UDF/커스텀 operator는 Flink runtime과의 상호작용을 포함하므로 더 어렵다. Flink는 이를 위해
**test harness** 를 제공하며, 입력 레코드/워터마크를 넣고 시간(processing/event)을 제어한 뒤 출력(메인/side output)을 검증할 수 있다.

### 주요 Test Harness 종류

- `OneInputStreamOperatorTestHarness` (DataStream 단일 입력 operator)
- `KeyedOneInputStreamOperatorTestHarness` (KeyedStream 단일 입력 operator)
- `TwoInputStreamOperatorTestHarness` (ConnectedStreams 두 입력 operator)
- `KeyedTwoInputStreamOperatorTestHarness` (Keyed + 두 입력 operator)

### Harness로 상태/타이머/워터마크 검증 흐름

- `processElement(value, timestamp)` 로 timestamped element 주입
- `processWatermark(wm)` 로 **event-time timer** 트리거(워터마크 전진)
- `setProcessingTime(t)` 로 **processing-time timer** 트리거
- `getOutput()` 또는 side output 조회로 assertion

```java
public class StatefulFlatMapTest {
    private OneInputStreamOperatorTestHarness<Long, Long> harness;

    @Before
    public void setup() throws Exception {
        StatefulFlatMapFunction udf = new StatefulFlatMapFunction();
        harness = new OneInputStreamOperatorTestHarness<>(
                new StreamFlatMap<>(udf)
        );
        harness.getExecutionConfig().setAutoWatermarkInterval(50);
        harness.open();
    }

    @Test
    public void testStateful() throws Exception {
        harness.processElement(2L, 100L);
        harness.processWatermark(100L);   // event time timer trigger
        harness.setProcessingTime(100L);  // processing time timer trigger

        // 출력 검증(예시)
        // assertThat(harness.getOutput(), containsInExactlyThisOrder(...));
        // side output도 확인 가능:
        // harness.getSideOutput(new OutputTag<>("invalidRecords"));
    }
}
```

### Keyed Harness 생성

`KeyedOneInputStreamOperatorTestHarness` / `KeyedTwoInputStreamOperatorTestHarness` 는 추가로 **KeySelector +
TypeInformation** 이 필요하다.

```
testHarness = new KeyedOneInputStreamOperatorTestHarness<>(
    new StreamFlatMap<>(statefulFlatMapFunction),
    new MyStringKeySelector(),
    Types.STRING
);
```

### 참고

`AbstractStreamOperatorTestHarness` 및 파생 클래스는 **public API가 아니며 변경될 수 있음**.

## Unit Testing ProcessFunction

ProcessFunction은 활용도가 높아, harness 생성을 더 쉽게 해주는 팩토리 `ProcessFunctionTestHarnesses` 를 제공한다. ProcessFunction을 operator로 감싼
뒤 element를 넣고 결과를 추출해 검증하는 패턴이다.

```java
public static class PassThroughProcessFunction extends ProcessFunction<Integer, Integer> {
    @Override
    public void processElement(Integer value, Context ctx, Collector<Integer> out) {
        out.collect(value);
    }
}

public class PassThroughProcessFunctionTest {
    @Test
    public void testPassThrough() throws Exception {
        OneInputStreamOperatorTestHarness<Integer, Integer> harness =
                ProcessFunctionTestHarnesses.forProcessFunction(new PassThroughProcessFunction());

        harness.processElement(1, 10);
        assertEquals(Collections.singletonList(1), harness.extractOutputValues());
    }
}
```

`ProcessFunctionTestHarnessesTest` 를 참고하면 `KeyedProcessFunction`, `KeyedCoProcessFunction`, `BroadcastProcessFunction` 등
다양한 변형 테스트 예시를 확인할 수 있다.

## Testing Flink Jobs

### JUnit Rule: MiniClusterWithClientResource

전체 잡을 로컬에 임베드된 mini cluster에서 실행하는 **통합 테스트**를 위해 `MiniClusterWithClientResource` 를 사용할 수 있다.

테스트 스코프 dependency 예시:

```xml

<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-test-utils</artifactId>
    <version>2.2.0</version>
    <scope>test</scope>
</dependency>
```

### MiniCluster 기반 통합 테스트 예시

- `StreamExecutionEnvironment` 구성(병렬도 포함)
- 테스트용 sink로 결과 수집 후 assertion
- Flink는 operator를 serialize해 분산 실행하므로, 예시에서는 **static 변수**로 결과를 회수

```java
public class ExampleIntegrationTest {

    @ClassRule
    public static MiniClusterWithClientResource flinkCluster =
            new MiniClusterWithClientResource(
                    new MiniClusterResourceConfiguration.Builder()
                            .setNumberSlotsPerTaskManager(2)
                            .setNumberTaskManagers(1)
                            .build());

    @Test
    public void testIncrementPipeline() throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
        env.setParallelism(2);

        CollectSink.values.clear();

        env.fromElements(1L, 21L, 22L)
                .map(new IncrementMapFunction())
                .addSink(new CollectSink());

        env.execute();

        assertTrue(CollectSink.values.containsAll(Arrays.asList(2L, 22L, 23L)));
    }

    private static class CollectSink implements SinkFunction<Long> {
        public static final List<Long> values =
                Collections.synchronizedList(new ArrayList<>());

        @Override
        public void invoke(Long value, SinkFunction.Context context) {
            values.add(value);
        }
    }
}
```

### MiniCluster 통합 테스트 팁

- 프로덕션 코드를 테스트에 복붙하지 않도록 **source/sink를 플러그인 형태로** 설계하고 테스트에서 테스트용 source/sink를 주입
- static 변수 대신 임시 디렉터리에 파일로 쓰는 방식도 가능
- event time/timer를 쓰는 잡은 테스트용 source로 **watermark 방출**을 구현하는 것도 고려
- 병렬 실행에서만 드러나는 버그를 잡기 위해 **parallelism > 1** 로 로컬 테스트 권장
- `@Rule` 보다 `@ClassRule` 권장: 여러 테스트가 같은 cluster를 공유해 스타트업/셔트다운 비용을 절약
- custom state 처리의 정확성을 보려면 checkpointing을 켜고, 테스트 전용 UDF에서 예외를 던져 **failure/restart** 시나리오를 검증할 수 있다
