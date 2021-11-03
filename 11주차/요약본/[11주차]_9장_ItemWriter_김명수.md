# 목차

- [그밖의 출력 방식을 위한 ItemWriter](#그밖의-출력-방식을-위한-ItemWriter)
  - [ItemWriterAdapter](#ItemWriterAdapter)
  - [PropertyExtractingDelegatingItemWriter](#PropertyExtractingDelegatingItemWriter)
  - [JmsItemWriter](#JmsItemWriter)
  - [SimpleMailMessageItemWriter](#SimpleMailMessageItemWriter)
- [여러 자원을 사용하는 ItemWriter](#여러-자원을-사용하는-ItemWriter)
  - [CompositeItemWriter](#CompositeItemWriter)
  - [ClassifierCompositeItemWriter](#ClassifierCompositeItemWriter)

---  

# 그밖의 출력 방식을 위한 ItemWriter

## ItemWriterAdapter

이미 구현과 테스트가 완료된 기존 컴포넌트를 활용할 수 있다.

**(1) 기존 서비스**

```java
@Slf4j
@Service
public class CustomerService {

    public void logCustomer(Customer customer) {
        logger.info("I just saved: {}", customer);
    }
}
```

**(2) ItemWriter 구성**

```java
@Bean
public ItemWriterAdapter<Customer> itemWriter(CustomerService service) {
    ItemWriterAdapter<Customer> adapter = new ItemWriterAdapter<>();

    adapter.setTargetObject(service);
    adapter.setTargetMethod("logCustomer");

    return adapter;
}
```

**(3) 실행 결과**

```log
I just saved: Customer{firstName=Richard, middleInitial=N, lastName=Darrow, address=5570 Isabella Ave, city=St. Louis, state=IL, zip=58540}
I just saved: Customer{firstName=Warren, middleInitial=L, lastName=Darrow, address=4686 Mt. Lee Drive, city=St. Louis, state=NY, zip=94935}
I just saved: Customer{firstName=Barack, middleInitial=G, lastName=Donnelly, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635}
I just saved: Customer{firstName=Ann, middleInitial=Z, lastName=Benes, address=2447 S. Greenwood Ave, city=Las Vegas, state=NY, zip=55366}
I just saved: Customer{firstName=Erica, middleInitial=Z, lastName=Gates, address=3141 Farnam Street, city=Omaha, state=CA, zip=57640}
I just saved: Customer{firstName=Warren, middleInitial=M, lastName=Williams, address=6670 S. Greenwood Ave, city=Hollywood, state=FL, zip=37288}
I just saved: Customer{firstName=Harry, middleInitial=T, lastName=Darrow, address=3273 Isabella Ave, city=Houston, state=FL, zip=97261}
I just saved: Customer{firstName=Steve, middleInitial=O, lastName=Darrow, address=8407 Infinite Loop Drive, city=Las Vegas, state=WA, zip=90520}
```

---

## PropertyExtractingDelegatingItemWriter

`ItemReader<T>`나 `ItemProcessor<I, O>` 객체를 그대로 쓰지 않고 특정 필드만 사용할 수 있다.

**(1) 기존 서비스**

```java
@Service
public class CustomerService {

    public void logCustomerAddress(String address, String city, String state, String zip) {
        System.out.printf("I just saved the address. address: %s, city: %s, state: %s, zip: %s\n",
                          address, city, state, zip);
    }
}
```

**(2) ItemWriter 구성하기**

```java
@Bean
public PropertyExtractingDelegatingItemWriter<Customer> itemWriter(CustomerService service) {
    PropertyExtractingDelegatingItemWriter<Customer> writer = new PropertyExtractingDelegatingItemWriter<>();

    writer.setTargetObject(service);
    writer.setTargetMethod("logCustomerAddress");
    writer.setFieldsUsedAsTargetMethodArguments(new String[] {
            "address", "city", "state", "zip"
    });

    return writer;
}
```

**(3) 실행 결과**

```log
I just saved the address. address: 5570 Isabella Ave, city: St. Louis, state: IL, zip: 58540
I just saved the address. address: 4686 Mt. Lee Drive, city: St. Louis, state: NY, zip: 94935
I just saved the address. address: 7844 S. Greenwood Ave, city: Houston, state: CA, zip: 38635
I just saved the address. address: 2447 S. Greenwood Ave, city: Las Vegas, state: NY, zip: 55366
I just saved the address. address: 3141 Farnam Street, city: Omaha, state: CA, zip: 57640
I just saved the address. address: 6670 S. Greenwood Ave, city: Hollywood, state: FL, zip: 37288
I just saved the address. address: 3273 Isabella Ave, city: Houston, state: FL, zip: 97261
I just saved the address. address: 8407 Infinite Loop Drive, city: Las Vegas, state: WA, zip: 90520
```

---  

## JmsItemWriter

JMS(Java Messaging Service)는 둘 이상의 엔드포인트 간에 통신하는 메시지 지향(message-oriented)적인 방식이다.

아래와 같은 Job을 구성하자.

**[Resource (customer.csv)] -> [Step ItemReader -> JmsItemWriter] -> [Customer Queue] -> [Step JmsItemReader -> ItemWriter]**

**(1) JMS 설정 하기**

```java
@Configuration
public class JmsConfiguration {

    @Bean
    public MessageConverter jacksonJmsMessageConverter() {
        System.out.println("## converter");
        MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
        converter.setTargetType(MessageType.TEXT);
        converter.setTypeIdPropertyName("_type");
        return converter;
    }

    @Bean
    public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
        CachingConnectionFactory factory = new CachingConnectionFactory(connectionFactory);
        factory.afterPropertiesSet();

        JmsTemplate jmsTemplate = new JmsTemplate(factory);
        jmsTemplate.setDefaultDestinationName("customers");
        jmsTemplate.setReceiveTimeout(5000L);

        return jmsTemplate;
    }
}
```

**(2) Job 구성 하기**

```java
@Bean
@StepScope
public ItemWriter<Customer> consoleItemWriter() {
    return items -> {
        System.out.println("## ItemWriter " + items.size());
        for (Customer item : items) {
            System.out.println("> Item: " + item);
        }
    };
}

@Bean
public JmsItemReader<Customer> jmsItemReader(JmsTemplate jmsTemplate) {
    return new JmsItemReaderBuilder<Customer>()
            .jmsTemplate(jmsTemplate)
            .itemType(Customer.class)
            .build();
}

@Bean
public JmsItemWriter<Customer> jmsItemWriter(JmsTemplate jmsTemplate) {
    return new JmsItemWriterBuilder<Customer>()
            .jmsTemplate(jmsTemplate)
            .build();
}

@Bean
public Step formatInputStep() {
    return stepBuilderFactory.get("formatInputStep")
                             .<Customer, Customer>chunk(3)
                             .reader(customerItemReader(null))
                             .writer(jmsItemWriter(null))
                             .build();
}

@Bean
public Step formatOutputStep() {
    return stepBuilderFactory.get("formatOutputStep")
                             .<Customer, Customer>chunk(3)
                             .reader(jmsItemReader(null))
                             .writer(consoleItemWriter())
                             .build();
}

@Bean
public Job job() throws Exception {
    return jobBuilderFactory.get("job")
                            .incrementer(new RunIdIncrementer())
                            .start(formatInputStep())
                            .next(formatOutputStep())
                            .build();
}
```

**(3) 실행 결과**

```java
## ItemWriter 3
> Item: Customer{firstName=Richard, middleInitial=N, lastName=Darrow, address=5570 Isabella Ave, city=St. Louis, state=IL, zip=58540}
> Item: Customer{firstName=Warren, middleInitial=L, lastName=Darrow, address=4686 Mt. Lee Drive, city=St. Louis, state=NY, zip=94935}
> Item: Customer{firstName=Barack, middleInitial=G, lastName=Donnelly, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635}
## ItemWriter 3
> Item: Customer{firstName=Ann, middleInitial=Z, lastName=Benes, address=2447 S. Greenwood Ave, city=Las Vegas, state=NY, zip=55366}
> Item: Customer{firstName=Erica, middleInitial=Z, lastName=Gates, address=3141 Farnam Street, city=Omaha, state=CA, zip=57640}
> Item: Customer{firstName=Warren, middleInitial=M, lastName=Williams, address=6670 S. Greenwood Ave, city=Hollywood, state=FL, zip=37288}
## ItemWriter 2
> Item: Customer{firstName=Harry, middleInitial=T, lastName=Darrow, address=3273 Isabella Ave, city=Houston, state=FL, zip=97261}
> Item: Customer{firstName=Steve, middleInitial=O, lastName=Darrow, address=8407 Infinite Loop Drive, city=Las Vegas, state=WA, zip=90520}
```

---  

# SimpleMailMessageItemWriter

아래와 같이 Mail관련 의존성 추가 후 ItemWriter를 작성할 수 있다.

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-mail</artifactId>
</dependency>
```

```java
@Bean
public SimpleMailMessageItemWriter emailItemWriter(MailSender mailSender) {
    return new SimpleMailMessageItemWriterBuilder()
            .mailSender(mailSender)
            .build();
}

@Bean
public Step emailStep() {
    return stepBuilderFactory.get("emailStep")
                             .<Customer, SimpleMailMessage>chunk(3)
                             .reader(null)
                             .processor((ItemProcessor<Customer, SimpleMailMessage>) customer -> {
                                 SimpleMailMessage mail = new SimpleMailMessage();
                                 mail.setFrom("prospringbatch@gmail.com");
                                 mail.setTo(customer.getEmail());
                                 mail.setSubject("Welcome!");
                                 mail.setText(String.format("Welcome %s %s", customer.getFirstName(),
                                                            customer.getLastName()));
                                 return mail;
                             })
                             .writer(emailItemWriter(null))
                             .build();
}
```

---  

# 여러 자원을 사용하는 ItemWriter

## MultiResourceItemWriter

---  

## CompositeItemWriter

`CompositeItemWriter`를 통해 다중 ItemWriter를 이용할 수 있다. 해당 Writer는 청크를 기준으로 생성 후 각각의 `ItemWriter::write()`를 호출한다.

**(1) ItemWriter 구성하기**

```java
@Bean
public ItemWriter<Customer> itemWriter1() {
    return customers -> {
        System.out.printf("## Writer-1 items: %d\n", customers.size());
        for (Customer customer : customers) {
            System.out.println("> " + customer);
        }
    };
}

@Bean
public ItemWriter<Customer> itemWriter2() {
    return customers -> {
        System.out.printf("## Writer-2 items: %d\n", customers.size());
        for (Customer customer : customers) {
            System.out.println("> " + customer);
        }
    };
}

@Bean
public CompositeItemWriter<Customer> compositeItemWriter() {
    return new CompositeItemWriterBuilder<Customer>()
            .delegates(
                    Arrays.asList(itemWriter1(), itemWriter2())
            ).build();
}
```

**(2) 실행 결과**

```log
## Writer-1 items: 3
> Customer{firstName=Richard, middleInitial=N, lastName=Darrow, address=5570 Isabella Ave, city=St. Louis, state=IL, zip=58540, email=null}
> Customer{firstName=Warren, middleInitial=L, lastName=Darrow, address=4686 Mt. Lee Drive, city=St. Louis, state=NY, zip=94935, email=null}
> Customer{firstName=Barack, middleInitial=G, lastName=Donnelly, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635, email=null}
## Writer-2 items: 3
> Customer{firstName=Richard, middleInitial=N, lastName=Darrow, address=5570 Isabella Ave, city=St. Louis, state=IL, zip=58540, email=null}
> Customer{firstName=Warren, middleInitial=L, lastName=Darrow, address=4686 Mt. Lee Drive, city=St. Louis, state=NY, zip=94935, email=null}
> Customer{firstName=Barack, middleInitial=G, lastName=Donnelly, address=7844 S. Greenwood Ave, city=Houston, state=CA, zip=38635, email=null}
...
```

---  

## ClassifierCompositeItemWriter

`ClassifierCompositeItemWriter`를 이용해서 Item에 따른 Writer를 선택할 수 있다.

```java
public class CustomerClassifier implements Classifier<Customer, ItemWriter<? extends Customer>> {

    private ItemWriter<Customer> fileItemWriter;
    private ItemWriter<Customer> jdbcItemWriter;

    @Override
    public ItemWriter<? extends Customer> classify(Customer customer) {
        if (customer.getState().matches("^[A-M].*")) {
            return fileItemWriter;
        }
        return jdbcItemWriter;
    }
}
```

```java
@Bean
public ClassifierCompositeItemWriter<Customer> classifierCompositeItemWriter() {
    Classifier<Customer, ItemWriter<? super Customer>> classifier = new CustomerClassifier();

    return new ClassifierCompositeItemWriterBuilder<Customer>()
            .classifier(classifier)
            .build();
}
```