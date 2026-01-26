# Working with State V2 (New APIs)

- Flink의 **State V2(새 State API)** 는 기존 API보다 **유연**하며, 특히 **비동기(asynchronous) 상태 접근**을 지원한다.
- 비동기 상태 연산은 대규모 상태를 효율적으로 처리하고, 필요 시 원격 파일시스템으로 spill하는 **분리형(disaggregated) 상태 관리**를 가능하게 하는 핵심 요소다.

## Keyed DataStream

### Keyed state 사용 조건

- keyed state를 쓰려면 먼저 `DataStream`에 **키를 지정**해 `KeyedStream`을 만들어야 한다: `keyBy(KeySelector)`
- 그리고 **비동기 상태 연산 활성화**를 위해 `KeyedStream.enableAsyncState()`를 호출해야 한다.
- KeySelector는 레코드 → 키를 반환하는 함수이며, 키는 어떤 타입이든 가능하지만 **결정적(deterministic)** 이어야 한다.
- Flink는 기본 모델이 key-value 쌍이 아니므로, 키는 데이터에 “붙이는” 물리적 필드가 아니라 **그룹핑을 위한 가상(virtual) 함수**로 이해하면 된다.

```java
// some ordinary POJO
public class WordCount {
    public String word;
    public int count;

    public String getWord() {
        return word;
    }
}

DataStream<WordCount> words = // [...]
        KeyedStream < WordCount > keyed = words
                .keyBy(WordCount::getWord).enableAsyncState();

```

## Using Keyed State V2

### 동기 vs 비동기 API

- 각 state 타입은 **동기(synchronous)** 와 **비동기(asynchronous)** 두 버전을 제공한다.
    - 동기 API: 상태 접근 완료까지 **블로킹**
    - 비동기 API: **논블로킹**, `StateFuture`를 반환
- 비동기 API가 더 효율적이며 **가능하면 비동기 사용 권장**.
- 같은 user function 내에서 **동기/비동기 혼용은 강력 비권장**(예측 어려운 실행 순서/성능 문제).

### 비동기 반환 타입: StateFuture / StateIterator

- `StateFuture<T>`: 상태 접근 결과가 나중에 채워지는 future
    - `thenAccept(Consumer<T>)`: 결과 소비 후 `StateFuture<Void>`
    - `thenApply(Function<T,R>)`: 결과 변환 후 `StateFuture<R>`
    - `thenCompose(Function<T, StateFuture<R>>)` : future 체이닝(flatten)
    - `thenCombine(StateFuture<U>, BiFunction<T,U,R>)`: 두 future 결합
    - `get()`처럼 스레드를 막는 API는 제공하지 않음(블로킹으로 인한 재귀적 정체 방지 목적)
    - 결과에 따라 분기하는 조건부 체이닝도 제공:
        - `thenConditionallyAccept / Apply / Compose / Combine`
- `StateIterator<T>`: 상태의 여러 요소를 **지연 로딩 형태로 순회**하기 위한 iterator
    - `isEmpty()`: 동기 체크
    - `onNext(Consumer<T>)`: 다음 요소를 받아 처리(완료 시 future)
    - `onNext(Function<T,R>)`: 다음 요소를 변환해 결과들을 컬렉션으로 모아 future로 반환

### 유틸리티: StateFutureUtils

- `completedFuture(T)`: 즉시 완료된 `StateFuture` 생성(체이닝 중 상수 반환 등에 사용)
- `completedVoidFuture()`: 즉시 완료된 `StateFuture<Void>`
- `combineAll(Collection<StateFuture<T>>)` : 여러 future 완료를 모아 `StateFuture<Collection<T>>`로 반환
- `toIterable(StateFuture<StateIterator<T>>)` : iterator 전체를 iterable로 변환(지연 로딩 이점이 사라질 수 있어 일반적으로 비권장)

## State Primitives (비동기 중심)

### 제공되는 keyed state 타입과 주요 비동기 메서드

- `ValueState<T>`
    - 조회: `asyncValue() -> StateFuture<T>`
    - 갱신: `asyncUpdate(T)`
- `ListState<T>`
    - 추가: `asyncAdd(T)`, `asyncAddAll(List<T>)`
    - 조회: `asyncGet() -> StateFuture<StateIterator<T>>`
    - 덮어쓰기: `asyncUpdate(List<T>)`
- `ReducingState<T>`
    - `asyncAdd(T)` 시 `ReduceFunction`으로 누적 집계(단일 값 유지)
- `AggregatingState<IN, OUT>`
    - `asyncAdd(IN)` 시 `AggregateFunction`으로 집계(입력/출력 타입 달라질 수 있음)
- `MapState<UK, UV>`
    - put: `asyncPut(UK,UV)`, `asyncPutAll(Map<UK,UV>)`
    - get: `asyncGet(UK)`
    - 순회: `asyncEntries() / asyncKeys() / asyncValues()`
    - 비었는지: `asyncIsEmpty()`
- 공통: `asyncClear()`는 현재 key 범위의 상태를 삭제

### StateDescriptor와 접근 방법(v2 패키지)

- 상태 핸들을 얻으려면 `StateDescriptor`가 필요(이름/타입/집계 함수 등 포함, 이름은 유니크해야 함).
- State V2는 **`org.apache.flink.api.common.state.v2` 패키지의 StateDescriptor/핸들 사용**으로 구분한다.
- 상태 접근은 `RuntimeContext`를 통해서만 가능하므로 **RichFunction**에서 사용한다.

## Example 패턴(요지)

```java
public class CountWindowAverage extends RichFlatMapFunction<Tuple2<Long, Long>, Tuple2<Long, Long>> {

    /**
     * The ValueState handle. The first field is the count, the second field a running sum.
     */
    private transient ValueState<Tuple2<Long, Long>> sum;

    @Override
    public void flatMap(Tuple2<Long, Long> input, Collector<Tuple2<Long, Long>> out) throws Exception {

        // access the state value
        sum.asyncValue().thenApply(currentSum -> {
            // if it hasn't been used before, it will be null
            Tuple2<Long, Long> current = currentSum == null ? Tuple2.of(0L, 0L) : currentSum;

            // update the count
            current.f0 += 1;

            // add the second field of the input value
            current.f1 += input.f1;

            return current;
        }).thenAccept(r -> {
            // if the count reaches 2, emit the average and clear the state
            if (r.f0 >= 2) {
                out.collect(Tuple2.of(input.f0, r.f1 / r.f0));
                sum.asyncClear();
            } else {
                sum.asyncUpdate(r);
            }
        });
    }

    @Override
    public void open(OpenContext ctx) {
        ValueStateDescriptor<Tuple2<Long, Long>> descriptor =
                new ValueStateDescriptor<>(
                        "average", // the state name
                        TypeInformation.of(new TypeHint<Tuple2<Long, Long>>() {
                        })); // type information
        sum = getRuntimeContext().getState(descriptor);
    }
}

```

```
// this can be used in a streaming program like this (assuming we have a StreamExecutionEnvironment env)
env.fromElements(Tuple2.of(1L, 3L), Tuple2.of(1L, 5L), Tuple2.of(1L, 7L), Tuple2.of(1L, 4L), Tuple2.of(1L, 2L))
        .keyBy(value -> value.f0)
        .enableAsyncState()
        .flatMap(new CountWindowAverage())
        .print();

// the printed output will be (1,4) and (1,5)
```

### 비동기 처리 흐름

- `asyncValue()`로 읽고 → `thenApply`로 계산 → `thenAccept`에서 `asyncUpdate` 또는 `asyncClear` 같은 후속 상태 연산을 수행하는 식으로
  **thenXXX 체이닝으로 단계적으로 로직을 분리**한다.
- 초기값(없으면 null)은 사용자가 직접 처리(예: null이면 (0,0)으로 시작).

## Execution Order(실행 순서)

### 비동기 상태 접근의 순서 특성

- 비동기 상태 접근은 레코드 A/B처럼 **서로 다른 입력 이벤트 간에는 완료 순서가 보장되지 않을 수 있음**.
- 다만 사용자 코드(예: `flatMap`, `thenXXX`에 전달된 람다)는 **태스크 스레드 단일 스레드에서 실행**되므로,
  사용자 코드 자체가 병렬로 동시에 실행되는 형태의 동시성 문제는 보통 없다.

### Flink가 보장하는 규칙

- **같은 키**에 대한 `flatMap`(entry) 호출은 **요소 도착 순서대로** 호출된다.
- 동일 체인으로 연결된 `thenXXX` 콜백은 **연결된 순서대로 실행**된다.
- 서로 다른 체인(분리된 체인) 간 실행 순서는 **보장되지 않음**.

## Best practices of asynchronous APIs

### 권장 사항

- 동기/비동기 혼용 피하기
- `thenXXX` **체이닝으로 결과 처리 + 다음 상태 접근**을 단계적으로 구성하기
- RichFunction의 **mutable 멤버 접근 최소화**
    - 비동기 완료 순서가 섞일 수 있어 멤버에 의존하면 예측 어려움
    - 대신 then 체인 결과로 데이터를 전달하거나
    - `StateFutureUtils.completedFuture(...)`, `thenApply(...)`로 값 전달
    - 또는 호출 단위로 초기화되는 캡처 컨테이너(예: `AtomicReference`) 사용

## State Time-To-Live (TTL)

### 공통 개념

- keyed state 모든 타입에 TTL 설정 가능, 컬렉션(List/Map)은 **엔트리 단위 TTL** 지원.
- `StateTtlConfig`를 만들고 `StateDescriptor.enableTimeToLive(ttlConfig)`로 활성화.
- TTL은 현재는 **processing time 기준**.

```java
import org.apache.flink.api.common.state.StateTtlConfig;
import org.apache.flink.api.common.state.ValueStateDescriptor;

import java.time.Duration;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .setUpdateType(StateTtlConfig.UpdateType.OnCreateAndWrite)
        .setStateVisibility(StateTtlConfig.StateVisibility.NeverReturnExpired)
        .build();

ValueStateDescriptor<String> stateDescript
```

### 주요 옵션

- TTL 값: `newBuilder(Duration)`의 첫 인자(필수)
- 업데이트 시점:
    - `OnCreateAndWrite`(기본): 생성/쓰기 시 갱신
    - `OnReadAndWrite`: 읽기에서도 갱신
- 만료값 가시성:
    - `NeverReturnExpired`(기본): 만료값은 읽기에서 반환하지 않음(엄격한 비공개/만료 요구에 유용)
    - `ReturnExpiredIfNotCleanedUp`: 정리 전이면 만료값 반환 가능
- TTL 메타데이터 저장으로 상태 저장소 사용량 증가
    - Heap: 추가 객체/long
    - RocksDB/ForSt: 값/엔트리당 추가 바이트(문서 기준 8 bytes)

### 정리(Cleanup)

- 기본적으로 만료값은 읽기 시 제거될 수 있고, 백엔드가 지원하면 백그라운드 정리도 가능.
- `disableCleanupInBackground()`로 백그라운드 정리 비활성화 가능.
- **ForSt State Backend**는 현재(문서 기준) 주로 **compaction 과정에서만 정리**한다.

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .disableCleanupInBackground()
        .build();
```

## Cleanup during compaction (ForSt)

```java
import org.apache.flink.api.common.state.StateTtlConfig;

StateTtlConfig ttlConfig = StateTtlConfig
        .newBuilder(Duration.ofSeconds(1))
        .cleanupInRocksdbCompactFilter(1000, Duration.ofHours(1))
        .build();
```

### 동작/튜닝 포인트

- ForSt는 비동기 compaction을 수행하며, Flink compaction filter가 TTL 만료 엔트리를 걸러낸다.
- `cleanupInRocksdbCompactFilter(queryTimeAfterNumEntries, periodicCompactionTime)`로 설정:
    - `queryTimeAfterNumEntries`: N개 처리마다 현재시각 조회(더 자주 조회하면 정리는 빨라질 수 있으나 JNI 비용으로 compaction 성능 저하)
    - `periodicCompactionTime`: 오래된 파일을 주기적으로 compaction해 **드물게 접근되는 만료 엔트리** 정리 촉진(짧을수록 compaction 증가)
- 디버그 로그: `log4j.logger.org.forstdb.FlinkCompactionFilter=DEBUG`
- 주의: compaction 중 TTL 필터는 오버헤드가 있으며, 특히 컬렉션 상태/가변 길이 요소에서는 JNI serializer 호출 등으로 비용이 커질 수 있음.

## Operator State / Broadcast State / CheckpointedFunction

### Operator State

- 병렬 연산자 인스턴스(서브태스크) 단위의 비-keyed state.
- 병렬도 변경 시 재분배 지원(주로 source/sink 구현에 사용).

### Broadcast State

- 규칙/룰 스트림을 전체 태스크로 broadcast하여 동일 상태를 유지하고,
  다른 스트림 처리에 참조하는 패턴에 적합.
- map 형태이며, broadcast+non-broadcast 입력을 받는 특정 연산자에서 사용.

### CheckpointedFunction

```java
void snapshotState(FunctionSnapshotContext context) throws Exception;

void initializeState(FunctionInitializationContext context) throws Exception;
```

- operator state는 `CheckpointedFunction`으로 관리 가능:
    - `snapshotState(...)`, `initializeState(...)`
- list-style operator state 기준 재분배:
    - even-split: 균등 분할
    - union: 전체 복제(대규모 리스트는 메타데이터/메모리 위험)

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

```
ListStateDescriptor<Tuple2<String, Integer>> descriptor =
    new ListStateDescriptor<>(
        "buffered-elements",
        TypeInformation.of(new TypeHint<Tuple2<String, Integer>>() {}));

checkpointedState = context.getOperatorStateStore().getListState(descriptor);
```

## Migrate from the Old State API

### 마이그레이션 절차

- `KeyedStream.enableAsyncState()` 호출
- StateDescriptor 및 state handle을 **v2 패키지로 교체**
- 기존 접근 로직을 **비동기 메서드(asyncXxx) 기반으로 재작성**
- 비동기 state 접근을 제대로 활용하려면 **ForSt State Backend 권장**
    - 다른 백엔드는 새 API를 쓰더라도 실제 접근이 동기일 수 있음
