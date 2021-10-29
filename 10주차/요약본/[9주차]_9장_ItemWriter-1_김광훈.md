# ItemWriter - 1
## 1. ItemWriter ??
#### 소개
ItemWriter 는 스프링 배치의 출력을 담당하는 기능을 한다.

스프링 배치 처리 결과를 포맷에 맞게 출력할 필요가 있을 때 스프링 배치가 제공하는 ItemWriter 를 사용한다.

과거에는 ItemReader 와 동일하게 각 아이템이 처리되는 대로 출력이 되었지만, 스프링 배치2 에서 청크 기반처리 방식이 도입되면서 ItemWriter 의 역할이 바뀌었다.

즉, 청크 기반이기 때문에 아이템을 묶음 단위로 Write 하게 된다.

```java
public interface ItemWriter<T> {
  void write(List<? extends T> items) throws Exception;
}
```

<br>


## 2. 데이터베이스 기반 ItemWriter
#### 소개
데이터베이스는 파일과 달리 트랜잭션이 적용되는 리소스이다.

파일 처리 기반과 달리, 물리적인 쓰기를 트랜잭션의 일부분으로 포함할 수 있다.

<br>

#### HibernateItemWriter
HibernateItemWriter 는 하이버네이트 세션 API 의 간단한 Wrapper 에 지나지 않는다.

청크가 완료되면 아이템 목록이 HibernateItemWriter 로 전달되며, HibernateItemWriter 내에서는 세션과 아직 연관되지 않은 각 아이템에 대해

하이버네이트의 Session.saveOrUpdate 메서드를 호출한다. 모든 아이템이 저장되거나 수정되면 HibernateItemWriter 는 Session 의 flush 메서드를 호출하여 변경 사항을 한 번에 실행한다.

<br>

#### JpaItemWriter
JpaItemWriter 는 Jpa 의 javax.persistence.EntityManager 를 감싼 간단한 Wrapper 에 불과하다.

청크가 완료가 되면 청크 내의 아이템 목록이 JpaItemWriter 로 전달된다.

JpaItemWriter 는 모든 아이템을 저장한 뒤 flush 를 호출하기 전에 아이템 목록 내 아이템을 순회하면서 아이템마다 EntityManager 의 merge 메서드를 호출한다.

HibernateItemWriter 와의 차이점은 생성자에서 SessionFactory 대신 EntityManager 를 저장한다.

그리고 initialize 메서드에서 HibernateTransactionManager 를 생성하는 대신 JpaTransactionManager 를 생성한다.

```java
@Bean
public JpaItemWriter<Pay2> jpaItemWriter() {
  JpaItemWriter<Pay2> jpaItemWriter = new JpaItemWriter<>();
  jpaItemWriter.setEntityManagerFactory(entityManagerFactory);
  return jpaItemWriter;
}
```