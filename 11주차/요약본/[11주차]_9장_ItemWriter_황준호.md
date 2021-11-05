## 9장. ItemWriter

### 그밖의 출력 방식을 위한 ItemWriter

- `ItemWriterAdapter`

  - 기존 스프링 서비스를 ItemWriter로 이용할 수 있다.
  - 다음 예는 기존에 존재하는 서비스인 `CustomerService::method`를 이용하는 코드

  ```java
  @Bean
  public ItemWriterAdapter<Customer> itemWriter(CustomerService customerService) {
    ItemWriterAdapter<Customer> customerItemWriterAdapter = new ItemWriterAdapter<>();
    customerItemWriterAdapter.setTargetObject(customerService);
    customerItemWriterAdapter.setTargetMethod("method"); // 단, 이 메서드는 인자로 Customer 하나만 받아야 한다
    return customerItemWriterAdapter;
  }
  ```

  - 하지만, 이미 존재하는 서비스가 `Customer`를 받아들이지 않는다면, 값을 추출해 전달해야 한다. 이때는 다음에서 소개하는 `PropertyExtractingDelegatingItemWriter`를 사용한다

- `PropertyExtractingDelegatingItemWriter`

  - 아이템에서 값을 추출한 후 서비스에 파라미터로 전달한다.

  ```java
  // 기존에 존재하는 서비스
  @Service
  public CustomerService {
    public void method(String address, String city, String state, String zip) {
      ...
    }
  }
  ```

  ```java
  @Bean
  public PropertyExtractingDelegatingItemWriter<Customer> itemWriter(CustomerService customerService) {
    PropertyExtractingDelegatingItemWriter<Customer> itemWriter 
      = new PropertyExtractingDelegatingItemWriter<>();
    itemWriter.setTargetObject(customerService);
    itemWriter.setTargetMethod("method");
    itemWriter.setFieldsUsedAsTargetMethodArguments(new String[]{"address", "city", "state", "zip"});
    return customerItemWriterAdapter;
  }
  ```

- `JmsItemWriter` - 생략

- `SimpleMailMessageItemWriter`

  - 이메일을 보낼 수 있다. 아래 예는 모든 신규 고객 정보를 불러온 후 신규 고객 모두에게 환영 이메일을 보내는 코드

  - 설정해야 하는 것들

    1. 자바 Mail 의존성 추가 

       `implementation 'org.springframework.boot:spring-boot-starter-mail'`

    2. `application.yml` 설정

       ```yaml
       spring:
         mail:
           host: smtp.gmail.com
           port: 587
           username: ...
           password: ...
           properties:
             mail:
               smtp:
                 auth: true
                 starttls:
                   enable: true
       ```

    3. 잡 config

       ```java
       @Bean
       public Step emailStep() {
         return stepBuilderFactory.get("emailStep")
             .<Customer, SimpleMailMessage>chunk(10)
             .reader(...)
             .processor(itemProcessor())
             .writer(emailItemWriter(null))
             .build();
       }
       
       @Bean
       public ItemProcessor<Customer, SimpleMailMessage> itemProcessor() {
         return customer -> {
           SimpleMailMessage mail = new SimpleMailMessage();
           mail.setFrom("service@gmail.com");
           mail.setTo(customer.getEmail());
           mail.setSubject("환영합니다!");
           mail.setText(String.format("%s %s님, 회원가입을 축하합니다!", 
             customer.getFirstName(), customer.getLastName()));
           return mail;
         };
       }
       
       @Bean
       public SimpleMailMessageItemWriter emailItemWriter(MailSender mailSender) {
         return new SimpleMailMessageItemWriterBuilder()
             .mailSender(mailSender)
             .build();
       }
       ```

### 여러 자원을 사용하는 ItemWriter

- `MultiResourceItemWriter`

  - 일정 수 기준으로 나눠서 여러 파일로 저장하고 싶을때 사용한다

  ```java
  @Bean
  public MultiResourceItemWriter<Customer> multiCustomerFileWriter() {
    return new MultiResourceItemWriterBuilder<Customer>()
        .name("multiCustomerFileWriter")
        .delegate(delegateItemWriter()) //실제 쓰기를 수행할 ItemWriter
        .itemCountLimitPerResource(25) //각 리소스에 쓰기를 수행할 아이템 수 (25개가 넘게 저장될 수도 있다)
        .resource(new FileSystemResource("chapter9/output/customers")) // 저장될 위치(루트 기준)
        .resourceSuffixCreator(index -> index + ".csv") // customers1.csv, customers2.csv... 이렇게 저장됨
        .build();
  }
  ```

  - 총 회원수가 100명, 청크 크기가 10, itemCountLimitPerResource가 25인 경우?
    - 파일은 총 4개가 생긴다. (`customers1.csv`, `customers2.csv`, `customers3.csv`, `customers4.csv`)
    - 각 파일에는 25명씩 저장되는 게 아니라 30명, 30명, 30명, 10명 저장된다
    - 매 청크 기준으로 쓰기를 수행할 때마다 쓰기 후 청크 경계에 도달했는지 체크하기 때문에 청크 중간에 새 리소스를 생성하지 않기 때문
  - 헤더와 푸터를 추가하는 부분은 생략

- `CompositeItemWriter`

  ```java
  @Bean
  public CompositeItemWriter<Customer> compositeItemWriter() {
    return new CompositeItemWriterBuilder<Customer>()
        .delegates(itemWriter1(), itemWriter2(), itemWriter3())
        .build();
  }
  ```

  - 쓰기 실행이 순차적으로 한번에 하나의 라이터에서 일어난다
    - itemWriter1수행 -> itemWriter2수행 -> itemWriter3수행
  - 모든 쓰기작업이 하나의 트랜잭션에서 일어나기 때문에 itemWriter2에서 실패하면 itemWriter1도 롤백된다
  - 100개를 썼다면 `JobRepository`에 쓰기 수가 300이 아닌 100이 기록된다.  (스프링 배치는 아이템수를 세기 때문)

- `ClassifierCompositeItemWriter`

  - A~M에 해당하는 주에 사는 고객은 플랫 파일, N~Z에 해당하는 주에 사는 고객은 데이터베이스에 쓰고 싶을땐?

  ```java
  @AllArgsConstructor
  public class CustomerClassifier implements Classifier<Customer, ItemWriter<? super Customer>> {
  
    private ItemWriter<Customer> fileItemWriter;
    private ItemWriter<Customer> jdbcItemWriter;
  
    @Override
    public ItemWriter<Customer> classify(Customer customer) {
      if (customer.getState().matches("^[A-M].*")) {
        return fileItemWriter;
      } else {
        return jdbcItemWriter;
      }
    }
  }
  ```

  ```java
  @Bean
  public Step classifierCompositeWriterStep() {
    return this.stepBuilderFactory.get("classifierCompositeWriterStep")
        .<Customer, Customer>chunk(10)
        .reader(...)
        .writer(classifierCompositeItemWriter())
        // ClassifierCompositeItemWriter은 ItemStream을 구현하고 있지 않기 때문에
        // 아래 stream을 빼먹으면 xml쓰기 시도시 오류가 난다.
        // 상태를 유지하며 작업을 수행할 수 있도록 수동으로 등록해줘야 한다.
        .stream(xmlItemWriter())
        .build();
  }
  
  @Bean
  public ClassifierCompositeItemWriter<Customer> classifierCompositeItemWriter() {
    Classifier<Customer, ItemWriter<? super Customer>> classifier
        = new CustomerClassifier(xmlItemWriter(), jdbcItemWriter());
  
    return new ClassifierCompositeItemWriterBuilder<Customer>()
        .classifier(classifier)
        .build();
  }
  ```

