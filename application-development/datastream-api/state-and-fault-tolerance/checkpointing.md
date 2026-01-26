# Checkpointing

### 왜 필요한가

- Flink의 함수/오퍼레이터는 **stateful**할 수 있으며, 상태(state)는 이벤트 처리 사이에 데이터를 유지해 복잡한 연산의 핵심이 된다.
- 장애 복구를 위해 Flink는 **상태를 체크포인트(checkpoint)** 로 저장한다.
- 체크포인트는 **상태 + 스트림에서의 처리 위치**를 함께 보존해, 장애가 나도 “장애가 없었던 것과 같은 의미론”으로 복구하게 해준다.

## Prerequisites

### 내구성(durable) 요구사항

- 체크포인트는 내구성 저장소와 상호작용하므로 일반적으로 다음이 필요:
    - **리플레이 가능한 영속 데이터 소스**: 일정 기간 레코드 재전송 가능(예: Kafka, RabbitMQ, Kinesis, PubSub, 파일시스템 등)
    - **상태 저장을 위한 영속 스토리지**: 보통 분산 파일시스템(HDFS, S3, NFS, Ceph 등)

## Enabling and Configuring Checkpointing

### 활성화

- 기본값은 **비활성화**
- `StreamExecutionEnvironment.enableCheckpointing(n)` 호출로 활성화
    - `n`: 체크포인트 간격(ms)

### 주요 설정 옵션

```
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// start a checkpoint every 1000 ms
env.enableCheckpointing(1000);

// advanced options:

// set mode to exactly-once (this is the default)
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// make sure 500 ms of progress happen between checkpoints
env.getCheckpointConfig().setMinPauseBetweenCheckpoints(500);

// checkpoints have to complete within one minute, or are discarded
env.getCheckpointConfig().setCheckpointTimeout(60000);

// only two consecutive checkpoint failures are tolerated
env.getCheckpointConfig().setTolerableCheckpointFailureNumber(2);

// allow only one checkpoint to be in progress at the same time
env.getCheckpointConfig().setMaxConcurrentCheckpoints(1);

// enable externalized checkpoints which are retained
// after job cancellation
env.getCheckpointConfig().setExternalizedCheckpointRetention(
    ExternalizedCheckpointRetention.RETAIN_ON_CANCELLATION);

// enables the unaligned checkpoints
env.getCheckpointConfig().enableUnalignedCheckpoints();

// sets the checkpoint storage where checkpoint snapshots will be written
Configuration config = new Configuration();
config.set(CheckpointingOptions.CHECKPOINT_STORAGE, "filesystem");
config.set(CheckpointingOptions.CHECKPOINTS_DIRECTORY, "hdfs:///my/checkpoint/dir");
env.configure(config);

// enable checkpointing with finished tasks
Configuration config = new Configuration();
config.set(CheckpointingOptions.ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH, true);
env.configure(config);
```

### checkpoint storage(저장 위치)

- 체크포인트 스냅샷을 **어디에 영속화할지** 설정
- 기본: JobManager 힙(메모리)
- 운영 환경 권장: **내구성 파일시스템** 사용(대규모 상태/장애 대비)

### exactly-once vs at-least-once

- `enableCheckpointing(n, mode)`로 보장 수준 선택 가능
- 일반적으로 **Exactly-once** 권장
- **At-least-once**는 “항상 수 ms 수준의 초저지연” 같은 특수한 경우에 고려

### checkpoint timeout

- 진행 중 체크포인트가 특정 시간 내 완료되지 않으면 **중단/실패 처리**

### minimum time between checkpoints(min pause)

- 체크포인트 사이에 최소 “휴지”를 둬서 처리 진행을 보장
- 예: 5000ms면 이전 체크포인트 완료 후 최소 5초 뒤에 다음 체크포인트 시작
- 효과:
    - 실제 체크포인트 주기는 이 값보다 작아질 수 없음
    - **동시 체크포인트 수가 1개**로 사실상 제한됨
- 실무 팁:
    - “interval”보다 “min pause” 기반이 간단한 경우가 많음(스토리지 지연 변동에 덜 민감)

### tolerable checkpoint failure number

- 연속 체크포인트 실패를 **몇 번까지 허용**할지
- 기본 0: 첫 실패에 잡 failover
- 일부 실패 원인(IO, async phase 실패, timeout 만료 등)에만 적용되며,
    - TaskManager의 sync phase 실패는 항상 failover 유발

### number of concurrent checkpoints

- 기본: 한 번에 1개(진행 중이면 다음 시작 안 함)
- 여러 개를 겹치게 하면:
    - 외부 호출 등으로 파이프라인 지연이 있어도 **짧은 간격(수백 ms)** 체크포인트가 가능
- 제약:
    - **min pause 설정과는 함께 사용할 수 없음**

### externalized checkpoints

- 주기 체크포인트를 외부에 보존(메타데이터를 영속 스토리지에 기록)
- 잡 실패 시 자동 삭제되지 않아, **실패 후 재시작 포인트**로 활용 가능
- 취소 시 삭제/보존 정책 선택(RETENTION 옵션)

### unaligned checkpoints

- 백프레셔 상황에서 체크포인트 시간을 크게 줄이기 위한 옵션
- 제약:
    - **Exactly-once**에서만
    - **동시 체크포인트 1개**일 때만

### checkpoints with finished tasks

- DAG 일부가 종료(예: bounded source 포함)되어도 체크포인트를 계속 수행 가능
- 운영 시 커스텀 오퍼레이터/UDF는 “finish 이후 체크포인트는 보통 비어야 한다”는 전제에 맞게 구현 필요

## Related Config Options

https://nightlies.apache.org/flink/flink-docs-release-2.2/docs/dev/datastream/fault-tolerance/checkpointing/#related-config-options

## Selecting Checkpoint Storage

### 저장소 선택의 의미

- 체크포인트는 타이머/커넥터/윈도우/사용자 상태 등 **모든 상태의 일관된 스냅샷**을 저장
- 기본은 JobManager 메모리지만,
- 운영 환경에서는 **고가용 파일시스템에 저장**하는 것이 강력 권장

```
Configuration config = new Configuration();
config.set(CheckpointingOptions.CHECKPOINT_STORAGE, "filesystem");
config.set(CheckpointingOptions.CHECKPOINTS_DIRECTORY, "...");
env.configure(config);
```

## State Checkpoints in Iterative Jobs

### 반복(iteration) 잡 주의

- Flink는 현재 **iteration 없는 잡**에서만 처리 보장을 제공
- iterative job에서 checkpointing 활성화는 예외를 유발
- 강제로 켤 수는 있으나, **루프 엣지에서 비행 중인 레코드/상태 변경은 장애 시 유실** 가능

## Checkpointing with parts of the graph finished

```
Configuration config = new Configuration();
config.set(CheckpointingOptions.ENABLE_CHECKPOINTS_AFTER_TASKS_FINISH, false);
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment(config);
```

### bounded 소스 포함 시(부분 완료) 동작

- Flink 1.14+에서 일부 태스크가 데이터 처리를 끝내도 체크포인트 계속 가능
- 완료된 태스크/서브태스크는 이후 체크포인트에 더 이상 기여하지 않음

### StreamOperator#finish 도입과 의미

- `finish()`는 버퍼/남은 상태를 **마지막으로 플러시하는 명확한 컷오프**
- finish 이후의 체크포인트는 보통 **비어 있어야** 하며(방출 경로가 없기 때문),
- 예외: 외부 트랜잭션 포인터(Exactly-once sink 등 2PC)처럼 “커밋을 위한 참조”는 유지 필요

### Operator State 영향(UnionListState)

- UnionListState는 “글로벌 오프셋 뷰” 같은 용도로 쓰이곤 함(예: Kafka 파티션 오프셋)
- 일부 서브태스크만 종료된 상태에서 UnionListState를 버리면 오프셋 유실 위험
- 그래서 Flink는 **UnionListState를 쓰는 경우**:
    - “모두 종료” 또는 “아무도 종료 안 함”일 때만 체크포인트가 성공하도록 특수 처리
- ListState 등은 close 이후 체크포인트에 담긴 상태가 복구 후 없을 수 있음을 유의

### 최종 체크포인트 대기(2PC 등)

- 2-phase commit 기반 오퍼레이터는 “모든 연산자가 end-of-data 도달” 후
    - 즉시 최종 체크포인트를 트리거하고,
    - 그 체크포인트 완료까지 태스크가 기다려 **커밋 일관성**을 보장

## Unify file merging mechanism for checkpoints(Experimental)

### 목적과 트레이드오프

- Flink 1.20에 도입된 실험적 기능: 작은 체크포인트 파일들을 **큰 파일로 병합**해
    - 파일 생성/삭제 폭증(메타데이터 관리 부담) 문제 완화
- 단점: **space amplification**(실제 점유가 상태 크기보다 커질 수 있음)
    - 상한은 `max-space-amplification`으로 제한 가능

### 적용 범위와 동작 개요

- keyed/operator/channel state에 적용
- 서브태스크/TaskManager 레벨 병합 지원
- 체크포인트 간 병합도 가능(across-checkpoint-boundary)
- 파일 풀(file pool)로 동시 쓰기 처리:
    - non-blocking: 즉시 파일 제공(파일 수 증가 가능)
    - blocking: 반환된 파일이 있을 때까지 대기
