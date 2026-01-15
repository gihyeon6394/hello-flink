# Intro to the DataStream API

이 트레이닝의 목표는 **DataStream API의 핵심을 이해하여 스트리밍 애플리케이션을 작성할 수 있게 하는 것**이다.  
세부적인 내부 구현보다는, 실제로 스트림을 만들고 변환·실행하는 데 필요한 기본 개념에 초점을 둔다.

### 무엇을 스트리밍할 수 있는가

DataStream API는 **직렬화 가능한 모든 데이터 타입**을 스트림으로 처리할 수 있다.

- **기본 타입**: String, Long, Integer, Boolean, Array
- **복합 타입**: Tuple, POJO
- 기타 타입은 **Kryo** 직렬화 사용
- Avro 등 외부 직렬화 포맷도 지원

### Java Tuples와 POJOs

Flink는 **Tuple과 POJO에 대해 고성능 네이티브 직렬화**를 제공한다.

#### Tuples

````
Tuple2<String, Integer> person = Tuple2.of("Fred", 35);

// zero based index!  
String name = person.f0;
Integer age = person.f1;

````

- Java에서 `Tuple0` ~ `Tuple25` 제공
- 필드 접근은 **0-based 인덱스**

#### POJOs

````
public class Person {
    public String name;  
    public Integer age;  
    public Person() {}
    public Person(String name, Integer age) {  
        . . .
    }
}  

Person person = new Person("Fred Flintstone", 35);

````

다음 조건을 만족하면 POJO로 인식된다.

- public 클래스이며 독립 클래스
- public 기본 생성자 존재
- 모든 필드가 public 이거나 getter/setter 제공

POJO는 **스키마 진화(schema evolution)** 도 지원한다.

### 간단한 예제

```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.api.common.functions.FilterFunction;

public class Example {

    public static void main(String[] args) throws Exception {
        final StreamExecutionEnvironment env =
                StreamExecutionEnvironment.getExecutionEnvironment();

        // 입력 스트림 생성 (실제로는 카프카, 소켓, 파일 등에서 생성)
        DataStream<Person> flintstones = env.fromData(
                new Person("Fred", 35),
                new Person("Wilma", 35),
                new Person("Pebbles", 2));

        DataStream<Person> adults = flintstones.filter(new FilterFunction<Person>() {
            @Override
            public boolean filter(Person person) throws Exception {
                return person.age >= 18;
            }
        });

        adults.print(); // 결과 출력 (실제로는 파일, DB, 메시지 큐 등)

        env.execute(); // 프로그램 실행
    }

    public static class Person {
        public String name;
        public Integer age;

        public Person() {
        }

        public Person(String name, Integer age) {
            // ...
        }

        public String toString() {
            // ...
        }
    }
}

```

입력 스트림에서 **성인만 필터링**하는 간단한 DataStream 애플리케이션 예제를 통해,

- ExecutionEnvironment 생성
- DataStream 생성
- 변환(filter)
- 결과 출력
- 실행(`execute()`)

이라는 기본 흐름을 확인할 수 있다.

### Stream Execution Environment

![img_6.png](img_6.png)

모든 Flink 애플리케이션은 **ExecutionEnvironment**가 필요하다.

- DataStream API 호출 → **Job Graph** 구성
- `execute()` 호출 시 JobManager로 전송
- JobManager가 병렬화하여 TaskManager에 분산 실행
- `execute()`를 호출하지 않으면 애플리케이션은 실행되지 않음

### 기본 스트림 소스

프로토타이핑 및 테스트용:

- `fromData(...)`
- 소켓 입력 (`socketTextStream`)
- 파일 소스 (`FileSource`)

실제 운영 환경에서는:

- Kafka, Kinesis, 파일 시스템 등
- 저지연·고처리량·재처리(replay) 지원 소스
- 외부 DB나 REST API는 주로 **스트림 보강(enrichment)** 용도

````
// 소켓에서 텍스트 라인 스트림 생성
DataStream<String> lines = env.socketTextStream("localhost", 9999);

// 파일에서 텍스트 라인 스트림 생성
FileSource<String> fileSource = FileSource.forRecordStreamFormat(
        new TextLineInputFormat(), new Path("file:///path")
    ).build();
DataStream<String> lines = env.fromSource(
    fileSource,
    WatermarkStrategy.noWatermarks(),
    "file-input"
);
````

### 기본 스트림 싱크

````
1> Fred: age 35
2> Wilma: age 35
````

- `print()` : 디버깅 및 학습용
    - 출력 앞의 `1>`, `2>` 는 서브태스크 ID
- 운영 환경:
    - FileSink
    - 데이터베이스
    - 메시지 큐 / Pub-Sub 시스템

### 디버깅

Flink는 **IDE 기반 로컬 디버깅**을 지원한다.

- 브레이크포인트 설정
- 변수 확인 및 단계별 실행
- Flink 내부 코드까지 추적 가능

이는 원격 클러스터 장애를 디버깅하기 전에 로컬에서 문제를 잡는 데 매우 유용하다.

### Hands-on

이 시점이면 간단한 DataStream 애플리케이션을 작성할 수 있다.

- flink-training-repo 클론
- 첫 번째 실습: **Filtering a Stream**
