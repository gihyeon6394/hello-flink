# Fault Tolerance via State Snapshots

## State Backends

- Flink의 **keyed state**는 키 기준으로 샤딩된 key/value 저장소처럼 동작하며, 각 키를 담당하는 **TaskManager 로컬**에 작업 사본(working copy)이 유지된다.
- **operator state**도 이를 필요로 하는 머신(들)의 로컬에 유지된다.
- 이 상태(state)는 **state backend**에 저장된다.

### 주요 구현체 비교

| Name                        | Working State 위치 | Snapshotting       | 특징                                              |
|-----------------------------|------------------|--------------------|-------------------------------------------------|
| EmbeddedRocksDBStateBackend | 로컬 디스크(tmp dir)  | Full / Incremental | 메모리보다 큰 state 지원, (경험칙) heap 기반 대비 ~10배 느릴 수 있음 |
| HashMapStateBackend         | JVM Heap         | Full               | 매우 빠름(메모리 기반)이나 큰 heap 필요, GC 영향 큼              |

### 동작/성능 포인트

- Heap 기반: 힙의 객체를 직접 읽고/쓰기 → 빠르지만 GC 부담.
- RocksDB 기반: 접근/업데이트 시 **직렬화·역직렬화**가 필요 → 비용 큼.
- RocksDB는 로컬 디스크 크기만큼 state를 담을 수 있고, **incremental snapshot**이 가능해 “크고 천천히 변하는 state”에서 유리.
- 두 backend 모두 **비동기(asynchronous) 스냅샷**을 지원해 스트림 처리를 크게 막지 않고 스냅샷을 뜰 수 있다.

## Checkpoint Storage

### 역할

- Flink는 주기적으로 모든 operator의 state를 **지속 가능한(persistent) 스냅샷**으로 만들고, 더 내구성 있는 저장소(예: 분산 파일시스템)로 복사한다.
- 장애가 나면 이 스냅샷으로 애플리케이션 전체 state를 복원해 “문제 없던 것처럼” 처리를 재개할 수 있다.

### 구현체

| Name                        | State Backup 위치     | 특징                                 |
|-----------------------------|---------------------|------------------------------------|
| FileSystemCheckpointStorage | 분산 파일 시스템           | 매우 큰 state 지원, 내구성 높음, **프로덕션 권장** |
| JobManagerCheckpointStorage | JobManager JVM Heap | 로컬에서 작은 state로 **테스트/실험**에 적합      |

## State Snapshots

### 용어 정리

- **Snapshot**: Flink 잡의 **전역적·일관된(global consistent)** 상태 이미지.
    - 각 source의 포인터(예: 파일 오프셋, Kafka 파티션 오프셋) + 각 stateful operator의 state 복사본을 포함.
- **Checkpoint**: 장애 복구를 위해 Flink가 자동으로 만드는 snapshot.
    - **빠른 복원**에 최적화되며 **incremental**일 수 있음.
- **Externalized Checkpoint**: 보통 체크포인트는 잡 실행 중 최근 n개만 유지되고, 잡 취소 시 삭제된다.
    - 설정으로 **보존(retain)** 하게 하면 사용자가 수동으로 해당 지점에서 재시작 가능.
- **Savepoint**: 사용자가 수동(또는 API)으로 트리거하는 snapshot.
    - 재배포/업그레이드/리사이즈 같은 운영 목적에 쓰이며 **항상 complete(full)** 이고 운영 유연성에 최적화.

## How does State Snapshotting Work?

### 비동기 배리어 스냅샷(Chandy-Lamport 변형)

![img.png](img.png)

- Flink는 **asynchronous barrier snapshotting**을 사용한다.
- Checkpoint coordinator(JobManager의 일부)가 체크포인트 시작을 지시하면:
    - Source가 자신의 오프셋을 기록하고,
    - 스트림에 번호가 붙은 **checkpoint barrier**를 삽입한다.
- Barrier는 job graph를 따라 흐르며, 각 체크포인트 기준으로 “이전/이후” 구간을 구분한다.
    - **Checkpoint n**에는 barrier n 이전 이벤트까지 처리한 결과로 만들어진 operator state가 포함되고, barrier 이후 이벤트는 포함되지 않는다.

### Barrier alignment

![img_1.png](img_1.png)

- 다중 입력(operator가 2개 입력을 받는 경우 등)은 스냅샷의 일관성을 위해 **barrier alignment**를 수행한다.
- 즉, 두 입력 스트림 모두에서 barrier가 도착할 때까지 정렬/대기하여 “양쪽 스트림에서 barrier 이전까지 처리한 상태”를 맞춘다.

### Copy-on-write와 비동기 스냅샷

- State backend는 **copy-on-write** 방식으로,
    - 스냅샷을 비동기로 저장하는 동안에도 최신 state로 계속 처리할 수 있게 한다.
- 스냅샷이 내구 저장소에 **완전히 기록**된 뒤에야 이전 버전 state는 GC/정리된다.

## Exactly Once Guarantees

### 장애 시 가능한 결과

- **At most once**: 복구를 거의 하지 않음 → 손실 가능.
- **At least once**: 손실은 없지만 **중복 결과** 가능.
- **Exactly once**: 손실/중복 모두 없음.

### “이벤트를 정확히 한 번 처리”의 의미

- Flink는 소스 데이터를 **되감아(rewind) 재생(replay)** 하며 복구한다.
- 따라서 “모든 이벤트가 실행 관점에서 정확히 한 번 처리된다”가 아니라,
    - **각 이벤트가 Flink가 관리하는 state에 정확히 한 번만 반영된다**는 뜻이다.

### 성능 트레이드오프

- **Barrier alignment**는 exactly once를 위해 필요하다.
- exactly once가 필요 없다면 `CheckpointingMode.AT_LEAST_ONCE`로 설정해 alignment를 비활성화하여 성능을 얻을 수 있다.

## Exactly Once End-to-end

### 조건

- 소스는 **replay 가능**해야 하고,
- 싱크는 **트랜잭션 지원**(transactional) 또는 **멱등(idempotent)** 해야 한다.
- 이 조건이 충족되어야 “source → Flink state → sink” 전체가 end-to-end exactly once가 된다.
