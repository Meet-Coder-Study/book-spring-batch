## 5장. `JobRepository`와 메타데이터

### JobRepository란?

- 스프링 배치 내에서 `JobRepository`를 말한 땐 둘 중 하나다

  1. `JobRepository` 인터페이스
  2. `JobRepository` 인터페이스를 구현해 데이터를 저장하는 데 사용되는 데이터 저장소 <- 이 절에선 주로 이걸 가리킨다

- 배치 잡 내부에서 바로 사용할 수 있는 데이터 저장소 2가지

  1. 관계형 데이터베이스

     ![Spring Batch Meta-Data ERD](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/images/meta-data-erd.png)

     - `BATCH_JOB_INSTANCE` : 잡의 논리적 실행
     - `BATCH_JOB_EXECUTION` : 잡의 실제 실행 기록
     - `BATCH_JOB_EXECUTION_PARAMS` : 잡이 매번 실행될때마다 사용된 잡 파라미터 기록
     - `BATCH_JOB_EXECUTION_CONTEXT` : `JobExecution`의 `ExecutionContext` 기록
     - `BATCH_STEP_EXECUTION` : 스텝의 시작, 완료, 상태에 대한 메타데이터 기록
     - `BATCH_STEP_EXECUTION_CONTEXT` : `StepExecution`의 `ExecutionContext` 기록

  2. 인메모리 저장소

     - 개발 단계나 단위테스트를 수행할 때 사용할 수 있게 `Map`기반 인메모리 데이터베이스를 제공한다. 다음 절에서 다룬다.

### 배치 인프라스트럭쳐 config

`@EnableBatchProcessing`을 사용하면 `JobRepository`를 사용할 수 있다.

`JobRepository`를 비롯한 모든 스프링배치 인프라스트럭쳐를 커스터마이징을 하고 싶다면 `BatchConfigurer` 인터페이스를 사용하면 된다

- `BatchConfigurer` 인터페이스

  - 스프링 배치 인프라스트럭쳐 컴포넌트 구성을 커스터마이징하는데 사용되는 전략 인터페이스이다. (일반적으론 `DefaultBatchConfigurer` 을 상속해서 커스터마이징한다.)
  - `@EnableBatchProcessing`을 사용했을때 빈이 추가되는 과정
    1. `BatchConfigurer` 구현체에서 빈을 생성한다
    2. `SimpleBatchConfiguration`에서 스프링의 `ApplicationContext`에 생성한 빈을 등록한다

- `JobRepository`커스터마이징하기

  - 보통 `ApplicationContext`에 두 개 이상의 데이터소스가 존재할때 커스터마이징한다.
  - 예) 업무 데이터 용도의 데이터소스와 `JobRepository`용 데이터소스가 별도로 존재할때

  ```java
  public class CustomBatchConfigurer extends DefaultBatchConfigurer {
  
    @Autowired
    @Qualifier("repositoryDataSource") //"repositoryDataSource"가 어딘가 있다고 가정.
    private DataSource dataSource;
  
    @Override
    protected JobRepository createJobRepository() throws Exception {
      JobRepositoryFactoryBean factoryBean = new JobRepositoryFactoryBean();
      factoryBean.setDatabaseType(DatabaseType.MYSQL.getProductName());
      // 접두어 "BATCH_" -> "FOO_"
      factoryBean.setTablePrefix("FOO_");
      // 데이터 생성시 트랜잭션 격리레벨 "ISOLATION_SERIALIZED" -> "ISOLATION_REPEATABLE_READ"
      factoryBean.setIsolationLevelForCreate("ISOLATION_REPEATABLE_READ");
      factoryBean.setDataSource(this.dataSource);
      // 보통 아래의 두 메서드는 스프링 컨테이너가 호출해준다. 근데 create...메서드 모두는 스프링 컨테이너에 노출되지 않는다. 그래서 개발자가 직접 호출해줘야 한다.
      factoryBean.afterPropertiesSet();
      return factoryBean.getObject();
    }
  }
  ```

- `TransactionManager`커스터마이징하기

  ```java
  public class CustomBatchConfigurer extends DefaultBatchConfigurer {
  
    @Autowired
    @Qualifier("batchTransactionManager") //"batchTransactionManager"가 어딘가 있다고 가정.
    private PlatformTransactionManager transactionManager;
  
    @Override
    public PlatformTransactionManager getTransactionManager() {
      return this.transactionManager;
    }
  }
  ```

- `JobExplorer`커스터마이징하기

  - 배치 메타데이터를 읽기 전용으로 제공하고 싶을 때

  ```java
  public class CustomBatchConfigurer extends DefaultBatchConfigurer {
  
    @Autowired
    @Qualifier("repositoryDataSource")
    private DataSource dataSource;
  
    //JobExplorer와 JobRepository는 동일한 데이터 저장소를 사용하므로 함께 커스터마이징을 하는 게 좋다
    @Override
    protected JobRepository createJobRepository() throws Exception {
      JobRepositoryFactoryBean factoryBean = new JobRepositoryFactoryBean();
      factoryBean.setDatabaseType(DatabaseType.MYSQL.getProductName());
      factoryBean.setTablePrefix("FOO_");
      factoryBean.setIsolationLevelForCreate("ISOLATION_REPEATABLE_READ");
      factoryBean.setDataSource(this.dataSource);
      factoryBean.afterPropertiesSet();
      return factoryBean.getObject();
    }
    
    @Override
    protected JobExplorer createJobExplorer() throws Exception {
      JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();
      factoryBean.setDataSource(this.dataSource);
      factoryBean.setTablePrefix("FOO_");
      factoryBean.afterPropertiesSet();
      return factoryBean.getObject();
    }
  }
  ```

- `JobLauncher`커스터마이징하기

  - 스프링 배치가 기본 제공하는 `SimpleJobLauncher` 외의 방식으로 커스터마이징하고 싶을때 (예 : 컨트롤러를 통해 잡 실행하려 할때)

- 데이터베이스 config

  ```yaml
  spring:
    datasource:
      driverClassName: ...
      url: ...
      username: ...
      password: ...
    batch:
      initialize-schema: ...
  ```

  - `initialize-schema` : 스프링부트가 스프링배치 스키마 스크립트를 실행하도록 지시
    - `always` : 애플리케이션을 실행할때마다 스크립트 실행. 개발 환경일때 사용하기 쉬움
    - `never` : 스크립트를 실행하지 않음
    - `embedded` : 내장 데이터베이스를 사용할때, 각 실행 시마다 데이터가 초기화된 데이터베이스 인스턴스를 사용한다는 가정으로 스크립트를 실행

### 잡 메타데이터 사용하기

어떻게 `JobRepository` 내의 정보를 얻을 수 있을까? 주로 `JobExplorer`을 사용한다.

- `JobExplorer`

  - `JobExplorer`은 `JobRepository`의 데이터에 접근하는 시작점이다.

  - 대부분의 배치 프레임워크 컴포넌트가 `JobRepository`를 사용해 잡 관련 정보에 접근하지만 `JobExplorer`은 데이터베이스에 직접 접근한다.

  - 예시 : 잡의 `JobInstance`가 얼마나 많이 실행됐는지, 각 `JobInstance`당 얼마나 많은 실제 실행이 있었는지 확인하는 config

  - ```java
    public class ExploringTasklet implements Tasklet {
    
      private JobExplorer explorer;
    
      public ExploringTasklet(JobExplorer explorer) {
        this.explorer = explorer;
      }
    
      @Override
      public RepeatStatus execute(StepContribution stepContribution, ChunkContext chunkContext) throws Exception {
        String jobName = chunkContext.getStepContext().getJobName();
        List<JobInstance> instances = explorer.getJobInstances(jobName, 0, Integer.MAX_VALUE);
        System.out.printf("%s 잡에는 %d개의 잡 인스턴스가 존재합니다.", jobName, instances.size());
    
        System.out.println("********************* 결과 *********************");
        for (JobInstance instance : instances) {
          List<JobExecution> jobExecutions = explorer.getJobExecutions(instance);
          System.out.printf("%d 인스턴스에는 %d개의 execution이 존재합니다.",
              instance.getInstanceId(), jobExecutions.size());
          for (JobExecution jobExecution : jobExecutions) {
            System.out.printf("\t%d execution의 ExitStatus 결과는 %s입니다.",
                jobExecution.getId(), jobExecution.getExitStatus());
          }
        }
        return RepeatStatus.FINISHED;
      }
    }
    ```

    ```java
    @Autowired
    private JobExplorer jobExplorer;
    
    @Bean
    public Job explorerJob() {
      return jobBuilderFactory.get("explorerJob")
        .start(explorerStep())
        .build();
    }
    
    @Bean
    public Step explorerStep() {
      return stepBuilderFactory.get("explorerStep")
        .tasklet(explorerTasklet())
        .build();
    }
    
    @Bean
    public Tasklet explorerTasklet() {
      return new ExploringTasklet(jobExplorer);
    }
    ```
