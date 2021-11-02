# 9장 ItemWriter -2

## 그밖의 출력 방식을 위한 ItemWriter

### ItemWriterAdaptor

- 기존 서비스를 ItemWriter로 사용하고자 할 때 사용.
- Write 대상이 기존 서비스이고 메소드 인자 타입이 아이템 타입을 그대로 사용할 수 있는 경우.
- 따라서, 처리중인 아이템 타입 하나의 인수만 받을 수 있다.

```java
@Service
public class CustomerService {

	public void logCustomer(Customer customer) {
		System.out.println(customer);
	}
}
```

```java
@Bean
public ItemWriterAdapter<Customer> itemWriter(CustomerService customerService) {
    ItemWriterAdapter<Customer> customerItemWriterAdapter = new ItemWriterAdapter<>();

    customerItemWriterAdapter.setTargetObject(customerService);
    customerItemWriterAdapter.setTargetMethod("logCustomer");

    return customerItemWriterAdapter;
}
```

`실습`

### PropertyExtractingDelegatingItemWriter

- ItemWriterAdaptor 에서는 기존 서비스가 도메인 객체를 받기 때문에 타입을 그대로 사용할 수 있지만 타입이 동일하지 않다면?
- Write 대상이 기존 서비스이고 메소드 인자 타입이 아이템 타입과 다를 경우 사용.
- PropertyExtractingDelegatingItemWriter는 이러한 경우에 아이템에서 값을 추출해서 넘겨줄 수 있다.

```java
@Service
public class CustomerService {

	public void logCustomerAddress(String address,
			String city,
			String state,
			String zip) {
		System.out.println(
				String.format("I just saved the address:\n%s\n%s, %s\n%s",
						address,
						city,
						state,
						zip));
	}
}
```

```java
@Bean
public PropertyExtractingDelegatingItemWriter<Customer> itemWriter(CustomerService customerService) {
    PropertyExtractingDelegatingItemWriter<Customer> itemWriter =
            new PropertyExtractingDelegatingItemWriter<>();

    itemWriter.setTargetObject(customerService);
    itemWriter.setTargetMethod("logCustomerAddress");
    itemWriter.setFieldsUsedAsTargetMethodArguments(
            new String[] {"address", "city", "state", "zip"});

    return itemWriter;
}
```
- setFieldsUsedAsTargetMethodArguments에 정의한 타입이 순서대로 도메인 서비스 메서드로 넘어간다.
- PropertyExtractingDelegatingItemWriter는 AbstractMethodInvokingDelegator 상속. 이에 따라 arguments를 set 할 수 있지만 동적으로 추출되기 때문에 사용하지 않음.

`실습`

###  JmsItemWriter

- JMS(Java Messaging Service)로 Write 할 때 사용.

- 실습 구조: file(CSV) -> step1 -> JMS Queue -> Step2 -> file(XML)

- 주의사항: 스프링부트가 제공하는 ConnectionFactory는 JmsTemplate와 잘 동작하지 않기 때문에 CachingCConnectionFactory를 대신 사용

`실습`

### SimpleMailMessageItemWriter

- 이메일로 Write 할 때 사용

- 실습 구조: file(CSV) -> step1 -> DB -> Step2 -> email

`실습`


## 여러 자원을 사용하는 ItemWriter

여러 리소스에 write 할 때 발생하는 문제
1. 처리할 아이템이 너무 많아서 부하가 심할 경우
2. 쓰기 대상의 종류가 여러개라서 그에 따른 커스텀 구현체를 만들어야 할 경우

### 아이템이 많아서 부하가 심할 경우

#### MultiResourceItemWriter

- 지정된 개수만큼 처리하고 새로운 리소스 생성 가능
- 실제 Write는 실제 Writer에게 위임하고 itemCountLimitPerResource(n)을 통해 리소스 분할하여 쓰기 가능

![](http://images-20200215.ebookreading.net/6/4/4/9781484237243/9781484237243__the-definitive-guide__9781484237243__images__215885_2_En_9_Chapter__215885_2_En_9_Fig10_HTML.png) 

```java
@Bean
public MultiResourceItemWriter<Customer> multiCustomerFileWriter(CustomerOutputFileSuffixCreator suffixCreator) throws Exception {

    return new MultiResourceItemWriterBuilder<Customer>()
            .name("multiCustomerFileWriter")
            .delegate(delegateItemWriter(null))
            .itemCountLimitPerResource(25)
            .resource(new FileSystemResource("Chapter09/target/customer"))
            .build();
}
```

- 생성 되는 파일의 suffix 지정 가능
```java
@Component
public class CustomerOutputFileSuffixCreator implements ResourceSuffixCreator {

	@Override
	public String getSuffix(int arg0) {
		return arg0 + ".xml";
	}
}
```

```java
@Bean
public MultiResourceItemWriter<Customer> multiCustomerFileWriter(CustomerOutputFileSuffixCreator suffixCreator) throws Exception {

    return new MultiResourceItemWriterBuilder<Customer>()
            .name("multiCustomerFileWriter")
            .delegate(delegateItemWriter(null))
            .itemCountLimitPerResource(25)
            .resource(new FileSystemResource("Chapter09/target/customer"))
            .resourceSuffixCreator(suffixCreator)
            .build();
}
```

`실습`

#### 헤더와 푸터 XML 프래그먼트

생성하는 여러 파일에 동시에 헤더와 푸터를 넣는 작업을 하고 싶다면?
==> 스프링 배치 콜백 함수 사용

XML일 경우 StaxWriterCallback 사용.

```java
@Component
public class CustomerXmlHeaderCallback implements StaxWriterCallback {

	@Override
	public void write(XMLEventWriter writer) throws IOException {
		XMLEventFactory factory = XMLEventFactory.newInstance();

		try {
			writer.add(factory.createStartElement("", "", "identification"));
			writer.add(factory.createStartElement("", "", "author"));
			writer.add(factory.createAttribute("name", "Michael Minella"));
			writer.add(factory.createEndElement("", "", "author"));
			writer.add(factory.createEndElement("", "", "identification"));
		} catch (XMLStreamException xmlse) {
			System.err.println("An error occured: " + xmlse.getMessage());
			xmlse.printStackTrace(System.err);
		}
	}
}
```
```java
@Bean
@StepScope
public StaxEventItemWriter<Customer> delegateItemWriter(CustomerXmlHeaderCallback headerCallback) throws Exception {

    Map<String, Class> aliases = new HashMap<>();
    aliases.put("customer", Customer.class);

    XStreamMarshaller marshaller = new XStreamMarshaller();

    marshaller.setAliases(aliases);

    marshaller.afterPropertiesSet();

    return new StaxEventItemWriterBuilder<Customer>()
    .name("customerItemWriter")
    .marshaller(marshaller)
    .rootTagName("customers")
    .headerCallback(headerCallback)
    .build();
}
```

플랫 파일일 경우 FlatFileFooterCallback 사용
단, ItemWriterListener 사용 시 MultiResourceItemWriter가 FlatFileItemWriter을 감싸고 있어서 itemsWrittenInCurrentFile의 초기화가 제때 불려지지 않기 때문에 이런겅우 AOP를 걸어준다.

```java
@Bean
public MultiResourceItemWriter<Customer> multiFlatFileItemWriter() throws Exception {

    return new MultiResourceItemWriterBuilder<Customer>()
            .name("multiFlatFileItemWriter")
            .delegate(delegateCustomerItemWriter(null))
            .itemCountLimitPerResource(25)
            .resource(new FileSystemResource("Chapter09/target/customer"))
            .build();
}
```

```java
@Component
@Aspect
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
@StepScope
public FlatFileItemWriter<Customer> delegateCustomerItemWriter(CustomerRecordCountFooterCallback footerCallback) throws Exception {
    BeanWrapperFieldExtractor<Customer> fieldExtractor = new BeanWrapperFieldExtractor<>();
    fieldExtractor.setNames(new String[] {"firstName", "lastName", "address", "city", "state", "zip"});
    fieldExtractor.afterPropertiesSet();

    FormatterLineAggregator<Customer> lineAggregator = new FormatterLineAggregator<>();

    lineAggregator.setFormat("%s %s lives at %s %s in %s, %s.");
    lineAggregator.setFieldExtractor(fieldExtractor);

    FlatFileItemWriter<Customer> itemWriter = new FlatFileItemWriter<>();

    itemWriter.setName("delegateCustomerItemWriter");
    itemWriter.setLineAggregator(lineAggregator);
    itemWriter.setAppendAllowed(true);
    itemWriter.setFooterCallback(footerCallback);

    return itemWriter;
}
```

`실습`

### CompositeItemWriter

Write를 여러 엔드포인트에 하고 싶을때 사용.

```java
@Bean
public CompositeItemWriter<Customer> compositeItemWriter() throws Exception {
    return new CompositeItemWriterBuilder<Customer>()
            .delegates(Arrays.asList(xmlDelegateItemWriter(null),
                    jdbcDelgateItemWriter(null)))
            .build();
}
```

`실습`

### ClassifierCompositeItemWriter

아이템의 유형에 따라 다른 엔드포인트에 쓰기를 하고 싶을 때 사용.

```java
public class CustomerClassifier implements
		Classifier<Customer, ItemWriter<? super Customer>> {

	private ItemWriter<Customer> fileItemWriter;
	private ItemWriter<Customer> jdbcItemWriter;

	public CustomerClassifier(StaxEventItemWriter<Customer> fileItemWriter, JdbcBatchItemWriter<Customer> jdbcItemWriter) {
		this.fileItemWriter = fileItemWriter;
		this.jdbcItemWriter = jdbcItemWriter;
	}

	@Override
	public ItemWriter<Customer> classify(Customer customer) {
		if(customer.getState().matches("^[A-M].*")) {
			return fileItemWriter;
		} else {
			return jdbcItemWriter;
		}
	}
}
```

```java
@Bean
	public ClassifierCompositeItemWriter<Customer> classifierCompositeItemWriter() throws Exception {
		Classifier<Customer, ItemWriter<? super Customer>> classifier = new CustomerClassifier(xmlDelegate(null), jdbcDelgate(null));

		return new ClassifierCompositeItemWriterBuilder<Customer>()
				.classifier(classifier)
				.build();
	}
```

CompositeItemWriter와 달리 ClassifierCompositeItemWriter는 ItemStream을 인터페이스를 구현하지 않는다. 따라서 상태를 가진 대상을 처리하기 위해서는 Stream을 처리할 writer를 등록해야 한다.

```java
@Bean
public Step classifierCompositeWriterStep() throws Exception {
    return this.stepBuilderFactory.get("classifierCompositeWriterStep")
            .<Customer, Customer>chunk(10)
            .reader(classifierCompositeWriterItemReader(null))
            .writer(classifierCompositeItemWriter())
            .stream(xmlDelegate(null))
            .build();
}
```
`실습`