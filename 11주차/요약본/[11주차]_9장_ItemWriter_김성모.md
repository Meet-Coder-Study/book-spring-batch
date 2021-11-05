# ItemWriter-2

## 그 밖의 ItemWriter

### ItemWriterAdapter

> 기존에 운영하던 서비스를 재사용하기 위하여 `ItemWriterAdapter` 를 사용하여 ItemWriter로 사용한다.

ItemWriterAdapter를 구현하기 위해서는 다음의 두 가지 의존성이 필요하다.
1. targetObject : 호출할 메서드를 가진 스프링 빈
2. targetMethod : 각 아이템을 처리할 때 호출할 메서드

단 targetMethod는 단 하나의 아규먼트만을 받아야만 한다.

### PropertyExtractingDelegatingItemWriter

`ItemWriterAdapter`가 단 하나의 아규먼트만을 넘기는 것에 반해 `PropertyExtractingDelegatingItemWriter` 을 
사용하면 파라미터를 지정해서 넘길 수 있다. 


### JmsItemWriter

둘 이상의 엔드포인트 간에 통신하는 메시지 지향적 방식이다. (JavaMessagingService)


### SimpleMailMessageItemWriter

이메일을 보낼 때 사용하는 ItemWriter이다. 

## 여러 자원을 사용하는 ItemWriter

### MultiResourceItemWriter

동일한 포맷의 다중 파일을 단일 스탭네에서 읽는 기능과 유사하게 다중 리소스를 생성하는 방법이다.

### CompositeItemWriter

하나의 스텝이 하나의 출력물을 만드는 것이 아닌 여러 엔드포인트의 작업을 진행할 때 사용한다.

### ClassifierCompositeItemWriter

서로 다른 유형의 레코드를 서로 다른 파서와 매퍼가 처리할 수 있도록 만드는 작업을 할 때 사용한다.

### ItemStream 인터페이스

주기적으로 상태를 저장하고 복원하는 역할을 하며 open, update, close 세 가지 메소드로 구성된다.

open 은 파일을 열고 close 는 파일을 닫는다.
update는 각 청크가 완료될 때 현재 상태를 기록한다.
 