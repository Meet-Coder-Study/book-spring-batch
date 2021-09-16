# 6장 JobRepository 와 메타데이터
## 1. JobRepository ??
### 1.1 기능
#### (1) 스프링 배치는 잡이 실행될 때 잡의 상태를 JobRepository 에 저장한다.
#### (2) 모니터링으로 사용할 수 있다.

### 1.2 정의
#### (1) Interface
#### (2) 데이터 저장소

### 1.3 RDB
#### (1) BATCH_JOB_INSTANCE 
- 스키마의 시작점
- 잡을 식별하는 고유 정보가 포함된 잡 파라미터로 잡을 처음 실행하면 생성

#### (2) BATCH_JOB_EXECUTION
- 배치 잡의 실제 실행 기록 저장

#### (3) BATCH_JOB_EXECUTION_PARAMS 
- 잡이 매번 실행될 때마다 사용된 잡 파라미터 저장

#### (4) BATCH_JOB_EXECUTION_CONTEXT
- Execution context 관련 정보 저장 
- 잡의 실행 정보 

#### (5) BATCH_STEP_EXECUTION
- 스텝의 시작, 완료, 상태에 대한 메타데이터 저장

#### (6) BATCH_STEP_EXECUTION_CONTEXT
= 스텝 수준에서 컴포넌틍 상태를 저장하는 데 사용 

## 2. 배치 인프라스트럭처 구성 
### 2.1 BatchConfigure 인터페이스 
#### (1) BatchConfigurer 구현체에서 빈을 생성
#### (2) 그다음 SimpleBatchConfiguration 에서 스프링의 ApplicationContext 에 생성한 빈 등록

### 2.2 TransactionManager
#### DefaultBatchConfigurer 가 기본적으로 setDataSource 수정자 내에서 DataSourceTransactionManager 자동 생성

### 2.3 JobExplorer
#### JobExplorer 는 JobRepository 가 다루는 데이터와 동일한 데이터를 읽기 전용으로만 보는 뷰
#### 기본적으로 데이터 접근 계층은 JobRepository 와 JobExplorer 간에 공유되는 동일한 공통 DAO 집합이다.

### 2.4 JobLauncher
#### JobLauncher 는 스프링 배치 잡을 실행하는 진입점
#### 스프링 부트는 기본적으로 스프링 배치가 제공하는 SimpleJobLauncher 를 사용한다. 
