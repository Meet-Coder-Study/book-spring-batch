## 9-1장. ItemWriter

### ItemWriter 소개

- `ItemWriter`는 `ItemReader`와는 달리 아이템을 건건이 쓰지 않는다. 그래서 `ItemReader`와는 달리 리스트를 받는다

  ```java
  public interface ItemWriter<T> {
    void write(List<? extends T> items) throws Exception;
  }
  ```

- 

  ![chunk-oriented-processing-with-item-processor](https://docs.spring.io/spring-batch/docs/current/reference/html/images/chunk-oriented-processing-with-item-processor.png)

### 파일 기반 ItemWriter

- `FlatFileItemWriter`

  - 구성 요소 : `Resource`와 `LineAggregator`구현체

    - `Resource` : 출력할 리소스
    - `LineAggregator`구현체 : Reader의 `LineMapper`에 해당함. (객체 -> 문자열)

  - 트랜잭션이 작동하는 방식

    - 트랜잭션 주기 내에서 실제 쓰기 작업을 가능한 한 늦게 수행하도록 설계됨
    - 쓰기 외 모든 작업을 완료한 뒤 디스크에 쓰기 직전에 커밋한다 (한번 쓰면 롤백할 수 없으니까)

  - 형식화된 텍스트 파일로 쓰기 (고객 정보 csv 파일을 읽어서 `"%s %s lives at %s %s in %s, %s."` 형식으로 쓰기)

    ```java
    // itemReader 부분은 생략
    
    @Bean
    public FlatFileItemWriter<Customer> formattedItemWriter() {
      return new FlatFileItemWriterBuilder<Customer>()
          .name("customerItemWriter")
          .resource(...) //출력될 리소스
          .formatted() // FormatterLineAggregator를 사용할 수 있도록 하는 별도의 빌더를 리턴해줌
          .format("%s %s lives at %s %s in %s, %s.")
          .names("firstName", "lastName", "address", "city", "state", "zip")
          .build();
    }
    ```

  - 구분자로 구분된 파일로 쓰기 (고객 정보 csv 파일을 읽어서 구분자를 `,`에서 `;`으로 바꾸기)

    ```java
    @Bean
    public FlatFileItemWriter<Customer> delimitedItemWriter() {
      return new FlatFileItemWriterBuilder<Customer>()
          .name("customerItemWriter")
          .resource(...) //출력될 리소스
          .delimited() // DelimitedLineAggregator를 사용할 수 있도록 하는 별도의 빌더를 리턴해줌
          .delimiter(";")
          .names("zip", "state", "city", "address", "lastName", "firstName")
          .build();
    }
    ```

  - `FlatFileItemWriter`의 여러가지 고급 옵션

    - `shouldDeleteIfEmpty`
      - false(기본값) : 스텝이 완료됐을때 아무 아이템도 안써졌으면 빈 파일로 남음
      - true : 스텝이 완료됐을때 아무 아이템도 안써졌으면 파일 삭제함
    - `shouldDeleteIfExist`
      - true(기본값) : 이름이 같은 파일이 있으면 삭제하고 새로 만듦
      - false : 이름이 같은 파일이 있으면 `ItemStreamException` 발생. 실행 결과를 매번 보호하고 싶을때 사용
    - `append`
      - false(기본값)
      - true : `shouldDeleteIfExist`을 자동으로 false로 설정함. 여러 스텝이 하나의 출력 파일에 쓸때 유용함
        - 결과 파일이 존재하지 않는 경우 : 새 파일 생성
        - 결과 파일이 존재하는 경우 : 기존 파일에 데이터 추가

- `StaxEventItemWriter`

  - `FlatFileItemWriter`와 동일하게 한 번에 청크 단위호 XML을 생성하며 로컬 트랜잭션이 커밋되기 직전에 파일에 쓴다

    ```
    // build.gradle
    implementation 'org.springframework:spring-oxm'
    implementation 'com.thoughtworks.xstream:xstream:1.4.10'
    ```

    ```java
    @Bean
    public StaxEventItemWriter<Customer> xmlCustomerWriter() {
      Map<String, Class<Customer>> aliases = new HashMap<>();
      aliases.put("customer", Customer.class);
    
      XStreamMarshaller marshaller = new XStreamMarshaller();
      marshaller.setAliases(aliases);
    
      return new StaxEventItemWriterBuilder<Customer>()
          .name("xmlOutputWriter")
          .resource(outputResource) // 기록할 리소스
          .marshaller(marshaller) // 마샬러(객체 -> xml)의 구현체
          .rootTagName("customers") // 마샬러가 생성할 각 xml 프래그먼트의 루트 태그 이름
          .build();
    }
    ```

### 데이터베이스 기반 ItemWriter

- `JdbcBatchItemWriter`

  - `JdbcTemplate`를 사용하며, `JdbcTemplate`의 배치 SQL 실행 기능을 사용해 한 번에 청크 하나에 대한 모든 SQL을 실행한다.

    ```java
    @Bean
    @StepScope
    public JdbcBatchItemWriter<Customer> jdbcCustomerWriter(DataSource dataSource) throws Exception {
      return new JdbcBatchItemWriterBuilder<Customer>()
          .dataSource(dataSource)
          .sql("INSERT INTO CUSTOMER (first_name, middle_name, last_name, address, city, state, zip_code) VALUES (?, ?, ?, ?, ?, ?, ?)")
          .itemPreparedStatementSetter(new CustomerItemPreparedStatementSetter())
          .build();
    }
    ```

    ```java
    public class CustomerItemPreparedStatementSetter implements ItemPreparedStatementSetter<Customer> {
    
      @Override
      public void setValues(Customer item, PreparedStatement ps) throws SQLException {
        ps.setString(1, item.getFirstName());
        ps.setString(2, item.getMiddleInitial());
        ps.setString(3, item.getLastName());
        ps.setString(4, item.getAddress());
        ps.setString(5, item.getCity());
        ps.setString(6, item.getState());
        ps.setString(7, item.getZipCode());
      }
    }
    ```

  - PreparedStatement말고 네임드 파라미터 접근법을 사용할수도 있다

    - `ItemPreparedStatementSetter` 의 구현체가 아닌 `ItemSqlParameterSourceProvider`의 구현체를 사용한다

    ```java
    @Bean
    public JdbcBatchItemWriter<Customer> jdbcCustomerWriter(DataSource dataSource) {
      return new JdbcBatchItemWriterBuilder<Customer>()
          .dataSource(dataSource)
          .sql("INSERT INTO CUSTOMER (first_name, middle_initial, last_name, address, city, state, zip) "
              + "VALUES (:firstName, :middleInitial, :lastName, :address, :city, :state, :zip)")
          .beanMapped() // 이걸 사용하면 아이템에서 값을 추출하는 작업을 하지 않아도 된다
          .build();
    }
    ```

- `HibernateItemWriter`

  - 설정해야 하는 것들

    1. 하이버네이트 의존성 포함

       ```
       // build.gradle
       
       implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
       ```

    2. `CurrentSessionContext` 구성

       ```yaml
       # application.yml
       
       spring:
         jpa:
           properties:
             hibernate:
               current_session_context_class: org.springframework.orm.hibernate5.SpringSessionContext
       ```

    3. `Customer` 객체 매핑(`@Id`, `@Entity` 등 이용)

    4. 하이버네이트를 지원하도록  `SessionFactory`와 새로운 `TransactionManager` 둘 다 구성

       - 일반적으로 제공되는 `DataSourceTransactionManager` 대신 `HibernateTransactionManager`를 제공하도록 구성

    5. `HibernateItemWriter` 구성

       ```java
       @Bean
       public HibernateItemWriter<Customer> hibernateItemWriter(EntityManagerFactory entityManager) {
         return new HibernateItemWriterBuilder<Customer>()
           // SessionFactory에 대한 참조는 필수 의존성
           .sessionFactory(entityManager.unwrap(SessionFactory.class))
           .build();
       }
       ```

- `JpaItemWriter`

  - `JpaItemWriter`는 모든 아이템을 저장한 뒤 flush를 호출하기 전에 아이템 목록을 순회하면서 아이템마다 `EntityManager.merge()`를 호출한다

  - 설정해야 하는 것들

    1. `JpaTransactionManager`를 생성하는  `BatchConfigurer` 구현체 작성

       - 이전 `HibernateBatchConfigurer`와의 차이점
         1. 생성자에서 `SessionFactory`대신 `EntityManager`를 저장한다
         2. initialize 메소드에서 `HibernateTransactionManager`대신 `JpaTransactionManager`를 생성한다

    2. `JpaItemWriter` 구성

       ```java
       @Bean
       public JpaItemWriter<Customer> jpaItemWriter(EntityManagerFactory entityManagerFactory) {
         JpaItemWriter<Customer> jpaItemWriter = new JpaItemWriter<>();
         jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
         return jpaItemWriter;
       }
       ```

### 스프링 데이터의 ItemWriter

- 몽고DB

  - 설정해야 하는 것

    1. Customer 매핑

       - id를 String으로 바꾸기(id에 long 사용불가)

    2. 의존성 추가 `implementation 'org.springframework.boot:spring-boot-starter-data-mongodb'`

    3. `application.yml`에서 몽고DB 설정

    4. `MongoItemWriter` 작성

       ```java
       @Bean
       public MongoItemWriter<Customer> mongoItemWriter<MongoOperations mongoTemplate) {
         return new MongoItemWriterBuilder<Customer>()
           .collection("customers") // 데이터베이스의 컬렉션 이름
           .template(mongoTemplate) //
           // delete 플래그도 있음 (true:매칭되는 아이템 삭제, false:매칭되는 아이템 저장(default))
           .build();
       }
       ```

  - 몽고DB가 트랜잭션을 지원하지 않기 때문에 커밋 직전까지 쓰기를 버퍼링하고 마지막 순간에 쓰기 작업을 수행한다

- 네오4j - 생략

- 피보탈 젬파이어와 아파치 지오드 - 생략

- 리포지터리

  - 슈퍼 인터페이스인 `CrudRepository`를 사용한다

  - 설정해야 하는 것들

    1. 의존성 추가

    2. 리포지터리 기능을 부트스트랩 하기 위해 스프링에게 `Repository`를 어디서 찾아야 하는지 알려줘야 한다

       ```java
       @Configuration
       @EnableJpaRepositories(basePackageClasses = Customer.class) // <--
       public class RepositoryJobConfig {
         ...
       }
       ```

    3. `RepositoryItemWriter` 작성

       ```java
       @Bean
       public RepositoryItemWriter<CustomerEntity> repositoryItemWriter(CustomerRepository repository) {
         return new RepositoryItemWriterBuilder<CustomerEntity>()
           .repository(repository)
           .methodName("save")
           .build();
       }
       ```

### 