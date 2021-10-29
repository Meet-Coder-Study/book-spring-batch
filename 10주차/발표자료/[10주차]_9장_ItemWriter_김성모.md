# ItemWriter

ItemReader에 의해 읽힌 데이터를 필터링하여 ItemWriter가 진행될 수 있도록 하기 위한 단계이다.

## ItemWriter 소개

청크 기반으로 묶음 단위로 아이템을 처리한다. ItemReader와 ItemProcessor가 아이템을 건건히 처리하며 청크 단위에 도달하면 ItemWriter가 처리한다.
![처리프로세스](https://user-images.githubusercontent.com/48056463/139166357-c7424f2d-02cd-4885-9afb-cc52860acd02.png)

청크 기반 처리 방식 시 파일 쓰기와 같은 트랜잭션이 적용되지 않는 리소스를 처리할 때 이미 쓰여진 내용을 롤백할 수 있는 방법이 없다. 하지만 스프링배치는 추가적인 보호조치에 대한 조치 취해진 ItemWriter를
제공해준다.

## 파일기반 ItemWriter

### FlatFileItemWriter

텍스트 파일로 출력을 만들 때 사용한다.

FlatFileItemWriter 는 출력할 리소스(`Resource`) 와 `LineAggregator` 구현체로 구성된다.
`LineAggregator`는 객체를 기반으로 출력 문자열을 생성하는 역할을 한다.

파일에 트랜잭션을 적용한 후 롤백 처리를 위하여 `FlatFileItemWriter` 는 롤백이 가능한 커밋 전 마지막 순간까지 출력 데이터의 저장을 지연시킨다.

이 기능은 `TransactionSynchronizationApter`의 `beforeCommit` 메서드로 구현되어 있다.


code 링크

### StaxEventItemWriter

XML 파일로 출력을 만들 때 사용한다.

의존성 추가
```
implementation 'com.thoughtworks.xstream:xstream:1.4.18'
implementation 'org.springframework:spring-oxm:5.3.11'
```

code 링크

### 데이터베이스기반 ItemWriter

위에서 봤던 파일과는 다르게 트랜잭션이 적용되는 리소스다.

#### JdbcBatchItemWriter

> 하나의 청크에 대한 배치업데이트 기능을 사용한다.

code 링크

#### HibernateItemWriter

의존성 추가
```
implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
```

#### JpaItemWriter

### 스프링 데이터의 ItemWriter

