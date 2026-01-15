# flink documentation

* reference : https://flink.apache.org/

## Apache Flink®

데이터 스트림에 대한 상태 기반(Stateful) 연산

Apache Flink는 무한(unbounded) 및 유한(bounded) 데이터 스트림을 대상으로 상태를 가지는 연산(stateful computations) 을 수행하기 위한 프레임워크이자 분산 처리 엔진입니다.
Flink는 모든 일반적인 클러스터 환경에서 실행될 수 있도록 설계되었으며, 메모리 기반 속도로 대규모 스케일의 연산을 수행할 수 있습니다.

# What is Apache Flink? — Architecture

Apache Flink는 **무한(unbounded) 및 유한(bounded) 데이터 스트림**을 대상으로 **상태 기반(stateful) 연산**을 수행하기 위한 프레임워크이자 분산 처리 엔진이다.  
모든 일반적인 클러스터 환경에서 실행 가능하도록 설계되었으며, **메모리 기반 속도**로 **대규모 확장성**을 제공한다.

### Unbounded / Bounded 데이터 처리

![img.png](img.png)

모든 데이터는 이벤트 스트림 형태로 생성된다. 신용카드 거래, 센서 데이터, 로그, 사용자 행동 데이터 모두 스트림이다.

- **Unbounded Stream**
    - 시작은 있지만 끝이 없음
    - 데이터가 생성되는 즉시 지속적으로 처리해야 함
    - 모든 데이터 도착을 기다릴 수 없음
    - 결과의 완전성을 위해 이벤트 시간 순서 같은 정렬 개념이 중요

- **Bounded Stream**
    - 시작과 끝이 명확함
    - 모든 데이터를 수집한 후 처리 가능
    - 정렬이 가능하므로 순서 제약이 없음
    - 전통적인 **배치 처리(batch processing)** 와 동일

Flink는 시간(time)과 상태(state)를 정밀하게 제어함으로써 unbounded 스트림에서 모든 유형의 애플리케이션을 실행할 수 있으며, bounded 스트림은 고정 크기 데이터셋에 최적화된 내부
알고리즘으로 처리하여 높은 성능을 제공한다.

### 어디서나 애플리케이션 배포

Flink는 분산 시스템으로, 실행을 위해 클러스터 자원이 필요하다.

- Hadoop YARN, Kubernetes 등 주요 리소스 매니저와 통합
- Standalone 클러스터 구성도 가능
- 리소스 매니저별 배포 모드를 통해 자연스러운 연동 지원
- 애플리케이션의 병렬도 설정을 기반으로 필요한 자원을 자동 요청
- 장애 발생 시 새로운 리소스를 요청하여 자동 복구
- 애플리케이션 제출 및 제어는 REST API 기반

### 대규모 확장성

Flink는 **대규모 상태 기반 스트리밍 애플리케이션** 실행을 전제로 설계되었다.

- 애플리케이션은 수천 개의 태스크로 병렬화되어 클러스터에서 동시 실행
- CPU, 메모리, 디스크, 네트워크 IO를 사실상 무제한 활용 가능
- 테라바이트 단위의 상태(state) 유지 가능
- 비동기·점진적 체크포인팅을 통해 낮은 지연시간과 **Exactly-Once 상태 일관성** 보장

실제 운영 환경에서:

- 하루 수조(trillions) 개 이벤트 처리
- 수 TB 상태 유지
- 수천 코어에서 실행 사례가 보고됨

### In-Memory 성능 활용

![img_1.png](img_1.png)
Flink의 상태 기반 애플리케이션은 **로컬 상태 접근**에 최적화되어 있다.

- 태스크 상태는 메모리에 유지
- 메모리 초과 시에도 디스크 기반 고효율 자료구조 사용
- 대부분의 연산이 로컬(주로 메모리) 상태 접근으로 수행되어 매우 낮은 지연시간 제공
- 주기적·비동기 체크포인팅을 통해 장애 시에도 Exactly-Once 상태 보장

### 목차

- About
  - [Applications](applications/README.md)
  - [Operations](operations/README.md)