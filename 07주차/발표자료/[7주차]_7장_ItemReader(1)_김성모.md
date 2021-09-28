# 7장 ItemReader

## ItemReader 인터페이스

```java
public interface ItemReader<T> {

    @Nullable
    T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;

}
```

스텝에 입력을 제공할 때, `ItemReader<T>` 인터페이스의 `read()` 메소드를 정의한다.

스프링 배치는 여러 입력 유형에 맞는 여러 구현체를 제공한다.

제공하는 입력 유형은 플랫 파일, 여러 데이터베이스, JMS 리소스, 기타 입력 소스 등 다양하다.

## 파일 입력

다양한 입력 유형을 처리할 때 어떤 파일 입력을 처리하는 지 방법에 대해 알아보자.

### 플랫 파일

`XML` 과 `JSON` 과는 다르게 데이터 포맷이나 의미를 정의하는 메타데이터가 없는 파일을 `플랫 파일`이라고 한다.

스프링 배치에서는 `FlatFileItemReader` 클래스를 사용한다.

`FlatFileItemReader` 는 메인 컴포넌트 두 개로 이뤄진다.

- Resource(읽을 대상 파일)
- LineMapper(ResultSet을 객체로 맵핑)

`LineMapper` 는 인터페이스로 구성되어 여러 구현체가 있는데, 그 중 가장 많이 사용하는 구현체는 `DefaultLineMapper` 이다.

`DefaultLineMapper` 는 파일에서 읽은 Raw String 을 두 단계 처리를 거쳐 도메인 객체로 변환한다.

1. LineTokenizer : 파일의 레코드 한 줄씩 파싱해 `FieldSet` 으로 만든다.
2. FieldSetMapper : `FieldSet` 을 도메인 객체로 매핑한다.

#### 고정 너비 파일

위에서 말했듯, 파일을 읽어서 레코드를 한 줄씩 파싱 후 도메인 객체에 매핑할 예정이다 예시를 위하여 고객 정보를 담은 resource 파일과 Customer 도메인 객체를 생성하자

```text
Aimee      CHoover    7341Vel Avenue          Mobile          AL35928
Jonas      UGilbert   8852In St.              Saint Paul      MN57321
Regan      MBaxter    4851Nec Av.             Gulfport        MS33193
Octavius   TJohnson   7418Cum Road            Houston         TX51507
Sydnee     NRobinson  894 Ornare. Ave         Olathe          KS25606
Stuart     KMckenzie  5529Orci Av.            Nampa           ID18562
Petra      ZLara      8401Et St.              Georgia         GA70323
Cherokee   TLara      8516Mauris St.          Seattle         WA28720
Athena     YBurt      4951Mollis Rd.          Newark          DE41034
Kaitlin    MMacias    5715Velit St.           Chandler        AZ86176
Leroy      XCherry    7810Vulputate St.       Seattle         WA37703
Connor     WMontoya   4122Mauris Av.          College         AK99743
Byron      XMedina    7875At Road             Rock Springs    WY37733
Penelope   YSandoval  2643Fringilla Av.       College         AK99557
Rashad     VOchoa     6587Lacus Street        Flint           MI96640
Jordan     UOneil     170 Mattis Ave          Bellevue        WA44941
Caesar     ODaugherty 7483Libero Ave          Frankfort       KY56493
Wynne      DRoth      8086Erat Street         Owensboro       KY50476
Robin      IRoberson  8014Pellentesque Street Casper          WY84633
Adrienne   CCarpenter 8141Aliquam Avenue      Tucson          AZ85057
```

```java

@Getter
@Setter
public class Customer {
    private String firstName;
    private String middleInitial;
    private String lastName;
    private String addressNumber;
    private String street;
    private String address;
    private String city;
    private String state;
    private String zipCode;

    public Customer() {
    }

    public Customer(String firstName, String middleName, String lastName, String addressNumber, String street, String city, String state, String zipCode) {
        this.firstName = firstName;
        this.middleInitial = middleName;
        this.lastName = lastName;
        this.addressNumber = addressNumber;
        this.street = street;
        this.city = city;
        this.state = state;
        this.zipCode = zipCode;
    }

    @Override
    public String toString() {
        return "Customer{" +
                "firstName='" + firstName + '\'' +
                ", middleInitial='" + middleInitial + '\'' +
                ", lastName='" + lastName + '\'' +
                ", address='" + address + '\'' +
                ", addressNumber='" + addressNumber + '\'' +
                ", street='" + street + '\'' +
                ", city='" + city + '\'' +
                ", state='" + state + '\'' +
                ", zipCode='" + zipCode + '\'' +
                '}';
    }
}
```

읽어들일 파일과 레코드를 읽어 매핑할 객체가 준비가 되었다면 간단한 ItemReader, ItemWriter, Step, Job 을 구성해보자

```java

@Configuration
public class FixedWidthJobConfiguration {
    @Autowired
    private JobBuilderFactory jobBuilderFactory;

    @Autowired
    private StepBuilderFactory stepBuilderFactory;

    @Bean
    @StepScope
    public FlatFileItemReader<Customer> fixedWidthItemReader(
            @Value("#{jobParameters['customerFile']}") Resource inputFile) {

        return new FlatFileItemReaderBuilder<Customer>()
                .name("fixedWidthItemReader")
                .resource(inputFile)
                .fixedLength()
                .columns(new Range[]{new Range(1, 11), new Range(12, 12), new Range(13, 22),
                        new Range(23, 26), new Range(27, 46), new Range(47, 62), new Range(63, 64),
                        new Range(65, 69)})
                .names(new String[]{"firstName", "middleInitial", "lastName",
                        "addressNumber", "street", "city", "state", "zipCode"})
                .targetType(Customer.class)
                .build();
    }

    @Bean
    public ItemWriter<Customer> fixedWidthItemWriter() {
        return (items) -> items.forEach(System.out::println);
    }

    @Bean
    public Step fixedWidthStep() {
        return this.stepBuilderFactory.get("fixedWidthStep")
                .<Customer, Customer>chunk(10)
                .reader(fixedWidthItemReader(null))
                .writer(fixedWidthItemWriter())
                .build();
    }

    @Bean
    public Job fixedWidthJob() {
        return this.jobBuilderFactory.get("fixedWidthJob")
                .start(fixedWidthStep())
                .build();
    }
}
```

#### 구분자로 구분된 파일

#### 커스텀 레코드 파싱

#### 여러 가지 레코드 포맷

#### 여러 줄에 걸친 레코드

#### 여러 개의 소스

### XML

### JSON