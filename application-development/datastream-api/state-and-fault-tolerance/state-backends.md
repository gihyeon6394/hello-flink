# State Backends

### 역할

- Flink는 **State Backend**를 통해 상태(state)를 **어떻게/어디에 저장할지**를 결정한다.
- 즉, 상태 저장 위치(메모리/오프힙 등)와 저장 방식이 백엔드 선택에 의해 정해진다.

## 상태 저장 위치와 관리 방식

### Heap vs Off-heap

- 상태는 **JVM 힙(Java heap)** 또는 **오프힙(off-heap)** 에 위치할 수 있다.
- 어떤 백엔드를 쓰느냐에 따라, Flink가 애플리케이션의 상태를 **직접 관리**할 수도 있다.
    - 예: 메모리 관리, 필요 시 디스크로 스필(spill) 등
    - 이를 통해 **매우 큰 상태**도 다룰 수 있게 된다.

## 설정 방식

### 기본 설정(클러스터/전역)

- 기본적으로는 **Flink 설정 파일**에서 모든 잡에 적용될 **기본 state backend**를 결정한다.

### 잡 단위 오버라이드

- 필요하면 잡별로 기본값을 **덮어쓸 수 있다**.

### Java 예시(잡 단위 설정)

````
Configuration config = new Configuration();
config.set(StateBackendOptions.STATE_BACKEND, "hashmap");
env.configure(config);
````

- `StateBackendOptions.STATE_BACKEND` 값을 지정해 백엔드를 선택한다.
    - 예: `"hashmap"`

