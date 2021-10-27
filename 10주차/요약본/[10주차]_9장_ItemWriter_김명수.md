# 목차

- [파일 기반 ItemWriter](#파일-기반-ItemWriter)
    - [FlatFileItemWriter](#FlatFileItemWriter)
- [데이터베이스 기반 ItemWriter](#데이터베이스-기반-ItemWriter)
    - [JdbcBatchItemWrite](#JdbcBatchItemWrite)
    - [HibernateItemWriter](#HibernateItemWriter)
    - [JpaItemWriter](#JpaItemWriter)

---  

# 파일 기반 ItemWriter

## FlatFileItemWriter

### 형식화된 텍스트 파일

**(1) output file 생성**

```shell
$ touch ./Chapter09/src/main/resources/output/formattedCustomers.txt
```

**(2) ItemWriter 구성**

```java
@Bean
@StepScope
public FlatFileItemWriter<Customer> itemWriter(
        @Value("#{jobParameters['outputFile']}") Resource outputFile) {
    return new FlatFileItemWriterBuilder<Customer>()
            .name("customerItemWriter")
            .resource(outputFile)
            .formatted()
            .format("%s %s lives at %s %s in %s, %s.")
            .names(new String[] {
                    "firstName",
                    "lastName",
                    "address",
                    "city",
                    "state",
                    "zip"
            })
            .build();
}
```  

**(3) 실행 결과 확인**

```shell
$ cat ./Chapter09/target/classes/output/formattedCustomers.txt
Richard Darrow lives at 5570 Isabella Ave St. Louis in IL, 58540.
Warren Darrow lives at 4686 Mt. Lee Drive St. Louis in NY, 94935.
Barack Donnelly lives at 7844 S. Greenwood Ave Houston in CA, 38635.
Ann Benes lives at 2447 S. Greenwood Ave Las Vegas in NY, 55366.
Erica Gates lives at 3141 Farnam Street Omaha in CA, 57640.
Warren Williams lives at 6670 S. Greenwood Ave Hollywood in FL, 37288.
Harry Darrow lives at 3273 Isabella Ave Houston in FL, 97261.
Steve Darrow lives at 8407 Infinite Loop Drive Las Vegas in WA, 90520.
```

### 구분자로 구분된 파일

**(1) output file 생성**

```shell
$ touch ./Chapter09/src/main/resources/output/delimitedCustomers.txt
```

**(2) ItemWriter 구성**

```java
@Bean
@StepScope
public FlatFileItemWriter<Customer> itemWriter(
        @Value("#{jobParameters['outputFile']}") Resource outputFile) {
    return new FlatFileItemWriterBuilder<Customer>()
            .name("customerItemWriter")
            .resource(outputFile)
            .delimited()
            .delimiter(";")
            .names(new String[] {
                    "firstName",
                    "lastName",
                    "address",
                    "city",
                    "state",
                    "zip"
            })
            .build();
}
```  

**(3) 실행 결과 확인**

```shell
$ cat ./Chapter09/target/classes/output/delimitedCustomers.txt
Richard;Darrow;5570 Isabella Ave;St. Louis;IL;58540
Warren;Darrow;4686 Mt. Lee Drive;St. Louis;NY;94935
Barack;Donnelly;7844 S. Greenwood Ave;Houston;CA;38635
Ann;Benes;2447 S. Greenwood Ave;Las Vegas;NY;55366
Erica;Gates;3141 Farnam Street;Omaha;CA;57640
Warren;Williams;6670 S. Greenwood Ave;Hollywood;FL;37288
Harry;Darrow;3273 Isabella Ave;Houston;FL;97261
Steve;Darrow;8407 Infinite Loop Drive;Las Vegas;WA;90520
```

---  

# 데이터베이스 기반 ItemWriter

## JdbcBatchItemWrite

**(1) ItemPreparedStatementSetter 구현하기**

```java
public class CustomerItemPreparedStatementSetter implements ItemPreparedStatementSetter<Customer> {

    @Override
    public void setValues(Customer customer, PreparedStatement ps) throws SQLException {
        ps.setString(1, customer.getFirstName());
        ps.setString(2, customer.getMiddleInitial());
        ps.setString(3, customer.getLastName());
        ps.setString(4, customer.getAddress());
        ps.setString(5, customer.getCity());
        ps.setString(6, customer.getState());
        ps.setString(7, customer.getZip());
    }
}
```

**(2) ItemWriter 구성하기**

```java
@Bean
@StepScope
public JdbcBatchItemWriter<Customer> itemWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Customer>()
            .dataSource(dataSource)
            .sql("INSERT INTO customer (first_name,"
                 + "middle_initial, "
                 + "last_name, "
                 + "address, "
                 + "city, "
                 + "state, "
                 + "zip) VALUES(?, ?, ?, ?, ?, ?, ?)")
            .itemPreparedStatementSetter(new CustomerItemPreparedStatementSetter())
            .build();
}

// name parameter를 사용하기
@Bean
@StepScope
public JdbcBatchItemWriter<Customer> itemWriter2(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Customer>()
            .dataSource(dataSource)
            .sql("INSERT INTO customer (first_name,"
                 + "middle_initial, "
                 + "last_name, "
                 + "address, "
                 + "city, "
                 + "state, "
                 + "zip) VALUES(:firstName, "
                 + ":middleInitial, "
                 + ":lastName, "
                 + ":address, "
                 + ":city, "
                 + ":state, "
                 + ":zip)")
            .beanMapped()
            .build();
}
```

**(3) 실행 결과 확인하기**

```shell
SELECT * FROM customer;
+----+------------+----------------+-----------+--------------------------+-----------+-------+-------+
| id | first_name | middle_initial | last_name | address                  | city      | state | zip   |
+----+------------+----------------+-----------+--------------------------+-----------+-------+-------+
|  1 | Richard    | N              | Darrow    | 5570 Isabella Ave        | St. Louis | IL    | 58540 |
|  2 | Warren     | L              | Darrow    | 4686 Mt. Lee Drive       | St. Louis | NY    | 94935 |
|  3 | Barack     | G              | Donnelly  | 7844 S. Greenwood Ave    | Houston   | CA    | 38635 |
|  4 | Ann        | Z              | Benes     | 2447 S. Greenwood Ave    | Las Vegas | NY    | 55366 |
|  5 | Erica      | Z              | Gates     | 3141 Farnam Street       | Omaha     | CA    | 57640 |
|  6 | Warren     | M              | Williams  | 6670 S. Greenwood Ave    | Hollywood | FL    | 37288 |
|  7 | Harry      | T              | Darrow    | 3273 Isabella Ave        | Houston   | FL    | 97261 |
|  8 | Steve      | O              | Darrow    | 8407 Infinite Loop Drive | Las Vegas | WA    | 90520 |
+----+------------+----------------+-----------+--------------------------+-----------+-------+-------+
8 rows in set (0.01 sec)
```

---  

## HibernateItemWriter

**(1) Hibernate 관련 설정하기**

```java
@Component
public class HibernateBatchConfigurer implements BatchConfigurer {

    private DataSource dataSource;
    private SessionFactory sessionFactory;
    private JobRepository jobRepository;
    private PlatformTransactionManager transactionManager;
    private JobLauncher jobLauncher;
    private JobExplorer jobExplorer;

    public HibernateBatchConfigurer(DataSource dataSource, EntityManagerFactory entityManagerFactory) {
        this.dataSource = dataSource;
        this.sessionFactory = entityManagerFactory.unwrap(SessionFactory.class);
    }

    @Override
    public JobRepository getJobRepository() {
        return this.jobRepository;
    }

    @Override
    public PlatformTransactionManager getTransactionManager() {
        return this.transactionManager;
    }

    @Override
    public JobLauncher getJobLauncher() {
        return this.jobLauncher;
    }

    @Override
    public JobExplorer getJobExplorer() {
        return this.jobExplorer;
    }

    @PostConstruct
    public void initialize() {
        try {
            HibernateTransactionManager transactionManager = new HibernateTransactionManager(sessionFactory);
            transactionManager.afterPropertiesSet();

            this.transactionManager = transactionManager;

            this.jobRepository = createJobRepository();
            this.jobExplorer = createJobExplorer();
            this.jobLauncher = createJobLauncher();

        } catch (Exception e) {
            throw new BatchConfigurationException(e);
        }
    }

    private JobLauncher createJobLauncher() throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();

        jobLauncher.setJobRepository(this.jobRepository);
        jobLauncher.afterPropertiesSet();

        return jobLauncher;
    }

    private JobExplorer createJobExplorer() throws Exception {
        JobExplorerFactoryBean jobExplorerFactoryBean = new JobExplorerFactoryBean();

        jobExplorerFactoryBean.setDataSource(this.dataSource);
        jobExplorerFactoryBean.afterPropertiesSet();

        return jobExplorerFactoryBean.getObject();
    }

    private JobRepository createJobRepository() throws Exception {
        JobRepositoryFactoryBean jobRepositoryFactoryBean = new JobRepositoryFactoryBean();

        jobRepositoryFactoryBean.setDataSource(this.dataSource);
        jobRepositoryFactoryBean.setTransactionManager(this.transactionManager);
        jobRepositoryFactoryBean.afterPropertiesSet();

        return jobRepositoryFactoryBean.getObject();
    }
}
```

**(2) ItemWriter 구성하기**

```shell
@Bean
@StepScope
public HibernateItemWriter<Customer> itemWriter(EntityManagerFactory entityManager) {
    return new HibernateItemWriterBuilder<Customer>()
            .sessionFactory(entityManager.unwrap(SessionFactory.class))
            .build();
}
```  

---  

## JpaItemWriter

**(1) JPA 관련 설정하기**

```java
@Configuration
public class JpaBatchConfigurer implements BatchConfigurer {

    private DataSource dataSource;
    private EntityManagerFactory entityManagerFactory;
    private JobRepository jobRepository;
    private PlatformTransactionManager transactionManager;
    private JobLauncher jobLauncher;
    private JobExplorer jobExplorer;

    public JpaBatchConfigurer(DataSource dataSource, EntityManagerFactory entityManagerFactory) {
        this.dataSource = dataSource;
        this.entityManagerFactory = entityManagerFactory;
    }

    @Override
    public JobRepository getJobRepository() {
        return this.jobRepository;
    }

    @Override
    public PlatformTransactionManager getTransactionManager() {
        return this.transactionManager;
    }

    @Override
    public JobLauncher getJobLauncher() {
        return this.jobLauncher;
    }

    @Override
    public JobExplorer getJobExplorer() {
        return this.jobExplorer;
    }

    @PostConstruct
    public void initialize() {
        try {
            JpaTransactionManager transactionManager = new JpaTransactionManager(entityManagerFactory);
            transactionManager.afterPropertiesSet();

            this.transactionManager = transactionManager;

            this.jobRepository = createJobRepository();
            this.jobExplorer = createJobExplorer();
            this.jobLauncher = createJobLauncher();

        } catch (Exception e) {
            throw new BatchConfigurationException(e);
        }
    }

    private JobLauncher createJobLauncher() throws Exception {
        SimpleJobLauncher jobLauncher = new SimpleJobLauncher();

        jobLauncher.setJobRepository(jobRepository);
        jobLauncher.afterPropertiesSet();

        return jobLauncher;
    }

    private JobExplorer createJobExplorer() throws Exception {
        JobExplorerFactoryBean jobExplorerFactoryBean = new JobExplorerFactoryBean();

        jobExplorerFactoryBean.setDataSource(dataSource);
        jobExplorerFactoryBean.afterPropertiesSet();

        return jobExplorerFactoryBean.getObject();
    }

    private JobRepository createJobRepository() throws Exception {
        JobRepositoryFactoryBean jobRepositoryFactoryBean = new JobRepositoryFactoryBean();

        jobRepositoryFactoryBean.setDataSource(dataSource);
        jobRepositoryFactoryBean.setTransactionManager(transactionManager);
        jobRepositoryFactoryBean.afterPropertiesSet();

        return jobRepositoryFactoryBean.getObject();
    }
}
```

**(2) ItemWriter 구성하기**

```java
@Bean
@StepScope
public JpaItemWriter<Customer> itemWriter(EntityManagerFactory entityManager) {
    JpaItemWriter<Customer> jpaItemWriter = new JpaItemWriter<>();

    jpaItemWriter.setEntityManagerFactory(entityManager);

    return jpaItemWriter;
}
```