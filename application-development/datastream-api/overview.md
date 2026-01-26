# Flink DataStream API Programming Guide #

## DataStream 프로그램 개요

- Flink의 DataStream 프로그램은 스트림 데이터에 **변환(transformations)** 을 적용하는 일반 프로그램이다(예: 필터링, 상태 업데이트, 윈도우 정의, 집계).
- 스트림은 **소스(source)** (메시지 큐, 소켓, 파일 등)에서 생성되고, 결과는 **싱크(sink)** (파일, 표준 출력, 외부 시스템 등)로 전달된다.
- 실행은 로컬 JVM 또는 다수 머신의 클러스터에서 가능하며, 단독 실행/다른 프로그램에 임베드된 형태 모두 가능하다.
- 시작은 “Flink 프로그램의 구성(anatomy)”을 이해하고, 그 위에 변환을 점진적으로 추가하는 방식이 권장된다.

## What is a DataStream?

### DataStream의 성격

- DataStream API는 Flink 내에서 데이터 컬렉션을 표현하는 **DataStream 클래스**에서 이름이 왔다.
- **불변(immutable)** 이며 **중복을 허용**하는 컬렉션처럼 생각할 수 있다.
- 데이터는 **유한(bounded)** 또는 **무한(unbounded)** 일 수 있으나, 다루는 API는 동일하다.
- Java Collection과 사용 느낌은 비슷하지만,
    - 생성 후 요소를 추가/삭제할 수 없고,
    - 내부 요소를 직접 열어보는 대신 **변환 연산(transformations)** 으로만 처리한다.
- 초기 DataStream은 소스로 만들고, `map`, `filter` 등의 API로 파생 스트림을 만들거나 결합한다.

## Anatomy of a Flink Program

### 기본 구성 단계

- Flink 프로그램은 보통 다음 흐름을 따른다:
    - 실행 환경 획득
    - 초기 데이터 로드/생성
    - 변환 정의
    - 결과 출력(싱크) 정의
    - 실행 트리거
- Java DataStream API 핵심 클래스는 `org.apache.flink.streaming.api`에 있다.

### StreamExecutionEnvironment

- 모든 Flink 프로그램의 기반이며, 대표적으로 다음 방식으로 얻는다:
    - `getExecutionEnvironment()` : 실행 컨텍스트에 맞게 로컬/클러스터 환경을 자동 선택(대부분 이걸로 충분)
    - `createLocalEnvironment()` : 로컬 JVM에서 명시적 실행
    - `createRemoteEnvironment(host, port, jarFiles...)` : 원격 클러스터 실행
- IDE/로컬에서 실행하면 로컬 환경이 만들어지고, JAR를 CLI로 제출하면 클러스터 환경으로 실행된다.

### 소스 예시(파일)와 변환 예시(map)

- 실행 환경은 파일을 라인 단위/CSV 등 다양한 방식으로 읽는 소스를 제공한다.
- 파일 소스로 DataStream을 만든 뒤, `map` 같은 변환으로 새로운 DataStream을 만든다(예: String → Integer 파싱).

### 싱크 예시와 실행 트리거

- 최종 결과 DataStream은 `FileSink`로 파일에 쓰거나 `print()`로 출력할 수 있다.
- 프로그램 정의가 끝나면 `env.execute()`로 실행을 트리거한다.
    - `execute()`는 잡 완료까지 기다린 뒤 `JobExecutionResult`(실행 시간, accumulator 결과 등)를 반환한다.
    - 기다리기 싫으면 `executeAsync()`로 비동기 제출하고 `JobClient`로 잡과 통신/결과 조회가 가능하다.

### Lazy 실행 모델의 중요성

- Flink는 **지연 실행(lazy evaluation)** 을 사용한다.
    - main이 실행될 때 데이터 로딩/변환이 즉시 수행되지 않고,
    - 각 연산이 데이터플로우 그래프에 추가된다.
- 실제 실행은 `execute()` 호출 시점에 일어나며, 로컬/클러스터 실행 여부는 실행 환경 타입에 달려 있다.
- 이 방식 덕분에 Flink는 프로그램을 하나의 “전체 계획 단위”로 최적화해 실행할 수 있다.

## Example Program

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
                .flatMap(new Splitter())
                .keyBy(value -> value.f0)
                .window(TumblingProcessingTimeWindows.of(Duration.ofSeconds(5)))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```

### 윈도우 워드카운트 예제 요약

- 소켓에서 텍스트를 받아 단어로 분리(`flatMap`)하고,
- 단어 기준으로 키 그룹(`keyBy`)한 뒤,
- **5초 텀블링 처리시간 윈도우**로 합계를 내고 출력한다.
- 로컬 실행 시 터미널에서 `nc -lk 9999`로 입력 스트림을 열고 단어를 입력해 결과를 확인한다.

## Data Sources

### 소스 개념과 확장

- 소스는 입력을 읽는 지점이며 `StreamExecutionEnvironment.addSource(...)`로 붙인다.
- 내장 소스가 다수 제공되고, 필요하면 커스텀 소스를 구현할 수 있다:
    - 비병렬: `SourceFunction`
    - 병렬: `ParallelSourceFunction` 또는 `RichParallelSourceFunction`

### 대표 내장 소스 종류

- 파일 기반:
    - `fromSource(FileSource...)` : 레코드 단위 파일 읽기
    - `readFile(...)` : 파일 입력 포맷에 따라 읽기(1회 또는 감시 모드 포함)
    - 내부적으로 “디렉터리 모니터링(단일 태스크)” + “데이터 읽기(병렬 리더)”로 분리되어 동작
- 주의사항(파일 소스):
    - `PROCESS_CONTINUOUSLY`에서 파일이 수정되면 **전체 재처리**될 수 있어 exactly-once 의미를 깨뜨릴 수 있다.
    - `PROCESS_ONCE`는 경로를 한 번 스캔하고 소스가 종료되며 이후 체크포인트가 더 생기지 않아, 장애 시 마지막 체크포인트 기준으로 재개되어 **복구가 느려질 수 있다**.
- 소켓 기반: `socketTextStream`
- 컬렉션 기반:
    - `fromData(Collection)`, `fromData(T...)`, `fromParallelCollection(...)`, `fromSequence(...)`
- 커스텀/커넥터 기반:
    - `addSource(...)`로 Kafka 소비자 등 연결(커넥터 참고)

## DataStream Transformations

### 변환 연산

- 사용 가능한 변환(operators)은 별도 “operators” 섹션에서 전체 목록을 참고한다.
- 일반적으로 `map/filter/flatMap`, `keyBy`, `window`, `sum/reduce` 등으로 파이프라인을 구성한다.

## Data Sinks

### 싱크 종류와 의미론 주의

- 싱크는 DataStream을 파일/소켓/외부 시스템으로 전달하거나 출력한다.
- 예:
    - `sinkTo(FileSink...)` : 파일 시스템에 신뢰성 있게 기록(체크포인팅 참여 가능)
    - `print()/printToErr()` : 표준 출력/에러 출력(병렬 시 태스크 식별자 포함 가능)
    - `writeUsingOutputFormat()/FileOutputFormat`, `writeToSocket`, `addSink`(커스텀 싱크/커넥터 싱크)
- 주의:
    - `write*()` 계열은 주로 **디버깅 목적**이며 체크포인팅에 참여하지 않아 보통 **at-least-once**이거나 실패 시 유실 가능성이 있다.
    - 파일로 **exactly-once** 전달이 필요하면 `FileSink` 사용이 권장된다.
    - `.addSink(...)`로 만든 커스텀 싱크는 체크포인팅에 참여하도록 구현하면 exactly-once 의미론을 달성할 수 있다.

## Execution Parameters

### 실행 설정

- `StreamExecutionEnvironment`는 `ExecutionConfig`를 통해 잡 런타임 설정을 제공한다.
- DataStream 관련 예: `setAutoWatermarkInterval(ms)`로 자동 워터마크 발행 간격을 설정/조회한다.

## Fault Tolerance

### 체크포인팅

- 체크포인팅 활성화/구성은 “State & Checkpointing”에서 다룬다.

## Controlling Latency

### 버퍼 타임아웃으로 지연/처리량 조절

- 기본적으로 네트워크 전송은 레코드를 1개씩 보내지 않고 **버퍼링**한다(처리량 최적화).
- 유입이 느리면 버퍼가 차기까지 기다려 지연이 커질 수 있어,
    - `env.setBufferTimeout(timeoutMillis)`로 버퍼 최대 대기 시간을 설정한다(오퍼레이터별 설정도 가능).
- 튜닝 가이드:
    - 처리량 최대화: `setBufferTimeout(-1)`(버퍼가 찰 때만 플러시)
    - 지연 최소화: 0에 가까운 값(예: 5~10ms)
    - `0`은 성능 급락을 유발할 수 있어 피하는 것이 좋다.

## Debugging

### 로컬 디버깅/테스트 지원

- 분산 실행 전 로컬에서 결과를 확인하며 개발하는 것이 일반적이다.
- **LocalStreamEnvironment**는 같은 JVM에서 Flink를 실행해 IDE 브레이크포인트 디버깅이 쉽다.
- **Collection Data Sources**는 테스트를 쉽게 하고, 검증 후 외부 소스/싱크로 교체하기 좋다.
    - 제한: 컬렉션 소스는 타입/이터레이터가 `Serializable`이어야 하고, **병렬 실행 불가(parallelism=1)** .
- 테스트/디버깅용으로 결과를 수집하는 싱크도 제공되며, 예로 `collectAsync()`로 결과를 이터레이터 형태로 받을 수 있다.
