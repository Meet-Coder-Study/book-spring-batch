## 8장. ItemProcessor

### ItemProcessor 소개

```java
public interface ItemProcessor<I, O> {
  O process(I item) throws Exception;
}
```

- `ItemProcessor`가 반환하는 타입(`O`)는 `ItemWriter`의 입력 타입
- `process()`가 `null`을 반환하면 해당 아이템의 이후 모든 처리가 중지된다
  - 스텝 내에서 해당 아이템의 이후 `ItemProcessor`들과 `ItemWriter`는 호출되지 않는다

### 스프링 배치가 제공하는 ItemProcessor 들

- `ValidatingItemProcessor` : 읽어온 데이터의 유효성 검증에 사용한다

  - 애너테이션을 이용해서 검증할 수 있다
    - `@NotNull`, `@Pattern`, `@Size` 등
  - 유효성 검증 기능은 `Validator`의 구현체가 제공하며, 이 인터페이스는 `void validate(T value);` 를 제공한다
  - 검증에 실패하면 `ValidationException`을 발생시킨다

  ```java
  @Bean
  public Step step() {
    return this.stepBuilderFactory.get("step")
        .<Customer, Customer>chunk(5)
        .reader(...)
        .processor(customerValidatingItemProcessor())
        .writer(...)
        .build();
  }
  
  @Bean
  public BeanValidatingItemProcessor<Customer> customerValidatingItemProcessor() {
    return new BeanValidatingItemProcessor<>();
  }
  ```

  - 만약 검증 로직을 직접 작성하고 싶다면? (예 : lastName이 유일해야 할 때)

  ```java
  public class UniqueLastNameValidator extends ItemStreamSupport implements Validator<Customer> {
  
    private Set<String> lastNames = new HashSet<>();
  
    @Override
    public void validate(Customer value) throws ValidationException {
      if (lastNames.contains(value.getLastName())) {
        throw new ValidationException("Duplicate last name was found: " + value.getLastName());
      }
      this.lastNames.add(value.getLastName());
    }
  
    // Execution 간에 상태를 유지하는 데 사용
    @Override
    public void update(ExecutionContext executionContext) {
      executionContext.put(getExecutionContextKey("lastNames"), this.lastNames);
    }
  
    // Execution 간에 상태를 유지하는 데 사용
    @Override
    public void open(ExecutionContext executionContext) {
      String lastNames = getExecutionContextKey("lastNames");
      if (executionContext.containsKey(lastNames)) {
        this.lastNames = (Set<String>) executionContext.get(lastNames);
      }
    }
  }
  ```

  ```java
  @Bean
  public Step step() {
    return this.stepBuilderFactory.get("step")
        .<Customer, Customer>chunk(5)
        .reader(...)
        .processor(customerValidatingItemProcessor())
        .writer(...)
        .stream(validator()) // <-- 
        .build();
  }
  
  // BeanValidatingItemProcessor -> ValidatingItemProcessor
  @Bean
  public ValidatingItemProcessor<Customer> customerValidatingItemProcessor() {
    return new ValidatingItemProcessor<>(validator());
  }
  
  @Bean
  public UniqueLastNameValidator validator() {
    UniqueLastNameValidator uniqueLastNameValidator = new UniqueLastNameValidator();
    uniqueLastNameValidator.setName("validator");
    return uniqueLastNameValidator;
  }
  ```

- `ItemProcessorAdapter` : 기존 서비스를 아이템 프로세서로 만들 수 있다

  - 예 : 이름을 대문자로 만드는 `UpperCaseNameService.upperCase(Customer customer)` 가 이미 있을때

  ```java
  @Bean
  public Step step() {
    return this.stepBuilderFactory.get("step")
        .<Customer, Customer>chunk(5)
        .reader(...)
        .processor(itemProcessor(null))
        .writer(...)
        .build();
  }
  
  @Bean
  public ItemProcessorAdapter<Customer, Customer> itemProcessor(UpperCaseNameService service) {
    ItemProcessorAdapter<Customer, Customer> adapter = new ItemProcessorAdapter<>();
    adapter.getTargetObject(service);
    adapter.getTargetMethod("upperCase");
    return adapter;
  }
  ```

- `ScriptItemProcessor` : 스크립트로 프로세서를 작성할 수 있다

  - 예 : 이름을 대문자로 만들기 (자바스크립트)

  ```java
  @Bean
  @StepScope
  public ScriptItemProcessor<Customer, Customer> itemProcessor(
    @Value("#{jobParameters['script']}") Resource script
  ) {
    ScriptItemProcessor<Customer, Customer> itemProcessor = new ScriptItemProcessor<>();
    itemProcessor.setScript(script);
    return itemProcessor;
  }
  ```

  이후, 잡을 실행할때 파라미터로 js파일의 위치를 입력하면 됨

- `CompositeItemProcessor` : 여러 프로세서를 체인처럼 연결할 수 있다

  - 이전 프로세서가 반환한 타입과 다음 프로세서의 입력 타입이 같아야 한다
  - 한 프로세서가 null을 반환하면 해당 아이템은 더이상 처리되지 않는다

  ```java
  @Bean
  public CompositeItemProcessor<Customer, Customer> itemProcessor() {
    CompositeItemProcessor<Customer, Customer> itemProcessor = new CompositeItemProcessor();
    itemProcessor.setDelegates(Arrays.asList(
      itemProcessor1(), itemProcessor2(), itemProcessor3()
    ));
    return itemProcessor;
  }
  ```

- `ClassifierCompositeItemProcessor` : 아이템마다 전달되는 itemProcessor를 다르게 하고 싶을때

  - 예 : 우편번호가 홀수인지, 짝수인지에 따라 전달되는 itemProcessor 다르게하기

  ```java
  @AllArgsConstructor
  public class ZipCodeClassifier implements Classifier<Customer, ItemProcessor<Customer, Customer>> {
  
    private ItemProcessor<Customer, Customer> oddItemProcessor;
    private ItemProcessor<Customer, Customer> evenItemProcessor;
  
    @Override
    public ItemProcessor<Customer, Customer> classify(Customer classifiable) {
      if (Integer.parseInt(classifiable.getZipCode()) % 2 == 0) {
        return evenItemProcessor;
      } else {
        return oddItemProcessor;
      }
    }
  }
  ```

  ```java
  @Bean
  public ClassifierCompositeItemProcessor<Customer, Customer> itemProcessor() {
    ClassifierCompositeItemProcessor<Customer, Customer> itemProcessor 
      = new ClassifierCompositeItemProcessor<>();
    itemProcessor.setClassifier(classifier());
    return itemProcessor;
  }
  
  @Bean
  public Classifier classifier() {
    return new ZipCodeClassifier(upperCaseItemProcessor(null), lowerCaseItemProcessor(null));
  }
  ```

### ItemProcessor 직접 만들기

- `ItemProcessor`가 null을 반환하면 해당 아이템을 필터링한다

- 스프링 배치는 필터링된 레코드수를 보관하고 `JobRepository`에 저장한다

- 아래 예시는 우편번호가 짝수일때 건너뛰는 프로세서를 직접 만든 예시.

  ```java
  public class EvenFilteringItemProcessor implements ItemProcessor<Customer, Customer> {
    @Override
    public Customer process(Customer item) throws Exception {
      return Integer.parseInt(item.getZipCode()) % 2 == 0 ? null : item;
    }
  }
  ```

  ```java
  @Bean
  public EvenFilteringItemProcessor itemProcessor() {
    return new EvenFilteringItemProcessor();
  }
  ```

- 다음 쿼리로 필터링된 아이템 수를 알 수 있다

  ```sql
  select STEP_EXECUTION_ID as ID, STEP_NAME, STATUS, COMMIT_COUNT, READ_COUNT, FILTER_COUNT, WRITE_COUNT
  from BATCH_STEP_EXECUTION;
  ```
