# 7장 ItemReader(2)

## 데이터베이스 입력

### JDBC

`Cursor` 와 `Paging` 기법을 통해 JDBC에서 수백 만건의 데이터를 모두 적재하지 않고 가져올 수 있다.

- Cursor

`RowMapper` 인터페이스를 구현하여 도메인 객체에 매핑시킨 후 `Cursor` 를 열어 스트리밍하게 데이터를 가져올 수 있다.

```java
public JdbcCursorItemReader<Customer> customerItemReader(Datasource datasource){
    return new JdbcCursorItemReaderBuilder<Customer>()
        .name("blah")
        .dataSource(dataSource)
        .sql("select * from customer")
        .rowMapper(new CustomerRowMapper())
        .build();
}
```

이제 `JdbcCursorItemReader`의 read 메소드를 호출하게 되면 데이터베이스는 로우 하나를 반환한다.

- Paging 처리

`JdbcPagingItemReader` 를 itemReader에서 구현해주면 된다.
또한 `SqlPaingQueryProviderFactoryBean` 으로 sql 을 작성한 뒤 위의 객체의 세팅값으로
queuryProvider에 주입해주면 된다.


### 하이버네이트

자에의 ORM 기술인 하이버네이트를 활용해 모델 객체를 데이터베이스 테이블로 매핑할 수 있다. 

배치 처리에서 하이버네이트를 사용하기 위해서는 스테이트풀 세션을 주의해야만 한다. (OOM 발생할 수 있음)

스프링배치가 제공하는 ItemReader는 이런 점을 해결하도록 개발되었다.

#### 하이버네이트 커서로 처리하기

사전 작업으로는 HibernateBatchConfigurer를 DefaultBatchConfigurer를 상속받아 
getTransactionManager()를 오버라이딩 시켜줄 수 있도록 해야한다.

transactionManager에 쓰일 객체 = new HibernateTransactionManager();

그 후 `HibernateCursorItemReader`를 구현하면 된다. 


#### 하이버네이트 페이징 기법 

`HibernatePagingItemReader` 를 구현하면 된다.

### JPA
 
위의 하이버네이트는 JPA 를 구현한 구현체이다.

페이징 처리는 `JpaPagingItemReader` 를 구현하게 되며 위 하이버네이트와 동일한 방식이다.

### 저장 프로시저

데이터베이스는 목적에 맞춘 저장 프로시저가 포함되어 있을 수 있다. 스프링 배치에서는 `StoredProcudereItemReader` 컴포넌트를 사용하여 구성할 수 있다. 

프로시저는 데이터베이스에서 만들어놓고 이를 배치에서 사용하게끔 호출한다고 보면 된다.

`StoredProcedureItemReader` 를 사용하며 프로시저명을 지정해주면 된다. 

### 스프링 데이터 

#### 몽고 DB

몽고 DB는 `MongoItemReader`로 페이지 기반이다.

#### 스프링데이터 레포지토리

`RepositoryItemReader`를 활용해 일관된 방식으로 레포지토리를 운용할 수 있다.

## 기존 서비스

기존 어플리케이션의 코드를 사용하기 위해서는 `ItemReaderAdapter` 를 활용하여 처리할 수 있다

## 커스텀 입력

위와 같이 여러 제공에도 불구하고 커스텀으로 처리해야하는 상황이 발생할 수 있다.
이땐 `ItemReader ` 인터페이스를 구현한 Custom ItemReader 를 만들 수 있다. 

중요 포인트는 `ItemReader` 의 read 메소드를 구현해주기만 하면 된다.

만약 리더의 상태를 저장해서 이 전에 종료된 지점부터 리더를 다시 시작하려면 `ItemStream`
인터페이스를 구현해야한다. `ItemStreamSuport` 라는 추상클래스도 함께 구현할 때 넣고 
read 메소드를 구현하게되면 Custom ItemReader 를 만들 수 있다.

## 에러 처리 

- 레코드 건너뛰기 : 에러가 발생했을 때 레코드를 건너뛰도록 step에서 설정할 수 있다.

- 잘못된 레코드 로그 남기기 : ItemListener를 만들어서 `@OnReadError` 를 처리하도록 작업한다.

- 입력이 없을 때 처리 : `@AfterStep` 을 구현하여 처리한다. 