# Handling Application Parameters

대부분의 Flink 배치/스트리밍 애플리케이션은 외부 configuration parameter에 의존한다.

- **입력/출력 소스**: 경로(path), 주소(address) 등
- **시스템 파라미터**: parallelism, runtime configuration
- **애플리케이션 전용 파라미터**: 보통 user function 내부에서 사용

Flink는 이를 위해 간단한 유틸리티 **ParameterTool**을 제공한다. 다만 필수는 아니며, Commons CLI / argparse4j 같은 다른 프레임워크도 Flink와 함께 잘 동작한다.

## Getting your configuration values into the ParameterTool

ParameterTool은 내부적으로 `Map<String, String>` 형태를 기대하므로, 다양한 방식의 설정 로딩을 쉽게 통합할 수 있다.

### From .properties files

Properties 파일을 읽어 key/value 쌍을 만든다.

```java
String propertiesFilePath = "/home/sam/flink/myjob.properties";
ParameterTool parameters = ParameterTool.fromPropertiesFile(propertiesFilePath);

File propertiesFile = new File(propertiesFilePath);
ParameterTool parameters2 = ParameterTool.fromPropertiesFile(propertiesFile);

InputStream propertiesFileInputStream = new FileInputStream(file);
ParameterTool parameters3 = ParameterTool.fromPropertiesFile(propertiesFileInputStream);
```

### From the command line arguments

커맨드라인에서 `--input hdfs:///mydata --elements 42` 같은 인자를 받을 수 있다.

```java
public static void main(String[] args) {
    ParameterTool parameters = ParameterTool.fromArgs(args);
    // .. regular code ..
}
```

### From system properties

JVM 실행 시 `-Dinput=hdfs:///mydata` 처럼 system properties를 전달할 수 있으며, 이를 ParameterTool로 초기화할 수 있다.

```java
ParameterTool parameters = ParameterTool.fromSystemProperties();
```

## Using the parameters in your Flink program

가져온 파라미터는 여러 방식으로 활용할 수 있다.

### Directly from the ParameterTool

ParameterTool은 값 조회를 위한 메서드를 제공한다.

- `getRequired(key)`: 필수 키 (없으면 실패)
- `get(key, default)`: 기본값 지원
- 타입 변환: `getLong(...)` 등
- 파라미터 개수: `getNumberOfParameters()`

```
ParameterTool parameters = // ...
parameters.getRequired("input");
parameters.get("output", "myDefaultValue");
parameters.getLong("expectedCount", -1L);
parameters.getNumberOfParameters();
```

메인 코드에서 바로 사용해 operator 설정에도 반영 가능하다(예: parallelism).

```java
ParameterTool parameters = ParameterTool.fromArgs(args);
int parallelism = parameters.get("mapParallelism", 2);

DataStream<Tuple2<String, Integer>> counts =
        text.flatMap(new Tokenizer()).setParallelism(parallelism);
```

또한 **ParameterTool은 serializable**이므로, user function에 전달해 함수 내부에서도 사용할 수 있다.

```java
ParameterTool parameters = ParameterTool.fromArgs(args);
DataStream<Tuple2<String, Integer>> counts =
        text.flatMap(new Tokenizer(parameters));
```

### Register the parameters globally

ExecutionConfig의 **global job parameters**로 등록하면,

- JobManager Web UI에서 configuration 값으로 확인 가능
- 모든 user function(특히 rich function)에서 접근 가능

전역 등록:

```
ParameterTool parameters = ParameterTool.fromArgs(args);

// set up the execution environment
final ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
env.getConfig().setGlobalJobParameters(parameters);
```

Rich user function에서 접근:

```java
public static final class Tokenizer
        extends RichFlatMapFunction<String, Tuple2<String, Integer>> {

    @Override
    public void flatMap(String value, Collector<Tuple2<String, Integer>> out) {
        ParameterTool parameters =
                ParameterTool.fromMap(getRuntimeContext().getGlobalJobParameters());

        parameters.getRequired("input");
        // .. do more ..
    }
}
```
