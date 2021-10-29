# 9장 ItemWriter

## 파일 기반

```java
@Bean
@StepScope
public FlatFileItemWriter<Customer> customerItemWriter(
        @Value("#{jobParameters['outputFile']}") Resource outputFile) {

    return new FlatFileItemWriterBuilder<Customer>()
            .name("customerItemWriter")
            .resource(outputFile)
            .formatted()
            .format("%s %s lives at %s %s in %s, %s.")
            .names(new String[] {"firstName", "lastName", "address", "city", "state", "zip"})
            .build();
}
```

```java
@Bean
@StepScope
public FlatFileItemReader<Customer> customerFileReader(
        @Value("#{jobParameters['customerFile']}")Resource inputFile) {

    return new FlatFileItemReaderBuilder<Customer>()
            .name("customerFileReader")
            .resource(inputFile)
            .delimited()
            .names(new String[] {"firstName",
                    "middleInitial",
                    "lastName",
                    "address",
                    "city",
                    "state",
                    "zip"})
            .targetType(Customer.class)
            .build();
}
```

## XML

```java
Bean
@StepScope
public StaxEventItemWriter<Customer> xmlCustomerWriter(
        @Value("#{jobParameters['outputFile']}") Resource outputFile) {

    Map<String, Class> aliases = new HashMap<>();
    aliases.put("customer", Customer.class);

    XStreamMarshaller marshaller = new XStreamMarshaller();

    marshaller.setAliases(aliases);

    marshaller.afterPropertiesSet();

    return new StaxEventItemWriterBuilder<Customer>()
            .name("customerItemWriter")
            .resource(outputFile)
            .marshaller(marshaller)
            .rootTagName("customers")
            .build();
}
```

## 데이터베이스 기반

### JDBC

```java
@Bean
public JdbcBatchItemWriter<Customer> jdbcCustomerWriter(DataSource dataSource) throws Exception {
    return new JdbcBatchItemWriterBuilder<Customer>()
            .dataSource(dataSource)
            .sql("INSERT INTO CUSTOMER (first_name, " +
                    "middle_initial, " +
                    "last_name, " +
                    "address, " +
                    "city, " +
                    "state, " +
                    "zip) VALUES (:firstName, " +
                    ":middleInitial, " +
                    ":lastName, " +
                    ":address, " +
                    ":city, " +
                    ":state, " +
                    ":zip)")
            .beanMapped()
            .build();
}
```

### JPA

```java
@Bean
public JpaItemWriter<Customer> jpaItemWriter(EntityManagerFactory entityManagerFactory) {
    JpaItemWriter<Customer> jpaItemWriter = new JpaItemWriter<>();

    jpaItemWriter.setEntityManagerFactory(entityManagerFactory);

    return jpaItemWriter;
}
```

## 스프링데이터

### Repository

```java
@Entity
@Table(name = "customer")
public class Customer implements Serializable {
	private static final long serialVersionUID = 1L;

	@Id
	@GeneratedValue(strategy = GenerationType.IDENTITY)
	private long id;
	private String firstName;
	private String middleInitial;
	private String lastName;
	private String address;
	private String city;
	private String state;
	private String zip;
	private String email;

	// Getter and Setter
}
```

```java
@Bean
public RepositoryItemWriter<Customer> repositoryItemWriter(CustomerRepository repository) {
    return new RepositoryItemWriterBuilder<Customer>()
            .repository(repository)
            .methodName("save")
            .build();
}
```