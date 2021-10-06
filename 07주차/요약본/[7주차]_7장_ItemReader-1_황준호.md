## 7장. ItemReader

### ItemReader 인터페이스

```java
public interface ItemReader<T> {
  T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;
}
```

- `ItemReader` 인터페이스는 전략 인터페이스다
- 스프링 배치는 플랫 파일, 여러 데이터베이스, JMS 리소스 등등 유형의 구현체를 제공한다
- `ItemReader`나 `ItemReader` 하위 인터페이스를 구현해서 커스텀하게 사용할수도 있다

### 파일 입력

#### 플랫 파일

- 가장 많이 사용하는 구현체인 `DefaultLineMapper`는 다음 두 개가 처리를 담당한다

  1. `LineTokenizer` : 파일의 한 줄 -> `FieldSet`
  2. `FieldSetMapper` : `FieldSet` -> 객체

- 고정 너비 파일

  ```java
  @Bean
  @StepScope
  public FlatFileItemReader<Customer> customerItemReader(@Value("#{jobParameters['customerFile']})") Resource inputFile) {
    return new FlatFileItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .resource(inputFile)
        .fixedLength()
        .columns(
            new Range[]{new Range(1, 11), new Range(12, 12), new Range(13, 22), new Range(23, 26),
                new Range(27, 46), new Range(47, 62), new Range(63, 64), new Range(65, 69)}
        )
        .names(
            new String[]{"firstName", "middleInitial", "lastName", "addressNumber", "street",
                "city", "state", "zipCode"}
        )
        .targetType(Customer.class)
        .build();
  }
  ```

- 필드가 구분자로 구분된 파일

  ```java
  @Bean
  @StepScope
  public FlatFileItemReader<Customer> customerItemReader(
    @Value("#{jobParameters['customerFile']})") Resource inputFile) {
    return new FlatFileItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .delimited()
        .names(new String[]{"firstName", "middleInitial", "lastName", "addressNumber", "street",
            "city", "state", "zipCode"})
        .targetType(Customer.class)
        .resource(inputFile)
        .build();
  }
  ```

  매핑을 커스텀하게 하고 싶으면?

  ```java
  @Bean
  @StepScope
  public FlatFileItemReader<Customer> customerItemReader(
    @Value("#{jobParameters['customerFile']})") Resource inputFile) {
    return new FlatFileItemReaderBuilder<Customer>()
        .name("customerItemReader")
        .delimited()
        .names(new String[]{"firstName", "middleInitial", "lastName", "addressNumber", "street",
            "city", "state", "zipCode"})
        .fieldSetMapper(new CustomFieldSetMapper())
        .resource(inputFile)
        .build();
  }
  ```

- `LineTokenizer`직접 구현

  - `LineTokenizer`을 implements하여 `FlatFileItemReader` config에서 설정한다

    ```java
    public interface LineTokenizer {
      FieldSet tokenizer(String line);
    }
    ```

  - 예

    ```java
    @Bean
    @StepScope
    public FlatFileItemReader<Customer> customerItemReader(
      @Value("#{jobParameters['customerFile']})") Resource inputFile) {
      return new FlatFileItemReaderBuilder<Customer>()
          .name("customerItemReader")
          .lineTokenizer(new CustomerFileLineTokenizer()) // <--
          .resource(inputFile)
          .build();
    }
    ```

- 여러 형식이 섞여 있는 경우

  - 예 : 접두사로 고객데이터, 거래데이터가 섞여 있는 경우

    ```
    CUST,Warren,Q,Darrow,8272 4th Street,New York,IL,76091
    TRANS,1165965,2011-01-22 00:13:29,51.43
    CUST,Ann,V,Gates,9247 Infinite Loop Drive,Hollywood,NE,37612
    CUST,Erica,I,Jobs,8875 Farnam Street,Aurora,IL,36314
    TRANS,8116369,2011-01-21 20:40:52,-14.83
    TRANS,8116369,2011-01-21 15:50:17,-45.45
    TRANS,8116369,2011-01-21 16:52:46,-74.6
    TRANS,8116369,2011-01-22 13:51:05,48.55
    TRANS,8116369,2011-01-21 16:51:59,98.53
    ```

  - ```java
    @Bean
    public PatternMatchingCompositeLineMapper lineTokenizer() {
      Map<String, LineTokenizer> lineTokenizers = new HashMap<>(2);
      lineTokenizers.put("CUST*", customerLineTokenizer());
      lineTokenizers.put("TRANS*", transactionLineTokenizer());
    
      Map<String, FieldSetMapper> fieldSetMappers = new HashMap<>(2);
      BeanWrapperFieldSetMapper<Customer> customerFieldSetMapper = new BeanWrapperFieldSetMapper<>();
      customerFieldSetMapper.setTargetType(Customer.class);
      fieldSetMappers.put("CUST*", customerFieldSetMapper);
      fieldSetMappers.put("TRANS*", new TransactionFieldSetMapper());
    
      PatternMatchingCompositeLineMapper lineMappers = new PatternMatchingCompositeLineMapper();
      lineMappers.setTokenizers(lineTokenizers);
      lineMappers.setFieldSetMappers(fieldSetMappers);
      return lineMappers;
    }
    
    @Bean
    public DelimitedLineTokenizer transactionLineTokenizer() {
      DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();
      lineTokenizer.setNames("prefix", "accountNumber", "transactionDate", "amount");
      return lineTokenizer;
    }
    
    @Bean
    public DelimitedLineTokenizer customerLineTokenizer() {
      DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();
      lineTokenizer.setNames(
          "firstName", "middleInitial", "lastName", "address", "city", "state", "zipCode"
      );
      lineTokenizer.setIncludedFields(1, 2, 3, 4, 5, 6, 7);
      return lineTokenizer;
    }
    ```

    ```java
    public class TransactionFieldSetMapper implements FieldSetMapper<Transaction> {
      public Transaction mapFieldSet(FieldSet fieldSet) {
        Transaction trans = new Transaction();
        trans.setAccountNumber(fieldSet.readString("accountNumber"));
        trans.setAmount(fieldSet.readDouble("amount"));
        trans.setTransactionDate(fieldSet.readDate("transactionDate", "yyyy-MM-dd HH:mm:ss"));
        return trans;
      }
    }
    ```

- 이후 예제 생략

#### XML

- XML파서로는 DOM파서와 SAX파서를 많이 이용한다
  - DOM파서는 부하가 커서 배치 처리엔 적합하지 않아 SAX파서를 자주 사용한다
  - SAX파서 : 특정 엘리먼트를 만나면 이벤트를 발생시키는 이벤트 기반 파서
  - 스프링 배치에서는  StAX파서를 사용한다
- 해야하는 것들
  - oxm관련 의존성 추가
  - 도메인 객체에 JAXB 애너테이션 추가
    - @XmlRootElement, @XmlElementWrapper, @XmlElement ...
  - 마샬러 설정
  - 아이템리더 설정

### JSON

- JsonItemReader는 3가지 의존성이 필요하다

  1. 배치를 재시작할때 사용하는 배치의 이름
  2. 파싱에 사용할 `JsonObjectReader`
  3. 읽어들일 리소스

  ```java
  @Bean
  @StepScope
  public JsonItemReader<Customer> customerFileReader(
      @Value("#{jobParameters['customerFile']}") Resource inputFile) {
  
    ObjectMapper objectMapper = new ObjectMapper();
    ObjectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));
  
    JacksonJsonObjectReader<Customer> jsonObjectReader = new JacksonJsonObjectReader<>(Customer.class);
    jsonObjectReader.setMapper(objectMapper);
  
    return new JsonItemReaderBuilder<Customer>()
        .name("customerFileReader") // <-- 1
        .jsonObjectReader(jsonObjectReader) // <-- 2
        .resource(inputFile) // <-- 3
        .build();
  }
  ```
