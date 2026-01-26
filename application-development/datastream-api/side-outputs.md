# Side Outputs

- Flink DataStream 연산은 **main stream** 외에 원하는 만큼의 **side output stream**을 추가로 만들 수 있다.
- side output의 데이터 타입은 main stream과 **같을 필요가 없고**, side output끼리도 **서로 다른 타입**을 가질 수 있다.
- 보통 “stream을 복제(replicate)한 뒤 각 복제본에서 filter로 필요한 것만 남기는” 방식 대신,
  **한 번의 처리에서 분기**하고 싶을 때 유용하다.

## OutputTag 정의

- side output을 식별하기 위해 먼저 **OutputTag**를 정의한다.
- OutputTag는 side output에 담길 **요소 타입을 포함**한다(typed).

```java
// type 정보를 분석할 수 있도록 anonymous inner class로 생성해야 함
OutputTag<String> outputTag = new OutputTag<String>("side-output") {
        };
```

## 어떤 함수에서 side output을 낼 수 있나

다음 함수들에서 `Context`를 통해 `ctx.output(OutputTag, value)` 형태로 side output을 emit 할 수 있다.

### Process 계열

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- KeyedCoProcessFunction

### Window Process 계열

- ProcessWindowFunction
- ProcessAllWindowFunction

## side output으로 데이터 emit 하기

- 아래 예시는 ProcessFunction에서
    - main output에는 `Integer`를 그대로 내보내고
    - side output에는 `String`을 `"sideout-" + value` 형태로 내보낸다.

```java
DataStream<Integer> input = ...;

final OutputTag<String> outputTag = new OutputTag<String>("side-output") {
};

SingleOutputStreamOperator<Integer> mainDataStream = input
        .process(new ProcessFunction<Integer, Integer>() {

            @Override
            public void processElement(
                    Integer value,
                    Context ctx,
                    Collector<Integer> out) throws Exception {

                // main output
                out.collect(value);

                // side output
                ctx.output(outputTag, "sideout-" + String.valueOf(value));
            }
        });
```

## side output stream 가져오기

- side output은 “해당 연산의 결과”에서 `getSideOutput(OutputTag)`로 꺼낸다.
- 반환되는 DataStream은 OutputTag의 타입에 맞게 **typed** 되어 있다.

```java
final OutputTag<String> outputTag = new OutputTag<String>("side-output") {
};

SingleOutputStreamOperator<Integer> mainDataStream = ...;

DataStream<String> sideOutputStream = mainDataStream.getSideOutput(outputTag);
```
