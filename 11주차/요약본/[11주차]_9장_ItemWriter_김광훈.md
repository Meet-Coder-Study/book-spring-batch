# ItemWriter - 2
## JmsItemWriter
JMS 는 둘 이상의 엔드포인트 간에 통신하는 메시지 지향적인 방식이다.

JMS 을 사용해 작업을 하려면 JMS 브로커를 사용해야 한다.

JMS 를 단일 스텝으로 사용한다면 큐에 올바르게 도착했는지 알 수 없기 때문에 두 개의 스텝을 사용한다.

첫 번쨰 스텝은 파일에 읽고 큐에 쓴다. 두 번째 스텝은 큐에서 데이터를 읽고 파일을 쓴다.

## SimpleMailMessageItemWriter
고객에게 메일을 보내려고 한다면, SimpleMailMessageItemWriter 를 사용하면 좋다.

JMS 와 비슷한 이유로 두 개의 스텝을 사용하는 것을 권장한다.

## CompositeItemWriter
CompositeItemWriter 를 사용해서 스텝 내에서 여러 ItemWriter 가 동일한 아이템에 대해 쓰기 작업을 수행할 수 있다.

## ClassifierCompositeItemWriter
ClassifierCompositeItemWriter 는 서로 다른 유형의 아이템을 확인하고 어떤 ItemWriter 를 사용해 쓰기 작업을 수행할지 판별한 후 적절한 라이터에 아이템을 전달할 수 있다.

## ItemStream 인터페이스
ItemStream 인터페이스는 주기적으로 상태를 저장하고 복원하는 역할을 한다.

해당 인터페이스는 open, update, close 세 가지 인터페이스로 구성되며, 상태를 가진 ItemReader 나 ItemWriter 에 의하여 구현된다.

예를 들어, 입력이나 출력에 파일을 상요한다면 open 메서드는 필요한 파일을 열고 close 메서드는 필요한 파일을 닫는다.

update 메서드는 각 청크가 완료가 될 때 현재 상황을 기록한다.

## ClassifierCompositeItemWriter vs CompositeItemWriter
둘의 차이점은 CompositeItemWriter 가 org.springframework.batch.item.ItemStream 인터페이스를 구현했다는 것이다.

CompositeItemWriter 에서 open 메서드는 필요에 따라 위임 ItemWriter 의 open 메서드를 반복적으로 호출한다.

close 와 update 메서드도 동일한 방식으로 동작한다. 반면 ClassifierCompositeItemWriter 는 ItemStream 의 메서드를 구현하지 않는다.

이 때문에 XML 파일이 열리지 않은 상태에서 XMLEventFactory 생성되거나 XML 쓰기가 시도되어 예외가 발생한다.