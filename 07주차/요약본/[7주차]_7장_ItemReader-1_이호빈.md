# 7-1장 ItemReader

- 스프링 배치는 거의 모든 유형의 입력 데이터를 처리할 수 있는 표준 방법을 제공한다

### ItemReader 인터페이스

- 스프링 배치의 ItemReader와 javax(JSR)의 ItemReader는 다르다.
    - JSR의 ItemReader는 ItemSteam과 ItemReader를 조합한 것
- ItemReader의 read()를 호출하면 Step 내에서 처리할 Item 한 개를 반환한다.

### 파일 입력

- 플랫파일
    - 한 개 이상의 레코드가 포함된 파일. 메타데이터가 없다
    - 파일 형태에 여러 경우가 있을 수 있다.
        - 고정 너비로 있는 경우, 구분자로 구분된 경우(csv), 여러 줄로 구성된 레코드를 읽는 경우, 다중 파일을 읽는 경우
    - FlatFileItemReader
        - 읽어들일 대상 파일을 나타내는 Resource, 필드의 묶음을 나타내는 LineMapper 인터페이스가 있다
        - LineMapper 구현체인 DefaultLineMapper를 통해서 읽을 수 있다.
            - String을 대상으로 도메인 객체로 변환해줄 수 있다
            - LineTokenizer로 해당 줄을 파싱하고, FieldSetMapper로 도메인 객체를 매핑한다
            - 커스텀 LineTokenizer, 커스텀 FieldSetMapper로 만들어줄 수 있다
                - 파싱 후, 원하는 대로 도메인 필드에 넣어줄 수 있다. (e.g. street + address = address 혹은 조건을 추가한다든지 등...)

> 각각 입력 파일에 대한 구현체들을 제공한다.
> 

### ItemReader 인터페이스 vs ItemStreamReader 인터페이스

- ItemReader
    - 데이터를 제공하고 다음으로 가게 해주는 인터페이스
    - read() 밖에 없다.
- ItemStreamReader
    - ItemReader와 ItemStream을 합친 것
- ItemStream
    - 에러가 발생했을 때 상태를 ExecutionContext에 저장하거나 ExecutionContext에 있는 상태를 꺼내서 복원하기 위함
    - 그래서 open, update, close 등이 있음

### XML

- XML 파일을 입력 받기 위해서 JAXB 라는 구현체를 이용한다. (여러 구현체가 있음)

[실습 링크](https://github.com/aegis1920/my-lab/tree/master/def-guide-spring-batch)
