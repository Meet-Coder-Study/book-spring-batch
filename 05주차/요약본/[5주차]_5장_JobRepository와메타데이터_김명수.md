# 배치 인프라스트럭처 구성하기

- [배치 인프라스트럭처 구성하기](#배치-인프라스트럭처-구성하기)
    - [BatchConfigurer 인터페이스](#BatchConfigurer-인터페이스)
    - [JobRepository 커스터마이징하기](#JobRepository-커스터마이징하기)
    - [TransactionManager 커스터마이징하기](#TransactionManager-커스터마이징하기)
    - [JobExplorer 커스터마이징하기](#JobExplorer-커스터마이징하기)
    - [JobLauncher 커스터마이징하기](#JobLauncher-커스터마이징하기)
    - [데이터베이스 구성하기](#데이터베이스-구성하기)
- [잡 메타데이터 사용하기](#잡-메타데이터-사용하기)

---  

# 배치 인프라스트럭처 구성하기

## BatchConfigurer 인터페이스

- `BatchConfigurer` 인터페이스는 스프링 배치 인프라스트럭처 컴포넌트의 구성을 커스터마이징하는 데 사용되는 전략 인터페이스(`strategy interace`)이다.
- `BatchConfigurer` -> `SimpleBatchConfiguration` 으로 빈을 등록하는데 보통 `BatchConfigurer`를 통해 커스터마이징 빈을 등록한다.
- `DefaultBatchConfigurer` 를 통해 필요한 빈만 재정의한다.

```java
public interface BatchConfigurer {

	JobRepository getJobRepository() throws Exception;

	PlatformTransactionManager getTransactionManager() throws Exception;

	JobLauncher getJobLauncher() throws Exception;

	JobExplorer getJobExplorer() throws Exception;
}
```   

---  

## JobRepository 커스터마이징하기

p.190의 표를 살펴보면 `JobRepositoryFactoryBean`의 여러가지 커스터마이징 옵션이 존재한다.

`DefaultBatchConfigurer`를 상속해서 아래와 같이 `JobRepository`를 재정의할 수 있다.

```java
@Slf4j
@Profile("job-repository-custom")
@Configuration
public class CustomBatchConfigurer extends DefaultBatchConfigurer {

    @Autowired
    @Qualifier("repositoryDataSource")
    private DataSource dataSource;

    @Override
    protected JobRepository createJobRepository() throws Exception {
        final JobRepositoryFactoryBean factoryBean = new JobRepositoryFactoryBean();

        factoryBean.setDatabaseType(DatabaseType.MYSQL.getProductName());
        factoryBean.setTablePrefix("FOO_");
        factoryBean.setIsolationLevelForCreate("ISOLATION_REPEATABLE_READ");
        factoryBean.setDataSource(dataSource);

        // BatchConfigurer의 메소드는 스프링에서 직접 노출되지 않으므로
        // InitilizerBean.afterPropertiesSet() 및 FactoryBean.getObject() 메소드를 호출해야한다.
        factoryBean.afterPropertiesSet();

        return factoryBean.getObject();
    }
}
```  

---  

## TransactionManager 커스터마이징하기

`TransactionManager`의 모든 구성 옵션을 다루는 대신 이미 정의된 빈을 기반으로 어떻게 커스터마이징 하는지 살펴보자.

```java
@Slf4j
@Profile("customize-configurer")
@Configuration
public class CustomBatchConfigurer extends DefaultBatchConfigurer {

    @Autowired
    @Qualifier("batchTransactionManager")
    private PlatformTransactionManager transactionManager;

    @Override
    public PlatformTransactionManager getTransactionManager() {
        return transactionManager;
    }
}
```

---  

## JobExplorer 커스터마이징하기

`JobExplorer`는 배치 메타데이터를 읽기 전용으로 제공한다.

```java
@Slf4j
@Profile("customize-configurer")
@Configuration
public class CustomBatchConfigurer extends DefaultBatchConfigurer {

    @Autowired
    @Qualifier("repositoryDataSource")
    private DataSource dataSource;

    @Override
    protected JobExplorer createJobExplorer() throws Exception {
        final JobExplorerFactoryBean factoryBean = new JobExplorerFactoryBean();

        factoryBean.setDataSource(dataSource);
        factoryBean.setTablePrefix("FOO_");

        // BatchConfigurer의 메소드는 스프링에서 직접 노출되지 않으므로
        // InitilizerBean.afterPropertiesSet() 및 FactoryBean.getObject() 메소드를 호출해야한다.
        factoryBean.afterPropertiesSet();

        return factoryBean.getObject();
    }
}
```  

---  

## JobLauncher 커스터마이징하기

- `JobLauncher`는 스프링 배치 잡을 실행하는 진입점이며 `SimpleJobLauncher`를 기본으로 사용한다.
- 스프링 부트의 기본 메커니즘으로 잡을 실행할 때는 대부분 JobLauncher를 커스터마이징 할 필요가 없다.
- 스프링 MVC 애플리케이션의 일부로써 특정 API 호출 시 잡을 실행하고 싶을 때 커스터마이징이 필요하다.

```java
public class SimpleJobLauncher implements JobLauncher, InitializingBean {

	protected static final Log logger = LogFactory.getLog(SimpleJobLauncher.class);

	private JobRepository jobRepository;
	private TaskExecutor taskExecutor;
    
    ...
  
    // 사용할 JobRepository를 지정한다.
	public void setJobRepository(JobRepository jobRepository) {
		this.jobRepository = jobRepository;
	}

    // TaskExecutor를 지정한다. 기본적으로 SyncTaskExecutor가 설정된다.(아래 afterPropertiesSet())
	public void setTaskExecutor(TaskExecutor taskExecutor) {
		this.taskExecutor = taskExecutor;
	}

	@Override
	public void afterPropertiesSet() throws Exception {
		Assert.state(jobRepository != null, "A JobRepository has not been set.");
		if (taskExecutor == null) {
			logger.info("No TaskExecutor has been set, defaulting to synchronous executor.");
			taskExecutor = new SyncTaskExecutor();
		}
	}
}
```  

---  

## 데이터베이스 구성하기

아래와 같은 설정으로 데이터 베이스를 구성할 수 있다.

```yaml
spring:
  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://localhost:53306/spring_batch
    username: root
    password: p@ssw0rd
  batch:
    jdbc:
      # ["always", "never", "embedded"]
      initialize-schema: always
```  

---  

# 잡 메타데이터 사용하기

스프링 배치는 내부적으로 여러 DAO를 사용해 `JobRepository` 테이블에 접근하지만  
프레임워크 개발자가 사용할 수 있도록 `JobExplorer`를 통해 훨씬 실용적인 API를 제공한다.

(p198에 JobExplorer에서 제공하는 여러 메소드에 대한 설명이 있다.)

**1. ExploringTasklet**

```java
@Slf4j
public class ExploringTasklet implements Tasklet {

    private final JobExplorer explorer;

    public ExploringTasklet(JobExplorer explorer) {
        this.explorer = explorer;
    }

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext chunkContext) throws Exception {
        final String jobName = chunkContext.getStepContext().getJobName();
        final List<JobInstance> instances = explorer.getJobInstances(jobName, 0, Integer.MAX_VALUE);

        logger.info("## There are {} job instances for the job {}", instances.size(), jobName);
        logger.info("They have had the following results");
        logger.info("****************************************************");
        for (JobInstance instance : instances) {
            final List<JobExecution> executions = explorer.getJobExecutions(instance);
            logger.info("> Instance {} had {} executions", instance.getInstanceId(), executions.size());

            for (JobExecution execution : executions) {
                logger.info("\tExecution {} resulted in ExitStatus {}", execution.getId(), execution.getExitStatus());
            }
        }

        return RepeatStatus.FINISHED;
    }
}
```  

**2. Job 구성**

```java
@Slf4j
@EnableBatchProcessing
@SpringBootApplication
public class DemoApplication {

  @Autowired
  private JobBuilderFactory jobBuilderFactory;

  @Autowired
  private StepBuilderFactory stepBuilderFactory;

  @Autowired
  private JobExplorer jobExplorer;

  @Bean
  public Tasklet explorerTasklet() {
    return new ExploringTasklet(jobExplorer);
  }

  @Bean
  public Step explorerStep() {
    return stepBuilderFactory.get("explorerStep")
                             .tasklet(explorerTasklet())
                             .build();
  }

  @Bean
  public Job explorerJob() {
    return jobBuilderFactory.get("explorerJob")
                            .start(explorerStep())
                            .incrementer(new RunIdIncrementer())
                            .build();
  }
}
```