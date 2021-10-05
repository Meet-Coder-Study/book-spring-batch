# 목차
- [데이터베이스 입력](#데이터베이스-입력)
  - [ItemReader Hierarchy](#ItemReader-Hierarchy) 
  - [JDBC](#JDBC)
    - [JDBC 커서 처리](#JDBC-커서-처리)
    - [JDBC 페이징 처리](#JDBC-페이징-처리)
  - [하이버네이트](#하이버네이트)
    - [하이버네이트 커서 처리](#하이버네이트-커서-처리)
    - [하이버네이트 패이징 처리](#하이버네이트-페이징-처리)
  - [JPA](#JPA)
    - [JPA 커서 처리](#JPA-커서-처리)
    - [JPA 페이징 처리](#JPA-페이징-처리)
  - [저장 프로시저](#저장-프로시저)
  - [스프링 데이터](#스프링-데이터)
- [기존 서비스](#기존-서비스)
- [커스텀 입력](#커스텀-입력)
- [에러 처리](#에러-처리)
  - [레코드 건너뛰기/잘못된 레코드 로그 남기기](#레코드-건너뛰기/잘못된-레코드-로그-남기기)
- [References](#References)

---  

# 데이터베이스 입력

데이터베이스가 배치 처리에서 훌륭한 입력 소스인 이유는 아래와 같습니다.
- 내장된 트랜잭션 기능 제공
- 다른 입력 포맷보다 훨씬 뛰어난 복구 기능을 기본으로 제공
- **엔터프라이즈 환경에서 사용하는 데이터가 시작부터 관계형 데이터베이스에 저장**

Batch는 일반 애플리케이션과는 다르게 많은 양의 데이터를 다룹니다.

만약 조회 쿼리에서 백만건의 데이터를 한번에 조회한다면 데이터베이스 부하 및 서버에서 해당 데이터를 모두 메모리에 유지해야합니다.

이러한 문제를 해결하기 위해 Spring Batch는 **Cursor-based ItemReader**와 **Paging ItemReader** 구현체들을 제공합니다.

![Chunk vs Paging](https://user-images.githubusercontent.com/25560203/135753725-d8dc05f0-8a25-4afc-803c-5d29a328f738.png)

- **Chunk-based**
  - 표준 `java.sql.ResultSet`으로 구현
  - `read`가 호출 될때 마다 ResultSet의 커서를 이동
- **Paging**
  - (고유 페이지 관련 쿼리를 통해 생성하여) 페이지라고 부르는 청크 크기만큼의 레코드를 가져옴
  - 하나의 페이지 레코드를 모두 읽은 경우, 새로운 다음 페이지 쿼리를 통해 데이터베이스에서 새 페이지 조회
  - 아래에서 소개될 XXXPagingItemReader는 Builder에서 기본값 pageSize를 10으로 설정하여 이용합니다.

<br />  

## ItemReader Hierarchy  

![ItemReader Hierarchy](https://user-images.githubusercontent.com/25560203/136023506-ee51484d-590f-4ff8-9b2e-46c7ef6e5901.png)  

<details>
<summary>:ballot_box_with_check: ItemReader</summary>  

[ItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/ItemReader.java)는 단일 아이템을 읽어오는 전략 인터페이스입니다.  

```java
package org.springframework.batch.item;

public interface ItemReader<T> {

  @Nullable
  T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;
}
```
</details>

<details>
<summary>:ballot_box_with_check: ItemStream</summary>

[ItemStream](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/ItemStream.java)는 주기적으로 상태를 저장하고 에러가 발생하면 복구하기 위한 마커 인터페이스입니다.  

```java
package org.springframework.batch.item;

public interface ItemStream {

	void open(ExecutionContext executionContext) throws ItemStreamException;

	void update(ExecutionContext executionContext) throws ItemStreamException;

	void close() throws ItemStreamException;
}
```
</details>  

<details>
<summary>:ballot_box_with_check: Example Data Sets</summary>  

해당 챕터에서는 데이터 베이스에 저장된 `Customer` 도메인 객체를 조회하는 `ItemReader`에 대하여 학습합니다.

전체 데이터는 [data-mysql.sql](https://github.com/AcornPublishing/definitive-spring-batch/blob/main/def-guide-spring-batch-master/Chapter07/src/main/resources/data-mysql.sql)에서 확인할 수 있습니다.

```java
public class Customer {

  private Long id;

  private String firstName;
  private String middleInitial;
  private String lastName;
  private String address;
  private String city;
  private String state;
  private String zipCode;

  ...
}  
```

```sql
CREATE TABLE IF NOT EXISTS customer (
    id            BIGINT      NOT NULL PRIMARY KEY,
    firstName     VARCHAR(11) NOT NULL,
    middleInitial VARCHAR(1),
    lastName      VARCHAR(20) NOT NULL,
    address       VARCHAR(45) NOT NULL,
    city          VARCHAR(16) NOT NULL,
    state         CHAR(2)     NOT NULL,
    zipCode       CHAR(5)
);
```
</details>  

## JDBC

### JDBC 커서 처리

[JdbcCursorItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JdbcCursorItemReader.java)를 통해 커서 기반의 ItemReader를 구현합니다.

![JdbcCursorItemReader](https://user-images.githubusercontent.com/25560203/135755688-c0d52b03-2d68-4365-a1ed-dfe3ed00ac20.png)

`JdbcCursorItemReaderBuilder`를 살펴보면 `DataSource`, `sql`, `RowMapper`, `PreparedStatementSetter` 등이 필요한 것을 확인할 수 있습니다.

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

해당 배치 예제에서는 특정 도시에 사는 고객만 처리합니다. 특정 도시(city)는 배치의 JobParameter로 전달받습니다.

**(1) RowMapper 구현하기**

쿼리 결과 도메인을 Customer 객체로 매핑하기 위해 (ResultSet -> Customer) 아래와 같은 `CustomerRowMapper`를 작성합니다.

```java
public class CustomerRowMapper implements RowMapper<Customer> {

    @Override
    public Customer mapRow(ResultSet resultSet, int i) throws SQLException {
        return Customer.builder()
                       .id(resultSet.getLong("id"))
                       .address(resultSet.getString("address"))
                       .city(resultSet.getString("city"))
                       .firstName(resultSet.getString("firstName"))
                       .lastName(resultSet.getString("lastName"))
                       .middleInitial(resultSet.getString("middleInitial"))
                       .state(resultSet.getString("state"))
                       .zipCode(resultSet.getString("zipCode"))
                       .build();
    }
}
```

**(2) PreparedStatementSetter 빈 구성 하기**

특정 도시만 사는 고객을 조회하기 위해(`where city = ?`) 스프링 프레임워크가 제공하는 `ArgumentPreparedStatementSetter`를 이용합니다.  
`ArgumentPreparedStatementSetter`는 Object 배열이 필요한데 아래와 같이 `SqlParameterValue` 타입 캐스트 여부에 따라 데이터 타입과 값을 설정합니다.

```java
public class ArgumentPreparedStatementSetter implements PreparedStatementSetter, ParameterDisposer {
  ...
  public ArgumentPreparedStatementSetter(@Nullable Object[] args) {
		this.args = args;
  }
  ...	
  protected void doSetValue(PreparedStatement ps, int parameterPosition, Object argValue) throws SQLException {
      if (argValue instanceof SqlParameterValue) {
          SqlParameterValue paramValue = (SqlParameterValue) argValue;
          StatementCreatorUtils.setParameterValue(ps, parameterPosition, paramValue, paramValue.getValue());
      }
      else {
          StatementCreatorUtils.setParameterValue(ps, parameterPosition, SqlTypeValue.TYPE_UNKNOWN, argValue);
      }
  }
  ...
}
```  

다음으로 `ArgumentPreparedStatementSetter`를 빈으로 등록합니다.

```java
@Bean
@StepScope
public ArgumentPreparedStatementSetter citySetter(@Value("#{jobParameters['city']}") String city) {
    return new ArgumentPreparedStatementSetter(new Object[] { city });
}
```  

**(3) ItemReader 빈 구성 하기**

위에서 생성한 `PreparedStatementSetter`와 `RowMapper`를 통해 `JdbcCursorItemReader`를 구성합니다.

```java
@Bean
public JdbcCursorItemReader<Customer> customerItemReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM customer WHERE city = ?")
            .rowMapper(new CustomerRowMapper())
            .preparedStatementSetter(citySetter(null))
            .build();
}
```  

**(4) 잡 실행 및 결과 확인**

```sql
// `city='Dover'`인 레코드는 총 11개 존재합니다.  
SELECT count(1) FROM customer WHERE city = 'Dover';
+----------+
| count(1) |
+----------+
|       11 |
+----------+


// 11개의 레코드에 대하여 Chunk Size==3이므로 4번의 커밋이 일어나는것을 확인할 수 있습니다.
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 07:37:15.323000 | 2021-10-04 07:37:15.430000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 07:37:15.432000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
```  

Spring batch 문서에 [Additional Properties](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/readersAndWriters.html#JdbcCursorItemReaderProperties)에는 `JdbcCursorItemReader`에 대한 추가적인 설정이 있습니다.  
(`queryTimeout`, `fetchSize` 등)

---  

### JDBC 페이징 처리

[JdbcPagingItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JdbcPagingItemReader.java)를 통해 페이징 기법으로 ItemReader를 구현합니다.

![JdbcPagingItemReader](https://user-images.githubusercontent.com/25560203/135814061-d6793558-ba5f-411f-b9dc-de23eeb85a4c.png)

**Note**: 데이터베이스에서 정해진 chunk 사이즈 만큼 조회하지만 ItemReader는 모두 하나의 아이템씩 처리합니다.

`JdbcPagingItemReader`의 필드를 살펴보면 `DataSource`, `PagingQueryProvider`, `RowMapper`, `pageSize` 등이 필요한 것을 확인할 수 있습니다.

`PagingQueryProvider`는 아래와 같은 Spec을 가지고 있습니다.

<details>
<summary>PagingQueryProvider</summary>

```java
package org.springframework.batch.item.database;
...
public interface PagingQueryProvider {

	void init(DataSource dataSource) throws Exception;
	String generateFirstPageQuery(int pageSize);
	String generateRemainingPagesQuery(int pageSize);
	String generateJumpToItemQuery(int itemIndex, int pageSize);
	int getParameterCount();
	boolean isUsingNamedParameters();
	Map<String, Order> getSortKeys();
	String getSortKeyPlaceHolder(String keyName);
	Map<String, Order> getSortKeysWithoutAliases();
}
```
</details>  

`PagingQueryProvider`는 페이징 관련 쿼리를 제공합니다. 스프링 배치는 DB2, Derby, H2, HSql, MySQL, Oracle, Postgres, SqlServer, Sybase 구현체를 제공합니다.

```java
package org.springframework.batch.item.database.support;

public class SqlPagingQueryProviderFactoryBean implements FactoryBean<PagingQueryProvider> {
  ...
  private Map<DatabaseType, AbstractSqlPagingQueryProvider> providers = new HashMap<>();
  
  {
      providers.put(DB2, new Db2PagingQueryProvider());
      providers.put(DB2VSE, new Db2PagingQueryProvider());
      providers.put(DB2ZOS, new Db2PagingQueryProvider());
      providers.put(DB2AS400, new Db2PagingQueryProvider());
      providers.put(DERBY,new DerbyPagingQueryProvider());
      providers.put(HSQL,new HsqlPagingQueryProvider());
      providers.put(H2,new H2PagingQueryProvider());
      providers.put(MYSQL,new MySqlPagingQueryProvider());
      providers.put(ORACLE,new OraclePagingQueryProvider());
      providers.put(POSTGRES,new PostgresPagingQueryProvider());
      providers.put(SQLITE, new SqlitePagingQueryProvider());
      providers.put(SQLSERVER,new SqlServerPagingQueryProvider());
      providers.put(SYBASE,new SybasePagingQueryProvider());
  }
  ...
}
```  

`SqlPagingQueryProviderFactoryBean`를 사용하여 인자로 전달받은 `DataSource`를 통해 데이터베이스를 자동 감지해 적절한 PagingQueryProvider로 반환해줍니다.  
(DataSource의 Connection의 getDatabaseProductName() 메소드를 통해 Database Vendor를 조회합니다)

<br />  

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) SqlPagingQueryProviderFactoryBean를 이용해 PagingQueryProvider 빈 등록하기**

```java
@Bean
public SqlPagingQueryProviderFactoryBean pagingQueryProvider(DataSource dataSource) {
    final SqlPagingQueryProviderFactoryBean factoryBean = new SqlPagingQueryProviderFactoryBean();

    factoryBean.setSelectClause("SELECT *");
    factoryBean.setFromClause("FROM customer");
    factoryBean.setWhereClause("WHERE city = :city"); // named parameter 사용
    factoryBean.setSortKey("lastName"); // 다음페이지에 추가됩니다.
    factoryBean.setDataSource(dataSource);

    return factoryBean;
}
```

**(2) ItemReader 빈 구성하기**

위에서 생성한 `QueryProvider`와 `pageSize`, `RowMapper` 등을 설정한 ItemReader를 빈으로 등록합니다.

```java
@Bean
@StepScope
public JdbcPagingItemReader<Customer> customerItemReader(DataSource dataSource,
                                                         PagingQueryProvider queryProvider,
                                                         @Value("#{jobParameters['city']}") String city) {
    final Map<String, Object> parameterValues = ImmutableMap.of("city", city);
    return new JdbcPagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .dataSource(dataSource)
            .queryProvider(queryProvider)
            .parameterValues(parameterValues)
            .pageSize(10)
            .rowMapper(new CustomerRowMapper())
            .build();
}
```  

**(3) 잡 실행 및 결과 확인**

Console Log 확인하면 `sortKey`(`lastName`)를 기준으로 다음 페이징에 사용됩니다.

```log
// Chunk-1 ItemReader query
2021-10-04 17:31:57.433 DEBUG 7407 --- [           main] o.s.batch.repeat.support.RepeatTemplate  : Starting repeat context.
2021-10-04 17:31:57.433 DEBUG 7407 --- [           main] o.s.batch.repeat.support.RepeatTemplate  : Repeat operation about to start at count=1
2021-10-04 17:31:57.435 DEBUG 7407 --- [           main] o.s.b.i.database.JdbcPagingItemReader    : Reading page 0
2021-10-04 17:31:57.435 DEBUG 7407 --- [           main] o.s.b.i.database.JdbcPagingItemReader    : SQL used for reading first page: [SELECT * FROM customer WHERE city = :city ORDER BY lastName ASC LIMIT 3]
2021-10-04 17:31:57.440 DEBUG 7407 --- [           main] o.s.b.i.database.JdbcPagingItemReader    : Using parameterMap:{city=Dover}
2021-10-04 17:31:57.447  INFO 7407 --- [           main] p6spy                                    : #1633336317447 | took 4ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
SELECT * FROM customer WHERE city = 'Dover' ORDER BY lastName ASC LIMIT 3;

// Chunk-2 ItemReader query
SELECT * FROM customer WHERE (city = 'Dover') AND ((lastName > 'Bradford')) ORDER BY lastName ASC LIMIT 3;
// Chunk-3 ItemReader query
SELECT * FROM customer WHERE (city = 'Dover') AND ((lastName > 'Langley')) ORDER BY lastName ASC LIMIT 3;
```  

`BATCH_STEP_EXECUTION`를 살펴보면 총 11개의 레코드를 처리했고 ChunkSize==3을 기준으로 4번의 커밋이 이루어졌습니다.  
또한 `BATCH_STEP_EXECUTION_CONTEXT`를 살펴보면 "customerItemReader.start.after":{"@class":"java.util.LinkedHashMap","lastName":"York"}를 통해 마지막 Item의 SortKey(lastName)을 기록합니다.

```shell
mysql > SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 08:31:57.374000 | 2021-10-04 08:31:57.542000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 08:31:57.544000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

-- Step Execution Context에 "customerItemReader.start.after":{"@class":"java.util.LinkedHashMap","lastName":"York"}를 통해 마지막 Item의 SortKey(lastName)을 기록합니다.
mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                                                                                                            | SERIALIZED_CONTEXT |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","customerItemReader.start.after":{"@class":"java.util.LinkedHashMap","lastName":"York"},"batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+

mysql> SELECT * FROM customer WHERE city = 'Dover' ORDER BY lastName DESC LIMIT 1;
+-----+-----------+---------------+----------+------------------+-------+-------+---------+
| id  | firstName | middleInitial | lastName | address          | city  | state | zipCode |
+-----+-----------+---------------+----------+------------------+-------+-------+---------+
| 233 | Belle     | Y             | York     | 709-1516 Eu, Av. | Dover | DE    | 64844   |
+-----+-----------+---------------+----------+------------------+-------+-------+---------+
```  

**JdbcPagingItemReader 주의 사항**

위의 내용에서 확인했듯이 JdbcPagingItemReader는 `NoOffset` 기반으로 데이터를 조회합니다.

아래와 같이 `sortKey`인 `lastName`이 같은게 있는 상황에서 청크의 마지막 아이템에 포함될 경우 다른 레코드가 누락될 수 있습니다.

```shell
+------+------------+---------------+-------------+----------------------------------------+------------------+-------+---------+
| id   | firstName  | middleInitial | lastName    | address                                | city             | state | zipCode |
+------+------------+---------------+-------------+----------------------------------------+------------------+-------+---------+
|    1 | Melinda    | A             | Frank       | P.O. Box 290, 520 Hendrerit. Ave       | Juneau           | AK    | 99658   | <-- 첫번쨰 Chunk Item1
|    2 | Stone      | B             | Todd        | Ap #365-6856 Vestibulum. St.           | Gary             | IN    | 83969   | <-- 첫번쨰 Chunk Item2
|    3 | Aileen     | C             | Hoffman     | 411-6828 Eu St.                        | Stamford         | CT    | 97699   | <-- 첫번쨰 Chunk Item3
|    4 | Jared      | D             | Hoffman     | 7638 Nulla. Rd.                        | Tuscaloosa       | AL    | 36105   | 
|    5 | Igor       | E             | Young       | Ap #296-2433 Diam. Av.                 | Norman           | OK    | 28800   | <-- 두번째 Chunk Item1

```

---  

## 하이버네이트

### 하이버네이트 커서 처리

[HibernateCursorItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/HibernateCursorItemReader.java)를 이용하여 커서 기반의 ItemReader를 구현합니다.

![HibernateCursorItemReader](https://user-images.githubusercontent.com/25560203/135823880-ec7e82af-812e-41e2-971a-c5b44afa409e.png)

하이버네이트 커서를 사용하려면 `sessionFactory`, `Customer 매핑`가 필요합니다.

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) 의존성 추가하기**

`spring-boot-starter-jpa`를 이용하여 하이버네이트 전용 의존성을 모두 가져옵니다.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```  

**(2) Entity 및 애플리케이션 설정**

```java
@Entity
@Table(name = "customer")
@Getter
@Setter(AccessLevel.PROTECTED)
@NoArgsConstructor
public class Customer {

    @Id
    private Long id;

    @Column(name = "firstName")
    private String firstName;
    @Column(name = "middleInitial")
    private String middleInitial;
    @Column(name = "lastName")
    private String lastName;
    @Column(name = "address")
    private String address;
    @Column(name = "city")
    private String city;
    @Column(name = "state")
    private String state;
    @Column(name = "zipCode")
    private String zipCode;

    ... 
}
```

```yaml
spring:
  jpa:
    hibernate:
      naming:
        implicit-strategy: org.hibernate.boot.model.naming.ImplicitNamingStrategyLegacyJpaImpl
        physical-strategy: org.hibernate.boot.model.naming.PhysicalNamingStrategyStandardImpl
```  

(어노테이션으로 적용했는데, yaml 설정도 필요가 있을지?!)

**(3) BatchConfigurer**

배치잡에서 사용할 `TransactionManager`를 커스터마이징합니다.  
스프링 배치 기본 `TransactionManager`는 `DataSourceTransactionManager`이므로 `DataSource`와 하이버네이트 세션을 아우르는 `TransactionManager`인  
`HibernateTransactionManager`가 필요합니다.

```java
@Component
public class HibernateBatchConfigurer extends DefaultBatchConfigurer {

    private final PlatformTransactionManager transactionManager;

    @Autowired
    public HibernateBatchConfigurer(DataSource dataSource,
                                    EntityManagerFactory entityManagerFactory) {
        super(dataSource);
        this.transactionManager = new HibernateTransactionManager(entityManagerFactory.unwrap(SessionFactory.class));
    }

    @Override
    public PlatformTransactionManager getTransactionManager() {
        // 4.2 이전 버전에서는 로그 메시지로 DataSourceTransactionManager 로 출력되는 버그 있음
        return this.transactionManager;
    }
}
```  

**(4) ItemReader 빈 구성하기**

`queryString`, `queryName`, `queryProvider`, `nativeQuery`로 쿼리를 제공합니다.

```java
@Bean
@StepScope
public HibernateCursorItemReader<Customer> cursorItemReader(EntityManagerFactory entityManagerFactory,
                                                            @Value("#{jobParameters['city']}") String city) {

    return new HibernateCursorItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
            .queryString("from Customer where city = :city")
            // .queryName(String) // 하이버네이트 구성에 포함된 네임드 하이버네이트 쿼리를 참조함
            // .queryProvider(HibernateQueryProvider<T>) // 하이버네이트 쿼리(HQL) 프로그래밍으로 빌드하는 기능 제공
            // .nativeQuery(String) // 네이티브 SQL 쿼리를 실행한 뒤 결과를 하이버네이트로 매핑하는데 사용
            .parameterValues(Collections.singletonMap("city", city))
            .build();
}
```  

**(5) 잡 실행 및 결과 확인**

```log
2021-10-04 18:30:45.324  INFO 11487 --- [           main] p6spy                                    : #1633339845324 | took 2ms | statement | connection 18| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select customer0_.id as id1_0_, customer0_.address as address2_0_, customer0_.city as city3_0_, customer0_.firstName as firstnam4_0_, customer0_.lastName as lastname5_0_, customer0_.middleInitial as middlein6_0_, customer0_.state as state7_0_, customer0_.zipCode as zipcode8_0_ from customer customer0_ where customer0_.city=?
select customer0_.id as id1_0_, customer0_.address as address2_0_, customer0_.city as city3_0_, customer0_.firstName as firstnam4_0_, customer0_.lastName as lastname5_0_, customer0_.middleInitial as middlein6_0_, customer0_.state as state7_0_, customer0_.zipCode as zipcode8_0_ from customer customer0_ where customer0_.city='Dover';
```

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 09:12:39.112000 | 2021-10-04 09:12:39.334000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 09:12:39.336000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```  

---  

### 하이버네이트 페이징 처리

[HibernatePagingItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/HibernatePagingItemReader.java)를 통해 페이징 기반의 ItemReader를 구현합니다.

![HibernatePagingItemReader](https://user-images.githubusercontent.com/25560203/135827126-7de1b513-b235-4b54-a372-71ff470ed8c1.png)

`HibernateCursorItemReader`와 비슷하지만 조회할 페이지 크기를 지정합니다.

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) ItemReader 빈 구성하기**

`queryString`, `queryName`, `queryProvider`를 통해 쿼리를 제공합니다. 또한 `pageSize`를 통해 페이지 크기를 지정합니다.

```java
@Bean
@StepScope
public HibernatePagingItemReader<Customer> pagingItemReader(EntityManagerFactory entityManagerFactory,
                                                            @Value("#{jobParameters['city']}") String city) {

    return new HibernatePagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
            .queryString("from Customer where city = :city")
            // .queryName(String)
            // .queryProvider()
            .parameterValues(Collections.singletonMap("city", city))
            .pageSize(CHUNK_SIZE)
            .build();
}
```  

**(2) 잡 실행 및 결과 확인**

`limit ?,?`로 offset 기반의 쿼리를 확인할 수 있습니다.

```log
// Chunk-1
2021-10-04 18:35:35.772  INFO 11861 --- [           main] p6spy                                    : #1633340135772 | took 2ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select ..fields.. from customer customer0_ where customer0_.city='Dover' limit 3;

// Chunk-2
2021-10-04 18:35:35.813  INFO 11861 --- [           main] p6spy                                    : #1633340135813 | took 2ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select ..fields... from customer customer0_ where customer0_.city='Dover' limit 3, 3;

// Chunk-3
2021-10-04 18:35:35.833  INFO 11861 --- [           main] p6spy                                    : #1633340135833 | took 2ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select ...fields... from customer customer0_ where customer0_.city='Dover' limit 6, 3;

// Chunk-4
2021-10-04 18:35:35.858  INFO 11861 --- [           main] p6spy                                    : #1633340135858 | took 4ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select ...fields... from customer customer0_ where customer0_.city='Dover' limit 9, 3;
```  

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 09:35:35.623000 | 2021-10-04 09:35:35.886000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 09:35:35.889000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
1 row in set (0.00 sec)
```  

---  

## JPA

### JPA 커서 처리

[JpaCursorItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JpaCursorItemReader.java)를 통해 커서 기반의 ItemReader를 구현합니다.

(JpaCursorItemReader는 스프링 배치 4.3 이후로 제공합니다.)

![JpaCursorItemReader](https://user-images.githubusercontent.com/25560203/135829758-f931724d-0d7a-41d8-84f4-5e8ec785ab42.png)

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) ItemReader 빈 구성하기**

`queryString`, `queryProvider`를 통해 쿼리를 제공합니다.

```java
@Bean
@StepScope
public JpaCursorItemReader<Customer> customerItemReaderByCursor(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {

    return new JpaCursorItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .entityManagerFactory(entityManagerFactory)
            .queryString("SELECT c FROM Customer c where c.city = :city")
            // .queryProvider()
            .parameterValues(Collections.singletonMap("city", city))
            .build();
}
```  

**(2) 잡 실행 및 결과 확인**

```log
2021-10-04 18:48:16.061  INFO 12941 --- [           main] p6spy                                    : #1633340896061 | took 3ms | statement | connection 18| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select ..fields... from customer customer0_ where customer0_.city='Dover';
```

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 09:48:15.915000 | 2021-10-04 09:48:16.189000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 09:48:16.191000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```

---  

### JPA 페이징 처리

[JpaPagingItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/JpaPagingItemReader.java)를 통해 페이징 기반의 ItemReader를 구현합니다.

![JpaPagingItemReader](https://user-images.githubusercontent.com/25560203/135830838-eafe5b03-7e9d-4497-97ee-52e574402721.png)

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) JpaQueryProvider 구현하기**

```java
@NoArgsConstructor
@AllArgsConstructor
public static class CustomerByCityQueryProvider extends AbstractJpaQueryProvider {

    private String cityName;

    public Query createQuery() {
        final EntityManager em = getEntityManager();
        return em.createQuery("SELECT c FROM Customer c WHERE c.city = :city")
                 .setParameter("city", cityName);
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        Assert.notNull(cityName, "City name is required");
    }
}
```

**(2) ItemReader 빈 구성하기**

JpaCursorItemReader와 마찬가지로 `queryString`, `queryProvider`를 통해 쿼리를 제공합니다.

```java
@Bean
@StepScope
public JpaPagingItemReader<Customer> createItemReaderByQueryProvider(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {
    final CustomerByCityQueryProvider queryProvider = new CustomerByCityQueryProvider(city);

    return new JpaPagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .entityManagerFactory(entityManagerFactory)
            // .queryString("SELECT c from Customer c where c.city = :city")
            .queryProvider(queryProvider)
            .parameterValues(Collections.singletonMap("city", city))
            .pageSize(CHUNK_SIZE)
            .build();
}
```  

**(3) 잡 실행 및 결과 확인**

Offset 기반으로 아이템을 조회하는 것을 확인할 수 있습니다.

```log
// Chunk-1
2021-10-04 19:09:16.937  INFO 14491 --- [           main] p6spy                                    : #1633342156936 | took 2ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select fields... from customer customer0_ where customer0_.city='Dover' limit 3;

// Chunk-2
2021-10-04 19:09:16.982  INFO 14491 --- [           main] p6spy                                    : #1633342156982 | took 1ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select fields... from customer customer0_ where customer0_.city='Dover' limit 3, 3;

// Chunk-3
2021-10-04 19:09:17.001  INFO 14491 --- [           main] p6spy                                    : #1633342157001 | took 1ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select fields... from customer customer0_ where customer0_.city='Dover' limit 6, 3;

// Chunk-4
2021-10-04 19:09:17.020  INFO 14491 --- [           main] p6spy                                    : #1633342157020 | took 1ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
select fields... from customer customer0_ where customer0_.city='Dover' limit 9, 3;
```

```sql
mysql> SELECT * FROM BATCH_STEP_EX;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       6 | copyCustomerStep |                1 | 2021-10-04 10:09:16.805000 | 2021-10-04 10:09:17.038000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 10:09:17.040000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```  

---  

# 저장 프로시저

[StoredProcedureItemReader](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/database/StoredProcedureItemReader.java)를 이용하여 저장된 프로시저를 호출합니다.

![StoredProcedureItemReader](https://user-images.githubusercontent.com/25560203/135836875-aec267ef-acce-470b-914a-ae1efe7db9a3.png)

StoredProcedureItemReader는 JdbcCursorItemReader를 바탕으로 설계됐으므로 구성 코드도 비슷합니다.

**특정 도시에 사는 고객을 읽는 ItemReader 구성하기**

**(1) 프로시저 생성하기**

```sql
DELIMITER //

CREATE PROCEDURE customer_list(IN cityOption CHAR(16))
BEGIN
    SELECT * FROM customer
    WHERE city = cityOption;
END //

DELIMITER ;
```

```sql
mysql> call customer_list('Dover');
+-----+-----------+---------------+----------+-------------------------------+-------+-------+---------+
| id  | firstName | middleInitial | lastName | address                       | city  | state | zipCode |
+-----+-----------+---------------+----------+-------------------------------+-------+-------+---------+
|  25 | Lunea     | Y             | Guzman   | 7280 Orci Rd.                 | Dover | DE    | 33714   |
|  75 | Inga      | W             | Bradford | Ap #610-453 Sollicitudin Road | Dover | DE    | 71301   |
....
```  

**(2) ItemReader 빈 구성하기**

```java
@Bean
@StepScope
public StoredProcedureItemReader<Customer> customerItemReader(DataSource dataSource,
                                                              @Value("#{jobParameters['city']}") String city) {

    return new StoredProcedureItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .dataSource(dataSource)
            .procedureName("customer_list")
            .parameters(new SqlParameter[] {
                    new SqlParameter("cityOption", Types.VARCHAR)
            })
            .preparedStatementSetter(
                    new ArgumentPreparedStatementSetter(new Object[] { city })
            )
            .rowMapper(new CustomerRowMapper())
            .build();
}
```

**(3) 잡 실행 및 결과 확인**

```log
2021-10-04 19:40:05.924  INFO 15924 --- [           main] p6spy                                    : #1633344005924 | took 3ms | statement | connection 20| url jdbc:mysql://localhost:53306/spring_batch?useSSL=false&characterEncoding=UTF-8&serverTimezone=UTC
{call customer_list('Dover')};
```

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION WHERE STEP_EXECUTION_ID=3;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 3 |       6 | copyCustomerStep |                3 | 2021-10-04 10:40:05.883000 | 2021-10-04 10:40:05.999000 | COMPLETED |            4 |         11 |            0 |          11 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-04 10:40:06.000000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT WHERE STEP_EXECUTION_ID=3;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 3 | {"@class":"java.util.HashMap","customerItemReader.read.count":12,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```

---  

## 스프링 데이터

[spring initializr](https://start.spring.io/)를 확인해보면 아래와 같이 Spring data 기반의 모듈이 존재합니다.

![Spring Data](https://user-images.githubusercontent.com/25560203/135838004-b428bf31-647a-4307-b39c-92588ea73ac6.png)

또한 [spring-batch/spring-batch-infrastructure](https://github.com/spring-projects/spring-batch/tree/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/data)를 살펴보면

`MongoDB`, `Neo4j`, `Repository(Spring data)` 기반의 ItemReader/ItemWriter를 제공합니다.

<br />  

> MongoDB 기반 ItemReader

```java
@Bean
@StepScope
public MongoItemReader<Map> tweetsItemReader(MongoOperations mongoTemplate,
        @Value("#{jobParameters['hashTag']}") String hashtag) {
    return new MongoItemReaderBuilder<Map>()
            .name("tweetsItemReader")
            .targetType(Map.class)
            .jsonQuery("{ \"entities.hashtags.text\": { $eq: ?0 }}")
            .collection("tweets_collection")
            .parameterValues(Collections.singletonList(hashtag))
            .pageSize(10)
            .sorts(Collections.singletonMap("created_at", Sort.Direction.ASC))
            .template(mongoTemplate)
            .build();
}
```  

> Spring Data Repository 기반 ItemReader

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {

    Page<Customer> findByCity(String city, Pageable pageRequest);
}

@Bean
@StepScope
public RepositoryItemReader<Customer> customerItemReader(
        CustomerRepository repository,
        @Value("#{jobParameters['city']}") String city) {

    return new RepositoryItemReaderBuilder<Customer>()
            .name("customertemReader")
            .arguments(Collections.singletonList(city))
            .methodName("findByCity")
            .repository(repository)
            .sorts(Collections.singletonMap("lastName", Direction.ASC))
            .build();
    /*
    -- select query
    select
        customer0_.id as id1_0_, ...
    from
        customer customer0_
    where customer0_.city='Dover' order by customer0_.lastName asc limit 10;

    -- count query
    select count(customer0_.id) as col_0_0_ from customer customer0_ where customer0_.city='Dover';
     */
}
```

```java
package org.springframework.batch.item.data;

public class RepositoryItemReader<T> extends AbstractItemCountingItemStreamItemReader<T> implements InitializingBean {
    ...
	protected List<T> doPageRead() throws Exception {
		Pageable pageRequest = PageRequest.of(page, pageSize, sort);

		MethodInvoker invoker = createMethodInvoker(repository, methodName);

		List<Object> parameters = new ArrayList<>();

		if(arguments != null && arguments.size() > 0) {
			parameters.addAll(arguments);
		}

		parameters.add(pageRequest);

		invoker.setArguments(parameters.toArray());

		Page<T> curPage = (Page<T>) doInvoke(invoker);

		return curPage.getContent();
	}
	...
}    
```  

---  

## 기존 서비스

대부분의 회사에서는 현재 서비스 중인 웹 또는 다른 형태의 자바 애플리케이션이 존재합니다. 해당 애플리케이션은 많은 분석, 설계, 테스트, 버그 수정 과정을 거쳤습니다.

이러한 이미 제공하던 검증된 코드를 사용하는 방법에 대하여 살펴봅니다.

[ItemReaderAdapter](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-infrastructure/src/main/java/org/springframework/batch/item/adapter/ItemReaderAdapter.java)를 사용해 기존 서비스 코드를 호출하는 ItemReader를 구성합니다.

![ItemReaderAdapter](https://user-images.githubusercontent.com/25560203/135840706-f00d3070-e9ca-4337-ab75-dc5837180d2c.png)

**전체 고객을 읽는 ItemReader 구성하기**

**(1) 기존 서비스**

```java
@Slf4j
@Component
public class CustomerService {

    private List<Customer> customers;
    private int curIndex;

    @PostConstruct
    private void setUp() {
        curIndex = 0;
        customers = new ArrayList<>(100);
        for (int i = 0; i < 100; i++) {
            // randome customer 생성
            customers.add(CustomerGenerator.createCustomer());
        }
    }

    public Customer getCustomer() {
        if (curIndex >= customers.size()) {
            return null;
        }
        return customers.get(curIndex++);
    }
}
```

**(2) ItemReader 빈 구성하기**

```java
@Bean
public ItemReaderAdapter<Customer> customerItemReader(CustomerService customerService) {
    final ItemReaderAdapter<Customer> adapter = new ItemReaderAdapter<>();

    adapter.setTargetObject(customerService);
    adapter.setTargetMethod("getCustomer");

    return adapter;
}
```  

---  

# 커스텀 입력

스프링 배치는 대부분의 타입의 ItemReader를 제공합니다. 하지만 상황에 따라서 직접 ItemReader를 구현할 상황도 존재합니다.

`read()` 를 구현하는 것은 쉽지만 재시작 등 상태를 유지하는 구현도 필요합니다.

offset 등의 리더 상태를 저장하기 위해 ItemStream 인터페이스를 구현합니다.

```java
public interface ItemStream {

	void open(ExecutionContext executionContext) throws ItemStreamException;
	void update(ExecutionContext executionContext) throws ItemStreamException;
	void close() throws ItemStreamException;
}
```  

**전체 고객을 읽는 ItemReader 구성하기**

**(1) CustomItemReader 구현**

- 생성자: 랜덤으로 100개의 Customer 객체를 생성합니다.
- read(): `curIndex`가 50인 경우 예외 / customers 보다 적은 경우 Customer 반환 / 이외 null
- open(): `ExecutionContext`로 부터 offset을 조회합니다.
- update(): `ExecutionContext`에 offset을 갱신합니다.


```java
@Slf4j
public class CustomerItemReader extends ItemStreamSupport implements ItemReader<Customer> {

    private List<Customer> customers;
    private int curIndex;
    private String INDEX_KEY = "current.index.customers";

    public CustomerItemReader() {
        customers = IntStream.range(0, 100)
                             .boxed()
                             .map(i -> CustomerGenerator.createCustomer())
                             .collect(Collectors.toList());
        curIndex = 0;
    }

    @Override
    public Customer read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        logger.info("## Read is called. curIndex: {}", curIndex);
        if (curIndex == 50) {
            throw new RuntimeException("This will end your execution");
        }

        if (curIndex < customers.size()) {
            return customers.get(curIndex++);
        }
        return null;
    }

    @Override
    public void close() throws ItemStreamException {
        logger.info("## Close is called.\n{}", ThreadUtil.getStackTrace());
    }

    @Override
    public void open(ExecutionContext executionContext) {
        logger.info("## Open is called.\n{}", ThreadUtil.getStackTrace());
        if (!executionContext.containsKey(getExecutionContextKey(INDEX_KEY))) {
            logger.info("## Initialize curIndex=0");
            curIndex = 0;
            return;
        }

        final int index = executionContext.getInt(getExecutionContextKey(INDEX_KEY));
        logger.info("## index from context: {}", index);
        if (index == 50) {
            curIndex = 51;
            return;
        }
        curIndex = index;
    }

    @Override
    public void update(ExecutionContext executionContext) {
        logger.info("## Update is called. curIndex: {}\n{}", curIndex, ThreadUtil.getStackTrace());
        executionContext.putInt(getExecutionContextKey(INDEX_KEY), curIndex);
    }
}
```

전체 플로우를 간략히 보면 아래와 같습니다.

![CustomItemReader](https://user-images.githubusercontent.com/25560203/135846896-4eb63e38-48df-4b7c-9fae-a81739b62544.png)

**(2) ItemReader 빈 구성하기**

```java
@Bean
public CustomerItemReader customerItemReader() {
    final CustomerItemReader customerItemReader = new CustomerItemReader();

    customerItemReader.setName("customerItemReader");

    return customerItemReader;
}
```  

**(3) 잡 실행 및 결과 확인**

```log
// (1) 최초 Open 호출
2021-10-04 20:29:35.138  INFO 19509 --- [           main] i.s.b.d.example9.CustomerItemReader      : ## Open is called.
io.spring.batch.database.example9.CustomerItemReader.open(CustomerItemReader.java:55)
org.springframework.batch.item.support.CompositeItemStream.open(CompositeItemStream.java:104)
org.springframework.batch.core.step.tasklet.TaskletStep.open(TaskletStep.java:311)
...
// (2) update() 호출(최초)
2021-10-04 20:29:35.139  INFO 19509 --- [           main] i.s.b.d.example9.CustomerItemReader      : ## Update is called. curIndex: 0
io.spring.batch.database.example9.CustomerItemReader.update(CustomerItemReader.java:73)
org.springframework.batch.item.support.CompositeItemStream.update(CompositeItemStream.java:76)
org.springframework.batch.core.step.tasklet.TaskletStep.doExecute(TaskletStep.java:251)
...
// (3) Chunk 시작 후 update() 호출
2021-10-04 20:29:35.157  INFO 19509 --- [           main] i.s.b.d.example9.CustomerItemReader      : ## Update is called. curIndex: 3
io.spring.batch.database.example9.CustomerItemReader.update(CustomerItemReader.java:73)
org.springframework.batch.item.support.CompositeItemStream.update(CompositeItemStream.java:76)
org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:447)
org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:331)
org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140)
org.springframework.batch.core.step.tasklet.TaskletStep$2.doInChunkContext(TaskletStep.java:273)

// (4) close() 호출
2021-10-04 20:29:35.168  INFO 19509 --- [           main] i.s.b.d.example9.CustomerItemReader      : ## Close is called.
io.spring.batch.database.example9.CustomerItemReader.close(CustomerItemReader.java:50)
org.springframework.batch.item.support.CompositeItemStream.close(CompositeItemStream.java:90)
org.springframework.batch.core.step.tasklet.TaskletStep.close(TaskletStep.java:306)
org.springframework.batch.core.step.AbstractStep.execute(AbstractStep.java:287)
org.springframework.batch.core.job.SimpleStepHandler.handleStep(SimpleStepHandler.java:152)
```

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                                 | SERIALIZED_CONTEXT |
+-------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.current.index.customers":48,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```  

---  

# 에러 처리  

## 레코드 건너뛰기/잘못된 레코드 로그 남기기  

배치를 처리하다가 실패한 경우 취할 수 있는 행동은 아래와 같다.  

1. 전체 배치 종료 처리
2. 에러 레코드 건너뛰기
   2.1. 어떠한 에러에 대하여 레코드를 건너뛸 것인가?
   2.2. 얼마나 많은 레코드를 건너뛸 수 있는가? (100만 건 중 1~2건을 건너뛰는 것과 50만 건을 건너뛰는 것은 다르다)
3. 에러 레코드 로그 남기기

Spring Batch에서는 [FaultTolerantStepBuilder](https://github.com/spring-projects/spring-batch/blob/main/spring-batch-core/src/main/java/org/springframework/batch/core/step/builder/FaultTolerantStepBuilder.java)를 이용하여 `SkipPolicy`를 구성할 수 있습니다.  

```java
package org.springframework.batch.core.step.skip;

public interface SkipPolicy {

	boolean shouldSkip(Throwable t, int skipCount) throws SkipLimitExceededException;
}
```  

`faultTolerant()`를 통해 아래와 같이 레코드 건너뛰기를 구성할 수 있습니다.  

```java
@Bean
public Step copyCustomerStep() {
    return stepBuilderFactory.get("copyCustomerStep")
                             .<Customer, Customer>chunk(CHUNK_SIZE)
                             .reader(customerFlatFileItemReader(null)) // custom query provider
                             .writer(customerItemWriter())
        
                              // FaultTolerantStepBuilder 구성
                             .faultTolerant()
                              // 스킵할 에러
                             .skip(Exception.class)
                              // 스킵하지 않을 에러
                             .noSkip(ParseException.class)
                              // 최대 스킵 레코드 수
                             .skipLimit(3)
        
                             .build();
}
```  

또한 `ItemListener`를 통해서 잘못된 레코드의 로그를 추가할 수 있습니다.  

배치 Job, Step의 실패에 대한 에러 로그(EXIT_MESSAGE)는 남기만 Skip 한 레코드에 대한 로그를 ExecutionContext 외에는 별도로 추가할 수 없습니다.  

<br />

**(1) FixedLength를 가진 파일 구성하기**  

총 9개의 레코드가 존재하고 이 중 4개가 유효하지 않습니다.  

```text
Aimee      CHoover    7341Vel Avenue          Mobile          AL35928                               
Jonas      UGilbert   8852In St.              Saint Paul      MN57321
Regan      MBaxter    4851Nec Av.             Gulfport        MS33193
Octavius   TJohnsoninvalid fixlength 7418Cum Road            Houston         TX51507               <-- error
Sydnee     NRobinson  894 Ornare. Ave         Olathe          KS25606 is also invalid length       <-- error
Stuart     KMckenzie  5529Orci Av.            Nampa           ID18562
Petra      ZLara      8401Et St.              Georgia         GA70323 is also invalid length       <-- error
Cherokee   TLara      8516Mauris St.          Seattle         WA28720  is also invalid length      <-- error
Athena     YBurt      4951Mollis Rd.          Newark          DE41034
```  

**(2) SkipPolicy 구현하기**  

```java
@Slf4j
public class FileVerificationSkipper implements SkipPolicy {

    @Override
    public boolean shouldSkip(Throwable t, int skipCount) throws SkipLimitExceededException {
        if (t instanceof FileNotFoundException) {
            return false;
        }
        logger.info("shouldSkip is called. throwable: {} / skipCount: {}", t.getClass().getSimpleName(), skipCount);
        return t instanceof ParseException && skipCount <= 3;
    }
}
```  

**(3) 에러 로그를 위한 ItemListener 구현**  

```java
@Slf4j
public class CustomerItemListener {

    @OnReadError
    public void onReadError(Exception e) {
        if (e instanceof FlatFileParseException) {
            final FlatFileParseException ffpe = (FlatFileParseException) e;
            final String message = "An error occurred while processing the "
                                   + ffpe.getLineNumber()
                                   + " line of the file. Below was the faulty input.\n"
                                   + ffpe.getInput()
                                   + "\n";
            logger.error(message);
            return;
        }
        logger.error("An error has occurred", e);
    }
}
```

**(4) ItemReader 빈 구성하기**  

```java
@Bean
public Step copyCustomerStep() {
    return stepBuilderFactory.get("copyCustomerStep")
                             .<Customer, Customer>chunk(CHUNK_SIZE)
                             .reader(customerFlatFileItemReader(null))
                             .listener(customerItemListener())
                             .writer(customerItemWriter())
                             .faultTolerant()
                             .skipPolicy(new FileVerificationSkipper())
                             .build();
}
```  

**(5) 잡 실행 및 결과 확인**  

```log
2021-10-05 21:09:20.555  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Write Chunk-1 items: #3
2021-10-05 21:09:20.556  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Current customer: Customer{firstName=Aimee, middleInitial=C, lastName=Hoover, addressNumber=7341, street=Vel Avenue, city=Mobile, state=AL, zipCode=35928, transactions=null}. Total Item Write: #1
2021-10-05 21:09:20.558  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Current customer: Customer{firstName=Jonas, middleInitial=U, lastName=Gilbert, addressNumber=8852, street=In St., city=Saint Paul, state=MN, zipCode=57321, transactions=null}. Total Item Write: #2
2021-10-05 21:09:20.559  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Current customer: Customer{firstName=Regan, middleInitial=M, lastName=Baxter, addressNumber=4851, street=Nec Av., city=Gulfport, state=MS, zipCode=33193, transactions=null}. Total Item Write: #3
2021-10-05 21:09:20.575 ERROR 6671 --- [           main] i.s.b.handleerror.CustomerItemListener   : An error occurred while processing the 4 line of the file. Below was the faulty input.
Octavius   TJohnsoninvalid fixlength 7418Cum Road            Houston         TX51507
2021-10-05 21:09:20.575  INFO 6671 --- [           main] i.s.b.h.FileVerificationSkipper          : shouldSkip is called. throwable: FlatFileParseException / skipCount: 0
2021-10-05 21:09:20.575 ERROR 6671 --- [           main] i.s.b.handleerror.CustomerItemListener   : An error occurred while processing the 5 line of the file. Below was the faulty input.
Sydnee     NRobinson  894 Ornare. Ave         Olathe          KS25606 is also invalid length
2021-10-05 21:09:20.575  INFO 6671 --- [           main] i.s.b.h.FileVerificationSkipper          : shouldSkip is called. throwable: FlatFileParseException / skipCount: 1
2021-10-05 21:09:20.576 ERROR 6671 --- [           main] i.s.b.handleerror.CustomerItemListener   : An error occurred while processing the 7 line of the file. Below was the faulty input.
Petra      ZLara      8401Et St.              Georgia         GA70323 is also invalid length
2021-10-05 21:09:20.576  INFO 6671 --- [           main] i.s.b.h.FileVerificationSkipper          : shouldSkip is called. throwable: FlatFileParseException / skipCount: 2
2021-10-05 21:09:20.576 ERROR 6671 --- [           main] i.s.b.handleerror.CustomerItemListener   : An error occurred while processing the 8 line of the file. Below was the faulty input.
Cherokee   TLara      8516Mauris St.          Seattle         WA28720  is also invalid length
2021-10-05 21:09:20.576  INFO 6671 --- [           main] i.s.b.h.FileVerificationSkipper          : shouldSkip is called. throwable: FlatFileParseException / skipCount: 3
2021-10-05 21:09:20.577  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Write Chunk-2 items: #2
2021-10-05 21:09:20.577  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Current customer: Customer{firstName=Stuart, middleInitial=K, lastName=Mckenzie, addressNumber=5529, street=Orci Av., city=Nampa, state=ID, zipCode=18562, transactions=null}. Total Item Write: #4
2021-10-05 21:09:20.577  INFO 6671 --- [           main] i.s.batch.handleerror.HandleErrorMain    : Current customer: Customer{firstName=Athena, middleInitial=Y, lastName=Burt, addressNumber=4951, street=Mollis Rd., city=Newark, state=DE, zipCode=41034, transactions=null}. Total Item Write: #5
```  

```sql
mysql> SELECT * FROM BATCH_JOB_EXECUTION;
+------------------+---------+-----------------+----------------------------+----------------------------+----------------------------+-----------+-----------+--------------+----------------------------+----------------------------+
| JOB_EXECUTION_ID | VERSION | JOB_INSTANCE_ID | CREATE_TIME                | START_TIME                 | END_TIME                   | STATUS    | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               | JOB_CONFIGURATION_LOCATION |
+------------------+---------+-----------------+----------------------------+----------------------------+----------------------------+-----------+-----------+--------------+----------------------------+----------------------------+
|                1 |       2 |               1 | 2021-10-05 12:09:20.410000 | 2021-10-05 12:09:20.458000 | 2021-10-05 12:09:20.612000 | COMPLETED | COMPLETED |              | 2021-10-05 12:09:20.613000 | NULL                       |
+------------------+---------+-----------------+----------------------------+----------------------------+----------------------------+-----------+-----------+--------------+----------------------------+----------------------------+
1 row in set (0.00 sec)

mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME        | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       4 | copyCustomerStep |                1 | 2021-10-05 12:09:20.501000 | 2021-10-05 12:09:20.595000 | COMPLETED |            2 |          5 |            0 |           5 |               4 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-05 12:09:20.597000 |
+-------------------+---------+------------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+

mysql> SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT;
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
| STEP_EXECUTION_ID | SHORT_CONTEXT                                                                                                                                                                                                                    | SERIALIZED_CONTEXT |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
|                 1 | {"@class":"java.util.HashMap","customerItemReader.read.count":10,"batch.taskletType":"org.springframework.batch.core.step.item.ChunkOrientedTasklet","batch.stepType":"org.springframework.batch.core.step.tasklet.TaskletStep"} | NULL               |
+-------------------+----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+--------------------+
```

---  

## 입력이 없을 때의 처리  

Spring Batch는 처음 `ItemReader::read()`가 호출 될때 null을 반환해도 정상 처리합니다.  

상황에 따라서 작성한 쿼리가 빈 결과를 반환하거나 파일이 비었을때 이를 알아야할 수도 있습니다.  

이때 `StepListener`의 `@AfterStep`을 등록하여 Step의 ExitStatus 변경할 수 있습니다.  

```java
public class EmptyInputStepFailer {

    @AfterStep
    public ExitStatus afterStep(StepExecution execution) {
        if (execution.getReadCount() > 0) {
            return execution.getExitStatus();
        }
        return ExitStatus.FAILED;
    }
}
```

---  

# References

- https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/readersAndWriters.html#itemReader
- https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/readersAndWriters.html#database
- https://book.naver.com/bookdb/book_detail.nhn?bid=18990242