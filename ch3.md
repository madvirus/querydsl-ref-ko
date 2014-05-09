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