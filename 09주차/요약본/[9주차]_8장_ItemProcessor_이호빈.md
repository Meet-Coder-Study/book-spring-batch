# 8장 ItemProcessor

- ItemReader로 읽은 데이터를 이용해 어떤 작업을 수행하는 컴포넌트
    - 비즈니스 로직을 구현하는 곳
- ItemWriter가 쓰기 처리를 하지 않도록 필터링할 수도 있다
- 입력의 유효성 검증을 할 수 있다.
    - 옛날 버전에서는 ItemReader에서 했었다. ValidatingItemReader 클래스를 상속해서 썼어서 입력 방법에 영향을 줬다.
    - 지금은 ItemProcessor에서 유효성 검증을 한다. ItemProcessor에서 하면 입력 방법에 상관없이 처리 전에 객체의 유효성 검증을 해줄 수 있다.
- 기존 서비스를 재사용하는 ItemProcessorAdapter를 제공한다.
- 특정 스크립트를 사용할 수도 있다.(ScriptItemProcessor)
- ItemProcessor 체인을 만들 수 있다. 단일 아이템으로 여러 작업을 순서대로 수행할 수 있다.
- 입력 Item의 타입과 반환하는 Item의 타입이 같을 필요가 없다. ItemProcessor가 반환하는 타입은 ItemWriter가 입력으로 사용하는 타입이 된다.
- ItemProcessor가 null을 반환하면 추가적인 여러 ItemProcessor나 ItemWriter는 호출되지 않는다.
- ItemReader는 입력 데이터가 없을 때 null을 반환. ItemProcessor가 null을 반환하면 다른 아이템 처리가 계속 된다.

### ValidatingItemProcessor

- 입력 아이템의 유효성 검증을 수행하는 Validator 인터페이스 구현체를 사용할 수 있다.
    - 유효성 검증에 실패하면 ValidationException이 발생한다.
- NotNull, Alphabetic, Numeric, Size 등등...
    - 검증 애너테이션을 적용하려면 spring-boot-starter-validation이라는 새로운 스타터를 사용해야 한다.
    - 유효성 검증 도구의 하이버네이트 구현체를 가지고 온다.

### ItemProcessorAdapter

- 기존 서비스를 ItemProcessor로 사용할 수 있다

### ScriptItemProcessor

- ScriptItemProcessor를 통해 JS 같은 스크립트 파일을 주입해줄 수 있다.

### CompositeItemProcessor

- 단일 ItemProcessor에 몰아두는 것이 아니라 ItemProcessor를 체인처럼 연결해서 책임을 분담시킬 수 있다.
- ItemProcessor와 마찬가지로 null을 반환하면 해당 Item은 더이상 처리되지 않는다.
- 유효성 검증을 통과하지 못한 아이템을 걸러내기만 하도록 ItemProcessor를 구성할 수 있다.
- 일부 아이템은 ItemProcessorA에게 전달하고 일부 아이템은 ItemProcessorB에게 전달하고 싶다면? → ClassifierCompositeItemProcessor를 쓰면 된다.

### 커스텀 ItemProcessor 구현

- 아이템을 필터링하는 커스텀 ItemProcessor 구현
- 읽은 얘들만 JobRepository에 기록한다.

```jsx
// 이렇게 구현 후
public class EvenFilteringItemProcessor implements ItemProcessor<Customer, Customer> {
	@Override
	public Customer process(Customer item) {
		return Integer.parseInt(item.getZip()) % 2 == 0 ? null : item;
	}
}

// Bean으로 등록
public EvenFilteringProcessor itemProcessor() {
	return new EvenFilteringItemProcessor();
}
```

[실습 코드](https://github.com/aegis1920/my-lab/tree/master/def-guide-spring-batch)
