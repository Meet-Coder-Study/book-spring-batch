# 목차

- [스프링 배치의 ItemProcessor 사용하기](#스프링-배치의-ItemProcessor-사용하기)
  - [BeanValidatingItemProcessor](#BeanValidatingItemProcessor)
  - [ValidatingItemProcessor](#ValidatingItemProcessor)
  - [ItemProcessorAdapter](#ItemProcessorAdapter)
  - [ScriptItemProcessor](#ScriptItemProcessor)
  - [CompositeItemProcessor](#CompositeItemProcessor)
- [ItemProcessor 직접 만들기](#ItemProcessor-직접-만들기)
---  

# 스프링 배치의 ItemProcessor 사용하기

## BeanValidatingItemProcessor

JSR-303(Bean 유효성 검증 사양)을 통해 Item 유효성을 검증할 수 있다.

**(1) Validation 어노테이션 적용하기**

```java
...
public class Customer {

    @NotNull(message = "First name is required")
    @Pattern(regexp = "[a-zA-Z]+", message = "First name must be alphabetical")
    private String firstName;

    // size와 pattern을 별도로 지정하여 에러 메시지를 구분하고 디버깅이 유용해진다.
    @Size(min = 1, max = 1)
    @Pattern(regexp = "[a-zA-Z]", message = "Middle initial must be alphabetical")
    private String middleInitial;

    @NotNull(message = "Last name is required")
    @Pattern(regexp = "[a-zA-Z]+", message = "Last name must be alphabeticla")
    private String lastName;

    @NotNull(message = "Address is required")
    @Pattern(regexp = "[0-9a-zA-Z\\. ]+")
    private String address;

    @NotNull(message = "City is required")
    @Pattern(regexp = "[a-zA-Z\\. ]+")
    private String city;

    @NotNull(message = "State is required")
    @Size(min = 2, max = 2)
    @Pattern(regexp = "[A-Z]{2}")
    private String state;

    @NotNull(message = "Zip is required")
    @Size(min = 5, max = 5)
    @Pattern(regexp = "\\d{5}")
    private String zip;
}
```  

**(2) ItemProcessor 구성하기**

```java
@Bean
public BeanValidatingItemProcessor<Customer> customerValidatingItemProcessor() {
    return new BeanValidatingItemProcessor<>();
}
```  

**(3) Input**

```csv
Richard,N,Darrow,5570 Isabella Ave,St. Louis,IL,58540
Barack,G,Donnelly,7844 S. Greenwood Ave,Houston,CA,38635
Ann,Z,Benes,2447 S. Greenwood Ave,Las Vegas,NY,55366
Laura,9S,Minella,8177 4th Street,Dallas,FL,04119          <-- validation fail record.
Erica,Z,Gates,3141 Farnam Street,Omaha,CA,57640
Warren,L,Darrow,4686 Mt. Lee Drive,St. Louis,NY,94935
Warren,M,Williams,6670 S. Greenwood Ave,Hollywood,FL,37288
Harry,T,Smith,3273 Isabella Ave,Houston,FL,97261
Steve,O,James,8407 Infinite Loop Drive,Las Vegas,WA,90520
Erica,Z,Neuberger,513 S. Greenwood Ave,Miami,IL,12778
Aimee,C,Hoover,7341 Vel Avenue,Mobile,AL,35928
Jonas,U,Gilbert,8852 In St.,Saint Paul,MN,57321
Regan,M,Darrow,4851 Nec Av.,Gulfport,MS,33193
Stuart,K,Mckenzie,5529 Orci Av.,Nampa,ID,18562
Sydnee,N,Robinson,894 Ornare. Ave,Olathe,KS,25606
```  

**(4) 결과 확인하기**

```shell
org.springframework.batch.item.validator.ValidationException: Validation failed for Customer{firstName=Laura, middleInitial=9S, lastName=Minella, address=8177 4th Street, city=Dallas, state=FL, zip=04119}: 
Field error in object 'item' on field 'middleInitial': rejected value [9S]; codes [Pattern.item.middleInitial,Pattern.middleInitial,Pattern.java.lang.String,Pattern]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.middleInitial,middleInitial]; arguments []; default message [middleInitial],[Ljavax.validation.constraints.Pattern$Flag;@26a262d6,[a-zA-Z]]; default message [Middle initial must be alphabetical]
Field error in object 'item' on field 'middleInitial': rejected value [9S]; codes [Size.item.middleInitial,Size.middleInitial,Size.java.lang.String,Size]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.middleInitial,middleInitial]; arguments []; default message [middleInitial],1,1]; default message [크기가 1에서 1 사이여야 합니다]
```  

```sql
 SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+--------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME    | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | LAST_UPDATED               |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+--------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+
|                 1 |       3 | copyFileStep |                1 | 2021-10-19 10:50:52.470000 | 2021-10-19 10:50:52.634000 | FAILED |            1 |          6 |            0 |           3 |               0 |                0 |                  0 |              1 | FAILED    | org.springframework.batch.item.validator.ValidationException: Validation failed for Customer{firstName=Laura, middleInitial=9S, lastName=Minella, address=8177 4th Street, city=Dallas, state=FL, zip=04119}: 
Field error in object 'item' on field 'middleInitial': rejected value [9S]; codes [Pattern.item.middleInitial,Pattern.middleInitial,Pattern.java.lang.String,Pattern]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.middleInitial,middleInitial]; arguments []; default message [middleInitial],[Ljavax.validation.constraints.Pattern$Flag;@26a262d6,[a-zA-Z]]; default message [Middle initial must be alphabetical]
Field error in object 'item' on field 'middleInitial': rejected value [9S]; codes [Size.item.middleInitial,Size.middleInitial,Size.java.lang.String,Size]; arguments [org.springframework.context.support.DefaultMessageSourceResolvable: codes [item.middleInitial,middleInitial]; arguments []; default message [middleInitial],1,1]; default message [??? 1?? 1 ???? ???]
        at org.springframework.batch.item.validator.SpringValidator.validate(SpringValidator.java:54)
        at org.springframework.batch.item.validator.ValidatingItemProcessor.process(ValidatingItemProcessor.java:84)
        at org.springframework.batch.core.step.item.SimpleChunkProcessor.doProcess(SimpleChunkProcessor.java:134)
        at org.springframework.batch.core.step.item.SimpleChunkProcessor.transform(SimpleChunkProcessor.java:319)
        at org.springframework.batch.core.step.item.SimpleChunkProcessor.process(SimpleChunkProcessor.java:210)
        at org.springframework.batch.core.step.item.ChunkOrientedTasklet.execute(ChunkOrientedTasklet.java:77)
        at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:407)
        at org.springframework.batch.core.step.tasklet.TaskletStep$ChunkTransactionCallback.doInTransaction(TaskletStep.java:331)
        at org.springframework.transaction.support.TransactionTemplate.execute(TransactionTemplate.java:140)
        at org.springframework.batch.core.step.tasklet.TaskletStep$2.doInChunkContext(TaskletStep.java:273)
        at org.springframework.batch.core.scope.context.StepContextRepeatCallback.doInIteration(StepContextRepeatCallback.java:82)
        at org.springframework.batch.repeat.support.RepeatTemplate.getNextResult(RepeatTemplate.java:375)
        at org.springframework.batch.repeat.support.RepeatTemplate.executeInternal(RepeatTemplate.java:215)
        at org.springframework.batch.repeat.support.RepeatTemplate.iterate(Repeat | 2021-10-19 10:50:52.635000 |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+--------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+----------------------------+
1 row in set (0.00 sec)
```  

---  

## ValidatingItemProcessor

`org.springframework.batch.item.validator.Validator`를 구현하여 직접 유효성을 검증한다.

**(1) `lastName` 중복 아이템 유효성 검사**

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

    @Override
    public void open(ExecutionContext executionContext) {
        String lastNames = getExecutionContextKey("lastNames");

        if (executionContext.containsKey(lastNames)) {
            this.lastNames = (Set<String>) executionContext.get(lastNames);
        }
    }

    @Override
    public void update(ExecutionContext executionContext) {
        Iterator<String> iterator = lastNames.iterator();
        Set<String> copiedLastNames = new HashSet<>();
        while (iterator.hasNext()) {
            copiedLastNames.add(iterator.next());
        }

        executionContext.put(getExecutionContextKey("lastNames"), copiedLastNames);
    }
}
```  

**(2) ItemProcessor 구성하기**

```java
@Bean
public UniqueLastNameValidator validator() {
    UniqueLastNameValidator validator = new UniqueLastNameValidator();

    validator.setName("validator");

    return validator;
}

@Bean
public ValidatingItemProcessor<Customer> customerValidatingItemProcessor() {
    return new ValidatingItemProcessor<>(validator());
}
```

---  

## ItemProcessorAdapter

**(1) 기존 Service**

```java
@Service
public class UpperCaseNameService {

    public Customer upperCase(Customer customer) {
        return Customer.builder()
                       .firstName(customer.getFirstName().toUpperCase())
                       .middleInitial(customer.getMiddleInitial().toUpperCase())
                       .lastName(customer.getLastName().toUpperCase())
                       .address(customer.getAddress())
                       .city(customer.getCity())
                       .state(customer.getState())
                       .zip(customer.getZip())
                       .build();
    }
}
```

**(2) ItemProcessor 구성**

```java
@Bean
public ItemProcessorAdapter<Customer, Customer> itemProcessor(UpperCaseNameService service) {
    ItemProcessorAdapter<Customer, Customer> adapter = new ItemProcessorAdapter<>();

    adapter.setTargetObject(service);
    adapter.setTargetMethod("upperCase");

    return adapter;
}
```  

**(3) 실행 결과**

```log
2021-10-19 20:32:17.614  INFO 8473 --- [           main] io.spring.batch.example3.AdapterMain     : Write items: 3
2021-10-19 20:32:17.616  INFO 8473 --- [           main] io.spring.batch.example3.AdapterMain     : Customer{firstName=RICHARD, middleInitial=N, lastName=DARROW, address=5570 Isabella Ave, city=St. Louis, state=IL, zip=58540}
2021-10-19 20:32:17.616  INFO 8473 --- [           main] io.spring.batch.example3.AdapterMain     : Customer{firstName=BARACK, middleInitial=G, lastName=DONNELLY, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635}
2021-10-19 20:32:17.616  INFO 8473 --- [           main] io.spring.batch.example3.AdapterMain     : Customer{firstName=ANN, middleInitial=Z, lastName=BENES, address=2447 S. Greenwood Ave, city=Las Vegas, state=NY, zip=55366}
```  


---  

## ScriptItemProcessor

- TODO: check supported script engines

**(1) js 파일 작성하기**

```js
item.setFirstName(item.getFirstName().toUpperCase());
item.setMiddleInitial(item.getMiddleInitial().toUpperCase());
item.setLastName(item.getLastName().toUpperCase());
item;
```

**(2) ItemProcessor 구성하기**

```java
@Bean
@StepScope
public ScriptItemProcessor<Customer, Customer> itemProcessor(
        @Value("#{jobParameters['script']}") Resource script) {
    ScriptItemProcessor<Customer, Customer> itemProcessor = new ScriptItemProcessor<>();

    itemProcessor.setScript(script);

    return itemProcessor;
}
```  

---  

## CompositeItemProcessor

- ItemProcessor-1: `ValidatingItemProcessor`를 이용하여 유효성 검사 / 실패한 아이템은 필터링
- ItemProcessor-2: `ItemProcessorAdapter`를 이용하여 기존 서비스를 활용한다.
- ItemProcessor-3: `ScriptItemProcessor`를 이용하여 address, city, state를 소문자로 변경한다.

**(1) 유효성 검사 ItemProcessor 구성하기**

```java
@Bean
public UniqueLastNameValidator validator() {
    UniqueLastNameValidator validator = new UniqueLastNameValidator();

    validator.setName("validator");

    return validator;
}

@Bean
public ValidatingItemProcessor<Customer> customerValidatingItemProcessor() {
    ValidatingItemProcessor<Customer> itemProcessor = new ValidatingItemProcessor<>(validator());

    // 유효성 검사 실패한 아이템은 필터링한다.
    itemProcessor.setFilter(true);

    return itemProcessor;
}
```  

**(2) ItemProcessorAdapter 구성하기**

```java
@Bean
public ItemProcessorAdapter<Customer, Customer> upperCaseItemProcessor(UpperCaseNameService service) {
    ItemProcessorAdapter<Customer, Customer> adapter = new ItemProcessorAdapter<>();

    adapter.setTargetObject(service);
    adapter.setTargetMethod("upperCase");

    return adapter;
}
```

**(3) ScriptItemProcessor 구성하기**

```js
item.setFirstName(item.getAddress().toLowerCase());
item.setMiddleInitial(item.getCity().toLowerCase());
item.setLastName(item.getState().toLowerCase());
item;
```

```java
@Bean
@StepScope
public ScriptItemProcessor<Customer, Customer> lowerCaseItemProcessor(
        @Value("#{jobParameters['script']}") Resource script) {
    ScriptItemProcessor<Customer, Customer> itemProcessor = new ScriptItemProcessor<>();

    itemProcessor.setScript(script);

    return itemProcessor;
}
```  

**(4) CompositeItemProcessor 구성하기**

```java
@Bean
public CompositeItemProcessor<Customer, Customer> itemProcessor() {
    CompositeItemProcessor<Customer, Customer> itemProcessor = new CompositeItemProcessor<>();

    itemProcessor.setDelegates(Arrays.asList(
            customerValidatingItemProcessor(),
            upperCaseItemProcessor(null),
            lowerCaseItemProcessor(null)
    ));

    return itemProcessor;
}
```  

**(5) 실행하기**

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME    | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       8 | copyFileStep |                1 | 2021-10-19 12:23:51.542000 | 2021-10-19 12:23:52.061000 | COMPLETED |            6 |         15 |            2 |          13 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-19 12:23:52.062000 |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
```  

---  

# ItemProcessor 직접 만들기

요구 사항: 홀수 우편번호만 기록하고, 짝수 우편번호는 필터링하는 `ItemProcessor` 구현

**(1) ItemProcessor 구현하기**

```java
public class EvenFilteringItemProcessor implements ItemProcessor<Customer, Customer> {

    @Override
    public Customer process(Customer item) throws Exception {
        return Integer.parseInt(item.getZip()) % 2 == 0 ? null : item;
    }
}
```  

**(2) ItemProcessor 구성하기**

```java
@Bean
public EvenFilteringItemProcessor itemProcessor() {
    return new EvenFilteringItemProcessor();
}
```

**(3) 실행하기**

```log
2021-10-19 21:29:38.973  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Write items: 1
2021-10-19 21:29:38.975  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Barack, middleInitial=G, lastName=Donnelly, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635}
2021-10-19 21:29:38.989  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Write items: 2
2021-10-19 21:29:38.989  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Laura, middleInitial=9S, lastName=Minella, address=8177 4th Street, city=Dallas, state=FL, zip=04119}
2021-10-19 21:29:38.989  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Warren, middleInitial=L, lastName=Darrow, address=4686 Mt. Lee Drive, city=St. Louis, state=NY, zip=94935}
2021-10-19 21:29:39.001  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Write items: 1
2021-10-19 21:29:39.001  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Harry, middleInitial=T, lastName=Smith, address=3273 Isabella Ave, city=Houston, state=FL, zip=97261}
2021-10-19 21:29:39.013  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Write items: 1
2021-10-19 21:29:39.013  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Jonas, middleInitial=U, lastName=Gilbert, address=8852 In St., city=Saint Paul, state=MN, zip=57321}
2021-10-19 21:29:39.027  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Write items: 1
2021-10-19 21:29:39.027  INFO 12353 --- [           main] i.s.b.example6.CustomItemProcessorMain   : Customer{firstName=Regan, middleInitial=M, lastName=Darrow, address=4851 Nec Av., city=Gulfport, state=MS, zip=33193}
```

```sql
mysql> SELECT * FROM BATCH_STEP_EXECUTION;
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
| STEP_EXECUTION_ID | VERSION | STEP_NAME    | JOB_EXECUTION_ID | START_TIME                 | END_TIME                   | STATUS    | COMMIT_COUNT | READ_COUNT | FILTER_COUNT | WRITE_COUNT | READ_SKIP_COUNT | WRITE_SKIP_COUNT | PROCESS_SKIP_COUNT | ROLLBACK_COUNT | EXIT_CODE | EXIT_MESSAGE | LAST_UPDATED               |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
|                 1 |       8 | copyFileStep |                1 | 2021-10-19 12:29:38.921000 | 2021-10-19 12:29:39.057000 | COMPLETED |            6 |         15 |            9 |           6 |               0 |                0 |                  0 |              0 | COMPLETED |              | 2021-10-19 12:29:39.059000 |
+-------------------+---------+--------------+------------------+----------------------------+----------------------------+-----------+--------------+------------+--------------+-------------+-----------------+------------------+--------------------+----------------+-----------+--------------+----------------------------+
```