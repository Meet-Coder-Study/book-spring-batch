
# ItemWriter

ItemReader에 의해 읽힌 데이터를 필터링하여 ItemWriter가 진행될 수 있도록 하기 위한 단계이다.

## ItemWriter 소개

청크 기반으로 묶음 단위로 아이템을 처리한다. 
ItemReader와 ItemProcessor가 아이템을 건건히 처리하며 청크 단위에 도달하면 ItemWriter가 처리한다.
(사진)

## 파일기반 ItemWriter

### FlatFileItemWriter
텍스트 파일로 출력을 만들 때 사용한다.

### StaxEventItemWriter
XML 파일로 출력을 만들 때 사용한다.

### 데이터베이스기반 ItemWriter

#### JdbcBatchItemWriter

#### HibernateItemWriter

#### JpaItemWriter

### 스프링 데이터의 ItemWriter

