# Project Configuration

- 이 섹션은 **Maven/Gradle** 같은 빌드 도구로 Flink 프로젝트를 구성하는 법, 필요한 **의존성(커넥터/포맷/테스트)** 추가, 그리고 몇 가지 **고급 설정 주제**를 다룬다.
- 모든 Flink 애플리케이션은 최소한 **Flink API 라이브러리**에 의존하며, 필요에 따라 **커넥터(Kafka/Cassandra 등)** 및 사용자 정의 함수 개발에 필요한 **서드파티 라이브러리**가
  추가된다.

## Getting started

### Maven으로 프로젝트 생성

- **Archetype**으로 생성하거나 제공되는 **quickstart 스크립트**를 사용한다.
- Archetype 방식은 프로젝트 이름, groupId/artifactId/package 등을 대화형으로 입력한다.

### Gradle

- Gradle도 지원되며, 이후 섹션에서 Gradle 기반 구성(의존성/패키징/IDE 연동)을 설명한다.

## Which dependencies do you need?

### 기본적으로 필요한 의존성 범주

- **Flink APIs**: 잡 개발용
- **Connectors & formats**: 외부 시스템 연동 및 데이터 인코딩/디코딩
- **Testing utilities**: 테스트 작성/실행
- 추가로, **custom function** 구현에 필요한 3rd party 의존성을 더할 수 있다.

## Flink APIs

### 주요 API와 대표 의존성 매핑

- Flink는 크게 **DataStream API**와 **Table API & SQL**을 제공하며, 단독 사용 또는 혼합이 가능하다.
- 선택한 API 조합에 따라 `flink-streaming-java`, `flink-table-api-java`, Scala용 Table API 모듈, 그리고 Table↔DataStream 브리지 모듈 등을
  추가한다.

## Running and packaging

### 실행에 필요한 모듈

- `main()`을 직접 실행해 잡을 돌리려면 클래스패스에 **flink-clients**가 필요하다.
- Table API 프로그램은 추가로 **flink-table-runtime**, **flink-table-planner-loader**가 필요할 수 있다.

### 패키징 원칙(uber/fat JAR)

- 일반적으로 **애플리케이션 코드 + 필요한 의존성(커넥터/포맷/서드파티)**을 하나로 묶은 **fat/uber JAR** 권장.
- 단, **Flink 자체가 제공하는 모듈(예: Java API들, 특정 runtime 모듈)**은 클러스터 배포판에 이미 있으므로 **job uber JAR에 포함하지 말 것**(중복/충돌 방지).
- 결과 JAR은 실행 중인 Flink 클러스터에 제출하거나 컨테이너 이미지에 쉽게 포함 가능.

## Gradle로 프로젝트 구성

### Requirements

- **Gradle 7.x**
- **Java 8(Deprecated) 또는 Java 11**

### IDE로 가져오기

- IntelliJ: Gradle 플러그인으로 Gradle 프로젝트 지원
- Eclipse: Buildship 플러그인 사용(일부 플러그인 요구로 Gradle 버전 하한 존재)
- 참고: Flink는 기본 JVM heap이 작을 수 있어 IDE 실행 시 **힙(-Xmx 등) 증설**이 필요할 수 있다.
- IntelliJ에서 실행 시, 경우에 따라 **Provided 스코프 의존성 포함 옵션**을 켜야 정상 실행된다(대안: main()을 호출하는 테스트 작성).

### 빌드/패키징

- `gradle clean shadowJar`로 **all.jar(애플리케이션 + 추가한 라이브러리/커넥터 포함)** 생성 가능.
- entry point(main class)가 기본값과 다르면 `build.gradle`의 **mainClassName**을 맞춰두는 것을 권장(실행 시 추가 지정 최소화).

## Adding dependencies to the project

### Gradle 의존성 추가 방식

- `build.gradle`의 `dependencies { ... }` 블록에 커넥터 등을 추가한다(예: Kafka 커넥터).

### 스코프(scope) 설정의 중요 포인트

- Flink **코어 의존성**은 보통 **provided**로 두는 것이 권장:
    - 컴파일에는 필요하지만, 배포 JAR에 포함하지 않음
    - 포함해버리면 JAR이 과도하게 커지거나, Flink 코어와 사용자 의존성 간 **버전 충돌** 위험이 생길 수 있음(클래스 로딩 구조와 충돌)
- 반대로, job에 함께 패키징해야 하는 애플리케이션 의존성은 **compile**(또는 이에 준하는)로 설정해 올바르게 포함되게 한다.

## Packaging the application

### 패키징 선택지(의존성 유무에 따라)

- Flink 배포판에 이미 있는 모듈만 쓰고 서드파티가 거의 없다면:
    - 반드시 uber/fat JAR이 필요하지 않을 수 있으며 `installDist`로 구성 가능
- 배포판에 없는 외부 의존성이 필요하다면:
    - (1) 배포판 클래스패스에 추가하거나, (2) 의존성을 **shade**하여 uber/fat JAR로 만든다
    - `installShadowDist` 등으로 단일 fat JAR을 만들고 `flink run -c ...`로 제출 가능

## Connectors and Formats

### 역할

- 커넥터로 외부 시스템에서 읽고/쓰고, 포맷으로 Flink 데이터 구조에 맞게 인코딩/디코딩한다.

### 사용 가능한 아티팩트(배포 형태)

- 커뮤니티 커넥터는 보통 Maven Central에 2종류로 제공:
    - `flink-connector-<NAME>`: **thin JAR**(커넥터 코드 중심, 서드파티 의존성 제외)
    - `flink-sql-connector-<NAME>`: **uber JAR**(서드파티 의존성 포함, 바로 사용 용이)
- 포맷도 유사하며, 일부 커넥터는 서드파티 의존성이 없어 uber 아티팩트가 없을 수도 있다.
- uber JAR은 주로 **SQL client** 사용에 초점이지만 DataStream/Table 애플리케이션에서도 사용 가능.

### 아티팩트 적용 방식 3가지

- thin JAR(+전이 의존성)을 job JAR에 **shade**
- uber JAR을 job JAR에 **shade**
- uber JAR을 Flink 배포판의 `/lib`에 복사(클러스터 전체에 공통 적용)
- 선택 기준:
    - job JAR에 shade하면 **잡 단위로 버전 통제**가 쉬움
    - thin JAR shade는 전이 의존성까지 더 세밀하게 버전 조정 가능(호환성 범위 내)
    - `/lib`에 넣으면 여러 잡에 대해 커넥터 버전을 **한 곳에서 관리** 가능

## Dependencies for Testing

### DataStream API 테스트

- DataStream 잡 테스트를 위해 `flink-test-utils`를 **test scope**로 추가한다.
- 이 모듈은 JUnit 테스트에서 잡을 실행할 수 있는 **MiniCluster** 등 유틸을 제공한다.

### Table API 테스트

- IDE에서 Table/SQL 프로그램을 로컬 테스트하려면 `flink-test-utils` 외에 `flink-table-test-utils`를 **test scope**로 추가한다.

## Advanced Configuration Topics

### Flink distribution 구조(의존성 배치 철학)

- Flink 코어 런타임(조정/네트워킹/체크포인팅/페일오버/API/오퍼레이터/리소스 관리 등)은 `flink-dist.jar`로 묶여 `/lib`에 포함된다.
- 코어 의존성을 작게 유지하고 충돌을 줄이기 위해, 코어에는 기본적으로 **커넥터나 부가 라이브러리(CEP/SQL/ML 등)를 최소화**한다.
- `/lib`에는 Table 실행에 필요한 JAR과 일부 커넥터/포맷이 기본 로딩되며, 제거 시 클래스패스에서 빠진다.
- `/opt`에는 선택(optional) JAR이 있고 `/lib`로 옮겨 활성화할 수 있다.

## Scala Versions

### 호환성 규칙

- Scala 버전은 서로 **바이너리 호환이 아니므로**, Scala에 의존하는 Flink 아티팩트는 `_2.12` 같은 접미사를 가진다.
- Java API만 쓰면 Scala 버전 제약이 상대적으로 덜하지만, Scala API를 쓰면 **애플리케이션 Scala 버전과 맞는** Flink 아티팩트를 선택해야 한다.
- 2.12.8 이후 일부 2.12.x는 바이너리 비호환 이슈가 있어, 더 최신 Scala로는 로컬 빌드 + 호환성 체크 스킵 옵션 등이 필요할 수 있다.

## Anatomy of Table Dependencies

### 기본 포함되는 Table 관련 JAR

- 배포판 `/lib`에는 SQL/Table 잡 실행에 필요한 JAR들이 기본 포함된다:
    - Table Java API uber, table runtime, planner loader 등
- 과거 하나의 `flink-table.jar`에서 여러 JAR로 분리되었고, 필요 시 planner 관련 JAR을 교체해 사용할 수 있다.
- Table Scala API 아티팩트는 기본 포함이 아닐 수 있어, Scala API로 커넥터/포맷을 쓸 때는 `/lib`에 수동 추가(권장) 또는 job uber JAR에 포함해야 할 수 있다.

## Table Planner and Table Planner Loader

### 두 종류의 planner 제공과 제약

- `/opt`의 `flink-table-planner_2.12...`(직접 접근 가능한 플래너)와
- `/lib`의 `flink-table-planner-loader...`(격리된 클래스패스 뒤에 숨긴 플래너)가 존재한다.
- 기본은 planner-loader 사용.
- 플래너 내부(io.apache.flink.table.planner 등)에 접근해야 한다면 JAR을 교체할 수 있지만, **Scala 버전 제약**을 받는다.
- 두 planner JAR은 **동시에 classpath에 존재하면 실패**한다.
- 향후 버전에서는 특정 planner 아티팩트 제공을 중단할 수 있으므로, **플래너 내부 의존을 줄이고 API 모듈 기반으로 마이그레이션**을 권장한다.

## Hadoop Dependencies

### 일반 원칙

- 보통 Hadoop 의존성을 **애플리케이션에 직접 추가할 필요는 없다**.
- Hadoop이 필요하면 “애플리케이션 의존성”이 아니라 “Flink 시스템 구성”에 포함되어야 하며, Flink는 `HADOOP_CLASSPATH`로 Hadoop 클래스를 참조한다.

### 이렇게 설계한 이유

- HDFS 체크포인트 설정, Kerberos 인증, YARN 배포 등은 사용자 코드 시작 전 **Flink 코어에서** 일어날 수 있다.
- Flink의 **inverted classloading**은 전이 의존성 충돌을 숨기고, 애플리케이션이 서로 다른 버전의 의존성을 쓰는 것을 돕는다(대규모 의존성 트리에서 특히 유리).

### 개발/테스트 시

- IDE에서 HDFS 접근 등으로 Hadoop이 필요하다면, Flink 코어 의존성과 유사하게 **test/provided 스코프**로 설정하는 방식이 권장된다.
