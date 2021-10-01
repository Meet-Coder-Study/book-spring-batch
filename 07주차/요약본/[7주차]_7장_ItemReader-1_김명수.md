# ItemReader Code Examples

- [FlatFile](#FlatFile)
    - [고정 너비 기반 파일](#고정-너비-기반-파일)
    - [구분자 기반 파일](#구분자-기반-파일)
    - [커스텀 바인딩](#커스텀-바인딩)
    - [커스텀 레코드 파싱](#커스텀-레코드-파싱)
    - [여러 가지 레코드 포맷](#여러-가지-레코드-포맷)
    - [XML 파일](#XML-파일)
    - [JSON 파일](#JSON-파일)

*전체 소스코드는 [Chapter07](https://github.com/zacscoding/spring-boot-example/tree/master/book/spring-batch/Chapter07)에서 확인 할 수 있습니다.*

---  

# FlatFile

아래와 같이 고객 정보를 나타내는 `Customer` 와 거래를 나타내는 `Transaction`을 정의

> Customer.java

```java
public class Customer {

    private String firstName;
    private String middleInitial;
    private String lastName;
    private String addressNumber;
    private String street;
    private String city;
    private String state;
    private String zipCode;
    private List<Transaction> transactions;
    
    ...
}
```   

> Transaction.java

```java
public class Transaction {

    private String accountNumber;
    private Date transactionDate;
    private Double amount;
    
    ...
}
```

고정 너비 기반의 파일, 구분자(`,`) 기반의 파일, Customer-Transaction 모두 존재하는 파일, XML, JSON 등의 고객 데이터가 존재할 때 각각에 맞는 **ItemReader**를 구성하자.

---  

## 고정 너비 기반 파일

아래와 같이 필드 마다 길이가 고정된 고객 정보 파일을 정의

> customerFixedWidth.txt

```text
Aimee      CHoover    7341Vel Avenue          Mobile          AL35928
Jonas      UGilbert   8852In St.              Saint Paul      MN57321
Regan      MBaxter    4851Nec Av.             Gulfport        MS33193
Octavius   TJohnson   7418Cum Road            Houston         TX51507
Sydnee     NRobinson  894 Ornare. Ave         Olathe          KS25606
```  

아래와 같이 `fixedLength()`를 이용하여 파일을 파싱할 수 있다.

```java
@Bean
@StepScope
public FlatFileItemReader<Customer> customerFlatFileItemReader(
        @Value("#{jobParameters['customerFile']}") Resource inputFile) {

    logger.info("Activated flat file reader: fixed length");

    return new FlatFileItemReaderBuilder<Customer>()
            // ItemStream 인터페이스는 애플리케이션 내 각 스텝의 ExecutionContext에 추가되는 특정 키의
            // 접두문자로 사용될 이름이 필요
            .name("customerItemReader")
            .resource(inputFile)
            // FixedLengthTokenizer는 각 줄을 파싱해 FieldSet으로 만드는 LineTokenizer의 구현체
            .fixedLength()
            .columns(createRange(new int[][] {
                    { 1, 11 },
                    { 12, 12 },
                    { 13, 22 },
                    { 23, 26 },
                    { 27, 46 },
                    { 47, 62 },
                    { 63, 64 },
                    { 65, 69 }
            }))
            .names("firstName",
                   "middleInitial",
                   "lastName",
                   "addressNumber",
                   "street",
                   "city",
                   "state",
                   "zipCode")
            .targetType(Customer.class)
            .build();
}

private Range[] createRange(int[][] minMaxArrays) {
    Range[] ranges = new Range[minMaxArrays.length];
    for (int i = 0; i < minMaxArrays.length; i++) {
        ranges[i] = new Range(minMaxArrays[i][0], minMaxArrays[i][1]);
    }
    return ranges;
}
```  

그 외에 단순히 Console 출력만 담당하는 **ItemWriter**를 등록하여 Job을 구성하자.

```java
public class BatchConfiguration {

    private final JobBuilderFactory jobBuilderFactory;
    private final StepBuilderFactory stepBuilderFactory;
    // `fixedLength`, `delimiter` 등 다양한 Reader 구성을 변경한다.
    private final FlatFileItemReader<Customer> reader;

    @Bean
    public Job copyCustomerJob() {
        return jobBuilderFactory.get("copyCustomerJob")
                                .incrementer(new RunIdIncrementer())
                                .start(copyFileStep())
                                .build();
    }

    @Bean
    public Step copyFileStep() {
        return stepBuilderFactory.get("copyFileStep")
                                 .<Customer, Customer>chunk(10)
                                 .reader(reader)
                                 .writer(customerConsoleItemWriter())
                                 .build();
    }

    @Bean
    public ItemWriter<Customer> customerConsoleItemWriter() {
        return items -> {
            logger.info("Write items: {}", items.size());
            items.forEach(item -> logger.info(item.toString()));
        };
    }
}
```  

---  

## 구분자 기반 파일

아래와 같이 `,`로 구분된 고객데이터가 있을때 **ItemReader**를 구성하자.

> customerDelimiter.txt

```text
Aimee,C,Hoover,7341,Vel Avenue,Mobile,AL,35928
Jonas,U,Gilbert,8852,In St.,Saint Paul,MN,57321
Regan,M,Baxter,4851,Nec Av.,Gulfport,MS,33193
Octavius,T,Johnson,7418,Cum Road,Houston,TX,51507
```  

`delimited()` 메소드를 통해 구분자를 지정해줄 수 있다. 기본값은 `,`이다.

```java
@Bean
@StepScope
public FlatFileItemReader<Customer> customerFlatFileItemReader(
        @Value("#{jobParameters['customerFile']}") Resource inputFile) {

    logger.info("Activated flat file reader: delimited");

    return new FlatFileItemReaderBuilder<Customer>()
            // ItemStream 인터페이스는 애플리케이션 내 각 스텝의 ExecutionContext에 추가되는 특정 키의
            // 접두문자로 사용될 이름이 필요
            .name("customerItemReader")
            .resource(inputFile)
            .delimited()
            .delimiter(",") // default ","
            .names("firstName",
                   "middleInitial",
                   "lastName",
                   "addressNumber",
                   "street",
                   "city",
                   "state",
                   "zipCode")
            .targetType(Customer.class)
            .build();
}
```  

---  

## 커스텀 바인딩

`Customer` 클래스에서 `addressNumber`, `street` 필드를 합친 `address` 필드로 변경하자.

```java
public class Customer {

    private String firstName;
    private String middleInitial;
    private String lastName;
    private String address;
    private String city;
    private String state;
    private String zipCode;
    
    ...
}
```  

이때 **FieldSetMapper** 인터페이스 구현을 `addressNumber`, `street` -> `address`로 변경하자.

```java
/**
 * {@link BeanWrapperFieldSetMapper} 대신 {@link CustomerFieldMapper}를 이용한다.
 */
@Bean
@StepScope
public FlatFileItemReader<Customer> customerFlatFileItemReader(
        @Value("#{jobParameters['customerFile']}") Resource inputFile) {
    logger.info("Activated flat file reader: custom field mapper");

    return new FlatFileItemReaderBuilder<Customer>()
            // ItemStream 인터페이스는 애플리케이션 내 각 스텝의 ExecutionContext에 추가되는 특정 키의
            // 접두문자로 사용될 이름이 필요
            .name("customerItemReader")
            .delimited()
            .names("firstName",
                   "middleInitial",
                   "lastName",
                   "addressNumber",
                   "street",
                   "city",
                   "state",
                   "zipCode")
            .fieldSetMapper(new CustomerFieldMapper())
            .resource(inputFile)
            .build();
}

public static class CustomerFieldMapper implements FieldSetMapper<Customer> {

    @Override
    public Customer mapFieldSet(FieldSet fieldSet) throws BindException {
        return Customer.builder()
                       .firstName(fieldSet.readString("firstName"))
                       .middleInitial(fieldSet.readString("middleInitial"))
                       .lastName(fieldSet.readString("lastName"))
                       .address(fieldSet.readString("addressNumber") + " " + fieldSet.readString("street"))
                       .city(fieldSet.readString("city"))
                       .state(fieldSet.readString("state"))
                       .zipCode(fieldSet.readString("zipCode"))
                       .build();
    }
}
```  

---  

## 커스텀 레코드 파싱

**LineTokenizer** 구현을 통해 레코드를 파싱해보자.

```java
@Bean
@StepScope
public FlatFileItemReader<Customer> customerFlatFileItemReader(
        @Value("#{jobParameters['customerFile']}") Resource inputFile) {
    logger.info("Activated flat file reader: custom line tokenizer");

    return new FlatFileItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .lineTokenizer(new CustomerFileLineTokenizer())
            .targetType(Customer.class)
            .resource(inputFile)
            .build();
}

public static class CustomerFileLineTokenizer implements LineTokenizer {

    private final String delimiter = ",";
    private final String[] names = {
            "firstName",
            "middleInitial",
            "lastName",
            "address",
            "city",
            "state",
            "zipCode"
    };
    private final FieldSetFactory fieldSetFactory = new DefaultFieldSetFactory();

    @Override
    public FieldSet tokenize(String record) {
        final String[] fields = record.split(delimiter);
        final List<String> parsedFields = new ArrayList<>();

        for (int i = 0; i < fields.length; i++) {
            if (i == 4) {
                parsedFields.set(i - 1, parsedFields.get(i - 1) + " " + fields[i]);
                continue;
            }
            parsedFields.add(fields[i]);
        }

        return fieldSetFactory.create(parsedFields.toArray(new String[0]), names);
    }
}
```  

---  

## 여러 가지 레코드 포맷

고객 정보 이외에 거래내역인 `Transaction` 데이터도 포함하는 정보를 살펴보자.

`CUST`로 시작하는 레코드는 고객을 나타내며, `TRANS`는 거래내역을 나타낸다.

> customerMultiFormat.csv

```text
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

`PatternMatchingCompositeLineMapper`을 통해 Pattern 마다 Tokenizer, FieldSetMapper를 추가해보자.

```java
@Bean
public PatternMatchingCompositeLineMapper lineTokenizer() {
    final Map<String, LineTokenizer> lineTokenizers = new HashMap<>(2);

    lineTokenizers.put("CUST*", customerLineTokenizer());
    lineTokenizers.put("TRANS*", transactionLineTokenizer());

    final Map<String, FieldSetMapper> fieldSetMappers = new HashMap<>(2);

    final BeanWrapperFieldSetMapper<Customer> customerFieldSetMapper = new BeanWrapperFieldSetMapper<>();
    customerFieldSetMapper.setTargetType(Customer.class);

    fieldSetMappers.put("CUST*", customerFieldSetMapper);
    fieldSetMappers.put("TRANS*", new TransactionFieldSetMapper());

    final PatternMatchingCompositeLineMapper lineMappers = new PatternMatchingCompositeLineMapper();

    lineMappers.setTokenizers(lineTokenizers);
    lineMappers.setFieldSetMappers(fieldSetMappers);

    return lineMappers;
}

@Bean
public DelimitedLineTokenizer transactionLineTokenizer() {
    final DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();

    lineTokenizer.setNames("prefix", "accountNumber", "transactionDate", "amount");

    return lineTokenizer;
}

@Bean
public DelimitedLineTokenizer customerLineTokenizer() {
    final DelimitedLineTokenizer lineTokenizer = new DelimitedLineTokenizer();

    lineTokenizer.setNames("firstName",
                           "middleInitial",
                           "lastName",
                           "address",
                           "city",
                           "state",
                           "zipCode");
    lineTokenizer.setIncludedFields(1, 2, 3, 4, 5, 6, 7);

    return lineTokenizer;
}

public static class TransactionFieldSetMapper implements FieldSetMapper<Transaction> {

    @Override
    public Transaction mapFieldSet(FieldSet fieldSet) throws BindException {
        return Transaction.builder()
                          .accountNumber(fieldSet.readString("accountNumber"))
                          .amount(fieldSet.readDouble("amount"))
                          .transactionDate(fieldSet.readDate("transactionDate", "yyyy-MM-dd HH:mm:ss"))
                          .build();
    }
}
```  

그러면 각 레코드마다 `Customer`, `Transaction`으로 변환이 된다.

또한 위임을 통해 `CUST` 데이터 마다 `TRANS` 데이터를 수집하는 `ItemStreamReader<Customer>`를 구현해보자.

```java
/**
 * {@link ItemReader}를 사용할 경우 읽을 대상 리소스를 열고 닫는 것을 제어할 뿐만 아니라 읽어들인 레코드와 관련된 상태를 {@link ExecutionContext}에
 * 관리해야 하므로 {@link ItemStreamReader}를 활용한다.
 */
class CustomerFileReader implements ItemStreamReader<Customer> {

    private Object curItem = null;
    private ItemStreamReader<Object> delegate;

    public CustomerFileReader(ItemStreamReader<Object> delegate) {
        this.delegate = delegate;
    }

    @Override
    public Customer read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException {
        if (curItem == null) {
            curItem = delegate.read();
        }

        final Customer item = (Customer) curItem;
        curItem = null;

        if (item != null) {
            item.setTransactions(new ArrayList<>());

            while (peek() instanceof Transaction) {
                item.getTransactions().add((Transaction) curItem);
                curItem = null;
            }
        }

        return item;
    }

    @Override
    public void open(ExecutionContext executionContext) throws ItemStreamException {
        delegate.open(executionContext);
    }

    @Override
    public void update(ExecutionContext executionContext) throws ItemStreamException {
        delegate.update(executionContext);
    }

    @Override
    public void close() throws ItemStreamException {
        delegate.close();
    }

    private Object peek() throws Exception {
        if (curItem == null) {
            curItem = delegate.read();
        }
        return curItem;
    }
}
```  

---  

## XML 파일

아래와 같이 의존성을 추가하자.

```xml
<dependency>
  <groupId>org.springframework</groupId>
  <artifactId>spring-oxm</artifactId>
</dependency>
<dependency>
  <groupId>javax.xml.bind</groupId>
  <artifactId>jaxb-api</artifactId>
  <version>2.2.11</version>
</dependency>
<dependency>
  <groupId>com.sun.xml.bind</groupId>
  <artifactId>jaxb-core</artifactId>
  <version>2.2.11</version>
</dependency>
<dependency>
  <groupId>com.sun.xml.bind</groupId>
  <artifactId>jaxb-impl</artifactId>
  <version>2.2.11</version>
</dependency>
<dependency>
  <groupId>javax.activation</groupId>
  <artifactId>activation</artifactId>
  <version>1.1.1</version>
</dependency>
```  

Customer 클래스에 JAXB Annotation을 아래와 같이 추가하자.

```java
@XmlRootElement
public class Customer {
  ...
  
  @XmlElementWrapper(name = "transactions")
  @XmlElement(name = "transaction")
  public void setTransactions(List<Transaction> transactions) {
    this.transactions = transactions;
  }
  
  ...
}
```

아래와 같은 Marshaller와 Reader를 구성하자.

```java
@Bean
@StepScope
public StaxEventItemReader<Customer> customerFileReader(@Value("#{jobParameters['customerFile']}")
                                                                Resource inputFile) {

    return new StaxEventItemReaderBuilder<Customer>()
            .name("customerFileReader")
            .resource(inputFile)
            .addFragmentRootElements("customer")
            .unmarshaller(customerMarshaller())
            .build();
}

private Jaxb2Marshaller customerMarshaller() {
    final Jaxb2Marshaller marshaller = new Jaxb2Marshaller();

    marshaller.setClassesToBeBound(Customer.class, Transaction.class);

    return marshaller;
}
```  

---  

## JSON 파일

```java
@Bean
@StepScope
public JsonItemReader<Customer> customerFileReader(@Value("#{jobParameters['customerFile']}")
                                                           Resource inputFile) {
    // dateformat 커스터 마이징 때문에 Bean 대신 별도로 생성한다.
    final ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.setDateFormat(new SimpleDateFormat("yyyy-MM-dd hh:mm:ss"));

    final JacksonJsonObjectReader<Customer> jsonObjectReader = new JacksonJsonObjectReader<>(objectMapper,
                                                                                             Customer.class);

    return new JsonItemReaderBuilder<Customer>()
            .name("customerFileReader")
            .jsonObjectReader(jsonObjectReader)
            .resource(inputFile)
            .build();
}
```

---  

# References

- [스프링 배치 완벽 가이드 2/e](https://book.naver.com/bookdb/book_detail.nhn?bid=18990242)