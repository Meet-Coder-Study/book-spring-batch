# 9-1장 ItemWriter

- ItemWriter는 스프링 배치의 출력 매커니즘
    - 원래는 Item을 읽고 처리되는대로 바로 출력했다. 그러나 지금은 Chunk단위로 출력한다.
- Chunk 단위를 매개변수로 받기 때문에 List로 받는다.

> 트랜잭션이 적용되지 않은 리소스를 처리한다면 Write에 실패했을 때 롤백할 방법이 없다. 그래서 추가적인 보호 조치를 취해야 한다.
> 

### 파일 기반 ItemWriter

- FlatFileItemWriter
    - 텍스트 파일 출력을 만들 때 사용하라고 스프링 배치가 만들어둔 ItemWriter의 구현체다
    - 트랜잭션 주기 내에서 실제 쓰기 작업을 가능한 늦게 수행하도록 설계해서 롤백이 그나마 수월하도록 구현했다.
        - 실제로 데이터를 기록할 때 TransactionSynchronizationAdapter의 beforeCommit 메서드를 사용해서 해당 매커니즘을 구현했다.
    - 즉, 쓰기 외 모든 작업을 완료한 뒤 Writer로 데이터를 실제로 디스크에 기록하기 직전에 PlatformTransactionManager가 트랜잭션을 커밋한다.
    - 디스크로 Flush되면 롤백할 수 없기때문에 딱 디스크에 기록 전에 이뤄지도록 한다.
    - 각종 파일 관리 옵션이 있다
        - 파일이 이미 만들어져 있을 때 삭제하고 다시 만든다든지
        - 파일이 이미 만들어져 있으면 여기에 추가한다든지 등등...
- StaxEventItemWriter
    - XML 작성 구현체
    - FlatFileItemWriter와 같이 실제 쓰기 작업을 가능한 늦게 수행한다.

### 데이터베이스 기반 ItemWriter

- JdbcBatchItemWriter
    - 하나의 Chunk에 대한 모든 SQL 문을 한 번에 실행하기 위해 PreparedStatement의 배치 업데이트 기능을 사용한다.
        - 이렇게 하면 실행 성능을 향상시키면서 실행 자체를 현재 트랜잭션에서 할 수 있다
    - `.beanMapped()` **를 사용해서 파라미터에 ?가 아니라 이름을 줄 수 있다
- HibernateItemWriter
- JpaItemWriter

### 스프링 데이터의 ItemWriter

- Mongo DB
    - 테이블이 없기 떄문에 Entity에서 JPA 애너테이션을 제거해야한다.
    - yml도 spring.data.mongodb.database로 알려줘야 한다.
    - MongoDB는 트랜잭션을 지원하지 않아서 커밋이 발생하기 직전까지 쓰기를 지연한다.
    
    ```sql
    @Bean
    public MongoItemWriter<Customer> mongoItemWriter(MongoOperations mongoTemplate) {
      return new MongoItemWriterBuilder<Customer>()
        .collection("customers")
        .template(mongoTemplate)
        .build();
    }
    ```
    
- Repository
    - 쓰기 작업을 할 때는 페이징이나 정렬에 관해 걱정할 필요가 없다. 그래서 그냥 CrudRepository를 사용하면 된다.

[실습코드](https://github.com/aegis1920/my-lab/tree/master/def-guide-spring-batch)
