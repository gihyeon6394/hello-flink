# Working with State #

### 목적

- Flink에서 **stateful(상태 기반) 프로그램**을 작성할 때 사용하는 API들을 소개한다.
- 개념적 배경은 *Stateful Stream Processing*을 참고한다.

## Keyed DataStream

### Keyed State를 쓰는 흐름

- **keyed state**를 사용하려면 먼저 `DataStream`에 **키를 지정**해 레코드와 상태를 **파티셔닝**해야 한다.
    - Java: `keyBy(KeySelector)`
    - Python: `key_by(KeySelector)`
- 키를 지정하면 `KeyedStream`이 되고, 여기서 keyed state 기반 연산이 가능해진다.

### KeySelector와 “가상 키”

- `KeySelector`는 레코드 1개를 입력으로 받아 **결정적(deterministic)** 연산으로 키를 반환한다.
- Flink 데이터 모델은 기본적으로 key-value 쌍이 아니라서, 데이터를 억지로 key/value로 포장할 필요가 없다.
- 키는 실제 데이터 위에 정의되는 **“가상(virtual) 키”**로, 그룹핑/파티셔닝 기준을 제공한다.

### Tuple/Expression Keys (Java 한정)

- Java API에는 tuple 필드 인덱스나 표현식 기반 키(튜플 키/표현식 키)도 있지만 **권장하지 않는다**.
- Java 람다 기반 `KeySelector`가 더 단순하고 런타임 오버헤드가 적을 수 있다.
- (Python DataStream API에서는 미지원)

```java
// some ordinary POJO
public class WC {
    public String word;
    public int count;

    public String getWord() {
        return word;
    }
}

DataStream<WC> words = // [...]
        KeyedStream < WC > keyed = words
                .keyBy(WC::getWord);
```

## Using Keyed State

### Keyed State의 특징

- keyed state는 **현재 처리 중인 레코드의 key 스코프**에 종속된다.
- 따라서 **KeyedStream에서만** 사용 가능하다.

### 제공되는 State Primitive

- `ValueState<T>`: 키별 단일 값 저장/갱신 (`update`, `value`)
- `ListState<T>`: 키별 리스트 저장 (`add`, `addAll`, `get`, `update`)
- `ReducingState<T>`: `ReduceFunction`으로 누적 집계된 단일 값 유지 (`add` 시 reduce)
- `AggregatingState<IN, OUT>`: `AggregateFunction`으로 누적 집계(입력 타입과 결과 타입이 달라질 수 있음)
- `MapState<UK, UV>`: 키별 맵(엔트리) 저장 (`put`, `get`, `entries/keys/values`, `isEmpty`)
- 공통: `clear()`로 **현재 key의 상태만** 제거

### 저장 위치와 키 의존성 주의

- state 객체는 “핸들(인터페이스)”일 뿐, 실제 저장은 메모리/디스크 등 backend에 있을 수 있다.
- 같은 함수 호출이라도 **입력 key가 다르면** 조회되는 state 값이 달라진다.

### StateDescriptor와 RuntimeContext

- state를 얻으려면 먼저 `StateDescriptor`(이름, 타입, (필요 시) 집계 함수 등)를 만든다.
    - 예: `ValueStateDescriptor`, `ListStateDescriptor`, `ReducingStateDescriptor`, `AggregatingStateDescriptor`,
      `MapStateDescriptor`
- state 접근은 `RuntimeContext`를 통해 이뤄지므로 **RichFunction에서만** 가능하다.
    - `getState`, `getListState`, `getReducingState`, `getAggregatingState`, `getMapState`

```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        Tuple2<Long, Long> currentSum = sum.value();

        // update the count
        currentSum.f0 += 1;

        // add the second field of the input value
        currentSum.f1 += input.f1;

        // update the state
        sum.update(currentSum);

        // if the count reaches 2, emit the average and clear the state
        if (currentSum.f0 >= 2) {
            out.collect(new Tuple2<>(input.f0, currentSum.f1 / currentSum.f0));
            sum.clear();
        }
    }

    @Override
    public void open(OpenContext ctx) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {
                        }), // type information
                        Tuple2.of(0L, 0L)); // default value of the state, if nothing was set
        sum = getRuntimeContext().getState(descriptor);
    }
}
```

```
// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(value -> value.f0)
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
```

## State Time-To-Live (TTL)

- 모든 keyed state 타입에 **TTL(만료 시간)** 을 설정할 수 있다.
- 컬렉션 타입(List/Map)은 **엔트리 단위(per-entry)** TTL을 지원(요소/엔트리별로 독립 만료).

### TTL 설정 방법

```
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;
import java.time.Duration;

StateTtlConfig ttlConfig = StateTtlConfig
    .newBuilder(Duration.ofSeconds(1))
    .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
    .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
    .build();
    
ValueStateDescriptor<String> stateDescriptor = new ValueStateDescriptor<>("text state", String.class);
stateDescriptor.enableTimeToLive(ttlConfig);
```

- `StateTtlConfig`를 만들고, `StateDescriptor.enableTimeToLive(ttlConfig)`로 활성화한다.
- 주요 옵션
    - TTL 값(필수)
    - UpdateType: TTL 갱신 시점
        - `OnCreateAndWrite`(기본): 생성/쓰기 시 갱신
        - `OnReadAndWrite`: 읽기에서도 갱신
    - StateVisibility: 만료값을 읽기에서 반환할지
        - `NeverReturnExpired`(기본): 만료값 미반환(존재하지 않는 것처럼 동작)
        - `ReturnExpiredIfNotCleanedUp`: 정리 전이면 만료값 반환 가능

### TTL 사용 시 주의사항

- TTL은 **processing time 기준만** 지원된다.
- TTL 설정은 체크포인트/세이브포인트에 “데이터로 포함”되는 게 아니라, **현재 실행 잡에서 state를 다루는 방식**이다.
- 짧은 TTL에서 긴 TTL로 조정하며 복원하는 것은 **데이터 오류 가능성** 때문에 권장되지 않는다.
- TTL 활성화 시 저장 공간이 늘어난다(Heap은 추가 객체/필드, RocksDB는 값/엔트리당 추가 바이트 등).
- TTL 활성화 시 `StateDescriptor`의 `defaultValue`(deprecated)는 **효과가 없어짐**(null/만료 시 기본값 처리는 사용자가 명시적으로 관리).

## Cleanup of Expired State

- 기본적으로 만료된 값은 **읽기 시점에 제거**되고,
- backend가 지원하면 **백그라운드 정리**도 수행된다(옵션으로 비활성화 가능).

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .disableCleanupInBackground()
        .build();
```

### Full Snapshot 시 정리

- 전체 스냅샷 생성 시 만료 상태를 제외해 **스냅샷 크기 감소** 가능
- RocksDB **incremental checkpointing**에는 적용되지 않는다.

```java
import org.apache.flink.api.common.state.StateTtlConfig;

import java.time.Duration;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .cleanupFullSnapshot()
        .build();
```

### Incremental Cleanup (주로 Heap)

- state 접근/레코드 처리 시마다 일부 엔트리를 검사하며 점진적으로 정리
- 접근/처리가 없으면 만료 상태가 남을 수 있고, 정리 비용이 **레코드 처리 지연**으로 이어질 수 있다.
- 현재는 **Heap에서만 의미 있게 동작**, RocksDB에 설정해도 효과가 없을 수 있다.
- 특정 구현/스냅샷 방식에서는 메모리 사용량이 증가할 수 있다.

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .cleanupIncrementally(10, true)
        .build();
```

### RocksDB Compaction Filter 기반 정리

- RocksDB backend 사용 시 compaction 과정에서 TTL 필터가 만료 엔트리를 제외한다.
- 타임스탬프 조회 빈도, periodic compaction 설정에 따라 정리 속도/성능이 트레이드오프가 있다.
- compaction 중 TTL 필터 실행은 compaction을 느리게 만들 수 있다(특히 컬렉션 상태는 요소 단위 검사 비용).

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .cleanupInRocksdbCompactFilter(1000, Duration.ofHours(1))
        .build();
```

### TTL Migration Compatibility

- Flink **2.2.0+**에서는 TTL 적용/비적용 상태 간 **복원 시 호환(마이그레이션)** 을 안전하게 지원한다.

## Operator State

- Operator state(비-keyed)는 **병렬 연산자 인스턴스(서브태스크)** 에 바인딩되는 상태.
- 병렬도 변경(rescale) 시 상태 재분배를 지원한다.
- 일반 앱에서는 덜 쓰이고, 주로 **source/sink 구현**(예: Kafka 파티션 오프셋 관리)에 활용된다.
- (Python DataStream API에서는 operator state 미지원)

## Broadcast State

### 특징

- broadcast stream을 통해 모든 다운스트림 태스크에 동일하게 전달되는 **특수한 operator state**
- 규칙/룰 스트림(저처리량)을 브로드캐스트하고, 다른 메인 스트림에 적용하는 패턴에 적합
- 차이점
    - map 형태
    - broadcast 입력 + non-broadcast 입력을 받는 특정 연산자에서만 사용
    - 서로 다른 이름의 broadcast state를 여러 개 가질 수 있음

## Using Operator State

### CheckpointedFunction

````java
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
````

- operator state를 쓰려면 보통 `CheckpointedFunction`을 구현한다.
    - `snapshotState(...)`: 체크포인트 시 state 스냅샷 저장
    - `initializeState(...)`: 초기화 및 복원 로직(복구 시에도 호출)
- 현재는 **list-style operator state**가 핵심이며, 재분배 스킴
    - Even-split: 전체 리스트를 병렬 인스턴스 수로 균등 분배
    - Union: 모든 인스턴스가 전체 리스트를 받음(카디널리티가 크면 메타데이터/메모리 문제 위험)

```java
public class BufferingSink
        implements SinkFunction<Tuple2<String, Integer>>,
        CheckpointedFunction {

    private final int threshold;

    private transient ListState<Tuple2<String, Integer>> checkpointedState;

    private List<Tuple2<String, Integer>> bufferedElements;

    public BufferingSink(int threshold) {
        this.threshold = threshold;
        this.bufferedElements = new ArrayList<>();
    }

    @Override
    public void invoke(Tuple2<String, Integer> value, Context context) throws Exception {
        bufferedElements.add(value);
        if (bufferedElements.size() >= threshold) {
            for (Tuple2<String, Integer> element : bufferedElements) {
                // send it to the sink
            }
            bufferedElements.clear();
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        checkpointedState.update(bufferedElements);
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        ListStateDescriptor<Tuple2<String, Integer>> descriptor =
                new ListStateDescriptor<>(
                        "buffered-elements",
                        TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {
                        }));

        checkpointedState = context.getOperatorStateStore().getListState(descriptor);

        if (context.isRestored()) {
            for (Tuple2<String, Integer> element : checkpointedState.get()) {
                bufferedElements.add(element);
            }
        }
    }
}
```

`````
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
`````

## Stateful Source Functions

### 체크포인트 락(원자성) 주의

- state 업데이트와 `collect()`를 **원자적으로** 수행해야 exactly-once에 유리하다.
- 이를 위해 소스의 `ctx.getCheckpointLock()`으로 lock을 얻어 동기화 블록에서 처리한다.

```java
public static class CounterSource
        extends RichParallelSourceFunction<Long>
        implements CheckpointedFunction {

    /**  current offset for exactly once semantics */
    private Long offset = 0L;

    /** flag for job cancellation */
    private volatile boolean isRunning = true;

    /** Our state object. */
    private ListState<Long> state;

    @Override
    public void run(SourceContext<Long> ctx) {
        final Object lock = ctx.getCheckpointLock();

        while (isRunning) {
            // output and state update are atomic
            synchronized (lock) {
                ctx.collect(offset);
                offset += 1;
            }
        }
    }

    @Override
    public void cancel() {
        isRunning = false;
    }

    @Override
    public void initializeState(FunctionInitializationContext context) throws Exception {
        state = context.getOperatorStateStore().getListState(new ListStateDescriptor<>(
                "state",
                LongSerializer.INSTANCE));

        // restore any state that we might already have to our fields, initialize state
        // is also called in case of restore.
        for (Long l : state.get()) {
            offset = l;
        }
    }

    @Override
    public void snapshotState(FunctionSnapshotContext context) throws Exception {
        state.update(Collections.singletonList(offset));
    }
}
```

### 외부 시스템과의 연동

- 체크포인트가 완전히 승인(ack)된 시점을 알아야 한다면 `CheckpointListener`를 사용한다.
