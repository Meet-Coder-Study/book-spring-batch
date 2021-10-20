
# ItemProcessor

ItemReader에 의해 읽힌 데이터를 필터링하여 ItemWriter가 진행될 수 있도록 하기 위한 단계이다.

## ItemProcessor 소개

스프링 배치는 읽기, 처리, 쓰기 간 고려해야하는 문제를 여러 부분으로 분리하여 몇 가지 고유 작업을 수행할 수 있도록 하였다.

- 입력의 유효성 검증 : ValidatingItemReader 사용
- 기존 서비스의 재사용 : ItemProcessorAdapter 제공
- 스크립트 실행 : ScriptItemProcessor 사용
- ItemProcessor 체이닝 

위의 모든 기능을 제공하는 것은 ItemProcessor 인터페이스이다. 

## 스프링 배치의 ItemProcessor 사용하기

### ValidatingItemProcessor

데이터를 읽은 후 비즈니스 규칙에 따른 유효성 검증을 할 때 사용할 수 있다. 

입력 데이터의 유효성 검증은 스프링 배치 Validator 인터페이스 구현체를 사용할 수 있으며 
검증에 실패한 경우 `ValidationException` 이 발생해 일반적인 스프링 배치 오류 처리가 절차대로 진행된다.

#### 입력 데이터의 유효성 검증
객체의 매핑 시 유효성 검증을 위한 방법으로는 JSR303 인 빈 유효성 검증을 위한 자바 사양을 통해 검증되는데 
@NotNull, @Pattern, @Size 등의 어노테이션을 속성에 넣어주어 검증 규칙을 정의할 수 있다.

이 기능을 동작하기 위하여 검증 메커니즘을 제공해야 하는데, BeanValidatingItemProcessor 가 이 일을 할 것이다.

스프링 배치에서 제공하는 기능이 아닌 직접 검증기를 구현해야하는 경우
ValidatingItemProcessor 를 구현하여 직접 만든 
ItemStreamSupport를 상속해 ItemStream 인터페이스를 구현함으로써 Validator 인터페이스 구현체를 주입할 것이다.

이렇게 만들어진 구현체는 @Bean으로 등록하여 사용한다.

### ItemProcessorAdapter
기존 서비스를 사용해서 검증을 하기 위한 작업을 진행하자.
ItemProcessorAdapter 를 만들어 service를 주입해주고 @Bean으로 만들어준다.

### ScriptItemProcessor

스크립트 언어는 수정이 용이해서 자주 변경되는 컴포넌트의 경우 유연성을 제공할 수 있다.
따라서 스프링 배치에서 스크립트를 사용해 유연하게 잡에 주입할 수 있다.

Resource로 스크립트 파일을 파라미터로 넣어주며 ItemProcessor에서 사용할 스크립트를 정해준다.

### CompositeItemProcessor

읽은 아이템들을 처리할 대 여러 단계에 걸쳐서 처리할 수 있을텐데 이를 효율적으로 재사용하며 
복합처리를 하기 위한 방법으로 사용한다.
이는 체이닝을 통해서 이루어지는데 이때 사용하는 것이 `CompositeItemProcessor` 이다.
`.setDelegates()` 함수를 통해 여러 검증 처리기를 너허줄 수 있다.

조건에 따라 다른 처리가 진행되어야 할 땐, `Classifier` 에 processor를 주입하고 `classify` 메소드를
오버라이딩하면 된다.


## ItemProcessor 직접 만들기

마찬가지로 ItemProcessor 인터페이스를 상속받은 커스텀 ItemProcessor 를 구현한다.
이 때 `process()` 메소드를 오버라이딩한다.

그 후 @Bean으로 커스텀 아이템 프로세서를 등록해주면 된다.
null 처리로 리턴되는 아이템은 스키핑될 것이며 그렇지 않으면 카운트가 올라가 context에 저장될 것이다.

