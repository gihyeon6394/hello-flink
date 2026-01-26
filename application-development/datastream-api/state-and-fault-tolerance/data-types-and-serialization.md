# Data Types & Serialization

### Flink의 타입/직렬화 특징

- Flink는 실행 계획을 최적화하기 위해 **데이터 타입을 분석**하며, 이를 위해 자체적인 **TypeInformation(타입 디스크립터)** , **제네릭 타입 추론**, **직렬화 프레임워크**를 제공한다.
- 타입 정보를 충분히 알수록 **더 효율적인 직렬화/데이터 레이아웃**을 선택할 수 있고, 사용자가 직렬화 프레임워크 등록에 덜 신경 쓰게 된다.
- 타입 정보는 보통 **execute()/print()/collect() 이전(프로그램 준비 단계)** 에 필요하다.

## 지원 데이터 타입

### 8가지 카테고리

- Java Tuples
- Java POJOs
- Primitive Types
- Common Collection Types
- Regular(General) Classes
- Values
- Hadoop Writables
- Special Types

## Tuples

- 고정 필드 수의 복합 타입(자바 API: `Tuple1` ~ `Tuple25`)
- 필드 타입은 임의의 Flink 타입 가능(중첩 튜플도 가능)
- 접근: `tuple.f4` 또는 `tuple.getField(pos)` (인덱스는 0부터)

```
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(value -> value.f0);
```

## POJOs

### POJO로 인식 조건

- 클래스는 `public`
- `public` 기본 생성자(무인자) 필요
- 필드가 `public` 이거나 `getFoo()/setFoo()` 형태의 getter/setter로 접근 가능
- 각 필드 타입이 **등록된 serializer로 직렬화 가능**해야 함

```java
public class WordWithCount {

    public String word;
    public int count;

    public WordWithCount() {
    }

    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}
```

```
DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy(value -> value.word);
```

### 장점과 직렬화

- Flink가 POJO 구조(필드)를 분석할 수 있어 **사용성/성능이 일반 타입보다 유리**
- 기본적으로 `PojoTypeInfo` + `PojoSerializer`(필요 시 Kryo fallback)
- Avro Specific Record/Reflect Type인 경우 `AvroTypeInfo` + `AvroSerializer`
- 테스트 유틸: `PojoTestUtils#assertSerializedAsPojo()`, Kryo fallback까지 막으려면 `assertSerializedAsPojoWithoutKryo()`

## Primitive Types

### 지원 범위

- `Integer`, `String`, `Double` 등 자바의 기본/박싱 타입을 지원

## Common Collection Types

### 전용(효율적) 직렬화 지원

- `Map`, `List`, `Set`, `Collection`에 대해 전용 직렬화 지원(메타데이터 분석/직렬화 비용을 줄임)
- 사용 조건
    - **구체적 타입 인자 필요**: `List<String>`는 OK, `List`, `List<T>`, `List<?>`는 NO
    - **인터페이스 타입으로 선언 권장**: `List<String>`는 OK, `LinkedList<String>`처럼 구현체 타입 선언은 NO
- 그 외 컬렉션/구현체 보존이 필요하면 일반 타입으로 처리되거나 **커스텀 serializer 등록**이 필요

## General Class Types

### 특징

- POJO로 인식되지 않는 대부분의 클래스는 **일반 타입(black box)** 으로 취급
- 내부 필드에 접근 불가(예: 효율적 정렬/키 처리에 불리)
- 기본적으로 **Kryo로 직렬화**
- 파일 핸들/스트림 등 직렬화 불가능한 리소스를 담은 타입은 제약이 있음

## Values

### 수동 직렬화 타입

- `org.apache.flink.types.Value`를 구현해 `read/write`로 직접 직렬화/역직렬화
- 일반 직렬화가 비효율적인 구조(예: 희소 벡터)에 유리
- `CopyableValue`는 내부 복제 로직도 제공
- 기본 제공 Value 타입(뮤터블): `IntValue`, `LongValue`, `StringValue` 등 → 객체 재사용으로 GC 부담 감소

## Hadoop Writables

### Hadoop Writable 지원

- `org.apache.hadoop.Writable` 구현 타입 사용 가능
- `write()/readFields()` 로 정의된 직렬화 로직을 사용

## Special Types

### Either

- 자바 API에 `Either(Left/Right)` 제공(오류 처리, 서로 다른 타입 출력 등에 유용)

## Type Erasure & Type Inference (Java 전용)

### 배경

- Java는 컴파일 후 제네릭 정보가 지워짐(type erasure) → 런타임에 `DataStream<String>`과 `DataStream<Long>`가 구분되지 않음
- Flink는 실행 준비 시점에 타입 정보가 필요하여, 리플렉션/시그니처 기반으로 타입을 복원하고 `TypeInformation`으로 보관
- 확인: `DataStream.getType()`

### 개발자가 도와야 하는 경우

- `fromCollection()` 같은 경우 타입 명시가 필요할 수 있음
- 제네릭 함수(`MapFunction<I,O>` 등)는 추가 타입 정보가 필요할 수 있음
- `ResultTypeQueryable`로 반환 타입을 명시 가능

## 타입 처리로 가능한 최적화

### 효과

- 타입을 많이 알수록 **직렬화 비용 감소**, **메모리/데이터 레이아웃 최적화**에 유리
- 대부분은 자동 처리되지만, 일부는 사용자가 개입해야 함

## 자주 겪는 이슈/해결

### 서브타입 등록(성능)

- 시그니처는 상위 타입인데 실행 시 하위 타입을 쓰면, 하위 타입을 등록해 성능 개선 가능
- 설정: `pipeline.serialization-config`로 serializer 등록(POJO/Kryo 등)

```yml
pipeline.serialization-config:
  # register serializer for POJO types
  - org.example.MyCustomType1: { type: pojo, class: org.example.MyCustomSerializer1 }
  # register serializer for generic types with Kryo
  - org.example.MyCustomType2: { type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer2 }
```

```
Configuration config = new Configuration();
config.set(PipelineOptions.SERIALIZATION_CONFIG,
    List.of("org.example.MyCustomType1: {type: pojo, class: org.example.MyCustomSerializer1}",
        "org.example.MyCustomType2: {type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer2}"));
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(config);

```

### 커스텀 serializer 등록(호환성)

- Kryo fallback이 모든 타입을 잘 처리하지 못함(예: 일부 Guava 컬렉션)
- `pipeline.serialization-config`로 추가 Kryo serializer 등록

```yaml
pipeline.serialization-config:
  - org.example.MyCustomType: { type: kryo, kryo-type: default, class: org.example.MyCustomSerializer }
```

### 타입 힌트 추가(Java)

- 추론 실패 시 `.returns(SomeType.class)` 또는 `.returns(new TypeHint<...>(){})`로 명시

## TypeInformation

### TypeInformation의 역할

- 타입의 기본 성질을 노출하고 serializer(및 일부 타입은 comparator) 생성
- 내부적으로 타입을 크게 구분
    - Basic types(프리미티브/박싱 + String/Date/BigDecimal/BigInteger 등)
    - 배열(primitive/object)
    - Composite types(Tuple, Row, POJO, 컬렉션 등)
    - Generic types(Kryo 처리)

## POJO 타입 규칙(“by-name” 필드 참조 가능 조건)

### 추가 규칙

- `public` + standalone(비정적 inner class 불가)
- `public` 무인자 생성자
- (상속 포함) 모든 non-static/non-transient 필드가
    - `public`(non-final) 이거나
    - public getter/setter(자바빈 규칙) 보유
- POJO로 못 알아보면 GenericType으로 처리되어 Kryo 사용

## TypeInformation / TypeSerializer 생성

### TypeInformation 생성

- 비제네릭: `TypeInformation.of(String.class)`
- 제네릭: `TypeInformation.of(new TypeHint<Tuple2<String, Double>>() {})`

````
TypeInformation<String> info = TypeInformation.of(String.class);
TypeInformation<Tuple2<String, Double>> info = TypeInformation.of(new TypeHint<Tuple2<String, Double>>(){});
````

### TypeSerializer 생성

- `typeInfo.createSerializer(serializerConfig)`
- 또는 RichFunction에서 `getRuntimeContext().createSerializer(typeInfo)`

## 람다(Java 8) 타입 추론 주의

```java
public class AppendOne<T> implements MapFunction<T, Tuple2<T, Long>> {

    public Tuple2<T, Long> map(T value) {
        return new Tuple2<T, Long>(value, 1L);
    }
}
```

### 주의점

- 람다는 구현 클래스가 명시되지 않아 추론이 더 까다로움
- 컴파일러에 따라 시그니처 정보가 불완전할 수 있어, 이상 동작 시 `.returns(...)`로 반환 타입을 명시

```java
DataStream<SomeType> result = stream
        .map(new MyGenericNonInferrableFunction<Long, SomeType>())
        .returns(SomeType.class);
```

## POJO 직렬화 고급 옵션

### Kryo/Avro 강제

- POJO 필드 중 Flink 기본 serializer가 없으면 Kryo fallback
- Avro 강제: `pipeline.force-avro: true` (flink-avro 모듈 필요)
- Kryo 강제: `pipeline.force-kryo: true`
- Kryo fallback 자체를 금지: `pipeline.generic-types: false` (Kryo가 필요한 타입 등장 시 예외)

```yaml
pipeline.force-avro: true
pipeline.force-kryo: true
pipeline.serialization-config:
  - org.example.MyCustomType: { type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer }
pipeline.generic-types: false # disable Kryo fallback
```

## TypeInfoFactory로 타입 시스템 확장

- `TypeInfoFactory`를 구현해 사용자 정의 타입 정보를 플러그인 가능
- 연결 방법
    - 설정: `pipeline.serialization-config`에서 `{type: typeinfo, class: ...}`
    - 또는 `@TypeInfo(...)` 애노테이션(설정이 더 높은 우선순위)
- 제네릭 파라미터 매핑이 필요하면 `TypeInformation#getGenericParameters` 구현 고려

```yaml
pipeline.serialization-config:
  - org.example.MyCustomType: { type: typeinfo, class: org.example.MyCustomTypeInfoFactory }
```

````java

@TypeInfo(MyTupleTypeInfoFactory.class)
public class MyTuple<T0, T1> {
    public T0 myfield0;
    public T1 myfield1;
}

public class MyPojo {
    public int id;

    @TypeInfo(MyTupleTypeInfoFactory.class)
    public MyTuple<Integer, String> tuple;
}

````

# State Schema Evolution

- Flink 스트리밍 애플리케이션은 장기간(또는 무기한) 실행되는 경우가 많아, 요구사항 변화에 맞춰 애플리케이션과 함께 **데이터 스키마도 진화**한다.
- 이 문서는 **state 타입의 데이터 스키마를 변경(진화)하는 방법**을 개괄하며, 제약은 상태 구조(ValueState, ListState 등)와 타입에 따라 달라질 수 있다.
- 본 내용은 **Flink 자체 타입 직렬화 프레임워크가 생성한 state serializer**를 사용하는 경우에만 해당한다.
    - 즉, StateDescriptor에 특정 `TypeSerializer`/`TypeInformation`을 직접 지정하지 않고 Flink가 타입을 추론해 serializer를 만드는 경우.

```
ListStateDescriptor<MyPojoType> descriptor =
    new ListStateDescriptor<>(
        "state-name",
        MyPojoType.class);

checkpointedState = getRuntimeContext().getListState(descriptor);
```

## 스키마 진화 가능 여부의 핵심

- state 스키마를 바꿀 수 있는지는, 저장된 state 바이트를 읽고/쓰는 **serializer가 스키마 진화를 지원하는지**에 달려 있다.
- Flink 타입 프레임워크가 생성한 serializer는(지원 범위 내에서) 이를 **투명하게 처리**한다.
- 커스텀 `TypeSerializer`를 구현해 스키마 진화를 지원하려면 **Custom State Serialization** 및 state serializer–state backend의 내부 동작(상호작용)을
  참고해야 한다.

## 스키마 진화 절차

1. 실행 중인 Flink 잡의 **savepoint 생성**
2. 애플리케이션에서 **state 타입/스키마 변경**(예: Avro 스키마 수정)
3. savepoint로부터 **잡 복구(restore)**
    - state에 **처음 접근할 때**, Flink가 스키마 변경 여부를 판단하고 필요 시 마이그레이션 수행

- 마이그레이션은 state별로 독립적으로 자동 진행된다.
    - 새 serializer의 스키마가 이전과 다르면
    - 이전 serializer로 state를 객체로 읽고
    - 새 serializer로 다시 바이트로 써서 변환한다.

## 스키마 진화 지원 데이터 타입

- 현재 스키마 진화는 **POJO, Avro 타입만 지원**된다.
- state 스키마 진화가 중요하다면 state 데이터 타입은 **Pojo 또는 Avro 사용을 권장**한다.
- 향후 더 많은 composite 타입으로 확장 계획이 있으며(FLINK-10896 참고) 지원 범위가 넓어질 수 있다.

## POJO 타입 스키마 진화 규칙

- 필드 삭제 가능: 삭제된 필드의 기존 값은 이후 체크포인트/세이브포인트에서 **버려진다**.
- 새 필드 추가 가능: 새 필드는 Java가 정의한 해당 타입의 **기본값(default)** 으로 초기화된다.
- 선언된 필드의 **타입 변경 불가**
- POJO의 **클래스명/패키지(네임스페이스) 변경 불가**
- POJO state 스키마 진화는 **Flink 1.8.0 초과 버전으로 savepoint 복구할 때만** 가능하며, 1.8.0 이하로 복구하면 스키마 변경이 불가하다.

## Avro 타입 스키마 진화 규칙

- Avro의 스키마 해석(해상) 규칙에서 **호환(compatible)** 으로 간주되는 변경이라면 Flink가 스키마 진화를 완전 지원한다.
- 제한: state 타입으로 쓰는 **Avro generated class는 복구 시 리로케이션(이동)되거나 네임스페이스가 달라지면 안 된다**.

## 스키마 마이그레이션 제한사항

- 키(key)의 스키마 진화는 지원되지 않음
    - 키 구조 변경은 비결정적 동작을 유발할 수 있다(예: POJO 키에서 필드 제거 시 서로 다른 키가 동일해질 수 있는데, Flink는 해당 값들을 병합할 방법이 없음).
    - 또한 RocksDB state backend는 `hashCode`가 아니라 **바이너리 객체 동일성**에 의존하므로, 키 객체 구조 변경은 비결정성을 초래할 수 있다.
- Kryo는 스키마 진화에 사용할 수 없음
    - Kryo 사용 시 비호환 변경 여부를 프레임워크가 검증할 수 없다.
    - 어떤 타입이 Kryo로 직렬화되는 자료구조 내부에 포함되면, 그 포함된 타입은 스키마 진화를 할 수 없다.
    - 예: POJO 안에 `List<SomeOtherPojo>`가 있고 List 및 원소가 Kryo로 직렬화되면 `SomeOtherPojo`의 스키마 진화는 지원되지 않는다.

# Custom Serialization for Managed State

- Flink 상태(state)에 **커스텀 직렬화**가 필요한 사용자를 위한 가이드
- 커스텀 serializer 제공 방법과, **스키마 진화(state schema evolution)** 를 가능하게 하는 구현 지침/베스트 프랙티스를 다룬다
- Flink 기본 serializer만 사용한다면 이 문서는 사실상 무시해도 된다

## 커스텀 state serializer 사용하기

- 관리되는 keyed/operator state 등록 시 `StateDescriptor`로 **state 이름**과 **타입 정보**를 지정하며, 보통 Flink가 이를 바탕으로 serializer를 생성한다
- 하지만 `StateDescriptor`에 **직접 TypeSerializer를 주입**해 Flink 타입 프레임워크를 우회할 수 있다
    - 예: `new ListStateDescriptor<>("state-name", new CustomTypeSerializer())`

```java
public class CustomTypeSerializer extends TypeSerializer<Tuple2<String, Integer>> {/*...*/
}

;

ListStateDescriptor<Tuple2<String, Integer>> descriptor =
        new ListStateDescriptor<>(
                "state-name",
                new CustomTypeSerializer());

checkpointedState =

getRuntimeContext().

getListState(descriptor);

```

## State serializer와 스키마 진화

- savepoint에서 복구할 때, Flink는 기존 state를 읽고/쓰기 위한 **serializer를 교체**할 수 있게 해 “특정 바이너리 포맷에 락인”되는 것을 방지한다
- 복구된 잡에서 state를 다시 접근할 때, 새 `StateDescriptor`가 제공하는 **새 serializer(스키마 B)** 가 등록될 수 있고, 이전에는 **옛 serializer(스키마 A)** 로
  저장돼 있을 수 있다
- 여기서 말하는 “스키마”는
    - state 타입의 데이터 모델(예: POJO 필드 추가/삭제)
    - 또는 그에 대응하는 직렬화 바이너리 포맷
    - 둘을 포괄한다
- 스키마가 바뀌는 대표 원인
    - state 타입의 데이터 스키마 변경(POJO 필드 추가/삭제 등) → 직렬화 포맷 업그레이드 필요
    - serializer의 설정(configuration) 변경

## TypeSerializerSnapshot 추상화

```java
public interface TypeSerializerSnapshot<T> {
    int getCurrentVersion();

    void writeSnapshot(DataOuputView out) throws IOException;

    void readSnapshot(int readVersion, DataInputView in, ClassLoader userCodeClassLoader) throws IOException;

    TypeSerializerSchemaCompatibility<T> resolveSchemaCompatibility(TypeSerializerSnapshot<T> oldSerializerSnapshot);

    TypeSerializer<T> restoreSerializer();
}

public abstract class TypeSerializer<T> {

    // ...

    public abstract TypeSerializerSnapshot<T> snapshotConfiguration();
}

```

- savepoint에는 **state 바이트 + serializer 스냅샷**을 함께 기록해야, 다음 실행에서 “기록된 스키마”를 알고 변경 여부를 감지할 수 있다
- 이를 표현하는 핵심 추상화가 `TypeSerializerSnapshot`
    - 스냅샷은 특정 시점의 “쓰기 스키마의 단일 진실(source of truth)”이며, 동일한 serializer를 복원하는 데 필요한 정보도 담는다
- 스냅샷은 **버전 관리**가 된다
    - 스냅샷 포맷이 바뀌면 버전을 올리고(`getCurrentVersion`)
    - 복구 시 `readSnapshot(readVersion, ...)`에서 과거 버전도 읽을 수 있게 처리한다

### 호환성 판정: resolveSchemaCompatibility

- 복구 시, 새 serializer 스냅샷이 옛 serializer 스냅샷을 입력으로 받아 호환성을 판단한다
- 결과는 아래 중 하나
    - `compatibleAsIs()`
        - 스키마가 동일(필요하면 내부 재설정으로 호환되게 만들 수도 있음)
    - `compatibleAfterMigration()`
        - 스키마가 다르지만 마이그레이션 가능
        - 옛 serializer로 바이트→객체로 읽고, 새 serializer로 객체→바이트로 다시 써서 변환
    - `incompatible()`
        - 스키마가 다르고 마이그레이션 불가 → state 접근 시 예외

### 이전 serializer 복원: restoreSerializer

- 마이그레이션이 필요한 경우, “이전 스키마를 이해하는 serializer”가 필요하다
- `TypeSerializerSnapshot#restoreSerializer()`가 그 역할(이전 스키마/설정을 인식하는 serializer 인스턴스 생성)

## Flink(State Backend)가 이 추상화들과 상호작용하는 방식

### Off-heap 백엔드(예: EmbeddedRocksDBStateBackend)

- 실행 중에는 등록된 serializer로 상태를 계속 읽고/쓰며, state 바이트는 백엔드에 **직렬화된 형태로 유지**
- savepoint 시점에
    - state 바이트(스키마 A) + serializer 스냅샷을 함께 저장
- 복구 시
    - state 바이트는 즉시 객체로 풀지 않고(여전히 스키마 A), 백엔드에 로드
    - 새 serializer(스키마 B) 등록 시 호환성 체크
    - `compatibleAfterMigration()`면 백엔드 내 모든 엔트리를 한꺼번에 A→B로 마이그레이션 후 처리 재개
    - `incompatible()`면 접근 실패(예외)

### Heap 백엔드(예: HashMapStateBackend)

- savepoint 시점에 state 객체를 **스키마 A로 직렬화**해 저장하고, serializer 스냅샷도 저장
- 복구 시
    - 옛 스냅샷으로 옛 serializer를 복원해(A 인식) 바이트→객체로 역직렬화하여 힙에 적재
    - 이후 새 serializer(스키마 B)로 다시 접근할 때 호환성 체크는 하되,
        - 힙 백엔드는 이미 객체 상태라 `compatibleAfterMigration()`여도 즉시 변환 작업은 없음
    - 다음 savepoint부터는 스키마 B로 직렬화되어 저장

## 미리 제공되는 스냅샷 베이스 클래스

- Flink는 대표 시나리오용 베이스 구현을 제공
    - `SimpleTypeSerializerSnapshot`
    - `CompositeTypeSerializerSnapshot`
- 베스트 프랙티스: **serializer마다 독립적인 snapshot 서브클래스**를 두는 것이 권장(공유 지양)

### SimpleTypeSerializerSnapshot

- serializer가 상태/설정이 없고, “클래스 자체가 스키마”인 경우
- 호환성 결과는 사실상 둘뿐
    - serializer 클래스 동일 → `compatibleAsIs()`
    - 다름 → `incompatible()`
- 스냅샷은 serializer 인스턴스 공급자(Supplier)를 받아, 복원 serializer 생성 및 타입 체크에 활용

```java
public class IntSerializerSnapshot extends SimpleTypeSerializerSnapshot<Integer> {
    public IntSerializerSnapshot() {
        super(() -> IntSerializer.INSTANCE);
    }
}

```

### CompositeTypeSerializerSnapshot

- 여러 “내부(nested) serializer”에 의존하는 composite serializer(예: Map/List/배열 등)에 적합
- outer serializer 스냅샷이 nested 스냅샷들을 포함해
    - nested별 호환성 판단
    - 전체 결과를 종합한다
- 기본 구현은 “outer 추가 정보 없음”을 가정하지만,
    - outer의 정적 설정(예: 배열 원소 클래스 등)을 스냅샷에 저장해야 하면
    - outer 스냅샷 read/write 및 outer 호환성 판정을 추가로 구현해야 한다
- 스냅샷에 클래스 정보를 저장할 때는 Java 직렬화 대신
    - **클래스명 문자열을 저장하고**
    - 복구 시 클래스 로딩으로 복원하는 방식이 권장된다

```java
public class MapSerializerSnapshot<K, V> extends CompositeTypeSerializerSnapshot<Map<K, V>, MapSerializer> {

    private static final int CURRENT_VERSION = 1;

    public MapSerializerSnapshot() {
        super(MapSerializer.class);
    }

    public MapSerializerSnapshot(MapSerializer<K, V> mapSerializer) {
        super(mapSerializer);
    }

    @Override
    public int getCurrentOuterSnapshotVersion() {
        return CURRENT_VERSION;
    }

    @Override
    protected MapSerializer createOuterSerializerWithNestedSerializers(TypeSerializer<?>[] nestedSerializers) {
        TypeSerializer<K> keySerializer = (TypeSerializer<K>) nestedSerializers[0];
        TypeSerializer<V> valueSerializer = (TypeSerializer<V>) nestedSerializers[1];
        return new MapSerializer<>(keySerializer, valueSerializer);
    }

    @Override
    protected TypeSerializer<?>[] getNestedSerializers(MapSerializer outerSerializer) {
        return new TypeSerializer<?>[]{outerSerializer.getKeySerializer(), outerSerializer.getValueSerializer()};
    }
}

public final class GenericArraySerializerSnapshot<C> extends CompositeTypeSerializerSnapshot<C[], GenericArraySerializer> {

    private static final int CURRENT_VERSION = 1;

    private Class<C> componentClass;

    public GenericArraySerializerSnapshot() {
        super(GenericArraySerializer.class);
    }

    public GenericArraySerializerSnapshot(GenericArraySerializer<C> genericArraySerializer) {
        super(genericArraySerializer);
        this.componentClass = genericArraySerializer.getComponentClass();
    }

    @Override
    protected int getCurrentOuterSnapshotVersion() {
        return CURRENT_VERSION;
    }

    @Override
    protected void writeOuterSnapshot(DataOutputView out) throws IOException {
        out.writeUTF(componentClass.getName());
    }

    @Override
    protected void readOuterSnapshot(int readOuterSnapshotVersion, DataInputView in, ClassLoader userCodeClassLoader) throws IOException {
        this.componentClass = InstantiationUtil.resolveClassByName(in, userCodeClassLoader);
    }

    @Override
    protected OuterSchemaCompatibility resolveOuterSchemaCompatibility(
            TypeSerializerSnapshot<C[]> oldSerializerSnapshot) {
        GenericArraySerializerSnapshot<C[]> oldGenericArraySerializerSnapshot =
                (GenericArraySerializerSnapshot<C[]>) oldSerializerSnapshot;
        return (this.componentClass == oldGenericArraySerializerSnapshot.componentClass)
                ? OuterSchemaCompatibility.COMPATIBLE_AS_IS
                : OuterSchemaCompatibility.INCOMPATIBLE;
    }

    @Override
    protected GenericArraySerializer createOuterSerializerWithNestedSerializers(TypeSerializer<?>[] nestedSerializers) {
        TypeSerializer<C> componentSerializer = (TypeSerializer<C>) nestedSerializers[0];
        return new GenericArraySerializer<>(componentClass, componentSerializer);
    }

    @Override
    protected TypeSerializer<?>[] getNestedSerializers(GenericArraySerializer outerSerializer) {
        return new TypeSerializer<?>[]{outerSerializer.getComponentSerializer()};
    }
}

```

## 구현 노트 & 베스트 프랙티스

### 1) 스냅샷은 classname으로 인스턴스화된다

- Flink는 savepoint에 기록된 classname으로 스냅샷을 생성하므로
    - 익명/중첩 클래스 사용을 피하고
    - public 기본 생성자(무인자)를 제공하는 것이 안전하다

### 2) 서로 다른 serializer가 같은 스냅샷 클래스를 공유하지 말 것

- 호환성 판정/이전 serializer 복원 로직이 복잡해지고 책임 분리가 깨진다
- “한 serializer의 스키마/설정/복원”은 그 serializer 전용 스냅샷에 캡슐화하는 게 바람직하다

### 3) 스냅샷 내용에 Java 직렬화를 사용하지 말 것

- Java 직렬화는 클래스의 바이너리 호환성 변화에 취약해, 스냅샷이 읽히지 않을 위험이 커진다
- 필요한 정보는 가능한 “문자열/원시값 등 안정적인 형태”로 저장하고, 복구 시 재구성한다

## 구버전 API 마이그레이션 가이드

### Flink 1.7 이전(Deprecated TypeSerializerConfigSnapshot)

- 과거에는 `TypeSerializerConfigSnapshot` + `TypeSerializer#ensureCompatibility(...)`로 관리되었고
- 이전 스냅샷은 “이전 serializer 인스턴스를 만들 수 없어”
    - savepoint에 serializer 인스턴스를 Java 직렬화로 함께 저장하는 방식이 필요해졌으며(비권장)
    - 결과적으로 serializer 업그레이드/스키마 마이그레이션 유연성이 떨어진다
- 권장 마이그레이션 절차
    - 새 `TypeSerializerSnapshot` 구현
    - `snapshotConfiguration()`에서 새 스냅샷 반환
    - 1.7 이전 savepoint로 복구 후 savepoint를 다시 떠서(구 스냅샷을 새 스냅샷으로 교체)
    - 그 다음부터 구 API 구현(구 스냅샷/ensureCompatibility)을 제거

### Flink 1.19 이전 호환성 판정 방향 변경

- 1.19 이전에는 “옛 스냅샷이 새 serializer를 받아” 판정하는 방식이 있어, 새 serializer 쪽에서 옛 serializer와의 호환을 선언하기 어려웠다
- 1.19부터는 방향이 반대로 바뀌어
    - `resolveSchemaCompatibility(TypeSerializerSnapshot oldSerializerSnapshot)`를 구현해야 한다
- 마이그레이션
    - 새 시그니처 메서드에 기존 로직을 옮기고
    - 예전 메서드(`resolveSchemaCompatibility(TypeSerializer newSerializer)`)는 제거

# 3rd Party Serializers

## Kryo 폴백과 외부(서드파티) 직렬화 연동

- Flink의 타입 직렬화기가 처리하지 못하는 커스텀 타입을 쓰면, Flink는 **Generic Kryo serializer로 폴백**한다
- 이때 Kryo에 **직접 커스텀 serializer**를 등록하거나, **Google Protobuf / Apache Thrift** 같은 직렬화 시스템을 Kryo와 함께 사용할 수 있다
- 등록 방법은 `pipeline.serialization-config`에 “타입 클래스 ↔ serializer 클래스” 매핑을 추가하는 것

```yaml
pipeline.serialization-config:
  - org.example.MyCustomType: { type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer }

```

```java
Configuration config = new Configuration();

// register the class of the serializer as serializer for a type
config.set(PipelineOptions.SERIALIZATION_CONFIG,
    List.of("org.example.MyCustomType: {type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer}"));

StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(config);
```

## serializer 등록 방법

### 설정 파일로 등록

- `pipeline.serialization-config`에 아래 형태로 추가
    - `org.example.MyCustomType: {type: kryo, kryo-type: registered, class: org.example.MyCustomSerializer}`

```yaml
pipeline.serialization-config:
  # register the Google Protobuf serializer with Kryo
  - org.example.MyCustomProtobufType: { type: kryo, kryo-type: registered, class: com.twitter.chill.protobuf.ProtobufSerializer }
  # register the serializer included with Apache Thrift as the standard serializer
  # TBaseSerializer states it should be initialized as a default Kryo serializer
  - org.example.MyCustomThriftType: { type: kryo, kryo-type: default, class: com.twitter.chill.thrift.TBaseSerializer }

```

### 코드로 등록

- `PipelineOptions.SERIALIZATION_CONFIG`에 동일한 매핑 문자열을 넣어 `StreamExecutionEnvironment`를 생성
- 등록되는 커스텀 serializer는 **Kryo의 `Serializer`를 상속**해야 한다
    - Protobuf/Thrift의 경우, 보통 이미 Kryo용 serializer 구현이 제공된다

## Protobuf / Thrift 예시

- Protobuf: `com.twitter.chill.protobuf.ProtobufSerializer`를 Kryo에 등록(보통 registered로 사용)
- Thrift: `com.twitter.chill.thrift.TBaseSerializer`를 Kryo에 등록(예시에서는 default serializer로 초기화 권장)


```xml

<dependency>
    <groupId>com.twitter</groupId>
    <artifactId>chill-thrift</artifactId>
    <version>0.7.6</version>
    <!-- exclusions for dependency conversion -->
    <exclusions>
        <exclusion>
            <groupId>com.esotericsoftware.kryo</groupId>
            <artifactId>kryo</artifactId>
        </exclusion>
    </exclusions>
</dependency>
        <!-- libthrift is required by chill-thrift -->
<dependency>
  <groupId>org.apache.thrift</groupId>
  <artifactId>libthrift</artifactId>
  <version>0.11.0</version>
  <exclusions>
      <exclusion>
          <groupId>javax.servlet</groupId>
          <artifactId>servlet-api</artifactId>
      </exclusion>
      <exclusion>
          <groupId>org.apache.httpcomponents</groupId>
          <artifactId>httpclient</artifactId>
      </exclusion>
  </exclusions>
</dependency>
```

```xml

<dependency>
	<groupId>com.twitter</groupId>
	<artifactId>chill-protobuf</artifactId>
	<version>0.7.6</version>
	<!-- exclusions for dependency conversion -->
	<exclusions>
		<exclusion>
			<groupId>com.esotericsoftware.kryo</groupId>
			<artifactId>kryo</artifactId>
		</exclusion>
	</exclusions>
</dependency>
<!-- We need protobuf for chill-protobuf -->
<dependency>
	<groupId>com.google.protobuf</groupId>
	<artifactId>protobuf-java</artifactId>
	<version>3.7.0</version>
</dependency>
```

## 의존성(Maven) 주의

- 위 예시를 쓰려면 `pom.xml`에 필요한 라이브러리를 추가해야 한다
    - Thrift: `chill-thrift` + `libthrift`
    - Protobuf: `chill-protobuf` + `protobuf-java`
- 문서의 버전은 예시이므로, 프로젝트 환경에 맞게 **버전 조정**이 필요하다
- `chill-*` 의존성에는 Kryo 충돌을 피하기 위한 **kryo exclusion** 예시가 포함되어 있다

## Kryo JavaSerializer 사용 시 이슈

- Kryo의 `JavaSerializer`를 커스텀 타입에 등록하면, 사용자 코드 JAR에 클래스가 있어도 **ClassNotFoundException**이 발생할 수 있다
    - 원인: `JavaSerializer`가 **잘못된 classloader**를 사용하는 알려진 이슈
- 해결: `org.apache.flink.api.java.typeutils.runtime.kryo.JavaSerializer`(Flink 재구현판)를 사용
    - 사용자 코드 classloader를 확실히 사용하도록 보장한다
- 관련 이슈: **FLINK-6025**
