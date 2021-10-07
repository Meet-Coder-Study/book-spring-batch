# 7장 ItemReader - 2부(데이터베이스~)

## 데이터베이스 입력

### JDBC

스프링 배치는 JDBC를 통하여 데이터를 가져올 때 커서와 페이징 방식을 지원한다


#### JDBC 커서 처리

```java
@Bean
public JdbcCursorItemReader<Customer> customerItemReader(DataSource dataSource) {
    return new JdbcCursorItemReaderBuilder<Customer>()
    .name("customerItemReader")
    .dataSource(dataSource)
    .sql("select * from customer where city = ?")
    .rowMapper(new CustomerRowMapper())
    .preparedStatementSetter(citySetter(null))
    .build();
}
```

#### JDBC 페이징 처리

```java
@Bean
@StepScope
public JdbcPagingItemReader<Customer> customerItemReader(DataSource dataSource,
        PagingQueryProvider queryProvider,
        @Value("#{jobParameters['city']}") String city) {

    Map<String, Object> parameterValues = new HashMap<>(1);
    parameterValues.put("city", city);

    return new JdbcPagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .dataSource(dataSource)
            .queryProvider(queryProvider)
            .parameterValues(parameterValues)
            .pageSize(10)
            .rowMapper(new CustomerRowMapper())
            .build();
}
```


### 하이버네이트

#### 하이버네이트로 커서 처리하기

```java
@Bean
@StepScope
public HibernateCursorItemReader<Customer> customerItemReader(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {

    return new HibernateCursorItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
            .queryString("from Customer where city = :city")
            .parameterValues(Collections.singletonMap("city", city))
            .build();
}
```

#### 하이버네이트를 사용해 페이징 기법으로 데이터베이스에 접근하기

```java
@Bean
@StepScope
public HibernatePagingItemReader<Customer> customerItemReader(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {

    return new HibernatePagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .sessionFactory(entityManagerFactory.unwrap(SessionFactory.class))
            .queryString("from Customer where city = :city")
            .parameterValues(Collections.singletonMap("city", city))
            .pageSize(10)
            .build();
}
```

### JPA

페이징 기법만 제공

```java
@Bean
@StepScope
public JpaPagingItemReader<Customer> customerItemReader(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {

    return new JpaPagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .entityManagerFactory(entityManagerFactory)
            .queryString("select c from Customer c where c.city = :city")
            .parameterValues(Collections.singletonMap("city", city))
            .build();
}
```

쿼리프로바이더 사용

```java
@Bean
@StepScope
public JpaPagingItemReader<Customer> customerItemReader(
        EntityManagerFactory entityManagerFactory,
        @Value("#{jobParameters['city']}") String city) {

    CustomerByCityQueryProvider queryProvider =
            new CustomerByCityQueryProvider();
    queryProvider.setCityName(city);

    return new JpaPagingItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .entityManagerFactory(entityManagerFactory)
            .queryProvider(queryProvider)
            .parameterValues(Collections.singletonMap("city", city))
            .build();
}
```

### 저장 프로시저

```java
@Bean
@StepScope
public StoredProcedureItemReader<Customer> customerItemReader(DataSource dataSource,
        @Value("#{jobParameters['city']}") String city) {

    return new StoredProcedureItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .dataSource(dataSource)
            .procedureName("customer_list")
            .parameters(new SqlParameter[]{new SqlParameter("cityOption", Types.VARCHAR)})
            .preparedStatementSetter(new ArgumentPreparedStatementSetter(new Object[] {city}))
            .rowMapper(new CustomerRowMapper())
            .build();
}
```

## 스프링 데이터

### 몽고DB

```java
@Bean
@StepScope
public MongoItemReader<Map> tweetsItemReader(MongoOperations mongoTemplate,
@Value("#{jobParameters['hashTag']}") String hashtag) {
    return new MongoItemReaderBuilder<Map>()
    .name("tweetsItemReader")
    .targetType(Map.class)
    .jsonQuery("{ \"entities.hashtags.text\": { $eq: ?0 }}")
    .collection("tweets_collection")
    .parameterValues(Collections.singletonList(hashtag))
    .pageSize(10)
    .sorts(Collections.singletonMap("created_at", Sort.Direction.ASC))
    .template(mongoTemplate)
    .build();
}
```

### 스프링 데이터 리포지터리

```java

@Bean
@StepScope
public RepositoryItemReader<Customer> customerItemReader(CustomerRepository repository,
        @Value("#{jobParameters['city']}") String city) {

    return new RepositoryItemReaderBuilder<Customer>()
            .name("customerItemReader")
            .arguments(Collections.singletonList(city))
            .methodName("findByCity")
            .repository(repository)
            .sorts(Collections.singletonMap("lastName", Sort.Direction.ASC))
            .build();
}

```

## 기존 서비스

```java
@Component
public class CustomerService {

	private List<Customer> customers;
	private int curIndex;

	private String [] firstNames = {"Michael", "Warren", "Ann", "Terrence",
			"Erica", "Laura", "Steve", "Larry"};
	private String middleInitial = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
	private String [] lastNames = {"Gates", "Darrow", "Donnelly", "Jobs",
			"Buffett", "Ellison", "Obama"};
	private String [] streets = {"4th Street", "Wall Street", "Fifth Avenue",
			"Mt. Lee Drive", "Jeopardy Lane",
			"Infinite Loop Drive", "Farnam Street",
			"Isabella Ave", "S. Greenwood Ave"};
	private String [] cities = {"Chicago", "New York", "Hollywood", "Aurora",
			"Omaha", "Atherton"};
	private String [] states = {"IL", "NY", "CA", "NE"};

	private Random generator = new Random();

	public CustomerService() {
		curIndex = 0;

		customers = new ArrayList<>();

		for(int i = 0; i < 100; i++) {
			customers.add(buildCustomer());
		}
	}

	private Customer buildCustomer() {
		Customer customer = new Customer();

		customer.setId((long) generator.nextInt(Integer.MAX_VALUE));
		customer.setFirstName(
				firstNames[generator.nextInt(firstNames.length - 1)]);
		customer.setMiddleInitial(
				String.valueOf(middleInitial.charAt(
						generator.nextInt(middleInitial.length() - 1))));
		customer.setLastName(
				lastNames[generator.nextInt(lastNames.length - 1)]);
		customer.setAddress(generator.nextInt(9999) + " " +
				streets[generator.nextInt(streets.length - 1)]);
		customer.setCity(cities[generator.nextInt(cities.length - 1)]);
		customer.setState(states[generator.nextInt(states.length - 1)]);
		customer.setZipCode(String.valueOf(generator.nextInt(99999)));

		return customer;
	}

	public Customer getCustomer() {
		Customer cust = null;

		if(curIndex < customers.size()) {
			cust = customers.get(curIndex);
			curIndex++;
		}

		return cust;
	}
}
```


```java
@Bean
public ItemReaderAdapter<Customer> customerItemReader(CustomerService customerService){

    ItemReaderAdapter<Customer> adapter = new ItemReaderAdapter<>();

    adapter.setTargetObject(customerService);
    adapter.setTargetMethod("getCustomer");

    return adapter;
}
```

## 커스텀 입력

```java
public class CustomerItemReader extends ItemStreamSupport implements ItemReader<Customer> {

	private List<Customer> customers;
	private int curIndex;
	private String INDEX_KEY = "current.index.customers";

	private String [] firstNames = {"Michael", "Warren", "Ann", "Terrence",
			"Erica", "Laura", "Steve", "Larry"};
	private String middleInitial = "ABCDEFGHIJKLMNOPQRSTUVWXYZ";
	private String [] lastNames = {"Gates", "Darrow", "Donnelly", "Jobs",
			"Buffett", "Ellison", "Obama"};
	private String [] streets = {"4th Street", "Wall Street", "Fifth Avenue",
			"Mt. Lee Drive", "Jeopardy Lane",
			"Infinite Loop Drive", "Farnam Street",
			"Isabella Ave", "S. Greenwood Ave"};
	private String [] cities = {"Chicago", "New York", "Hollywood", "Aurora",
			"Omaha", "Atherton"};
	private String [] states = {"IL", "NY", "CA", "NE"};

	private Random generator = new Random();

	public CustomerItemReader() {
		curIndex = 0;

		customers = new ArrayList<>();

		for(int i = 0; i < 100; i++) {
			customers.add(buildCustomer());
		}
	}

	private Customer buildCustomer() {
		Customer customer = new Customer();

		customer.setId((long) generator.nextInt(Integer.MAX_VALUE));
		customer.setFirstName(
				firstNames[generator.nextInt(firstNames.length - 1)]);
		customer.setMiddleInitial(
				String.valueOf(middleInitial.charAt(
						generator.nextInt(middleInitial.length() - 1))));
		customer.setLastName(
				lastNames[generator.nextInt(lastNames.length - 1)]);
		customer.setAddress(generator.nextInt(9999) + " " +
				streets[generator.nextInt(streets.length - 1)]);
		customer.setCity(cities[generator.nextInt(cities.length - 1)]);
		customer.setState(states[generator.nextInt(states.length - 1)]);
		customer.setZipCode(String.valueOf(generator.nextInt(99999)));

		return customer;
	}

	@Override
	public Customer read() {
		Customer cust = null;

		if(curIndex == 50) {
			throw new RuntimeException("This will end your execution");
		}

		if(curIndex < customers.size()) {
			cust = customers.get(curIndex);
			curIndex++;
		}

		return cust;
	}

	public void close() throws ItemStreamException {
	}

	public void open(ExecutionContext executionContext) throws ItemStreamException {
		if(executionContext.containsKey(getExecutionContextKey(INDEX_KEY))) {
			int index = executionContext.getInt(getExecutionContextKey(INDEX_KEY));

			if(index == 50) {
				curIndex = 51;
			} else {
				curIndex = index;
			}
		} else {
			curIndex = 0;
		}
	}

	public void update(ExecutionContext executionContext) throws ItemStreamException {
		executionContext.putInt(getExecutionContextKey(INDEX_KEY), curIndex);
	}
}
```

```java
@Bean
public CustomerItemReader customerItemReader() {
    CustomerItemReader customerItemReader = new CustomerItemReader();

    customerItemReader.setName("customerItemReader");

    return customerItemReader;
}
```

## 에러 처리

### 레코드 건너뛰기

```java
@Bean
public Step copyFileStep() {
    return this.stepBuilderFactory.get("copyFileStep")
    .<Customer, Customer>chunk(10)
    .reader(customerItemReader(null))
    .writer(itemWriter())
    .faultTolerant()
    .skip(ParseException.class)
    .build();
}
```

### 잘못된 레코드 로그 남기기

```java
public class CustomerItemListener  {

	private static final Log logger = LogFactory.getLog(CustomerItemListener.class);

	@OnReadError
	public void onReadError(Exception e) {
		if(e instanceof FlatFileParseException) {
			FlatFileParseException ffpe = (FlatFileParseException) e;

			StringBuilder errorMessage = new StringBuilder();
			errorMessage.append("An error occured while processing the " +
					ffpe.getLineNumber() +
					" line of the file.  Below was the faulty " +
					"input.\n");
			errorMessage.append(ffpe.getInput() + "\n");

			logger.error(errorMessage.toString(), ffpe);
		} else {
			logger.error("An error has occurred", e);
		}
	}
}
```

```java
@Bean
public Step copyFileStep() {
    return this.stepBuilderFactory.get("copyFileStep")
    .<Customer, Customer>chunk(10)
    .reader(customerItemReader(null))
    .writer(itemWriter())
    .faultTolerant()
    .skipLimit(100)    
    .skip(Exception.class)
    .listener(customerListener())
    .build();
}
```

### 입력이 없을 때의 처리

```java
@AfterStep
public ExitStatus afterStep(StepExecution execution){
    if(execution.getReadCount() > 0){
        return execution.getExitStatus();
    } else {
        return ExitStatus.FAILED;
    }
}
```