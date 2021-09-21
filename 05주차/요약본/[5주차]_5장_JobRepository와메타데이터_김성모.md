# 5장 JobRepository와 메타데이터

## JobRepository란?

> 스프링 배치 내에서 JobRepository 인터페이스를 구현해 데이터를 저장하는 데 사용되는 데이터 저장소이다. 

### 관계형 데이터베이스 사용하기

관계형 데이터베이스는 스프링 배치에서 기본적으로 제공하는 JobRepository 이다.

![JobRepository - schema](https://user-images.githubusercontent.com/48056463/133590242-b2e5628c-0046-454d-a0dd-6a8d6f27f2de.png)


1. 테이블 설명  

- 시작점인 `BATCH_JOB_INSTANCE`

```sql
CREATE TABLE BATCH_JOB_INSTANCE  (
  JOB_INSTANCE_ID BIGINT  PRIMARY KEY ,
  VERSION BIGINT,
  JOB_NAME VARCHAR(100) NOT NULL ,
  JOB_KEY VARCHAR(2500)
);
```

- 배치 잡의 실제 실행 기록 `BAATCH_JOB_EXECUTION`

> 잡이 실행될 대 마다 새 레코드 생성되고 잡이 진행되는 동안 주기적으로 업데이트 된다. 

```sql
CREATE TABLE BATCH_JOB_EXECUTION  (
  JOB_EXECUTION_ID BIGINT  PRIMARY KEY ,
  VERSION BIGINT,
  JOB_INSTANCE_ID BIGINT NOT NULL,
  CREATE_TIME TIMESTAMP NOT NULL,
  START_TIME TIMESTAMP DEFAULT NULL,
  END_TIME TIMESTAMP DEFAULT NULL,
  STATUS VARCHAR(10),
  EXIT_CODE VARCHAR(20),
  EXIT_MESSAGE VARCHAR(2500),
  LAST_UPDATED TIMESTAMP,
  JOB_CONFIGURATION_LOCATION VARCHAR(2500) NULL,
  constraint JOB_INSTANCE_EXECUTION_FK foreign key (JOB_INSTANCE_ID)
  references BATCH_JOB_INSTANCE(JOB_INSTANCE_ID)
) ;
```

위 테이블과 연관된 세 개의 테이블 

- 연관 1. BATCH_JOB_EXECUTION_CONTEXT

> 잡이 여러 번 실행되는 상황에서 관련 정보를 보관하는 장소 

```sql
CREATE TABLE BATCH_JOB_EXECUTION_CONTEXT  (
  JOB_EXECUTION_ID BIGINT PRIMARY KEY,
  SHORT_CONTEXT VARCHAR(2500) NOT NULL,
  SERIALIZED_CONTEXT CLOB,
  constraint JOB_EXEC_CTX_FK foreign key (JOB_EXECUTION_ID)
  references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
) ;
```


- 연관 2. BATCH_JOB_EXECUTION_PARAMS

> 매번 실행될 때 마다 사용된 잡 파라미터를 보관하는 테이블 

```sql
CREATE TABLE BATCH_JOB_EXECUTION_PARAMS  (
	JOB_EXECUTION_ID BIGINT NOT NULL ,
	TYPE_CD VARCHAR(6) NOT NULL ,
	KEY_NAME VARCHAR(100) NOT NULL ,
	STRING_VAL VARCHAR(250) ,
	DATE_VAL DATETIME DEFAULT NULL ,
	LONG_VAL BIGINT ,
	DOUBLE_VAL DOUBLE PRECISION ,
	IDENTIFYING CHAR(1) NOT NULL ,
	constraint JOB_EXEC_PARAMS_FK foreign key (JOB_EXECUTION_ID)
	references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
);
```

- 연관 3. BATCH_STEP_EXCUTION_CONTEXT

```sql
CREATE TABLE BATCH_STEP_EXECUTION_CONTEXT  (
  STEP_EXECUTION_ID BIGINT PRIMARY KEY,
  SHORT_CONTEXT VARCHAR(2500) NOT NULL,
  SERIALIZED_CONTEXT CLOB,
  constraint STEP_EXEC_CTX_FK foreign key (STEP_EXECUTION_ID)
  references BATCH_STEP_EXECUTION(STEP_EXECUTION_ID)
) ;
```


- 잡에서와 마찬가지로 스텝의 상태를 저장하는 BATCH_STEP_EXECUTION

```sql
CREATE TABLE BATCH_STEP_EXECUTION  (
  STEP_EXECUTION_ID BIGINT  PRIMARY KEY ,
  VERSION BIGINT NOT NULL,
  STEP_NAME VARCHAR(100) NOT NULL,
  JOB_EXECUTION_ID BIGINT NOT NULL,
  START_TIME TIMESTAMP NOT NULL ,
  END_TIME TIMESTAMP DEFAULT NULL,
  STATUS VARCHAR(10),
  COMMIT_COUNT BIGINT ,
  READ_COUNT BIGINT ,
  FILTER_COUNT BIGINT ,
  WRITE_COUNT BIGINT ,
  READ_SKIP_COUNT BIGINT ,
  WRITE_SKIP_COUNT BIGINT ,
  PROCESS_SKIP_COUNT BIGINT ,
  ROLLBACK_COUNT BIGINT ,
  EXIT_CODE VARCHAR(20) ,
  EXIT_MESSAGE VARCHAR(2500) ,
  LAST_UPDATED TIMESTAMP,
  constraint JOB_EXECUTION_STEP_FK foreign key (JOB_EXECUTION_ID)
  references BATCH_JOB_EXECUTION(JOB_EXECUTION_ID)
) ;
```

- BATCH_STEP_EXECUTION_CONTEXT

```sql
CREATE TABLE BATCH_STEP_EXECUTION_CONTEXT  (
  STEP_EXECUTION_ID BIGINT PRIMARY KEY,
  SHORT_CONTEXT VARCHAR(2500) NOT NULL,
  SERIALIZED_CONTEXT CLOB,
  constraint STEP_EXEC_CTX_FK foreign key (STEP_EXECUTION_ID)
  references BATCH_STEP_EXECUTION(STEP_EXECUTION_ID)
) ;
```


## 배치 인프라스터럭쳐 구성하기

> @EnableBatchProcessing 을 사용하면 기본적으로 추가 구성없이 배치가 제공하는 JobRepository를 사용할 수 있다. 

### BatchConfigurer 만들기 
```java
public interface BatchConfigurer {

	JobRepository getJobRepository() throws Exception;

	PlatformTransactionManager getTransactionManager() throws Exception;

	JobLauncher getJobLauncher() throws Exception;

	JobExplorer getJobExplorer() throws Exception;
}
```

위의 인터페이스를 구현한 구현체에서 빈을 생성하고 `SimpleBatchConfiguration`에서 스프링 `ApplicationContext`에 생성한 빈을 등록한다.

기본적으로 스프링배치가 제공하는 `DefaultBatchConfigurer`를 사용하면 모든 기본 옵션이 제공된다. 따라서 몇 개의 구성만 재정의한다면 `DefaultBatchConfigurer` 를 상속해 적절한 메소드를 재정의하는 것이 더 쉽다.


### JoRepository 커스터마이징하기

`JobRepositoryFactoryBean` 이라는 `FactoryBean` 을 통해서 `JobRepository`는 생성된다.

`JobRepositoryFactoryBean` 에서 제공하는 커스터마이징을 set으로 변경하여 커스터마이징할 수 있다.

### TransactionManager 커스터마이징하기


위에서 `DefaultBatchConfigurer` 를 상속해 커스터마이징을 진행하면 쉽다고 말하였는데, 이를 위하여 transaction bean 을 가져와서 getTransactionMnager()에 @Override 해주면 된다.

```java
 public class CustomBatchConfigurer extends DefaultBatchConfigurer {
 
   @Autowired
   @Qualifier("batchTransactionManager")
   private PlatformTransactionManager transactionManager;
 
   @Override
   public PlatformTransactionManager getTransactionManager() {
     return this.transactionManager;
   }
 }
 ```

### JobExplorer 커스터마이징하기

```java
  public class CustomBatchConfigurer extends DefaultBatchConfigurer {
  
    @Autowired
    @Qualifier("batchTransactionManager")
    private DataSource dataSource;
    
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


### JobLauncher 커스터마이징하기

스프링 배치 잡을 실행하는 진입점을 변경해야할 때 구성한다.


## 잡 메타데이터 사용하기

