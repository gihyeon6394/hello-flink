# The Broadcast State Pattern #

- **Broadcast State**를 실전에서 사용하는 패턴을 설명한다.
- 대표 유스케이스: **규칙/룰(rule) 스트림**을 모든 태스크에 **브로드캐스트**해서 로컬에 저장하고, 다른 **데이터(이벤트) 스트림**에 적용해 매칭/필터링/평가를 수행.

### 예시 시나리오(색/도형 + 규칙)

- Item 스트림: `Item(Color, Shape)` 형태의 이벤트
- Rule 스트림: “같은 색에서 사각형 다음 삼각형” 같은 **패턴 규칙**들이 시간에 따라 변함
- 목표: **같은 Color** 내에서 Rule에 맞는 Item 쌍을 찾아내기

## Provided APIs

### 1) 비브로드캐스트(메인) 스트림 키잉

- 같은 색끼리 같은 물리 머신/파티션에 가도록 **Color로 keyBy**:
    - `itemStream.keyBy(Color)` → `KeyedStream<Item, Color>`

```java
// key the items by color
KeyedStream<Item, Color> colorPartitionedStream = itemStream
                .keyBy(new KeySelector<Item, Color>() {
                    /*...*/
                });
```

### 2) Rule 스트림 브로드캐스트 + Broadcast State 생성

- Rule 스트림을 모든 downstream 태스크로 broadcast하고,
- `MapStateDescriptor`로 **Broadcast State(맵 형태)** 를 정의해 룰을 저장:
    - 키 예: ruleName(String)
    - 값 예: Rule 객체

```java
// a map descriptor to store the name of the rule (string) and the rule itself.
MapStateDescriptor<String, Rule> ruleStateDescriptor = new MapStateDescriptor<>(
                "RulesBroadcastState",
                BasicTypeInfo.STRING_TYPE_INFO,
                TypeInformation.of(new TypeHint<Rule>() {
                }));

// broadcast the rules and create the broadcast state
BroadcastStream<Rule> ruleBroadcastStream = ruleStream
        .broadcast(ruleStateDescriptor);
```

### 3) 두 스트림 연결 후 매칭 로직 적용

```java
DataStream<String> output = colorPartitionedStream
        .connect(ruleBroadcastStream)
        .process(
                // type arguments in our KeyedBroadcastProcessFunction represent: 
                //   1. the key of the keyed stream
                //   2. the type of elements in the non-broadcast side
                //   3. the type of elements in the broadcast side
                //   4. the type of the result, here a string

                new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {
                    // my matching logic
                }
        );
```

- 비브로드캐스트 스트림에서 `connect(broadcastStream)` 호출
- 반환된 connected stream에 `process()`를 호출해 로직 구현
- 비브로드캐스트 스트림 타입에 따라 사용하는 함수가 달라짐:
    - keyed 스트림이면 `KeyedBroadcastProcessFunction`
    - non-keyed 스트림이면 `BroadcastProcessFunction`

## BroadcastProcessFunction vs KeyedBroadcastProcessFunction

### 공통 구조

```java
public abstract class BroadcastProcessFunction<IN1, IN2, OUT> extends BaseBroadcastProcessFunction {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;
}

public abstract class KeyedBroadcastProcessFunction<KS, IN1, IN2, OUT> {

    public abstract void processElement(IN1 value, ReadOnlyContext ctx, Collector<OUT> out) throws Exception;

    public abstract void processBroadcastElement(IN2 value, Context ctx, Collector<OUT> out) throws Exception;

    public void onTimer(long timestamp, OnTimerContext ctx, Collector<OUT> out) throws Exception;
}

```

- 둘 다 “양쪽 입력”을 처리하는 메서드 2개를 구현:
    - `processBroadcastElement(...)`: **broadcast(rule) 쪽 입력** 처리
    - `processElement(...)`: **non-broadcast(item) 쪽 입력** 처리

### Context 차이(중요)

- non-broadcast 쪽: **ReadOnlyContext** (브로드캐스트 상태를 읽기만)
- broadcast 쪽: **Context** (브로드캐스트 상태 읽기/쓰기 가능)

### Context에서 공통으로 가능한 것들

- 브로드캐스트 상태 접근: `getBroadcastState(descriptor)`
- 이벤트 타임스탬프 조회: `timestamp()`
- 현재 watermark 조회: `currentWatermark()`
- 현재 processing time 조회: `currentProcessingTime()`
- 사이드 아웃풋 방출: `output(OutputTag, value)`
- 주의: `getBroadcastState()`에 넘기는 descriptor는 `broadcast(...)`에 사용한 것과 동일해야 함

### 왜 broadcast 쪽만 상태 수정 가능한가?

- Flink는 **태스크 간 직접 통신(cross-task communication)** 이 없으므로,
  모든 병렬 태스크에서 Broadcast State의 내용이 동일하게 유지되려면:
    - **브로드캐스트 입력(모든 태스크가 동일 입력을 받는 쪽)** 에서만 상태를 갱신하게 제한
    - 그리고 `processBroadcastElement()` 로직은 모든 병렬 인스턴스에서 **완전히 동일하고 결정적(deterministic)** 이어야 함
- 핵심 규칙:
    - `processBroadcastElement()`의 상태 업데이트는 **태스크마다 동일한 결과**를 만들어야 한다.

## KeyedBroadcastProcessFunction만 제공하는 추가 기능

### 타이머(timer) 사용

- keyed 스트림 기반이므로 `processElement()`의 ReadOnlyContext에서 **timer service 접근 가능**
- 타이머가 발화하면 `onTimer(...)`가 호출되며,
    - 이벤트/처리시간 타이머 구분
    - 타이머에 연관된 key 조회 등이 가능
- 제한:
    - **타이머 등록은 `processElement()`에서만 가능**
    - broadcast 입력에는 key가 없으므로 `processBroadcastElement()`에서 타이머 등록 불가

### applyToKeyedState

- broadcast 쪽 Context에 `applyToKeyedState(...)` 제공:
    - 특정 keyed state descriptor에 대해 **모든 key의 상태에 함수 적용** 가능
- 단, 문서 기준으로 PyFlink는 미지원

## 예시 로직의 핵심 아이디어(개념)

````
new KeyedBroadcastProcessFunction<Color, Item, Rule, String>() {

    // store partial matches, i.e. first elements of the pair waiting for their second element
    // we keep a list as we may have many first elements waiting
    private final MapStateDescriptor<String, List<Item>> mapStateDesc =
        new MapStateDescriptor<>(
            "items",
            BasicTypeInfo.STRING_TYPE_INFO,
            new ListTypeInfo<>(Item.class));

    // identical to our ruleStateDescriptor above
    private final MapStateDescriptor<String, Rule> ruleStateDescriptor = 
        new MapStateDescriptor<>(
            "RulesBroadcastState",
            BasicTypeInfo.STRING_TYPE_INFO,
            TypeInformation.of(new TypeHint<Rule>() {}));

    @Override
    public void processBroadcastElement(Rule value,
                                        Context ctx,
                                        Collector<String> out) throws Exception {
        ctx.getBroadcastState(ruleStateDescriptor).put(value.name, value);
    }

    @Override
    public void processElement(Item value,
                               ReadOnlyContext ctx,
                               Collector<String> out) throws Exception {

        final MapState<String, List<Item>> state = getRuntimeContext().getMapState(mapStateDesc);
        final Shape shape = value.getShape();
    
        for (Map.Entry<String, Rule> entry :
                ctx.getBroadcastState(ruleStateDescriptor).immutableEntries()) {
            final String ruleName = entry.getKey();
            final Rule rule = entry.getValue();
    
            List<Item> stored = state.get(ruleName);
            if (stored == null) {
                stored = new ArrayList<>();
            }
    
            if (shape == rule.second && !stored.isEmpty()) {
                for (Item i : stored) {
                    out.collect("MATCH: " + i + " - " + value);
                }
                stored.clear();
            }
    
            // there is no else{} to cover if rule.first == rule.second
            if (shape.equals(rule.first)) {
                stored.add(value);
            }
    
            if (stored.isEmpty()) {
                state.remove(ruleName);
            } else {
                state.put(ruleName, stored);
            }
        }
    }
}
````

### 상태 구성

- Broadcast State(룰): ruleName → Rule
- Keyed State(부분 매칭 저장): ruleName → “첫 번째 요소 후보들” 리스트
    - 같은 Color(key) 범위에서만 관리되므로, 색깔별로 독립적으로 매칭이 진행됨

### 처리 흐름

- `processBroadcastElement(rule)`:
    - 룰을 Broadcast State에 put/update
- `processElement(item)`:
    - 현재 브로드캐스트된 모든 룰을 순회하며
    - item의 shape이 룰의 second에 해당하고, 대기 중 first가 있으면 매칭 결과 emit
    - item의 shape이 룰의 first면 대기 리스트에 추가
    - 대기 리스트가 비면 해당 ruleName 엔트리를 제거해 상태를 정리

## Important Considerations

### 태스크 간 통신 없음

- Broadcast State 갱신은 broadcast 쪽에서만 가능
- 각 태스크에서 동일 입력을 동일 방식으로 처리해야 일관성이 유지됨

### 브로드캐스트 이벤트 도착 순서가 태스크마다 다를 수 있음

- 모든 요소가 “결국” 모든 태스크에 전달되지만,
  **도착 순서는 태스크별로 달라질 수 있음**
- 따라서 Broadcast State 업데이트 로직은 **이벤트 순서에 의존하면 안 됨**

### 모든 태스크가 Broadcast State를 체크포인트한다

- 체크포인트 시 모든 태스크가 자신의 broadcast state를 저장
- 장점: 복구 시 한 파일로 몰리는 **핫스팟** 방지
- 단점: 체크포인트 크기가 **병렬도 p만큼 증가**
- 복구/리스케일 시 Flink가 **중복/누락 없이** 재분배되도록 보장
    - 같은/작은 병렬도: 각 태스크가 자기 state를 읽음
    - 스케일 업: 추가 태스크가 기존 태스크들의 체크포인트를 라운드로빈으로 읽음

### RocksDB(및 유사) 백엔드에 올려두지 않는다

- Broadcast State는 런타임에 **인메모리로 유지**
- 따라서 메모리 프로비저닝(사이즈/증가율)을 충분히 고려해야 함
- 이는 일반적으로 operator state 전반에 해당하는 성질이기도 함
