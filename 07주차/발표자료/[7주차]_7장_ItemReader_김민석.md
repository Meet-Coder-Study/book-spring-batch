# 7장 ItemReader - 1부 (~json)

청크 기반 스텝은 ItemReader - ItemProcessor - ItemWriter 로 구성.

스프링 배치는 거의 모든 유형의 입력 데이터를 읽을 수 있는 기반 코드를 제공함과 동시에 커스터마이징 할 수 있는 기능도 지원함.

7장에서는 ItemReader가 제공하는 다양한 기능을 알아본다.

## ItemReader 인터페이스

```java
public interface ItemReader<T> {

    @Nullable
    T read() throws Exception, UnexpectedInputException, ParseException, NonTransientResourceException;

}
```

- 스프링 배치는 처리할 입력 유형에 맞는 ItemReader의 여러 구현체 제공
![image](https://user-images.githubusercontent.com/6725753/135055056-aaac8995-8e5a-4b0d-98a3-08fdca72d1bb.png)
  ![image](https://user-images.githubusercontent.com/6725753/135055276-8d4d2882-cc7c-441a-86eb-79ce775aba7c.png)

- 스텝 내에서 동작 방식
  - 스프링 배치가 read 메서드 호출
  - 아이템 한개 반환
  - 청크의 limit 개수 만큼 반복
  - 청크 개수 만큼 확보되면 ItemProcessor -> ItemWriter로 청크 전달

## 파일 입력

스프링 배치는 파일을 고성능 IO 로 읽어 올 수 있는 선언적 방식의 리더를 제공

### 플랫 파일

플랫 파일 : 구분자가 없는 한개 이상의 레코드로 구성된 텍스트 파일

FlatFileItemReader 사용.

FlatFileItemReader은 내부적으로 Resource(파일)와 LineMapper(DefaultLineMapper) 구현체로 구성.

LineMapper는 LineTokenizer와 FieldSetMapper로 구성.

FlatFileItemReader는 빌더를 사용하여 여러 옵션을 줄 수 있다.

- 동작방식
  - 레코드 한 개에 해당하는 문자열이 LineMapper로 전달
  - LineTokenizer가 원시 문자열을 파싱해서 FieldSet으로 만든다
  - FieldSetMapper가 FieldSet을 토대로 도메인 객체로 매핑


#### 고정 너비 파일

고정 너비 파일 : 레코드의 각 항목이 고정된 길이로 표현

```text
Aimee      CHoover    7341Vel Avenue          Mobile          AL35928
Jonas      UGilbert   8852In St.              Saint Paul      MN57321
Regan      MBaxter    4851Nec Av.             Gulfport        MS33193
Octavius   TJohnson   7418Cum Road            Houston         TX51507
Sydnee     NRobinson  894 Ornare. Ave         Olathe          KS25606
Stuart     KMckenzie  5529Orci Av.            Nampa           ID18562
```

`<Lab1>`

#### 필드가 구분자로 구분된 파일

delimited file : 레코드의 각 항목이 특정한 딜리미터로 구분되어 있는 텍스트 파일

```text
Aimee,C,Hoover,7341,Vel Avenue,Mobile,AL,35928
Jonas,U,Gilbert,8852,In St.,Saint Paul,MN,57321
Regan,M,Baxter,4851,Nec Av.,Gulfport,MS,33193
Octavius,T,Johnson,7418,Cum Road,Houston,TX,51507
Sydnee,N,Robinson,894,Ornare. Ave,Olathe,KS,25606
Stuart,K,Mckenzie,5529,Orci Av.,Nampa,ID,18562
```

`<Lab2>`

건물번호와 거리명을 붙여서 단일 필드로 매핑

FieldSetMapper의 커스텀 구현체 사용.

`<Lab3>`

#### 커스텀 레코드 파싱

건물번호와 거리명을 합칠 때 커스텀 LineTokenizer 구현체 사용.

커스텀 LineTokenizer는 아래와 같은 경우에도 사용
- 특이한 파일 포맷 파싱
- 서드파티 파일 포맷 파싱
- 특수한 타입 변환 요구 조건 처리

#### 여러 가지 레코드 포맷

한 파일 안에 여러 레코드가 섞여 있을 경우 PatternMatchingCompositeLineMapper 사용.

기존 DefaultLineMapper는 단일 필드매퍼와 토크나이저를 사용했지만 PatternMatchingCompositeLineMapper는 Map을 통해서 패턴 매칭을 사용하여 다양한 매퍼와 토크나이저 사용 가능.

```text
CUST,Warren,Q,Darrow,8272 4th Street,New York,IL,76091
TRANS,1165965,2011-01-22 00:13:29,51.43
CUST,Ann,V,Gates,9247 Infinite Loop Drive,Hollywood,NE,37612
CUST,Erica,I,Jobs,8875 Farnam Street,Aurora,IL,36314
TRANS,8116369,2011-01-21 20:40:52,-14.83
TRANS,8116369,2011-01-21 15:50:17,-45.45
TRANS,8116369,2011-01-21 16:52:46,-74.6
TRANS,8116369,2011-01-22 13:51:05,48.55
TRANS,8116369,2011-01-21 16:51:59,98.53
```

`<Lab4>`


#### 여러 줄에 걸친 레코드

하나의 커스터머에 여러개의 트랜잭션을 가진 도메인 객체로 변환하고 싶을때는?

`<Lab5>`

#### 여러 개의 소스

여러 개의 파일에서 읽어야 하는 경우라면?

`<Lab6>`


### XML

파일 포맷의 차이는 메타데이터의 차이.

XML은 태그라는 메타데이터를 사용하여 데이터를 완전히 설명 함.

XML 파서는 대표적으로 DOM과 SAX가 있다. DOM은 한번에 읽어 들이는 방식으로 효율적이지 않기 때문에 이벤트 기반의 SAX 사용.

스프링 배치에서는 문서 내 각 섹션을 독립적으로 파싱하는 StAX 파서 사용.

```text
StAX(Streaming API for XML)는 XML 문서를 처리하는 자바 API로서, 기존 DOM 및 SAX에 추가된 API이다.

기존 XML API는 2가지 방식이었다.

트리 기반 : 문서 전체를 트리(Tree) 구조로 메모리로 읽어서 랜덤하게 접근이 가능하다.
이벤트 기반 : 문서의 한 항목식 이벤트가 발생하여 응용 프로그램에서 처리한다.
2가지 방식은 각각 보완작용을 한다. 트리 기반(DOM)은 문서의 구조 해석이 가능하고, 이벤트 기반(SAX)은 메모리를 적게 사용하면서 신속한 작동이 가능하다.

StAX는 이러한 방식의 중간 방식으로 설계되었다. StAX 방식은 프로그램의 동작점, 즉 문서의 한지점을 가리키는 커서가 있는 방식이다. 이러한 이유로 응용 프로그램은 필요에 따라 정보를 추출할 수 있게 된다. (pull 형) 이것이 기존 SAX와 같은 이벤트 기반 API와 다른 점이다. SAX는 파서에서 응용 프로그램으로 데이터를 보내는 방식이다. (push형)

https://ko.wikipedia.org/wiki/StAX
```

ItemReader 구현체는 StaxEventItemReader 사용

```java
public class StaxEventItemReader<T> extends AbstractItemCountingItemStreamItemReader<T> implements
ResourceAwareItemReaderItemStream<T>, InitializingBean {
}

public interface ResourceAwareItemReaderItemStream<T> extends ItemStreamReader<T> {

  void setResource(Resource resource);
}

public interface ItemStreamReader<T> extends ItemStream, ItemReader<T> {

}

public interface ItemStream {

  void open(ExecutionContext executionContext) throws ItemStreamException;
  void update(ExecutionContext executionContext) throws ItemStreamException;
  void close() throws ItemStreamException;
}
```

XML 바인딩은 JAXB 사용.

`<Lab7>`

### JSON

XML과 방식은 거의 유사.

`<Lab8>`