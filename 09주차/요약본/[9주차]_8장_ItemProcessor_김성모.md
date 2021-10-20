
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
ItemStreamSupport를 상속해 ItemStream 인터페이스를 구현함으로써 Validator 인터페이스 구현체를 주입할 것이다.