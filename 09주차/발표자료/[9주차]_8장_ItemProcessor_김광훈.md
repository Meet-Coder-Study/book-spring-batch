# 8장 ItemProcessor
## 1. ItemProcessor 란 ?
#### o ItemProcessor 는 Spring Batch 내에서 입력 데이터를 이용하여 어떤 작업을 수행하는 컴포넌트이다.
#### o ItemReader 로 읽은 데이터를 사용하여 특정 작업을 수행하도록 할 때 사용한다.
#### o ItemProcessor 는 멱등(idempotent) 이여야 한다.
#### o Work Flow
<img width="800" src="https://user-images.githubusercontent.com/60383031/137929258-c93c5a27-17b7-4360-8d0f-04e85b329baa.png">

<br>

#### o ItemReader Interface
<img width="800" src="https://user-images.githubusercontent.com/60383031/137614576-09bbbc8a-825e-43c1-994a-82bf2b49379e.png">

- 사용 방법
    ```java
      // 람다 사용하지 않은 예제
      @Bean
      public ItemProcessor<Teacher, String> processor() {
        return new ItemProcessor<Teacher, String>() {
          @Override
          public String process(Teacher teacher) throws Exception {
            return teacher.getName();
          }
        };
      }
   
      // 람다를 사용한 예제
      @Bean
      public ItemProcessor<Teacher, String> processor() {
        return teacher -> teacher.getName();
      }
   ```
    - 인터페이스에 추상 메소드가 process 하나만 있기 떼문에 위와 같이 람다식을 사용할 수 있다.
    

- 입력 아이템과 리턴 아이템의 타입이 같지 않아도 된다.
- ItemProcessor 가 반환하는 타입은 ItemWriter 가 사용하는 타입이여야 한다.
- ItemProcessor 가 null 을 반환하면 해당 아이템의 이후 모든 처리가 중지된다.
    - 즉, null 을 반환하면 ItemWriter 에 전달되지 않는다. 
    - 공식문서
      
      <br>
      <img width="800" src="https://user-images.githubusercontent.com/60383031/137615025-4c778973-b8e7-4929-939f-77a551cfbe2f.png">
    
        
<br>

#### o ItemReader Interface Implements
<img width="800" src="https://user-images.githubusercontent.com/60383031/137615149-843f4cd0-c632-4293-9544-5fd5537e5aa6.png">

<br>


## 2. ItemProcessor 사용하기 예제 (ValidatingItemProcessor)
#### o 유효성 검증은 Reader/Writer 가 아닌 Processor 에서 처리하는 것이 좋다.
#### o Validator Interface
<img width="800" src="https://user-images.githubusercontent.com/60383031/137615368-d0dc395c-846d-4f7e-a43f-f9db44eb351c.png">

#### o Validator Interface Implements
<img width="800" src="https://user-images.githubusercontent.com/60383031/137615457-01182927-347c-47f3-86db-1d593eb6d77a.png">

- Flow: check validator support ---> validate ---> if has error ---> throw exception

#### o BeanValidatingItemProcessor
<img width="800" src="https://user-images.githubusercontent.com/60383031/137615958-089a6d02-e10f-4a61-ad5b-8e1941aa7e54.png">


#### o Custom Validator
```java
public class UniqueLastNameValidator extends ItemStreamSupport implements Validator<Customer> {

	private Set<String> lastNames = new HashSet<>();

	@Override
	public void validate(Customer value) throws ValidationException {
		if(lastNames.contains(value.getLastName())) {
			throw new ValidationException("Duplicate last name was found: " + value.getLastName());
		}

		this.lastNames.add(value.getLastName());
	}

	@Override
	public void update(ExecutionContext executionContext) {
		executionContext.put(getExecutionContextKey("lastNames"), this.lastNames);
	}

	@Override
	public void open(ExecutionContext executionContext) {
		String lastNames = getExecutionContextKey("lastNames");

		if(executionContext.containsKey(lastNames)) {
			this.lastNames = (Set<String>) executionContext.get(lastNames);
		}
	}
}
```

<br>


## 3. ItemProcessorAdapter
#### o ItemProcessorAdapter 를 사용하여 이미 개발된 다양한 서비스가 ItemProcessor 역할을 하도록 만들 수 있다.
#### o 예제
- UpperCaseNameService
```java
@Service
public class UpperCaseNameService {

	public Customer upperCase(Customer customer) {
		Customer newCustomer = new Customer(customer);

		newCustomer.setFirstName(newCustomer.getFirstName().toUpperCase());
		newCustomer.setMiddleInitial(newCustomer.getMiddleInitial().toUpperCase());
		newCustomer.setLastName(newCustomer.getLastName().toUpperCase());

		return newCustomer;
	}

}
```

- ItemProcessorAdapter
```java
@Bean
public ItemProcessorAdapter<Customer, Customer> itemProcessor(UpperCaseNameService service) {
  ItemProcessorAdapter<Customer, Customer> adapter = new ItemProcessorAdapter<>();

  adapter.setTargetObject(service);     // Instance 지정 
  adapter.setTargetMethod("upperCase"); // Instance 의 method 지정

  return adapter;
}
```
#### o ItemProcessorAdapter

<img width="800" src="https://user-images.githubusercontent.com/60383031/137616684-ac10f4d7-e28f-45b5-8891-7a3fbfbb0a05.png">


#### o AbstractMethodInvokingDelegator
<img width="800" src="https://user-images.githubusercontent.com/60383031/137616921-dedd7407-9010-4e04-912f-7801a0364cbc.png">


<br>


#### o AbstractMethodInvokingDelegator extends
<img width="800" src="https://user-images.githubusercontent.com/60383031/137617006-2fe1a847-bf75-4f8a-8a7c-40c8d3fb954a.png">

- ItemProcessor 이외에 Reader, Writer Adapter 에도 사용

<br>

## 4. ScriptItemProcessor
#### o Java Script 를 ItemProcessor 에 사용하여 유연하게 배치 잡에 사용할 수 있다.
#### o 예제
```java
@Bean
@StepScope
public ScriptItemProcessor<Customer, Customer> itemProcessor(@Value("#{jobParameters['script']}") Resource script) {
  ScriptItemProcessor<Customer, Customer> itemProcessor = new ScriptItemProcessor<>();

  itemProcessor.setScript(script);

  return itemProcessor;
}
```

<br>

## 5. CompositeItemProcessor
#### o 스텝 내에서 여러 ItemProcessor 를 체인처럼 연결하여 책임을 분담하는 역할을 한다.
<img width="800" src="https://user-images.githubusercontent.com/60383031/137770706-1734a957-75c1-45e3-985e-8741e71280be.png">

#### o 예제
```java
@Bean
public CompositeItemProcessor<Customer, Customer> itemProcessor() {
  CompositeItemProcessor<Customer, Customer> itemProcessor =
    new CompositeItemProcessor<>();

    itemProcessor.setDelegates(Arrays.asList(
        customerValidatingItemProcessor(),
        upperCaseItemProcessor(null),
        lowerCaseItemProcessor(null)));
    
    return itemProcessor;
}
```
- customerValidatingItemProcessor()
    - 잘못된 레코드를 필터링하도록 입력 데이터의 유효성 검증을 수행
    

- upperCaseItemProcessor()
    - 사용자 이름을 대문자로 변경

    
- lowerCaseItemProcessor()
    - 스크립트 파일을 사용하여 address, city, state 피드 값을 소문자로 변경


<br>

## 6. CustomItemProcessor
#### o ItemProcessor 는 처리할 비즈니스 로직이 담기는 곳이다.
#### o 그러므로 사실상 항상 직접 구현해야 한다. ValidatingItemProcessor, ItemProcessorAdapter 는 사실상 거의 사용하지 않고 CompositeItemProcessor 는 가끔 사용한다. (조졸두님 블로그 참고)
#### o 예제
- 홀수 짝수를 구분하는 로직을 살펴본다.
- 홀수일 때만 긹하고 짝수는 필터링하는 ItemProcessor 를 살펴보자.
```java
public class EvenFilteringItemProcessor implements ItemProcessor<Customer, Customer> {

	@Override
	public Customer process(Customer item)  {
		return Integer.parseInt(item.getZip()) % 2 == 0 ? null: item;
	}
}
```

- ItemProcessor 를 구현하려면 ItemProcessor Interface 를 구현하는 클래스를 만들어야 한다.
- ItemProcessor 가 null 을 반환하기만 하면 해당 아이템이 필터링되도록 함으로써 과정을 단순화한다.
- null 을 던지면 그 이후에 수행되는 ItemProcessor / ItemWriter 에게 전달되지 않는다. 