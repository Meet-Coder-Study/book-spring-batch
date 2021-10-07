## 7장. ItemReader(2)

### 데이터베이스 입력

- 데이터베이스가 배치 처리에서 훌륭한 입력 소스인 이유
  1. 데이터베이스는 내장된 트랜잭션 기능을 제공한다
     - 이는 대체로 성능이 우수하고 파일보다 훨씬 확장성이 좋다
  2. 다른 입력 포맷보다 훨씬 뛰어난 복구 기능을 기본적으로 제공한다

#### JDBC

- 레코드가 수백만건이 있을때 이를 모두 메모리에 적재하는 건 위험하므로 한 번에 처리할 만큼의 레코드만 로딩하는 방법을 제공한다.

  1. 커서 (1개씩)
     - `ResultSet`으로 구현된다
     - `next()` 메서드를 호출할때마다 데이터베이스에서 레코드를 가져와 반환한다
  2. 페이징 (여러개씩)
     - 커서 방식과는 달리 여러개의 레코드를 한꺼번에 가져올 수 있다

- JDBC 커서 처리

  - 필요한 작업 : 리더 구성, `RowMapper` 구현체 작성

  ```java
  public class CustomerRowMapper implements RowMapper<Customer> {
  
    @Override
    public Customer mapRow(ResultSet rs, int rowNum) throws SQLException {
      // ResultSet으로부터 값을 뽑아서 Customer 객체 생성 후 return
    }
  }
  ```

  ```java
  @Bean
  public JdbcCursorItemReader<Customer> customerItemReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .dataSource(dataSource)
        .sql("select * from customer where city = ?")
        .rowMapper(new CustomerRowMapper())
        .preparedStatementSetter(citySetter(null)) // 받은 객체 배열을 ?에 순서대로 넣는다
        .build();
  }
  
  @Bean
  @StepScope
  public PreparedStatementSetter citySetter(@Value("#{jobParameters['city']}") String city) {
    return new ArgumentPreparedStatementSetter(new Object[]{city});
  }
  ```

  - 스프링 배치가 `JdbcCursorItemReader.read()` 메서드를 호출하면 로우 하나 읽어서 도메인 객체로 매핑함
  - 단점
    - 백만 단위의 레코드를 처리한다면 요청할때마다 네트워크 오버헤드가 추가됨
    - `ResultSet`은 쓰레드 안전이 보장되지 않음 -> 다중 쓰레드 환경에서는 사용할 수 없다

- JDBC 페이징 처리

  - 페이징 기법을 사용하려면 `PagingQueryProvider`의 구현체를 제공하야 한다.
    - 하지만 각 데이터베이스마다 다른 구현체를 제공한다. 그러므로 선택지가 2개 있다
      1. 사용하려는 데이터베이스 전용 `PagingQueryProvider`를 사용한다.
         - `MySqlPagingQueryProvider`, `H2PagingQueryProvider` 등등
      2. `SqlPagingQueryProviderFactoryBean`를 사용하여 사용 중인 데이터베이스를 자동으로 감지하도록 한다

  ```java
    @StepScope
    @Bean
    public JdbcPagingItemReader<Customer> customJdbcPagingItemReader(
        DataSource dataSource,
        PagingQueryProvider queryProvider,
        @Value("#{jobParameters['city']}") String city
    ) {
      Map<String, Object> parameterValues = new HashMap<>(1);
      parameterValues.put("city", city);
      return new JdbcPagingItemReaderBuilder<Customer>()
          .name("customerItemReader")
          .dataSource(dataSource)
          .queryProvider(queryProvider)
          .parameterValues(parameterValues)
          .pageSize(10)
          .rowMapper(new CustomerRowMapper())
          .build();
    }
  
    // 제공되는 DataSource로 작업 중인 데이터베이스 타입을 결정한다
    @Bean
    public SqlPagingQueryProviderFactoryBean pagingQueryProvider(DataSource dataSource) {
      SqlPagingQueryProviderFactoryBean factoryBean = new SqlPagingQueryProviderFactoryBean();
      factoryBean.setSelectClause("select *");
      factoryBean.setFromClause("from customer");
      factoryBean.setWhereClause("where city = :city");
      // 페이징 기법에선 order by절이 반드시 필요하다. 정렬키가 ResultSet내에서 중복되면 안된다(?)
      factoryBean.setSortKey("lastName");
      factoryBean.setDataSource(dataSource);
      return factoryBean;
    }
  ```

#### 하이버네이트

- 배치 처리에서 하이버네이트를 사용하는 건 웹 애플리케이션에서 하이버네이트를 사용하는 것만큼 직관적이지 않다

  - 웹 애플리케이션에서 : 요청이 서버에서 오면 세션을 연다 -> 모든 처리를 동일한 세션에서 처리 -> 뷰를 클라이언트에게 반환하면서 세션을 닫음
  - 배치 처리에서 : 있는 그대로 사용하면 stateful 세션구현체를 사용하게 된다 -> 백만건을 처리한다면 아이템을 캐시에 쌓으면서 `OutOfMemoryException` 이 발생함 -> 하이버네이트 기반 `ItemReader`는 이런 문제를 해결함

- 커서 처리

  - `spring-boot-starter-data-jpa` 의존성 추가 (데이터 접근용x 매핑용o)

    - 해당 의존성을 추가하면 하이버네이트 전용 의존성도 가져올 수 있다

  - 엔티티 매핑

    - `@Entity`, `@Table`, `@Id` 등으로 매핑

  - 배치 잡에서 사용할 `TransactionManager` 커스텀

    - 스프링 배치는 기본적으로 `TransactionManager`로 `DataSourceTransactionManager`를 제공한다
    - 하지만 예제에선 일반적인 `DataSource` 커넥션과 하이버네이트 세션을 아우르는 `TransactionManager`가 필요하다 (왜인지는.. 모르겠음)
    - 이런 목적으로 사용할 수 있는 `HibernateTransactionManager` 을 제공한다. (`DefaultBatchConfigurer`로 구성할 수 있다)

    ```java
    @Component
    public class HibernateBatchConfigurer extends DefaultBatchConfigurer {
    
      private DataSource dataSource;
      private SessionFactory sessionFactory;
      private PlatformTransactionManager transactionManager;
    
      public HibernateBatchConfigurer(
          DataSource dataSource, EntityManagerFactory entityManagerFactory
      ) {
        super(dataSource);
        this.dataSource = dataSource;
        this.sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
        this.transactionManager = new HibernateTransactionManager(this.sessionFactory);
      }
    
      // 이제 스프링 배치는 이 메서드가 반환하는 트랜잭션 매니저를 사용한다
      @Override
      public PlatformTransactionManager getTransactionManager() {
        return this.transactionManager;
      }
    }
    ```

  - `HibernateCursorItemReader` config

    ```java
    @Bean
    @StepScope
    public HibernateCursorItemReader<Customer> customerItemReader(
        EntityManagerFactory entityManagerFactory, @Value("#{jobParameters['city']}") String city
    ) {
      return new HibernateCursorItemReaderBuilder<Customer>()
          .name("customerItemReader")
          .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
        	//여기선 queryString를 썼지만 queryName, queryProvider, nativeQuery도 사용 가능
          .queryString("from Customer where city = :city")
          .parameterValues(Collections.singletonMap("city", city))
          .build();
    }
    ```

- 페이징 기법으로 데이터베이스 접근

  - 커서기법과의 차이점은 `HibernateCursorItemReader`대신 `HibernatePagingItemReader` 을 구성해야 한다는 것과 페이지 크기를 지정하는것이다

  ```java
  @Bean
  @StepScope
  public HibernatePagingItemReader<Customer> customerItemReader(
      EntityManagerFactory entityManagerFactory, @Value("#{jobParameters['city']}") String city
  ) {
    return new HibernatePagingItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
        // 이 예시엔 order by 기준이 없음. 지정되지 않으면 id순. 지정하려면 아래 sql에 order by 추가하기
        .queryString("from Customer where city = :city")
        .parameterValues(Collections.singletonMap("city", city))
        .pageSize(10)
        .build();
  }
  ```

#### JPA

- 하이버네이트는 커서기법 제공O, 페이징기법 제공O

- JPA는 커서기법 제공X, 페이징기법 제공O

- `JpaPagingItemReader`가 필요한 4가지 의존성

  1. `ExecutionContext`내 엔트리의 접두어로 사용되는 이름
  2. 스프링부트가 제공하는 `entityManager`
  3. 실행할 쿼리
  4. 파라미터

  ```java
  @StepScope
  @Bean
  public JpaPagingItemReader<Customer> jpaPagingItemReader(
      EntityManagerFactory entityManagerFactory, @Value("#{jobParameters['city']}") String city
  ) {
    return new JpaPagingItemReaderBuilder<Customer>()
        .name("jpaPagingItemReader")
        .entityManagerFactory(entityManagerFactory)
        .queryString("select c from Customer c where c.city = :city")
        .parameterValues(Collections.singletonMap("city", city))
        .build();
  }
  ```

  `queryString` 말고도 `AbstractJpaQueryProvider`을 이용해서 조회 쿼리를 설정할수도 있다

  ```java
  @StepScope
  @Bean
  public JpaPagingItemReader<Customer> jpaPagingItemReader(
      EntityManagerFactory entityManagerFactory, @Value("#{jobParameters['city']}") String city
  ) {
    CustomerByCityQueryProvider queryProvider = new CustomerByCityQueryProvider();
    queryProvider.setCityName(city);
  
    return new JpaPagingItemReaderBuilder<Customer>()
        .name("jpaPagingItemReader")
        .entityManagerFactory(entityManagerFactory)
        .queryProvider(queryProvider)
        .build();
  }
  ```

  ```java
  @Setter
  public class CustomerByCityQueryProvider extends AbstractJpaQueryProvider {
  
    private String cityName;
  
    @Override
    public Query createQuery() {
      EntityManager manager = getEntityManager();
      Query query = manager.createQuery("select c from Customer c where c.city = :city");
      query.setParameter("city", cityName);
      return query;
    }
  
    @Override
    public void afterPropertiesSet() throws Exception {
      Assert.notNull(cityName, "City name is required");
    }
  }
  ```

#### 저장 프로시저

- 저장 프로시저?

  - 데이터베이스 전용 코드의 집합
  - 스프링배치는 저장 프로시저에서 데이터를 조회하는 용도로 `StoredProcedureItemReader`을 제공한다

  ```sql
  DELIMITER //
  
  CREATE PROCEDURE customer_list(IN cityOption CHAR(16))
    BEGIN
      SELECT * FROM CUSTOMER
      WHERE city = cityOption;
    END //
  
  DELIMITER ;
  ```

  ```java
  @Bean
  @StepScope
  public StoredProcedureItemReader<Customer> customerItemReader(
      DataSource dataSource, @Value("#{jobParameters['city']}") String city
  ) {
    return new StoredProcedureItemReaderBuilder<Customer>()
        .name("StoredProcedureItemReader")
        .dataSource(dataSource)
        .procedureName("customer_list")
        .parameters(new SqlParameter[]{new SqlParameter("cityOption", Types.VARCHAR)})
        .preparedStatementSetter(new ArgumentPreparedStatementSetter(new Object[]{city}))
        .rowMapper(new CustomerRowMapper())
        .build();
  }
  ```

  ```java
  public class CustomerRowMapper implements RowMapper<Customer> {
  
    @Override
    public Customer mapRow(ResultSet rs, int rowNum) throws SQLException {
      ...
    }
  }
  ```

#### 스프링 데이터

- 몽고DB

  - 몽고DB의 특징

    - 테이블을 사용하지 않음
    - 각 데이터베이스는 한 개 또는 그 이상의 컬렉션으로 구성돼 있음
    - 각 컬렉션은 일반적으로 JSON또는 BSON포멧인 문서의 그룹으로 이뤄진다
    - 자바스크립트 또는 JSON기반 쿼리 언어로 검색할 수 있다
    - 속도가 매우 빠르며 사용자가 처리하는 데이터에 맞춰 스키마를 변경할 수 있다
    - 높은 가용성과 확장성 : 기본으로 리플리케이션과 샤딩을 제공한다
    - 지리공간정보 지원 : 몽고DB의 쿼리언어는 특정 지점이 어떤 경계내에 속하는지를 결정하는 질의를 지원한다

  - config

    1. 의존성 추가

       `implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'`

    2. 잡 정의

       ```java
       @Bean
       public Step mongoDbStep() {
         return this.stepBuilderFactory.get("mongoDbStep")
             .<Map, Map>chunk(10)
             .reader(tweetsItemReader(null, null))
             .writer(itemWriter())
             .build();
       }
       
       @Bean
       @StepScope
       public ItemReader<? extends Map> tweetsItemReader(
           MongoOperations mongoTemplate, @Value("#{jobParameters['hashTag']}") String hashtag
       ) {
         return new MongoItemReaderBuilder<Map>()
             .name("tweetsItemReader") //잡을 재시작할 수 있도록 ExecutionContext에 상태를 저장하는데 사용
             .targetType(Map.class) //반환되는 문서를 역직렬화할 대상 클래스
             .jsonQuery("{ \"entities.hashtags.text\": {$eq: ?0 }}")
             .collection("tweets_collection") //쿼리 대상 컬렉션
             .parameterValues(Collections.singletonList(hashtag)) //쿼리에 필요판 파라미터
             .pageSize(10)
             .sorts(Collections.singletonMap("created_at", Direction.ASC)) //정렬기준 필드와 정렬 방법
             .template(mongoTemplate) //쿼리 실행 대상 MongoOperations 구현체
             .build();
       }
       ```

    3. 프로퍼티 추가

       `spring.data.mongodb.database: tweets` 추가

- 스프링 데이터 레포지터리

  - 스프링 데이터가 제공하는 인터페이스 중 하나를 상속하는 인터페이스를 정의하기만 하면 기본적인 CRUD작업을 할 수 있다
  - 스프링 배치가 스프링 데이터의 `PagingAndSortingRepository`를 사용하기 때문에 스프링 데이터와 호환이 좋다
  - `RepositoryItemReader`를 사용하면 스프링 데이터 레포지터리를 지원하는 어떤 데이터 저장소건 상관없이 질의를 할 수 있다

  ```java
  public interface CustomerRepository extends JpaRepository<Customer, Long> {
    Page<Customer> findByCity(String city, Pageable pageRequest);
  }
  ```

  ```java
  @Bean
  @StepScope
  public RepositoryItemReader<Customer> customerItemReader(
      CustomerRepository repository, @Value("#{jobParameters['city']}") String city
  ) {
    return new RepositoryItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .arguments(Collections.singletonList(city))
        .methodName("findByCity")
        .repository(repository)
        .sorts(Collections.singletonMap("lastName", Sort.Direction.ASC))
        .build();
  }
  ```

### 기존 서비스

- 이미 구현되어 있고 테스트도 충분히 마친 코드가 있다면 그대로 이용하고 싶을 것이다.
- `ItemReaderAdapter`을 이용하면 기존 스프링 서비스를 호출해서 `ItemReader`에 데이터를 공급할 수 있다
- `ItemReaderAdapter`을 이용할땐 다음 두가지를 염두해야 한다.
  1. 사용하는 서비스가 컬렉션을 반환한다면 개발자가 직접 컬렉션 내 객체를 하나씩 꺼내면서 처리해야 한다
  2. 입력 데이터를 모두 처리하면 null을 반환해야 한다. (스프링 배치에 해당 스텝의 입력을 모두 소비했음을 나타냄)

```java
@Component
public class CustomerService {
  public Customer getCustomer() {
    //호출될때마다 Customer을 반환, 다 소진되면 null 반환홤
  }
}
```

```java
@Bean
public ItemReaderAdapter<Customer> customerItemReader(CustomerService customerService) {
  ItemReaderAdapter<Customer> adapter = new ItemReaderAdapter<>();
  adapter.setTargetObject(customerService);
  adapter.setTargetMethod("getCustomer");
  return adapter;
}
```

### 커스텀 입력

- 상태를 유지하고 싶을 땐 커스텀 `ItemReader`를 구현해야 한다
- `ItemStreamSupport`를 상속받음으로써 `ItemStream`을 구현하고, `ItemReader`를 구현해야 한다

```java
public class CustomerItemReader extends ItemStreamSupport implements ItemReader<Customer> {

  private List<Customer> customers;
  private final String INDEX_KEY = "current.index.customers";
  private int curIndex;
  
  public CustomerItemReader() {
    customers = new ArrayList<>(); //100명의 고객 정보가 들어 있다고 가정.
    curIndex = 0;
  }

  @Override
  public Customer read() {
    // customers를 하나씩 읽고 반환, curIndex++
    // curIndex가 50이면 예외 발생
  }

  // ItemReader에서 필요한 상태를 초기화하려고 호출
  // 리더의 이전 상태를 알기 위해 ExecutionContext를 사용할 수 있다.
  // 예 : 잡을 재시작할때 이전 상태를 복원, 데이터베이스 연결 등
  @Override
  public void open(ExecutionContext executionContext) {
    if (executionContext.containsKey(getExecutionContextKey(INDEX_KEY))) { //값이 설정되어 있으면 재시작한것
      int index = executionContext.getInt(getExecutionContextKey(INDEX_KEY));
      if (index == 50) {
        curIndex = 51;
      } else {
        curIndex = index;
      }
    } else {
      curIndex = 0;
    }
  }

  // 스프링 배치가 잡의 상태를 갱신하는 처리에 사용됨
  // 리더의 현재 상태(어떤 레코드가 현재 처리중인지)를 알기 위해 ExecutionContext를 사용할 수 있다.
  // 아래 예제는 현재 처리중인 레코드를 나타내는 키-값 쌍을 추가함
  @Override
  public void update(ExecutionContext executionContext) {
    executionContext.putInt(getExecutionContextKey(INDEX_KEY), curIndex);
  }
  
  // 리소스를 닫는데 사용됨
  @Override
  public void close() {
  }
}
```

```java
@Bean
public CustomerItemReader customerItemReader() {
  CustomerItemReader customerItemReader = new CustomerItemReader();
  customerItemReader.setName("customerItemReader");
  return customerItemReader;
}
```

- 이렇게 설정하면 `BATCH_STEP_EXECUTION` 테이블에서 커밋 카운트를 확인할 수 있다

### 에러 처리

- 레코드 건너뛰기

  - 특정 예외가 발생했을 때 건너뛰는 기능을 제공한다
  - 건너뛸지 결정할때 고려사항
    1. 어떤 조건에서 건너뛸 것인가? (어떤 예외를 무시할 것인가?)
    2. 건너뛰는 걸 얼마나 허용할 것인가?
       - `skipLimit(10)` 이면 10번을 넘어가면 실패로 기록됨

  ````java
  @Bean
  public Step step() {
    return this.stepBuilderFactory.get("step")
      .<Customer, Customer>chunk(10)
      .reader(...)
      .writer(...)
      .faultTolerant() // 재시도 기능 활성화
      .skip(Exception.class)
      .noSkip(ParseException.class) // ParseException을 제외한 모든 예외 skip
      .skipLimit(10) // skip이 10번을 넘어가면 실패로 기록
      .build();
  }
  ````

  - 예외마다 skipLimit을 다르게 지정할 수도 있다. ( `SkipPolicy`인터페이스를 구현)

  ```java
  public CustomSkipper implements SkipPolicy {
    
    public boolean shouldSkip(Throwable exception, int skipCount) throws SkipLimitExceedException {
      if (exception instanceof FileNotFoundException) { //FileNotFoundException은 안 건너뜀
        return false
      } else if (exception instanceof ParseException && skipCount <= 10) { //ParseException은 10번까지 건너뜀
        return true;
      } else {
        return false;
      }
    }
  }
  ```

- 로그 남기기

  - `@OnReadError` 로 어떤 메시지를 로그로 남길지 정의하고 `listener`로 등록하면 된다

  ```java
  @Bean
  public Step step() {
    return this.stepBuilderFactory.get("step")
      .<Customer, Customer>chunk(10)
      .reader(...)
      .writer(...)
      .faultTolerant()
      .skipLimit(100)
      .skip(Exception.class)
      .listener(new CustomListener()) // <--
      .build();
  }
  ```

  ```java
  @Slf4j
  public class CustomListener {
    @OnReadError
    public void onReadError(Exception e) {
      if (e instanceof FlatFileParseException) {
        // 데이터베이스를 이용하는 처리를 할 땐 스프링 배치가 담당할 예외 처리가 많지 않으므로 충분한 정보를 담아야 한다
        log.error(...); 
      } else {
        log.error(...);
      }
    }
  }
  ```

- 레코드를 읽었는데 결과가 없는 경우

  - 레코드가 없어도 스프링배치는 완료된 것으로 처리한다 (null이 반환되어도 평상시 null을 받았을 때와 동일하게 처리하기 때문)
  - `@AfterStep`에서 조회한 레코드 수를 확인한 후 레코드가 없다면 실패처리를 한다

  ```java
  @Slf4j
  public class CustomListener {
    // 읽은 레코드 수가 0이라면 실패 처리
    @AfterStep
    public ExitStatus afterStep(StepExecution execution) {
      if (execution.getReadCount() > 0) {
        return execution.getExitStatus();
      } else {
        return ExitStatus.FAILED;
      }
    }
  }
  ```
