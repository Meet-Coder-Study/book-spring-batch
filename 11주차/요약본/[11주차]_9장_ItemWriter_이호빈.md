# 9-2장 ItemWriter

### ItemWriterAdapter

- 기존 스프링 서비스를 ItemWriter로 사용할 때 쓴다.
    - ItemWriterAdapter는 기존 서비스를 살짝 감싼 Wrapper 역할을 한다

```java
@Bean
public ItemWriterAdapter<Customer> itemWriter(CustomerService customerService) {
	ItemWriterAdapter<Customer> customerItemrWriterAdapter = new ItemWriterAdapter<>();
	customerItemrWriterAdapter.setTargetObject(customerService); // 서비스 클래스
	customerItemrWriterAdapter.setTargetMethod("logCustomer"); // 메서드 명

	return customerItemrWriterAdapter;
}

// ...
// .reader(customerFileReader(null))
// .writer(null) // 프록시로 인해 시작시에는 들어간다.
// .build();
```

### PropertyExtractingDelegatingItemWriter

- ItemWriterAdapter를 사용할 때 processor 혹은 reader에서 넘어온 item을 그대로 사용하게 되는데 item의 모든 필드를 넘겨받고 싶지 않을 수 있다.
- PropertyExtractingDelegatingItemWriter는 원하는 필드만 파라미터로 받고 싶을 때 사용할 수 있다.

```java
@Bean
public PropertyExtractingDelegatingItemWriter<Customer> itemWriter(CustomerService customerService) {
	PropertyExtractingDelegatingItemWriter<Customer> itemWriter = new PropertyExtractingDelegatingItemWriter<>();
	itemWriter.setTargetObject(customerService); // 서비스 클래스
	itemWriter.setTargetMethod("logCustomer"); // 메서드 명
	itemWriter.setFieldsUsedAsTargetMethodArguments(new String[] {"address", "city", "state", "zip"});

	return itemWriter;
}

// ...
// .reader(customerFileReader(null))
// .writer(null) // 프록시로 인해 시작시에는 들어간다.
// .build();
```

### JmsItemWriter

- Java Messaging Service는 엔드포인트 간에 통신하는 메시지 지향 방식이다.
- pub-sub 모델을 사용해 다른 기술과 통신할 수 있다
- Jms 브로커가 필요한데 예제에서는 아파치 액티브 MQ를 사용했다.

```java
// 스프링이 자체적으로 기능을 제공하는 게 있지만 잘 안된다고 한다?

@Bean
public MessageConverter jacksonJmsMessageConverter() {
	MappingJackson2MessageConverter converter = new MappingJackson2MessageConverter();
	converter.setTargetType(MessageType.TEXT);
	converter.setTypeIdPropertyName("_type");
	return converter;
}

@Bean
public JmsTemplate jmsTemplate(ConnectionFactory connectionFactory) {
	CachingConnectionFactory cachingConnectionFactory =  new CachingConnectionFactory(connectionFactory);
	cachingConnectionFactory.afterPropertiesSet();

	JmsTemplate jmsTemplate = new JmsTemplate(cachingConnectionFactory);
	jmsTemplate.setDefaultDestinationName("customers");
	jmsTemplate.setReceiveTimeout(5000L);

	return jmsTemplate;
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
```

### SimpleMailMessageItemWriter

- 이메일을 보낼 수 있는 ItemWriter
- spring boot starter mail을 사용하며 구글 SMTP 서버를 사용한다
- 각종 설정을 해주고 processor 단계에서 보낼 메일을 작성한다.

```java
@Bean
public SimpleMailMessageItemWriter emailItemWriter(MailSender mailSender) {
	return new SimpleMailMessageItemWriterBuilder()
		.mailSender(mailSender)
		.build();
}
```

### MultiResourceItemWriter

- 아이템 개수마다 새 파일을 생성해야될 때 쓰인다.
    - 예를 들어, 총 아이템 개수가 100개라면 25개마다 새 파일을 만들도록 설정할 수 있다.
- MultiResourceItemWriter의 write 메서드가 호출되면 해당 Writer는 현재 리소스가 이미 생성된 상태로 열려있는지 확인한다. (생성되지 않았다면 새 파일을 생성한다.)
- 그리고 itemWriter에 위임(delegate)한다
- 아이템의 쓰기 작업이 이뤄지면 파일에 기록된 아이템 개수가 새 리소스 생성을 위해 구성된 임계값에 도달했는지 확인한다.
    - 새 리소스를 생성하기 전에 청크가 끝날 때까지 기다린다. (청크가 끝나고 나서 새 리소스를 생성한다)
    - 아이템 개수가 15개 일때 새 파일을 생성하도록 해도, 청크 단위가 20개라면 20개까지 처리하고 생성한다.

```java
@Bean
public MultiResourceItemWriter<Customer> multiCustomerFileWriter(CustomerOutputFileSuffixCreator suffixCreator) throws Exception {
	return new MultiResourceItemWriterBuilder<Customer>()
		.name("multiCustomerFileWriter")
		.delegate(delegateItemWriter()) // 위임할 ItemWriter
		.itemCountLimitPerResource(25) // 리소스(파일) 당 아이템 개수
		.resource(new FileSystemResource("Chapter09/target/customer")) // 리소스 경로
		.resourceSuffixCreator(suffixCreator()) // 확장자
		.build();
}

@Component
public class CustomerOutputFileSuffixCreator implements ResourceSuffixCreator {
	@Override
	public String getSuffix(int arg0) {
		return arg0 + ".xml";
	}
}
```

### FlatFile에 Header와 Footer 추가하기

- FlatFileHeaderCallback과 FlatFileFooterCallback을 사용해야 한다.
- Aspect를 사용해야 한다. (open 메서드 수행 전에 실행되어야 하므로)

### CompositeItemWriter

- 여러 ItemWriter를 동일한 아이템에 쓰기 작업을 수행할 수 있다.

```java
@Bean
public CompositeItemWriter<Customer> compositeItemWriter() throws Exception {
	return new CompositeItemWriterBuilder<Customer>()
		.delegates(Arrays.asList(xmlDelegateItemWriter(null), jdbcDelegateItemWriter(null)))
		.build();
}
```

- 쓰기 실행은 순차적으로 일어나며 동일한 트랜잭션에서 발생한다. 중간에 쓰기 작업을 수행하지 못했다면 전체 청크가 롤백된다.

### ClassifierCompositeItemWriter

- 아이템을 분류(Classfier)해서 어떤 ItemWriter를 쓸 지 판별해 전달할 수 있다

```java
@Bean
public class CustomerClassifier implements Classifier<Customer, ItemWriter<? Super Customer>> {
		private ItemWriter<Customer> fileItemWriter;
		private ItemWriter<Customer> jdbcItemWriter;

		// 생성자 주입

		@Override
		public ItemWriter<Customer> classify(Customer customer) {
				if (customer.getState().matches("^[A-M].*")) {
						return fileItemWriter;
				} else {
						return jdbcItemWriter;
				}
		}
}

@Bean
public ClassifierCompositeItemWriter<Customer> classifierCompositeItemWriter() throws Exception {
	Classifier<Customer, ItemWriter<? super Customer>> classifier = new CustomerClassifier(xmlDelegate(null), jdbcDelegate(null));
	
	return ClassifierCompositeItemWriterBuilder<Customer>()
		.classifier(classifier)
		.build();
}
```

- ClassifierCompositeItemWriter는 ItemStream의 메서드를 구현하지 않기 때문에 해당 ItemReader나 ItemWriter를 stream으로 등록해야 한다.

```java
// ...
	this.stepBuilderFactory..
		.reader()
		.writer()
		.stream(xmlDelegate(null)) // 이 부분이 추가됐다.
		.build();
```
