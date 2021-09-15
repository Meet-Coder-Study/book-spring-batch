# 5장 - JobRepository와 메타데이터

## JobRepository가 뭔데?

크게 아래 2가지를 의미한다

- JobRepository 인터페이스
    - Job의 Repository 개념 (CRU가 있음)
    - 애플리케이션 내부에 구성하는 것을 빼고는 해당 인터페이스를 직접 다룰 필요가 거의 없음

    ```java
    public interface JobRepository {

    	boolean isJobInstanceExists(String jobName, JobParameters jobParameters);
    	JobInstance createJobInstance(String jobName, JobParameters jobParameters);
    	JobExecution createJobExecution(JobInstance jobInstance, JobParameters jobParameters, String jobConfigurationLocation);
    	void update(JobExecution jobExecution);

      ...

    	StepExecution getLastStepExecution(JobInstance jobInstance, String stepName);
    	JobExecution getLastJobExecution(String jobName, JobParameters jobParameters);
    ```

- JobRepository 데이터 저장소
    - JobRepository 구현체가 사용하는 DB라고 말하지만 **Job과 Step에 관련된 메타데이터의 저장소**가 의미가 더 뚜렷
    - MySQL, H2 뿐만 아니라 java.util에 있는 Map이 될 수도 있음!?!?

## 왜 Job, Step의 메타데이터를 저장해야 되는데?

- 스프링 배치는 대량 데이터 처리를 잘 하기 위해서 나왔음
- 대량 데이터 처리인 만큼 성공, 실패의 의미가 매우 커서 기록을 잘 해놔야 함
    - 예를 들어, 중간에 실패했다면 어디서 실패했는지, Job이 처리되는 시간, 왜 실패했는지 등등...

> 즉, 기록을 잘해야하기 때문에 Job, Step이라는 단위로 JobRepository가 저장하고 관리함

## JobRepository를 구현하는데 필요한 Schema

- 6개의 테이블이 존재함
- BATCH_JOB_INSTANCE, BATCH_JOB_EXECUTION, BATCH_JOB_EXECUTION_CONTEXT, BATCH_JOB_EXECUTION_PARAMS, BATCH_STEP_EXECUTION, BATCH_STEP_EXECUTION_CONTEXT

![스크린샷 2021-09-12 오후 11 23 44](https://user-images.githubusercontent.com/22311557/132996466-fcd245f2-293c-403a-801e-b797fca4c660.png)

### BATCH_JOB_INSTANCE

- Batch Job의 시작점
- Job을 **처음 실행하면** 단일 JobInstance 레코드가 해당 테이블에 등록된다
- Job의 이름, optimistic lock의 version, JobInstance의 고유 key

### BATCH_JOB_EXECUTION
- 배치 잡의 실제 실행 기록
- optimistic lock의 version, Job 실행 시작 시간, 실행 완료 시간, Job의 종료 코드, 종료 코드와 관련된 메시지 등등...
- Job이 실행될 때마다 해당 테이블에 생성됨

### BATCH_JOB_EXECUTION_CONTEXT

- BATCH_JOB_EXECUTION와 연관
- JobExecution의 ExecutionContext를 저장하는 곳
    - Job 컴포넌트의 상태를 저장
    - 저장될 때 Jackson 2로 직렬화돼서 저장됨

### BATCH_JOB_EXECUTION_PARAMS

- Job이 매번 실행될 때마다 사용된 Job의 모든 파라미터들을 저장

### BATCH_STEP_EXECUTION

- 눈치챘겠지만 BATCH_JOB_EXECUTION과 비슷하나 실제 실행 로직이 들어가는 부분이므로  여러 정보가 더 저장된다
- Step의 실행에 대한 메타데이터 저장
- Step의 실행 시작 시간, 실행 완료 시간 이외에도 스텝 실행 중에 커밋된 트랜잭션 수, read Item 수, write Item 수, skip된 Item 수, process 과정 중 skip된 Item 수, 롤백된 트랜잭션 수 등등... 과 같이 거의 모든 모든 데이터가 저장된다.

### BATCH_STEP_EXECUTION_CONTEXT

- StepExecution의 ExecutionContext을 저장하는 곳
    - Step 컴포넌트의 상태를 저장
    - 얘도 직렬화돼서 저장됨

## 위에 말한 JobRepository의 Schema는 누가 생성하는 걸까?

application.yml에 spring.batch.jdbc.initialize-schema 속성이 스키마 생성 스크립트를 실행시켜줌

- always : 애플리케이션을 실행할 때  항상 스키마 생성 스크립트가 실행됨 (이미 존재하면 실행 X)
    - 찾아보면 요런 식으로 되어있음 (spring batch core 의존성에 존재)

    ```sql
    CREATE TABLE BATCH_JOB_INSTANCE  (
    	JOB_INSTANCE_ID BIGINT  NOT NULL PRIMARY KEY ,
    	VERSION BIGINT ,
    	JOB_NAME VARCHAR(100) NOT NULL,
    	JOB_KEY VARCHAR(32) NOT NULL,
    	constraint JOB_INST_UN unique (JOB_NAME, JOB_KEY)
    ) ENGINE=InnoDB;

    ...
    ```

    - 매번 실행하면 같은 테이블을 실행하므로 오류가 나는데 always는 오류가 나면 무시함. 그래서 계속 실행해도 이미 존재하면 무시해버리는 것
- never : 스크립트를 실행하지 않음
- embedded : 내장 DB를 사용할 때만 스키마 생성 스크립트가 실행됨 (아마 테스트용인듯?)

## 근데... 사용하려면 이런 JobRepository Schema에 직접 접근해야하나?

- 아니다. JobExplorer라는 조회용 API를 제공하고 있다.

![스크린샷 2021-09-13 오전 1 37 09](https://user-images.githubusercontent.com/22311557/132996505-9747e291-5b76-497d-b762-95b6332bfde3.png)


- 등등...

### 예시

```kotlin
@EnableBatchProcessing
@SpringBootApplication
class ExploringJob(
    @Autowired private val jobBuilderFactory: JobBuilderFactory,
    @Autowired private val stepBuilderFactory: StepBuilderFactory,
    @Autowired private val jobExplorer: JobExplorer, // 주입해주고 사용
) {

    @Bean
    fun explorerJob(): Job {
        return this.jobBuilderFactory
            .get("explorerJob")
            .start(explorerStep())
            .build()
    }

    @Bean
    fun explorerStep(): Step {
        return this.stepBuilderFactory
            .get("explorerStep")
            .tasklet(explorerTasklet())
            .build()
    }

    @Bean
    fun explorerTasklet(): Tasklet {
        return ExploringTasklet(this.jobExplorer)
    }
}

class ExploringTasklet(
    private val explorer: JobExplorer,
) : Tasklet {
    override fun execute(contribution: StepContribution, chunkContext: ChunkContext): RepeatStatus? {
        val jobName = chunkContext.stepContext.jobName
        val instances = explorer.getJobInstances(jobName, 0, Int.MAX_VALUE)

        println("There are ${instances.size} job instances for the job $jobName")
        println("*****************결과는 아래있음*****************")

        instances.forEach {
            val jobExecutions = this.explorer.getJobExecutions(it)

            println("instance $it had ${jobExecutions.size}")

            jobExecutions.forEach { execution -> println("executionId : ${execution.id}, ExitStatus : ${execution.exitStatus}") }
        }
        return RepeatStatus.FINISHED
    }
}

fun main(args: Array<String>) {
    runApplication<ExploringJob>(*args)
}
```

## Spring Batch Config를 커스터마이징할 순 없을까?

- 당연히 가능하다
- BatchConfigurer에 해당하는 컴포넌트들을 커스터마이징할 수 있다

```java
public interface BatchConfigurer {

	JobRepository getJobRepository() throws Exception;

	PlatformTransactionManager getTransactionManager() throws Exception;

	JobLauncher getJobLauncher() throws Exception;

	JobExplorer getJobExplorer() throws Exception;
}
```

- BatchConfigurer의 구현체인 DefaultBatchConfigurer를 사용하면 모든 기본 옵션이 제공된다. 일반적으로는 저 4개 중에서 한 두 가지만 설정을 바꾸므로 모두 새로 만들기보다는 DefaultBatchConfigurer를 상속해서 재정의하는 게 더 쉽다.
- 그리고 해당 컴포넌트들은 각 이름의 FactoryBean을 통해 생성됨
- FactoryBean은 각 속성을 커스터마이징할 수 있음

### JobRepository 커스터마이징

- JobRepository의 DataSource를 다르게 할 때 많이 씀
    - 예를 들어, 상용 DB에 JobRepository를 넣기에는 역할이 분명하지 않으므로 다른 DB에 JobRepository를 넣는다던가...
- 책 예시에는 안 나와있지만 setSerializer를 통해 ExecutionContext Serializer 구현체를 설정해줄 수 있음 (위에서 말한 거)

```java
class CustomJobRepositoryBatchConfigurer(
    @Autowired @Qualifier("repositoryDataSource")
    private val dataSource: DataSource,
): DefaultBatchConfigurer() {
    override fun createJobRepository(): JobRepository {
        val factoryBean = JobRepositoryFactoryBean()
        factoryBean.setDataSource(this.dataSource)
        factoryBean.setTablePrefix("FOO_")
        factoryBean.setIsolationLevelForCreate("ISOLATION_REPEATABLE_READ")
        factoryBean.afterPropertiesSet()
        return factoryBean.`object`
    }
}
```

### TransactionManger 커스터마이징

- 마찬가지로 다른 TransactionManager를 사용하고 싶을 때
- TransactionManager가 생성되지 않은 경우에는 DefaultBatchConfigurer가 자동적으로 DataSourceTransactionManger를 생성

```java
class CustomTransactionManagerBatchConfigurer(
    @Autowired @Qualifier("batchTransactionManager")
    private val transactionManager: PlatformTransactionManager,
): DefaultBatchConfigurer() {
    override fun getTransactionManager(): PlatformTransactionManager {
        return this.transactionManager
    }
}
```

### JobExplorer 커스터마이징

- JobExplorer란 DB에 저장된 JobRepository 관련 데이터를 조회(읽기) 전용으로 제공해줌
- JobRepository와 JobExplorer 간에 공유되는 동일한 DAO 집합이다. 그래서 JobRepository를 커스터마이징한 것과 속성이 같다.

```java
class CustomJobExplorerBatchConfigurer(
    @Autowired @Qualifier("batchJobExplorer")
    private val dataSource: DataSource,
): DefaultBatchConfigurer(dataSource) {
    override fun createJobExplorer(): JobExplorer {
        val factoryBean = JobExplorerFactoryBean()
        factoryBean.setDataSource(this.dataSource)
        factoryBean.setTablePrefix("FOO_")
        factoryBean.afterPropertiesSet()
        return factoryBean.`object`
    }
}
```

### JobLauncher 커스터마이징

- JobLauncher란 스프링 배치 Job을 실행하는 진입 점
- 기본적으로 SimpleJobLauncher를 사용한다. 책에는 안 나와있지만 SimpleJobLauncher를 상속해서 커스터마이징하면 될 것 같다. (다른 JobLauncher가 없음)
    - SimpleJobLauncher에 setJobRepository, setTaskExecutor가 있다. 얘네들을 커스터마이징
- CommandLine으로 실행하는 것이 아닌 MVC 애플리케이션의 일부분으로 컨트롤러를 통해 실행할 때 사용

## 번외

### SimpleBatchConfiguration 디버깅해보기

- "BatchConfigurer 구현체에서 빈을 생성한다. 그 다음 SimpleBatchConfiguration에서 스프링의 ApplicationContext에 생성한 빈을 등록한다." 라고 책에 쓰여있음
    - 근데 실제로 디버깅해보니 시작하자마자
    - SimpleBatchConfiguration에서 createLazyProxy로 프록시가 먼저 생성됨
    - BatchConfigurer의 구현체인 BasicBatchConfigurer에서 아까 만든 프록시로 initialize된다
    - 그리고 다시 SimpleBatchConfiguration에서 initialize된다
    - 더 깊게 파면 한도끝도 없을 것 같아 스톱함ㅎㅎ

### JobRepository의 구현체로 java.util.Map이 된다는데 언제 되는가?

- 아래 사진처럼 setDataSource를 아예 안 적을 때가 있는데 이럴 때 Map이 구현체로 쓰인다고 함
- MapJobRepositoryFactoryBean이 이용됨

    ![스크린샷 2021-09-13 오전 1 43 26](https://user-images.githubusercontent.com/22311557/132996509-7f9630cb-67c2-4a74-8f06-024c617ce196.png)

- 당연히 메모리에 저장되므로(인메모리) 상용에서는 사용하지 말 것!
- 출처 : [https://stackoverflow.com/questions/44238232/define-an-in-memory-jobrepository](https://stackoverflow.com/questions/44238232/define-an-in-memory-jobrepository)

## 요약

- JobRepository Schema의 구조 알기

    ![스크린샷 2021-09-12 오후 11 23 44](https://user-images.githubusercontent.com/22311557/132996466-fcd245f2-293c-403a-801e-b797fca4c660.png)

- JobRepository Schema에 접근은 JobExplorer를 통해 하자
- Spring Batch Config의 커스터마이징은 DefaultBatchConfigurer를 상속해서 구현하며 JobRepository, TransactionManager, JobLauncher, JobExplorer를 설정해줄 수 있다

> [실습 링크](https://github.com/aegis1920/my-lab/tree/master/def-guide-spring-batch)
