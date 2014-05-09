# 3 일반 사용법

본 장은 레퍼런스 튜토리얼에서 다루지 않은 내용을 설명한다.

## 3.1. 쿼리 생성
Querydsl에서 Query를 생성하려면 표현식 인자를 이용해서 query 메서드를 호출한다. query 메서드는 모듈에 따라 다르고 이미 튜토리얼에서 설명했으므로, 본 절에서는 표현식에 초점을 맞출 것이다.

표현식을 생성할 때에는 도메인 모듈에서 생성한 표현식 타입의 필드와 메서드를 이용한다. 코드 생성을 할 수 없는 경우, 표현식을 생성하기 위한 범용 방법을 사용하면 된다.

### 3.1.1 복잡한 조건(predicates)

복잡한 불리언 표현식을 작성하려면 com.mysema.query.BooleanBuilder 클래스를 사용한다. 이 클래스는 Predicate을 구현하고 있고 메서드 체인 형식으로 사용할 수 있다.
```
public List<Customer> getCustomer(String... names){
    QCustomer customer = QCustomer.customer;
    JPAQuery query = new JPAQuery(entityManager).from(customer);
    BooleanBuilder builder = new BooleanBuilder();
    for (String name : names){
        builder.or(customer.name.eq(name));
    }
    query.where(builder); // customer.name eq name1 OR customer.name eq name2 OR ...
    return query.list(customer);
}
```
BooleanBuilder는 상태변경이 되며(mutable) 초기에는 null을, 각 and나 or 뒤에는 오퍼레이션의 결과를 표현한다.

### 3.1.2 동적 표현식
com.mysema.query.support.Expressions 클래스는 동적인 표현식 생성을 위한 정적 팩토리 클래스이다. 팩토리 메서드는 리턴 타입에 따라 이름을 지었으므로 쉽게 유추할 수 있다.

일반적으로 동적 경로, 커스텀 구문이나 커스텀 오퍼레이션과 같이 Fluent DSL 형식을 사용할 수 없는 경우에 한해 Expressions 클래스를 사용한다.

다음 표현식을 보자.
```
QPerson person = QPerson.person;
person.firstName.startsWith("P");
```
만약 Q타입 생성이 가능하지 않으면 다음과 같이 위와 동일한 표현식을 만들 수 있다.
```
Path<Person> person = Expressions.path(Person.class, "person");
Path<String> personFirstName = Expressions.path(String.class, person, "firstName");
Constant<String> constant = Expressions.constant("P");
Expressions.predicate(Ops.STARTS_WITH, personFirstName, constant);
```
Path 인스턴스는 변수와 프로퍼티를 의미하고, Constant는 상수를, Operation은 오퍼레이션을 표현하며, TemplateExpression 인스턴스를 사용해서 String 템플릿으로 표현식을 표현할 수 있다.

### 3.1.3 동적 경로
Expressions 기반의 표현식 생성 외에 Querydsl은 동적 경로 생성을 위한 더 표현력이 좋은 API를 제공한다.

동적 경로 생성을 위해 com.mysema.query.types.path.PathBuilder 클래스를 사용할 수 있다. 이 클래스는 EntityPathBase 클래스를 확장하고 있고 경로 생성을 위해 클래스 생성과 별칭 사용 대신에 사용가능하다.

Expressions API와 비교해서 PathBuilder는 알지 못하는 오퍼레이션이나 커스텀 구문을 직접적으로 지원하진 않지만, 그 구문은 좀더 일반적인 DSL에 가깝다.

Strign 프로퍼티:
```
PathBuilder<User> entityPath = new
PathBuilder<User>(User.class, "entity");
// fully generic access
entityPath.get("userName");
// .. or with supplied type
entityPath.get("userName", String.class);
// .. and correct signature
entityPath.getString("userName").lower();
List property with component type:

entityPath.getList("list", String.class).get(0);
```

복합 표현식 타입 사용:
```
entityPath.getList("list", String.class, StringPath.class).get(0).lower();
```

키와 값 타입을 갖는 맵 프로퍼티:
```
entityPath.getMap("map", String.class, String.class).get("key");
```

복합 표현식 타입 사용:
```
entityPath.getMap("map", String.class, String.class, StringPath.class).get("key").lower();
```

### 3.1.4 Case 표현식
case-when-then-else 표현식을 만들 땐, 다음과 같이 CaseBuilder 클래스를 사용한다.

```
QCustomer customer = QCustomer.customer;
Expression<String> cases = new CaseBuilder()
    .when(customer.annualSpending.gt(10000)).then("Premier")
    .when(customer.annualSpending.gt(5000)).then("Gold")
    .when(customer.annualSpending.gt(2000)).then("Silver")
    .otherwise("Bronze");
// The cases expression can now be used in a projection or condition
```
equals-operations을 가진 case 표현식은 다음과 같이 단순한 형태를 사용하면 된다.
```
QCustomer customer = QCustomer.customer;
Expression<String> cases = customer.annualSpending
    .when(10000).then("Premier")
    .when(5000).then("Gold")
    .when(2000).then("Silver")
    .otherwise("Bronze");
// The cases expression can now be used in a projection or condition
```
JDOQL에서는 아직 Case 표현식을 지원하지 않는다.

### 3.1.5 Casting 표현식
표현식 타입에서 지네릭 시그너처를 피하기 위해, 타입 계층을 단순화시킨다. 그 결과로 모든 생성된 쿼리 타입은 com.mysema.query.types.path.EntityPathBase나 com.mysema.query.types.path.BeanPath를 직접 상속받으며, 논리적인 상위 타입으로 타입 변환할 수 없다.

자바 타입 변환을 직접 사용하는 대신, _super 필드를 통해서 상위 타입에 대한 레퍼런스에 접근할 수 있다. 생성된 쿼리 타입이 한 개 상위 타입을 가질 경우, _super 필드를 사용할 수 있다.

```
// from Account
QAccount extends EntityPathBase<Account>{
    // ...
}

// from BankAccount extends Account
QBankAccount extends EntityPathBase<BankAccount>{

    public final QAccount _super = new QAccount(this);

    // ...
}
```

상위 타입에서 하위 타입으로 변환하려면 EntityPathBase 클래스의 as 메서드를 사용하면 된다.
```
QAccount account = new QAccount("account");
QBankAccount bankAccount = account.as(QBankAccount.class);
```

### 3.1.6 리터럴 선택
Constant 표현식을 통해 리터럴을 선택할 수 있다. 다음은 간단한 예다.
```
query.list(Expressions.constant(1),
           Expressions.constant("abc"));
```
서브쿼리에서 Constant 표현식을 종종 사용한다.

## 3.2 결과 처리
Querydsl은 결과 처리를 커스터마이징 하기 위해 행 기반 변환을 위한 FactoryExpressions과 집합을 위한 ResultTransformer를 제공하고 있다.

com.mysema.query.types.FactoryExpression 인터페이스는 빈 생성, 생성자 호출 그리고 더 복잡한 객체를 생성하기 위해 사용된다. com.mysema.query.types.Projections 클래스를 이용해서 FactoryExpression 구현체 기능에 접근할 수 있다.

com.mysema.query.ResultTransformer 인터페이스의 주요 구현체는 GroupBy 클래스이다.

### 3.2.1 다중 컬럼 리턴
Querydsl 3.0 부터 다중 컬럼 결과를 위한 기본 타입은 com.mysema.query.Tuple 이다. Tuple은 타입에 안전한 Map을 제공하고, 이를 통해 Tuple 행 객체로부터 컬럼 데이터에 접근할 수 있다.

```
List<Tuple> result = query.from(employee).list(employee.firstName, employee.lastName);
for (Tuple row : result) {
     System.out.println("firstName " + row.get(employee.firstName));
     System.out.println("lastName " + row.get(employee.lastName));
}}
```
위 예제를 QTuple 클래스를 이용하면 다음과 같이 작성할 수 있다.
```
List<Tuple> result = query.from(employee).list(new QTuple(employee.firstName, employee.lastName));
for (Tuple row : result) {
     System.out.println("firstName " + row.get(employee.firstName));
     System.out.println("lastName " + row.get(employee.lastName)); 
}}
```

### 3.2.2 빈 생성(population)
쿼리 결과로부터 빈을 생성하고 싶다면, Bean 프로젝션을 사용하면 된다.
```
List<UserDTO> dtos = query.list(
    Projections.bean(UserDTO.class, user.firstName, user.lastName));
```
setter 메서드 대신 필드에 직접 접근해야 한다면 다음 코드를 사용하면 된다.
```
List<UserDTO> dtos = query.list(
    Projections.fields(UserDTO.class, user.firstName, user.lastName));
```

### 3.2.3 생성자 사용
생성자 기반의 행 변환을 하는 방법은 다음과 같다.
```
List<UserDTO> dtos = query.list(
    Projections.bean(UserDTO.class, user.firstName, user.lastName));
```
지네릭 생성자 표현식을 사용하는 대신, QueryProjection 어노테이션을 적용한 생성자를 사용할 수도 있다.
```
class CustomerDTO {

  @QueryProjection
  public CustomerDTO(long id, String name){
     ...
  }

}
```
그리고, 이 클래스를 다음과 같이 쿼리에서 사용 가능하다.
```
QCustomer customer = QCustomer.customer;
JPQLQuery query = new HibernateQuery(session);
List<CustomerDTO> dtos = query.from(customer).list(new QCustomerDTO(customer.id, customer.name));
```
이 예제가 Hibernate용 코드이긴 하지만, 다른 모든 모듈에서도 이 기능을 사용할 수 있다.

만약 QueryProjection 어노테이션이 적용된 타입이 엔티티(@Entity) 타입이 아니라면, 예제처럼 생성자 프로젝션을 사용할 수 있다. 하지만, 어노테이션이 적용된 타입이 엔티티(@Entity) 타입이라면 쿼리 타입의 정적 create 메서드를 실행해서 생성자 프로젝션을 만들 필요가 있다.
```
@Entity
class Customer {

  @QueryProjection
  public Customer(long id, String name){
     ...
  }

}
QCustomer customer = QCustomer.customer;
JPQLQuery query = new HibernateQuery(session);
List<Customer> dtos = query.from(customer).list(QCustomer.create(customer.id, customer.name));
```
코드 생성을 할 수 없다면, 다음과 같이 생성자 프로젝션을 생성할 수 있다.
```
List<Customer> dtos = query.from(customer)
    .list(ConstructorExpression.create(Customer.class, customer.id, customer.name));   
```

### 3.2.4 결과 집합(aggregation)
com.mysema.query.group.GroupBy 클래스는 메모리에서 쿼리 결과에 대한 집합 연산을 수행하는 집합 함수를 제공한다. 다음은 사용 예이다.

부모 자식 관계에 대한 집합 연산
```
import static com.mysema.query.group.GroupBy.*;

Map<Integer, List<Comment>> results = query.from(post, comment)
    .where(comment.post.id.eq(post.id))
    .transform(groupBy(post.id).as(list(comment)));
```
이 코드는 post id와 관련된 커멘트를 매핑한다.

다중 결과 컬럼
```
Map<Integer, Group> results = query.from(post, comment)
    .where(comment.post.id.eq(post.id))
    .transform(groupBy(post.id).as(post.name, set(comment.id)));
```
이 코드는 post id와 Group을 매핑한다. Group은 post name과 comment id를 갖는다.

Group is the GroupBy equivalent to the Tuple interface.

더 많은 에제는 [여기](https://github.com/querydsl/querydsl/blob/master/querydsl-collections/src/test/java/com/mysema/query/collections/GroupByTest.java)에서 찾아볼 수 있다.


## 3.3 코드 생성

Querydsl은 JPA, JDO, Mongodb 모듈에서 코드 생성을 위해 자바6의 APT 어노테이션 처리 기능을 사용한다.
이 절에서는 코드 생성을 위한 다양한 설정 옵션과 APT에 대한 대안을 설명한다.

### 3.3.1 경로 초기화
기본적으로 Querydsl은 처음 2레벨의 레퍼런스 프로퍼티만 초기화한다. 더 깊은 경로로 초기화해야 한다면, com.mysema.query.annotations.QueryInit 어노테이션을 도메인 타입에 적용해야 한다. 더 깊은 레벨로 초기화가 필요한 프로퍼티에 QueryInit 어노테이션을 적용한다. 다음은 적용 예를 보여주고 있다.
```
@Entity
class Event {
    @QueryInit("customer.address")
    Account account;
}

@Entity
class Account{
    Customer customer;
}

@Entity
class Customer{
    String name;
    Address address;
    // ...
}
```
이 예제는 Event 경로가 루트 경로인 /로 초기화될 때, account.customer 경로의 초기화를 실행한다. 경로 초기화 포맷은 와일드카드 문자(customer.* 또는 그냥 * 등)를 지원한다.

자동 경로 초기화는 수동 초기화를 대신하며, 엔티티 필드가 final이어선 안 된다. 선언적 포맷은 쿼리 타입의 모든 최상위 레벨 인스턴스에 적용할 수 있고 final 엔티티 필드의 사용을 가능하게 해주는 이점이 있다.

선호하는 초기화 방식은 자동 경로 초기화지만, 다음에 설명할 Config 어노테이션을 이용해서 수동 초기화를 활성화시킬 수 있다.

### 3.3.2 커스터마이징
패키지나 타입에 Config 어노테이션을 사용해서 Querydsl의 직렬화를 커스터마이징할 수 있다. Querydsl은 어노테이션이 적용된 패키지와 타입의 직렬화 방식을 변경한다.

직렬화 옵션은 표 3.1과 같다.

표 3.1 Config 옵션

Name | Description
---|---
entityAccessors | public final 필드 대신 엔티티 경로로 사용할 접근 메서드 (기본값: false)
listAccessors | listProperty(int index) 형식의 메서드 (기본값: false)
mapAccessors | mapProperty(Key key) 형식의 접근 메서드 (기본값: false)
createDefaultVariable | 기본 변수 생성 (기본값: true)
defaultVariableName | 기본 변수의 이름

다음은 몇 가지 예이다.

엔티티 타입 직렬화 커스터마이징:
```
@Config(entityAccessors=true)
@Entity
public class User {
    //...
}
```
패키지 내용 커스터마이징:
```
@Config(listAccessors=true)
package com.mysema.query.domain.rel;

import com.mysema.query.annotations.Config;
```

만약 직렬화 설정을 글로벌하게 변경하고 싶다면, 다음의 APT 옵셥을 사용하면 된다.

표 3.2. APT 옵션

이름 | 설명
---|---
querydsl.entityAccessors | 레퍼런스 필드 접근
querydsl.listAccessors | 인덱스를 이용한 리스트 직접 접근
querydsl.mapAccessors | enable accessors for direct key based map access
querydsl.prefix | 쿼리 타입을 이용한 접두어 (기본값: Q)
querydsl.suffix | 쿼리 타입을 위한 접미사
querydsl.packageSuffix | 쿼리 타입 패키지를 위한 접미사
querydsl.createDefaultVariable | 기본 변수 만들지 여부
querydsl.unknownAsEmbeddable | 알 수 없는 어노테이션 비적용 클래스를 embeddable로 처리할지 여부 (기본값: false)
querydsl.includedPackages | 코드 생성에 포함될 패키지 목록 (콤마로 구분) (default: all)
querydsl.includedClasses | 코드 생성에 포함될 클래스 이름 목록 (콤마로 구분) (default: all)
querydsl.excludedPackages | 코드 생성에서 제외할 패키지 이름 (콤마로 구분) (default: none)
querydsl.excludedClasses | 코드 생성에서 제외할 클래스 이름 (콤마로 구분) (default: none)

다음은 메이븐 APT 플러그인 옵션 사용 예이다.
```
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.0.9</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.mysema.query.apt.jpa.JPAAnnotationProcessor</processor>
            <options>
              <querydsl.entityAccessors>true</querydsl.entityAccessors>
            </options>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

### 3.3.3 커스텀 타입 매핑
경로 타입을 바꾸고 싶으면 커스텀 타입 매핑을 사용하면 된다. 특정 String 경로에 대해 비교나 String 연산을 막아야 하거나 커스텀 타입을 위해 Date/Time 지원이 추가되어야 하는 경우에 커스텀 타입 매핑을 유용하게 사용할 수 있다. Joda time API와 JDK(java.util.Date, Calendar 그리고 하위 타입)의 시간 타입은 기본으로 지원하며, 다른 API가 필요할 경우 이 기능을 사용하면 된다.

다음은 예시이다.
```
@Entity
public class MyEntity{
    @QueryType(PropertyType.SIMPLE)
    public String stringAsSimple;

    @QueryType(PropertyType.COMPARABLE)
    public String stringAsComparable;

    @QueryType(PropertyType.NONE)
    public String stringNotInQuerydsl;
}
```
PropertyType.NONE은 쿼리 타입 생성시 프로퍼티를 생략할 때 사용된다. @Transient나 @QueryTransient 어노테이션이 적용된 프로퍼티가 영속 대상에서 빠지는 것과 차이가 난다. PropertyType.NONE은 단지 Querydsl 쿼리 타입에서 해당 프로퍼티를 제외한다.

### 3.3.4 위임 메서드(Delegate methods)

정적 메서드를 위임 메서드로 선언하려면, 해당하는 도메인 타입을 값으로 갖는 QueryDelegate 어노테이션을 정적 메서드에 적용하고,
정적 메서드의 첫 번째 파라미터로 해당하는 Querydsl 쿼리 타입을 제공한다.

다음은 예제이다.

```
@QueryEntity
public static class User{

    String name;

    User manager;

}
```

```
@QueryDelegate(User.class)
public static BooleanPath isManagedBy(QUser user, User other){
    return user.manager.eq(other);
}
```

QUser 쿼리 타입의 생성된 메서드는 다음과 같다.
```
public BooleanPath isManagedBy(QUser other) {
    return com.mysema.query.domain.DelegateTest.isManagedBy(this, other);
}
```

내장 타입을 확장하는데 위임 메서드를 사용할 수도 있다. 다음은 몇 가지 예제이다.

```
public class QueryExtensions {

    @QueryDelegate(Date.class)
    public static BooleanExpression inPeriod(DatePath<Date> date, Pair<Date,Date> period){
        return date.goe(period.getFirst()).and(date.loe(period.getSecond()));
    }

    @QueryDelegate(Timestamp.class)
    public static BooleanExpression inDatePeriod(DateTimePath<Timestamp> timestamp, Pair<Date,Date> period){
        Timestamp first = new Timestamp(DateUtils.truncate(period.getFirst(), Calendar.DAY_OF_MONTH).getTime());
        Calendar second = Calendar.getInstance();
        second.setTime(DateUtils.truncate(period.getSecond(), Calendar.DAY_OF_MONTH));
        second.add(1, Calendar.DAY_OF_MONTH);
        return timestamp.goe(first).and(timestamp.lt(new Timestamp(second.getTimeInMillis())));
    }

}
```
내장 타입을 위한 위임 메서드를 선언하면, 위임 메서드를 알맞게 사용하는 하위 클래스가 만들어진다.

```
public class QDate extends DatePath<java.sql.Date> {

    public QDate(BeanPath<? extends java.sql.Date> entity) {
        super(entity.getType(), entity.getMetadata());
    }

    public QDate(PathMetadata<?> metadata) {
        super(java.sql.Date.class, metadata);
    }

    public BooleanExpression inPeriod(com.mysema.commons.lang.Pair<java.sql.Date, java.sql.Date> period) {
        return QueryExtensions.inPeriod(this, period);
    }

}

public class QTimestamp extends DateTimePath<java.sql.Timestamp> {

    public QTimestamp(BeanPath<? extends java.sql.Timestamp> entity) {
        super(entity.getType(), entity.getMetadata());
    }

    public QTimestamp(PathMetadata<?> metadata) {
        super(java.sql.Timestamp.class, metadata);
    }

    public BooleanExpression inDatePeriod(com.mysema.commons.lang.Pair<java.sql.Date, java.sql.Date> period) {
        return QueryExtensions.inDatePeriod(this, period);
    }

}
```

### 3.3.5 어노테이션 비적용 타입
@QueryEntities 어노테이션을 만들면, 어노테이션이 적용되지 않은 타입에 대해서도 Querydsl 쿼리 타입을 생성하는 것이 가능하다. QueryEntities 어노테이션을 선택한 패키지에 넣고, value 속성에 복제할 클래스를 값으로 지정한다.

실제로 타입을 생성하려면 com.mysema.query.apt.QuerydslAnnotationProcessor를 사용한다. 메이븐 설정 방법은 다음과 같다.

```
<project>
  <build>
  <plugins>
    ...
    <plugin>
      <groupId>com.mysema.maven</groupId>
      <artifactId>apt-maven-plugin</artifactId>
      <version>1.0.9</version>
      <executions>
        <execution>
          <goals>
            <goal>process</goal>
          </goals>
          <configuration>
            <outputDirectory>target/generated-sources/java</outputDirectory>
            <processor>com.mysema.query.apt.QuerydslAnnotationProcessor</processor>
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```

### 3.3.6 클래스패스 기반 코드 생성
어노테이션이 적용된 자바 소스를 사용할 수 없는 경우(예를 들어, Scala나 Groovy와 같은 다른 JVM 언어를 사용했거나, 바이트코드 조작을 이용해서 어노테이션을 추가한 경우 등), GenericExporter 클래스를 사용해서 클래스패스에서 어노테이션이 적용된 클래스를 스캔하고 검색된 클래스를 위한 쿼리 타입을 생성할 수 있다.
GenericExporter를 사용하려면 querydsl-codegen 모듈을 의존에 추가해주어야 한다. (더 정확하게는 com.mysema.querydsl:querydsl-codegen:${querydsl.version} 모듈)

다음은 JPA를 위한 예제이다.
```
GenericExporter exporter = new GenericExporter();
exporter.setKeywords(Keywords.JPA);
exporter.setEntityAnnotation(Entity.class);
exporter.setEmbeddableAnnotation(Embeddable.class);
exporter.setEmbeddedAnnotation(Embedded.class);
exporter.setSupertypeAnnotation(MappedSuperclass.class);
exporter.setSkipAnnotation(Transient.class);
exporter.setTargetFolder(new File("target/generated-sources/java"));
exporter.export(DomainClass.class.getPackage());
```
이 코드는 DomainClass의 패키지 및 그 하위패키지에 위치한 모든 JPA 어노테이션 적용 클래스를 찾아 target/generated-sources/java 디렉토리에 쿼리 타입을 생성한다.

#### 3.3.6.1 메이븐 사요법

querydsl-maven-plugin의 generic-export, jpa-exportㅡjdo-export 골을 통해 GenericExporter를 사용할 수 있다.

각 골은 Querydsl, JPA, JDO 어노테이션에 매핑된다.

설정 엘리먼트는 다음과 같다.

표 3.3 메이븐 설정

타입 | 엘리먼트 | 설명
---|---|---
File | targetFolder | 생성된 소스가 위치할 대상 폴더
boolean | scala | Scala 소스를 생성하려면 true (기본값: false)
String[] | packages | 엔티티 클래스를 검색할 패키지
boolean | handleFields | 필드를 프로퍼티로 처리할 경우 true (기본값: true)
boolean | handleMethods | getter를 프로퍼티로 처리할 경우 true (기본값: true)
String | sourceEncoding | 생성할 소스 파일의 캐릭터 인코딩
boolean | testClasspath | 테스트 클래스패스를 사용하려면 true

다음은 JPA 어노테이션이 적용된 클래스를 위한 예이다.
```
<plugin>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-maven-plugin</artifactId>
  <version>${querydsl.version}</version>  
  <executions>
    <execution>    
      <phase>process-classes</phase>
      <goals>
        <goal>jpa-export</goal>
      </goals>
      <configuration>
        <targetFolder>target/generated-sources/java</targetFolder>
        <packages>
          <package>com.example.domain</package>
        </packages>
      </configuration>      
    </execution>
  </executions>
</plugin>
```
위 메이븐 설정은 com.example.domain 및 그 하위 패키지의 JPA 어노테이션 적용 클래스를 찾아 target/generated-sources/java 디렉토리에 코드를 생성한다.

생성 후에, 직접 생성된 소스를 컴파일하려면 그 소스 폴더를 위한 compile 골을 사용하면 된다.

```
<execution>
  <goals>
    <goal>compile</goal>
  </goals>
  <configuration>
    <sourceFolder>target/generated-sources/scala</targetFolder>
  </configuration>
</execution>
```

compile 골은 다음 설정 엘리먼트를 갖는다.

표 3.4. 메이븐 설정

타입 | 엘리먼트 | 설명
---|---|---
File | sourceFolder | 생성된 소스를 갖는 소스 폴더
String | sourceEncoding | 소스의 캐릭터 인코딩
String | source | 컴파일러의 -source 옵션
String | target | 컴파일러의 -target 옵션
boolean | testClasspath | 테스트 클래스패스를 사용할 경우 true
Map | compilerOptions | 컴파일러 옵션

sourceFolder를 제외한 모든 옵션은 선택사항이다.

#### 3.3.6.2 스카라 지원

스카라 출력을 원하면, 다음 설정을 사용하자.

```
<plugin>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-maven-plugin</artifactId>
  <version>${querydsl.version}</version>
  <dependencies>
    <dependency>
      <groupId>com.mysema.querydsl</groupId>
      <artifactId>querydsl-scala</artifactId>
      <version>${project.version}</version>
    </dependency>
    <dependency>
      <groupId>org.scala-lang</groupId>
      <artifactId>scala-library</artifactId>
      <version>${scala.version}</version>
    </dependency>
  </dependencies>
  <executions>
    <execution>
      <goals>
        <goal>jpa-export</goal>
      </goals>
      <configuration>
        <targetFolder>target/generated-sources/scala</targetFolder>
        <scala>true</scala>
        <packages>
          <package>com.example.domain</package>
        </packages>
      </configuration>
    </execution>
  </executions>
</plugin>
```

## 3.4 별칭 사용법

코드 생성을 할 수 없는 경우, 표현식 생성을 위한 경로 참조 목적으로 별칭 객체를 사용할 수 있다. 자바빈 객체의 프록시를 생성하고 프록시 객체의 getter 메서드를 경로로 사용할 수 있다.

다음 예제는 생성된 타입을 이용한 표현식 생성을 대신해서 어떻게 별칭 객체를 사용할 수 있는 보여준다.

먼저 다음 예제는 APT가 생성된 도메인 타입을 이용해서 쿼리를 실행하는 코드를 보자.
```
QCat cat = new QCat("cat");
for (String name : query.from(cat,cats)
    .where(cat.kittens.size().gt(0))
    .list(cat.name)){
    System.out.println(name);
}
```
그리고, 다음은 Cal 클래스를 이용해서 별칭 인스턴스를 사용하는 예이다. $ 메서드 파라미터인 c.getKittens()는 내부적으로 프로퍼티 경로 c.kittens로 변환된다.
```
Cat c = alias(Cat.class, "cat");
for (String name : query.from($(c),cats)
    .where($(c.getKittens()).size().gt(0))
    .list($(c.getName()))){
    System.out.println(name);
}
```

별칭 기능을 사용하려면 다음의 두 import를 추가해야 한다.

```
import static com.mysema.query.alias.Alias.$;
import static com.mysema.query.alias.Alias.alias;
```

다음 예는 앞 예제의 변형이다. $ 메서드 안에서 리스트의 size()에 접근하고 있다.
```
Cat c = alias(Cat.class, "cat");
for (String name : query.from($(c),cats)
    .where($(c.getKittens().size()).gt(0))
    .list($(c.getName()))){
    System.out.println(name);
}
```

기본 데이터 타입이 아니고 final이 아닌 모든 별칭 프로퍼티는 별칭 자체이다. 따라서, $ 메서드 내에서 기본 데이터 타입이나 final 타입이 나오기 전까지 메서드를 이어서 호출할 수 있다. 다음은 예이다.

```
$(c.getMate().getName())
```
이 코드는 내부적으로 c.mate.name으로 변환된다.

하지만, 아래 코드는 올바르게 변환되지 않는데, 그 이유는 toLowerCase() 호출이 추적되지 않기 때문이다.
```
$(c.getMate().getName().toLowerCase())
```

또한, 별칭 타입에서는 getters, size(), contains(Object), get(int)만 호출할 수 있음에 유의하자. 모든 다른 메서드 호출은 익셉션을 발생시킨다.


