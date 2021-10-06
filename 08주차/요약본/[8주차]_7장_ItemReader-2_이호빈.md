# 7-2장 ItemReader (데이터베이스 입력)

대부분의 내용은 [실습 코드](https://github.com/aegis1920/my-lab/tree/master/def-guide-spring-batch)에 있습니다.

- DB는 배치 처리에서 훌륭한 입력소스다
    - DB에 내장된 트랜잭션
    - 파일보다 확장성이 좋으며 복구 기능이 좋다

## JDBC

- 스프링이 제공하는 JdbcTemplate을 사용하면
    - 전체 ResultSet을 한 row씩 순서대로 가져오면서 필요한 도메인 객체로 변환해 메모리에 저장한다
    - 전체 데이터를 한 번에 메모리에 저장하지 않기 위해 스프링 배치는 Cursor와 Paging을 제공한다.

### Cursor

- DB에서 하나의 레코드를 한 번에 가져옴(항상 1 row)
- next()를 호출할 때마다 실행됨. 그리고 다음 레코드로 커서를 옮김

### Paging

- DB에서 Chunk 크기(e.g. 10 row)만큼의 레코드를 한 번에 가져옴
- 고유한 SQL 쿼리를 통해 생성됨

### JDBC Cursor 처리

- JdbcTemplate이 ResultSet을 Domain 객체로 매핑하는데 RowMapper 구현체가 필요하다
- RowMapper 구현체는 Spring Framework Core가 제공하는 JDBC 지원 표준 컴포넌트다
- JdbcCursorItemReader는 ResultSet을 생성하면서 Cursor를 연 다음, 스프링 배치가 read()를 호출할 때마다 Domain 객체로 매핑할 row를 가져온다.
- ResultSet은 ThreadSafe 하지 않으므로 다중 스레드 환경에서는 사용할 수 없다.

### JDBC Paging 처리

- 스프링 배치가 Page라는 청크로 결과 목록을 반환한다
- 한 번에 SQL 쿼리 하나를 실행해 레코드를 하나씩 가져오는 대신, 각 페이지마다 새로운 쿼리를 실행한 뒤 쿼리 결과를 한 번에 메모리에 적재한다??
- 페이지 크기(반환될 레코드 개수)와 페이지 번호(현재 처리중인 페이지 번호)를 가지고 쿼리를 할 수 있어야 한다.
- PagingQueryProvider 인터페이스의 구현체를 제공해야 한다. DB마다 그 구현체가 다른데 SqlPagingQueryProviderFactoryBean을 사용하면 사용하는 DB가 어떤 건지 감지할 수 있다
- Paging 기법은 각 페이지의 쿼리를 실행할 때마다 동일한 레코드 정렬 순서를 보장하려면 순서가 중요하다. 그래서 sortKey가 있다.
- SqlPagingQueryProviderFactoryBean에서 DataSource를 받는 이유는 이 DataSource를 사용해 DB 타입을 결정하기 때문

### Hibernate

- XML 파일 또는 애너테이션을 사용해서 객체를 데이터베이스 테이블로 매핑하는 구성을 한다
- 그래서 DB 구조를 알지 못해도 객체 구조 기반으로 쿼리를 작성할 수 있다
- 하이버네이트의 기본 세션은 stateful이다. 즉, 아이템 백만 건을 조회하고 사용한다면 해당 아이템들을 모두 캐시에 쌓으면서 OOME가 발생한다.
- 스프링 배치가 제공하는 하이버네이트 기반 ItemReader는 Commit할 때 세션을 Flush해서 배치처리를 잘할 수 있도록 한다.

### Hibernate Cursor 처리

- sessionFactory, Customer 엔티티, HibernateCursorItemReader가 필요하다
- pageSize()만 추가하면 끝

### JPA

- Cursor 기반 방법을 제공하지 않는다.
- 커스텀 BatchConfigurer 구현체를 생성할 필요도 없다. JpaTransactionManger 구성을 알아서 처리하기 때문

### 저장 프로시저

- DB에 저장되는 해당 DB 전용 함수
- StoredProcedureItemReader로 프로시저를 만들어 SQL로 정의된 함수를 실행시켜줄 수 있다

### 몽고 DB

- 기본으로 레플리케이션, 샤딩을 제공한다
- MongoItemReader는 page 기반 ItemReader다
- `spring-boot-starter-data-mongodb` 의존성을 추가한다
- application.yml에 DB 설정 : `spring.data.mongodb.database: tweets`

```sql
@Bean
@StepScope
public MongoItemReader<Map> tweetsItemReader(MongoOperations mongoTemplate, @Value("#{jobParameters['hashTag']}") String hashtag) {
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

@Bean
public Step copyFileStep() {
    return this.stepBuilderFactory.get("copyFileStep")
                .<Map, Map>chunk(10)
                .reader(tweetsItemReader(null, null))
                .writer(itemWriter())
                .build();

}

```

### Spring Data Repository

- 일관성을 위해 Repository 추상화를 했다.
- 그래서 인터페이스를 상속하기만 하면 기본적인 CRUD를 할 수 있다
- 스프링 배치와 스프링 데이터가 호환성이 좋은 이유는 둘 다 PagingAndSortingRepository를 활용하기 때문이다
- RepositoryItemReader를 사용하면 어떤 DB 건 해당 DB에 질의할 수 있다
- Pageable은 한 페이지만큼 요청하는데 필요한 파라미터를 캡슐화한다.

### 기존 서비스를 이용해 ItemReader에 데이터를 공급하는 법

- ItemReaderAdapter를 사용한다.
- 매번 호출할 때마다 반환되는 객체는 ItemReader가 반환하는 객체다.
- 만약, 기존 서비스가 어떤 객체의 컬렉션을 반환한다면 단일 아이템으로 만들어야 하므로 직접 컬렉션 내 객체를 하나씩 꺼내면서 처리해야된다.
- 입력 데이터를 모두 처리하면 서비스 메서드는 반드시 null을 반환해야 한다. null을 반환하면 해당 Step의 입력을 모두 소비했음을 알린다

### 커스텀 입력

- 커스텀 ItemReader는 Job을 실행할 때마다 목록의 처음부터 재시작한다. 이는 좋지 않은 게 레코드 100만개 중 50만개만 처리하다 에러가 발생했을 때는 에러가 발생한 Chunk부터 다시 시작하도록 해야한다.
- 이를 구현하려면 ItemStream을 구현해야한다.
- ItemStream 인터페이스
    - open()
        - 필요한 상태를 초기화한다. (Job을 재시작할 때 이전 상태를 복원한다)
        - 복원한다는 건 그만큼 레코드 개수를 건너뛴다는 뜻
        - 특정 파일을 열거나 DB 연결을 한다
        - ExecutionContext에 접근할 수 있다
            - Reader의 현재 상태를 알려준다
    - update()
        - Job의 상태를 갱신할 때 사용한다.
        - ExecutionContext에 접근할 수 있다
    - close()
        - 리소스를 닫는다

### 에러 처리

- 에러가 발생했을 때 할 수 있는 처리
- 예외를 던져 배치를 멈춤
- 특정 예외가 발생했을 때 레코드를 건너뛰게 할 수 있음
- 몇 번까지 예외를 허용할지도 설정해줄 수 있음
- 리스너로 입력이 없을 때 처리하거나 잘못된 레코드에 로그를 남겨줄 수 있다.
