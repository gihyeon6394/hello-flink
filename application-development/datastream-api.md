# DataStream API

- Flink DataStream API Programming Guide

## Flink DataStream API Programming Guide

### DataStream 프로그램 개요

- Flink의 DataStream 프로그램은 스트림에 대해 **변환(transformations)** 을 적용하는 일반 프로그램이다(예: filter, 상태 업데이트, 윈도우 정의, 집계).
- 스트림은 **source**(메시지 큐, 소켓, 파일 등)에서 생성되고, 결과는 **sink**(파일, 표준 출력, 외부 시스템 등)로 전달된다.
- 실행은 로컬 JVM/단일 실행/다중 머신 클러스터 등 다양한 컨텍스트에서 가능하며, 독립 실행 또는 다른 프로그램에 임베드될 수도 있다.
- 자체 프로그램을 만들 때는 “Flink 프로그램의 구성(anatomy)”을 이해한 뒤 변환을 점진적으로 추가하는 흐름이 권장된다.

### What is a DataStream?

- DataStream은 Flink 프로그램에서 데이터 컬렉션을 표현하는 클래스이며, **중복 허용** + **불변(immutable)** 컬렉션처럼 생각할 수 있다.
- 데이터는 **유한(finite)** 또는 **무한(unbounded)** 일 수 있지만, 이를 다루는 API는 동일하다.
- Java Collection과 사용 방식은 비슷하지만,
    - 생성 후 요소를 추가/삭제할 수 없고,
    - 내부 요소를 직접 “조회”하기보다 **API 연산(변환)** 으로만 처리한다.
- 초기 DataStream은 source로 만들고, `map`, `filter` 같은 변환으로 파생 스트림을 만들거나 결합한다.

### Anatomy of a Flink Program

- Flink 프로그램은 보통 다음 단계로 구성된다:
    - 실행 환경(ExecutionEnvironment) 획득
    - 초기 데이터 로드/생성
    - 데이터 변환(Transformations) 정의
    - 결과 출력 위치(Sink) 정의
    - 실행 트리거(execute)
- Java DataStream API 핵심 클래스는 `org.apache.flink.streaming.api` 패키지에 있다.

### Execution Environment

- `StreamExecutionEnvironment`가 모든 Flink 프로그램의 기반이다.
- 대표 생성 방식:
    - `getExecutionEnvironment()` : 실행 컨텍스트에 맞게 로컬/클러스터 환경을 자동 선택(IDE/로컬 실행이면 로컬, CLI로 클러스터 제출이면 클러스터 환경 반환).
    - `createLocalEnvironment()` : 로컬 JVM 내 실행 환경 명시
    - `createRemoteEnvironment(host, port, jarFiles...)` : 원격 클러스터 실행 환경
- 실행 환경 타입에 따라 `execute()`가 로컬에서 실행되거나 클러스터에 제출된다.

### Sources, Transformations, Sinks 흐름

- Source에서 DataStream을 만든 후 변환을 적용해 결과 스트림을 만들고, Sink로 외부에 출력한다.
- 예시(개념 수준):
    - 파일을 source로 읽어 DataStream 생성
    - `map` 등으로 타입 변환/처리
    - `FileSink`로 파일 출력 또는 `print()`로 표준 출력

### Program Execution의 핵심: Lazy Evaluation

- Flink는 **지연 실행(lazy)** 방식이다.
    - main이 실행될 때 실제 데이터 로딩/변환이 즉시 수행되지 않고,
    - 각 연산이 **데이터플로우 그래프**로 구성된다.
- 실제 실행은 `env.execute()`(또는 `executeAsync()`) 호출로 트리거된다.
- 지연 실행 덕분에 Flink가 전체 작업을 하나의 단위로 계획하여 실행할 수 있다.

### execute() vs executeAsync()

- `execute()`:
    - 작업 완료까지 대기하고 `JobExecutionResult`(실행 시간, accumulator 결과 등)를 반환한다.
- `executeAsync()`:
    - 비동기로 제출하고 `JobClient`를 반환하며, 필요하면 결과를 추후 조회할 수 있다.

### Example Program 요약(윈도우 워드카운트)

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;

import java.time.Duration;

import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter()) // e.g. "hello bye hello" -> ("hello",1),("bye",1),("hello",1)
                .keyBy(value -> value.f0) // e.g. <"hello", [("hello",1),("hello",1)]>, <"bye", [("bye",1)]>
                .window(TumblingProcessingTimeWindows.of(Duration.ofSeconds(5)))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word : sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```

- 소켓에서 들어오는 텍스트를 읽어 단어로 분리한 뒤,
- 단어를 key로 묶어 **5초 텀블링 처리시간 윈도우**로 합계를 내고,
- 결과를 출력하는 스트리밍 예제다.
- 로컬 테스트는 netcat 등으로 소켓 입력 스트림을 열어 확인한다.

### Data Sources

- 입력은 `StreamExecutionEnvironment.addSource(...)`로 붙일 수 있다.
- 내장 source 외에도:
    - 비병렬 소스는 `SourceFunction`
    - 병렬 소스는 `ParallelSourceFunction` 또는 `RichParallelSourceFunction`으로 커스텀 구현 가능
- 주요 내장 소스 유형:
    - 파일 기반: FileSource 등(레코드 단위 읽기, 한 번 처리/지속 감시 모드 지원)
    - 소켓 기반: `socketTextStream`
    - 컬렉션 기반: `fromCollection`, `fromElements`, `fromSequence`, 병렬 이터레이터 기반 등
    - 커스텀/커넥터 기반: `addSource(...)`로 Kafka 등 연결(커넥터 사용)

### File-based source 동작과 주의점

- 내부적으로 “디렉터리 모니터링(단일 태스크)”과 “데이터 읽기(병렬 리더)”로 분리되어 동작한다.
- `PROCESS_CONTINUOUSLY`(지속 감시) 모드에서 파일이 수정되면 **파일 전체를 재처리**할 수 있어 exactly-once 의미를 깨뜨릴 수 있다.
- `PROCESS_ONCE`(1회 처리) 모드에서는 소스가 경로 스캔 후 종료되며, 이후 체크포인트가 더 이상 생기지 않아 장애 복구 시 마지막 체크포인트 기준 재시작으로 **복구가 느려질 수 있다**.

### DataStream Transformations

- 변환 연산(operators)은 별도 “operators” 섹션에서 전체 목록을 참고한다.
- 기본적으로 `map`, `filter`, `flatMap`, `keyBy`, `window`, `sum` 같은 연산으로 파이프라인을 구성한다.

### Data Sinks

- sink는 DataStream 결과를 파일/소켓/외부 시스템으로 내보내거나 출력한다.
- 예:
    - `sinkTo(FileSink...)` : 파일 시스템에 신뢰성 있게 기록(체크포인팅 참여 가능)
    - `print()` / `printToErr()` : 표준 출력/에러 출력(병렬 실행 시 태스크 식별자 프리픽스 가능)
    - `writeUsingOutputFormat`, `writeToSocket`, `addSink`(커스텀 싱크/커넥터 싱크)
- 주의:
    - `write*()` 계열은 주로 디버깅용이며 **체크포인팅에 참여하지 않아** 보통 at-least-once 의미(또는 실패 시 유실 가능)가 될 수 있다.
    - 파일 시스템으로 정확히 한 번(exactly-once) 전달이 중요하면 **FileSink** 또는 체크포인팅에 참여하는 싱크 구현을 사용한다.

### Execution Parameters

- `StreamExecutionEnvironment`의 `ExecutionConfig`로 런타임 관련 잡 설정을 조정한다.
- DataStream API 관련 예: `setAutoWatermarkInterval(ms)`로 자동 워터마크 발행 간격 설정.

### Controlling Latency (Buffer Timeout)

- 네트워크 전송은 원소를 1개씩 보내지 않고 **버퍼링**하여 처리량(throughput)을 높인다.
- 유입이 느리면 버퍼가 차기까지 대기해 지연(latency)이 커질 수 있으므로,
    - `env.setBufferTimeout(timeoutMillis)`로 버퍼를 최대 대기 시간 후 강제 플러시할 수 있다(오퍼레이터별 설정도 가능).
- 튜닝 가이드:
    - 처리량 최대화: `-1`(버퍼가 찰 때만 전송)
    - 지연 최소화: 0에 가까운 값(예: 5~10ms) 권장
    - 단, `0`은 성능 급락을 유발할 수 있어 피하는 것이 좋다.

### Debugging (로컬 개발/테스트)

- 분산 실행 전 로컬에서 결과 검증/디버깅하는 것이 권장된다.
- **LocalStreamEnvironment**는 같은 JVM에서 Flink를 띄워 IDE에서 브레이크포인트 디버깅이 쉽다.
- **Collection Source**는 테스트를 쉽게 하며, 검증 후 실제 외부 source/sink로 교체하기 좋다.
    - 제한: 현재 컬렉션 소스는 타입/이터레이터가 `Serializable`이어야 하며, **병렬 실행 불가(parallelism=1)**.
- 테스트/디버깅을 위해 결과를 모으는 sink도 제공되며, 예로 `collectAsync()`로 결과를 이터레이터 형태로 수집할 수 있다.
