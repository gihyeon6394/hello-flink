# State TTL Migration Compatibility

### Flink 2.2.0부터의 변경점

- Apache Flink **2.2.0부터** 기존 상태(state)에 대해 **State TTL(Time-to-Live)** 을 *켜거나/끄는 작업*을 **복원(restore) 시점의 예외 없이(seamless)
  ** 처리할 수 있다.
- 과거에는 TTL 설정 변경이 직전 스냅샷/세이브포인트와 **직렬화 포맷 불일치**를 일으켜 `StateMigrationException`이 발생할 수 있었다.
- **2.2.0 이후** 주요 state backend 전반에서 TTL 마이그레이션 호환이 완성되었다.

## Version Overview

### 버전별 지원 범위

- **2.0.0**: TTL/Non-TTL 직렬화 호환을 위한 `TtlAwareSerializer` 도입
- **2.1.0**: `RocksDBKeyedStateBackend`에서 TTL 마이그레이션 지원 추가
- **2.2.0**: `HeapKeyedStateBackend`에서 TTL 마이그레이션 지원 추가
- 결론: **Flink 2.2.0+** 에서 주요 state backend 전반에 대해 TTL 상태 마이그레이션 호환 지원

## Motivation

### 기존 문제의 원인

- 과거에는 `StateDescriptor`에서 TTL을 켜거나 끄면 **TTL 상태와 Non-TTL 상태의 직렬화 포맷이 달라** 복원 시 호환성 오류가 발생했다.

## Compatibility Behavior

### 이제 가능한 복원 패턴

- **Non-TTL로 생성된 state**를 **TTL-enabled descriptor**로 **복원 가능**
- **TTL로 생성된 state**를 **TTL 비활성 descriptor**로 **복원 가능**
- 직렬화기(serializer)와 state backend가 TTL 메타데이터의 존재 유무를 **투명하게 처리**한다.

## Supported Migration Scenarios

### 지원 시나리오와 동작

| Migration Type                         | Available Since               | Behavior                                                                     |
|----------------------------------------|-------------------------------|------------------------------------------------------------------------------|
| Non-TTL state → TTL-enabled descriptor | 2.1.0 (RocksDB), 2.2.0 (Heap) | Previous state restored as non-expired. TTL applied on new updates/accesses. |
| TTL state → Non-TTL descriptor         | 2.1.0 (RocksDB), 2.2.0 (Heap) | TTL metadata is ignored. State becomes permanently visible.                  |

## Limitations

### 주의할 점(호환/적용 범위)

- TTL 파라미터 변경(예: 만료 시간, 업데이트 시점 갱신 정책 등)은 **항상 호환되지 않을 수** 있으며, 경우에 따라 **serializer 마이그레이션**이 필요하다.
- TTL은 **소급 적용되지 않는다**:
    - Non-TTL에서 복원된 기존 엔트리는 **다음 access/update 이후**에야 만료 대상이 된다.
- 이 호환성은 **그 외 직렬화기 비호환 변경이 없다는 전제**에서 성립한다.

## Example

### TTL 추가 적용 예

```
ValueStateDescriptor<String> descriptor = new ValueStateDescriptor<>("user-state", String.class);
descriptor.enableTimeToLive(StateTtlConfig.newBuilder(Time.hours(1)).build());
```

- 기존 TTL 없는 `ValueStateDescriptor`에 TTL을 추가해도(2.2.0+) 복원이 성공하며,
    - 복원된 기존 데이터는 유지되고
    - 이후부터 TTL 규칙이 적용된다.
