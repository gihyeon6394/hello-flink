# Builtin Watermark Generators

### WatermarkGenerator 추상화와 내장 구현

- Flink는 이벤트 타임 처리를 위해 **타임스탬프 할당**과 **워터마크 발행**을 사용자 코드로 제어할 수 있으며, 이를 위해 `WatermarkGenerator`를 구현할 수 있다.
- 구현 부담을 줄이기 위해 Flink는 **미리 구현된(timestamp assigner/워터마크) 전략**을 제공한다.
- 내장 구현은 바로 사용 가능할 뿐 아니라, **커스텀 구현의 참고 예시**로도 유용하다.

## Monotonously Increasing Timestamps

### 단조 증가(오름차순) 타임스탬프

````
WatermarkStrategy.forMonotonousTimestamps();
````

- 가장 단순한 periodic 워터마크 케이스는, 특정 소스 태스크가 보는 이벤트 타임스탬프가 **항상 오름차순**으로 들어오는 경우다.
- 이 경우 **현재 이벤트 타임스탬프 자체가 워터마크**로 동작할 수 있다(더 과거 타임스탬프가 이후에 올 일이 없으므로).
- 조건은 “전체 스트림”이 아니라 **병렬 소스 태스크(파티션/스플릿) 단위로 오름차순이면 충분**하다.
    - 예: Kafka 파티션을 소스 병렬 인스턴스가 1:1로 읽는 구성이라면, **파티션 내부에서만** 타임스탬프가 오름차순이면 된다.
- Flink는 셔플/유니온/커넥트/머지 등으로 병렬 스트림이 합쳐질 때도 **워터마크 병합 메커니즘**으로 올바른 워터마크를 만든다.
- 사용 예:
    - `WatermarkStrategy.forMonotonousTimestamps()`

## Fixed Amount of Lateness

### 고정 지연(Out-of-Order) 허용

````
WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10));
````

- 또 다른 periodic 전략은, 워터마크가 스트림에서 관측된 **최대 이벤트 타임스탬프보다 일정 시간만큼 뒤처지도록** 만드는 방식이다.
- 스트림의 **최대 지연(최대 out-of-order 정도)** 을 미리 알고 있는 경우에 적합하다.
    - 예: 테스트용 커스텀 소스에서 타임스탬프가 일정 범위로만 퍼져 있는 경우 등
- Flink는 이를 위해 `BoundedOutOfOrdernessWatermarks`(전략 헬퍼로는 `forBoundedOutOfOrderness`)를 제공한다.
    - 인자 `maxOutOfOrderness`는 **허용 가능한 최대 지연**을 의미한다.
    - 지연(lateness)은 대략 `t_w - t`(직전 워터마크 시각 - 이벤트 시각)로 해석할 수 있으며,
        - lateness > 0 이면 이벤트는 “늦게 도착”한 것으로 간주된다.
    - 늦은 이벤트는 기본적으로 해당 윈도우의 최종 결과 계산에서 **무시**될 수 있으며, 필요하면 allowed lateness 설정으로 처리 방식을 조정한다.
- 사용 예:
    - `WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(10))`
