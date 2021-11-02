9.ItemWriter
--

#### 스프링 데이터(Spring Data) Repository

```java
@Data
@Entity
@Table(name = "customer")
public class Customer implements Serializable {
    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;

    private String firstName;

    private String lastName;

    private String city;

    private String zip;
}
```

```java
public interface CustomerRepository extends CrudRepository<Customer, Long> {
}
```

```java
@EnableJpaRepositories(basePackages = "me.ham.ch9.lab5")
public class RepositoryItemWriterLab5Configuration {
    // ...
    
    @Bean
	public RepositoryItemWriter<Customer> repositoryItemWriter(CustomerRepository customerRepository) {
		return new RepositoryItemWriterBuilder<Customer>()
                .repository(customerRepository)
                .methodName("save")
                .build();
	}
}
```

| id | first\_name | last\_name | city | zip |
| :--- | :--- | :--- | :--- | :--- |
| 22 | hosik | kim | seoul | 12311 |
| 23 | hosik | ham | seoul | 22222 |
| 24 | hosik | lee | seoul | 12312 |
| 25 | hosik | yoo | seoul | 44444 |
| 26 | hosik | tee | seoul | 21231 |
| 27 | hosik | sss | busan | 22222 |

---

#### ItemWriterAdapter

ItemWriterAdapter가 호출하는 메서드는 처리 중인 아이템 타입만 받아 들인다.   
예를 들어 Customer 객체를 처리한다면, 호출되는 메서드는 Customer 타입의 argument 하나만 가져야 한다.

* targetObject : 호출할 메서드를 가지고 있는 스프링 빈
* targetMethod : 각 아이템을 처리할 때 호출할 메서드

```java
@Slf4j
@Service
public class CustomerService {
    public void logCustomer(Customer customer){
        log.info("[CustomerService logCustomer]:{}", customer);
    }
}
```

```java
    @Bean
    public ItemWriterAdapter<Customer> itemWriterAdapter(CustomerService customerService) {
        ItemWriterAdapter<Customer> customerItemWriterAdapter = new ItemWriterAdapter<>();
    
        customerItemWriterAdapter.setTargetObject(customerService);
        customerItemWriterAdapter.setTargetMethod("logCustomer");
        return customerItemWriterAdapter;
    }
```

---

#### PropertyExtractingDelegatingItemWriter

요청한 아이템 속성만 전달할 때 사용하는 ItemWriter

```java
    @Bean
	public PropertyExtractingDelegatingItemWriter<Customer> propertyExtractingDelegatingItemWriter(CustomerServiceLab7 customerServiceLab7) {
        PropertyExtractingDelegatingItemWriter<Customer> propertyExtractingDelegatingItemWriter = new PropertyExtractingDelegatingItemWriter<>();

        propertyExtractingDelegatingItemWriter.setTargetObject(customerServiceLab7);
        propertyExtractingDelegatingItemWriter.setTargetMethod("logCustomer");
        propertyExtractingDelegatingItemWriter.setFieldsUsedAsTargetMethodArguments(new String[] {
                "firstName",
                "lastName",
                "city",
                "zip"
        });
		return propertyExtractingDelegatingItemWriter;
	}
```

```java
@Slf4j
@Service
public class CustomerServiceLab7 {
    public void logCustomer(Customer customer){
        log.info("[CustomerService logCustomer]:{}", customer);
    }

    public void logCustomer(String firstName, String lastName, String city, String zip){
        log.info("firstName: {}, lastName: {}, city: {}, zip: {}", firstName, lastName, city, zip);
    }
}
```

---

#### JmsItemWriter (Java Message Service)

발행 - 구독 모델을 사용하여 둘 이상의 엔드포인트 간에 통신하는 메시지 지향적인 방식이다.

* MessageConverter   
-> 지점 간 전송을 위해 메시지를 JSON으로 변환한다.
* JmsTemplate   
-> Java Message Service 접근을 단순화시켜주는 헬퍼 클래스 역할

```xml
    <!-- pom.xml -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-activemq</artifactId>
    </dependency>
    <dependency>
        <groupId>org.apache.activemq</groupId>
        <artifactId>activemq-broker</artifactId>
    </dependency>
```

```java
    @Bean
    public Step jmsItemWriterStepLab8() {
        return this.stepBuilderFactory.get("jmsItemWriterStepLab8")
                .<Customer, Customer>chunk(CHUNK_SIZE)
                .reader(itemReaderLab8())
                .processor(jmsItemProcessor())
                .writer(jmsItemWriter())
                .build();
    }

    @Bean
    public JmsItemWriter<Customer> jmsItemWriter() {
        return new JmsItemWriterBuilder<Customer>()
                .jmsTemplate(jmsTemplate())
                .build();
    }

    @Bean
    public MessageConverter jacksonJmsMessageConverter() {
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);
        converter.setTypeIdPropertyName("_type");
        return converter;
    }

    @Bean
    public JmsTemplate jmsTemplate() {
        ActiveMQConnectionFactory connectionFactory = new ActiveMQConnectionFactory("vm://localhost?broker.persistent=false");
    
        JmsTemplate jmsTemplate = new JmsTemplate(connectionFactory);
        jmsTemplate.setDefaultDestinationName("customers");
        jmsTemplate.setReceiveTimeout(5000L);
    
        return jmsTemplate;
    }
```

```text
2021-11-02 19:46:07.742  INFO 29164 --- [           main] ...JmsItemWriterLab8Configuration : jmsItemProcessor::Customer(firstName=hosik, lastName=kim, city=seoul, zip=12311)
2021-11-02 19:46:07.748  INFO 29164 --- [           main] ...JmsItemWriterLab8Configuration : jmsItemProcessor::Customer(firstName=hosik, lastName=ham, city=seoul, zip=22222)
2021-11-02 19:46:07.748  INFO 29164 --- [           main] ...JmsItemWriterLab8Configuration : jmsItemProcessor::Customer(firstName=hosik, lastName=lee, city=seoul, zip=12312)
2021-11-02 19:46:07.748  INFO 29164 --- [           main] ...JmsItemWriterLab8Configuration : jmsItemProcessor::Customer(firstName=hosik, lastName=yoo, city=seoul, zip=44444)
2021-11-02 19:46:07.748  INFO 29164 --- [           main] ...JmsItemWriterLab8Configuration : jmsItemProcessor::Customer(firstName=hosik, lastName=tee, city=seoul, zip=21231)
2021-11-02 19:46:07.748 DEBUG 29164 --- [           main] o.s.batch.item.jms.JmsItemWriter         : Writing to JMS with 5 items.
```

---

#### MultiResourceItemWriter

지정된 레코드만큼 처리한 후, 새로운 리소스를 생성하는 기능을 제공한다.   
처리한 레코드 수에 따라 출력 리소스를 동적으로 만든다.

ex) 30개의 row를 읽어, 10개씩 출력 -> 3개의 파일이 동적으로 만들어짐

```java
    @Bean
	public MultiResourceItemWriter<Customer> multiCustomerFileWriter(CustomerOutputFileSuffixCreator suffixCreator)  {
		return new MultiResourceItemWriterBuilder<Customer>()
				.name("multiCustomerFileWriter")
				.delegate(flatFileItemWriterLab9())
                // 5개씩 동적으로 파일을 만든다.
				.itemCountLimitPerResource(5)
				.resource(new ClassPathResource(""))
				.resourceSuffixCreator(suffixCreator)
				.build();
	}

    @Bean
    @StepScope
    public FlatFileItemWriter<Customer> flatFileItemWriterLab9() {
        return new FlatFileItemWriterBuilder<Customer>()
                .name("customerItemWriter")
                .resource(new ClassPathResource("ch9/lab9/outputFile.txt"))
                .delimited()
                .delimiter(",")
                .names(new String[] {
                        "firstName"
                        ,"lastName"
                        ,"city"
                        ,"zip"})
                .build();
    }
```

```java
    @Component
    public class CustomerOutputFileSuffixCreator implements ResourceSuffixCreator {
    
    	@Override
    	public String getSuffix(int fileName) {
    		return fileName + ".txt";
    	}
    }
```

inputFile.txt
```csv
hosik,kim,seoul,12311
hosik,ham,seoul,22222
hosik,lee,seoul,12312
hosik,yoo,seoul,44444
hosik,tee,seoul,21231
hosik,sss,busan,22222
```

classes1.txt
```txt
hosik,kim,seoul,12311
hosik,ham,seoul,22222
```
classes2.txt
```txt
hosik,lee,seoul,12312
hosik,yoo,seoul,44444
```
classes3.txt
```txt
hosik,tee,seoul,21231
hosik,sss,busan,22222
```

---

#### 플랫 파일 내의 헤더와 푸터

```java
@Aspect
@Component
@ConditionalOnProperty(value = "job.name", havingValue = "headerFooterItemWriterJobLab10")
public class CustomerRecordCountFooterCallback implements FlatFileFooterCallback {

	private int itemsWrittenInCurrentFile = 0;

	@Override
	public void writeFooter(Writer writer) throws IOException {
		writer.write("This file contains " +
				itemsWrittenInCurrentFile + " items");
	}

	@Before("execution(* org.springframework.batch.item.file.FlatFileItemWriter.write(..))")
	public void beforeWrite(JoinPoint joinPoint) {
		List<Customer> items = (List<Customer>) joinPoint.getArgs()[0];

		this.itemsWrittenInCurrentFile += items.size();
	}

	@Before("execution(* org.springframework.batch.item.file.FlatFileItemWriter.open(..))")
	public void resetCounter() {
		this.itemsWrittenInCurrentFile = 0;
	}
}
```
```java
        @Bean
        public FlatFileItemWriter<Customer> flatFileItemWriterLab10(CustomerRecordCountFooterCallback callback) {
            return new FlatFileItemWriterBuilder<Customer>()
                    .name("customerItemWriter")
                    .resource(new ClassPathResource("ch9/lab9/outputFile.txt"))
                    .delimited()
                    .delimiter(",")
                    .names(new String[] {
                            "firstName"
                            ,"lastName"
                            ,"city"
                            ,"zip"})
                    .footerCallback(callback)
                    .build();
        }
```

outputFile.txt
```text
hosik,kim,seoul,12311
hosik,ham,seoul,22222
hosik,lee,seoul,12312
hosik,yoo,seoul,44444
hosik,tee,seoul,21231
hosik,sss,busan,22222
This file contains 0 items
```