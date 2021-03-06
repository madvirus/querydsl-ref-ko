# 2. 튜토리얼
일반적인 시작 안내 문서 대신 Querydsl이 지원하는 주요 백엔드 기술에 대한 안내 문서를 제공한다.
## 2.1 JPA 쿼리
Querydsl은 영속 도메인 모델에 대해 쿼리할 수 있는 범용적인 정적 타입 구문을 정의하고 있다. JDO와 JPA는 Querydsl이 지원하는 주요 기술이다. 이 안내 문서에서는 JPA와 함께 Querydsl을 사용하는 방법을 설명한다.

Querydsl은 JPQL과 Criteria 쿼리를 모두 대체할 수 있다. Querydsl은 Criteria 쿼리의 동적인 특징과 JPQL의 표현력을 타입에 안전한 방법으로 제공한다.

### 2.1.1 메이븐 통합

메이븐 프로젝트의 의존 설정에 다음을 추가한다.

```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-jpa</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```
다음으로 메이븐 APT 플러그인을 설정한다.
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
          </configuration>
        </execution>
      </executions>
    </plugin>
    ...
  </plugins>
  </build>
</project>
```
JPAAnnotationProcessor는 javax.persistence.Entity 어노테이션을 가진 도메인 타입을 찾아서 쿼리 타입을 생성한다.

도메인 타입으로 Hibernate 어노테이션을 사용하면, APT 프로세서로  com.mysema.query.apt.hibernate.HibernateAnnotationProcessor를 사용해야 한다.

`mvn clean install`을 실행하면, target/generated-sources/java 디렉토리에 Query 타입이 생성된다.

이클립스를 사용할 경우, `mvn eclipse:eclipse`을 실행하면 target/generated-sources/java 디렉토리가 소스 폴더에 추가된다.

생성된 Query 타입을 이용하면 JPQ 쿼리 인스턴스와 쿼리 도메인 모델 인스턴스를 생성할 수 있다.

### 2.1.2 Ant 통합

클래스패스에 full-deps에 포함된 jar 파일들을 위치시키고, 다음 태스크를 이용해서 Querydsl 코드를 생성한다.

```
    <!-- APT based code generation -->
    <javac srcdir="${src}" classpathref="cp">
      <compilerarg value="-proc:only"/>
      <compilerarg value="-processor"/>
      <compilerarg value="com.mysema.query.apt.jpa.JPAAnnotationProcessor"/>
      <compilerarg value="-s"/>
      <compilerarg value="${generated}"/>
    </javac>

    <!-- compilation -->
    <javac classpathref="cp" destdir="${build}">
      <src path="${src}"/>
      <src path="${generated}"/>
    </javac>
```

src를 메인 소스 폴더로 변경하고, generated를 생성된 소스를 위한 폴더로 변경하고, build를 클래스 생성 폴더로 변경한다.

### 2.1.3 Roo에서 Querydsl JPA 사용하기

스프링 Roo에서 Querydsl JPA를 사용한다면, com.mysema.query.apt.jpa.JPAAnnotationProcessor 대신 com.mysema.query.apt.roo.RooAnnotationProcessor를 사용할 수 있다. RooAnnotationProcessor는 @Entity가 적용된 클래스 대신 @RooJpaEntity와 @RooJpaActiveRecord 어노테이션이 적용된 클래스를 처리한다.

### 2.1.4 hbm.xml 파일에서 쿼리 모델 생성하기

하이버네이트에서 XML 기반 설정을 사용하고 있다면, Querydsl 모델을 생성하기 위해 XML 메타정보를 사용할 수 있다.

com.mysema.query.jpa.codegen.HibernateDomainExporter가 이 기능을 제공한다.

```
HibernateDomainExporter exporter = new HibernateDomainExporter(
  "Q",                     // name prefix
  new File("target/gen3"), // target folder
  configuration);          // instance of org.hibernate.cfg.Configuration

exporter.export();
```
HibernateDomainExporter는 리플렉션을 이용해서 도메인의 프로퍼티 타입을 확인하기 때문에, HibernateDomainExporter를 실행하려면 클래스패스에 도메인 타입이 위치해야 한다.

모든 JPA 어노테이션은 무시되지만, @QueryInit이나 @QueryType과 같은 Querydsl 어노테이션은 처리한다.

### 2.1.5 쿼라 타입 사용하기

Querydsl을 이용해서 쿼리를 작성하려면, 변수와 Query 구현체를 생성해야 한다. 먼저 변수부터 시작해보자.

다음과 같은 도메인 타입이 있다고 가정하다.

```
@Entity
public class Customer {
    private String firstName;
    private String lastName;

    public String getFirstName(){
        return firstName;
    }

    public String getLastName(){
        return lastName;
    }

    public void setFirstName(String fn){
        firstName = fn;
    }

    public void setLastName(String ln)[
        lastName = ln;
    }
}
```

Querydsl은 Customer와 동일한 패키지에 QCustomer라는 이름을 가진 쿼리 타입을 생성한다. Querydsl 쿼리에서 Customer 타입을 위한 정적 타입 변수로 QCustomer를 사용한다.

QCustomer는 기본 인스턴스 변수를 갖고 있으며, 다음과 같이 정적 필드로 접근할 수 있다.

```
QCustomer customer = QCustomer.customer;
```

다음처럼 Customer 변수를 직접 정의할 수도 있다.

```
QCustomer customer = new QCustomer("myCustomer");
```

### 2.1.6 쿼리

Querdsl JPA 모듈은 JPA와 Hibernate API를 모두 지원한다.

JPA API를 사용하려면 다음과 같이 JPAQuery 인스턴스를 사용하면 된다.

```
// where entityManager is a JPA EntityManager
JPAQuery query = new JPAQuery(entityManager);
```

Hibernate를 사용한다면, HibernateQuery를 사용하면 된다.

```
// where session is a Hibernate session
HibernateQuery query = new HibernateQuery(session);
```

JPAQuery와 HibernateQuery는 둘 다 JPQLQuery 인터페이스를 구현하고 있다.

firstName 프로퍼티가 Bob인 Customer를 조회하고 싶다면 다음의 쿼리를 사용하면 된다.

```
QCustomer customer = QCustomer.customer;
JPAQuery query = new JPAQuery(entityManager);
Customer bob = query.from(customer)
  .where(customer.firstName.eq("Bob"))
  .uniqueResult(customer);
```

from 메서드는 쿼리 대상(소스)을 지정하고, where 부분은 필터를 정의하고, uniqueResult는 프로젝션을 정의하고, 1개 결과만 리턴하라고 지시한다.

여러 소스로부터 쿼리를 만들고 싶다면 다음처럼 쿼리를 사용한다.

```
QCustomer customer = QCustomer.customer;
QCompany company = QCompany.company;
query.from(customer, company);
```

여러 필터를 사용하는 방법은 다음과 같다.
```
query.from(customer)
    .where(customer.firstName.eq("Bob"), customer.lastName.eq("Wilson"));
```
또는, 다음과 같이 해도 된다.
```
query.from(customer)
    .where(customer.firstName.eq("Bob").and(customer.lastName.eq("Wilson")));
```
위 코드를 JPQL 쿼리로 작성하면 다음과 같을 것이다.
```
from Customer as customer
    where customer.firstName = "Bob" and customer.lastName = "Wilson"
```
필터 조건을 or로 조합하고 싶다면 다음 패턴을 사용한다.
```
query.from(customer)
    .where(customer.firstName.eq("Bob").or(customer.lastName.eq("Wilson")));
```

### 2.1.7 조인

Querydsl은 JPQL의 이너 조인, 조인, 레프트 조인, 풀조인을 지원한다. 조인 역시 타입에 안전하며 다음 패턴에 따라 작성한다.

```
QCat cat = QCat.cat;
QCat mate = new QCat("mate");
QCate kitten = new QCat("kitten");
query.from(cat)
    .innerJoin(cat.mate, mate)
    .leftJoin(cat.kittens, kitten)
    .list(cat);
```
위 쿼리를 JPQL로 작성하면 다음과 같다.
```
from Cat as cat
    inner join cat.mate as mate
    left outer join cat.kittens as kitten
```
조인을 사용하는 또 다른 예다.
```
query.from(cat)
    .leftJoin(cat.kittens, kitten)
    .on(kitten.bodyWeight.gt(10.0))
    .list(cat);
```
위 코드의 JPQL 버전은 다음과 같다.
```
from Cat as cat
    left join cat.kittens as kitten
    on kitten.bodyWeight > 10.0
```

### 2.1.8 일반 용법

JPQLQuery 인터페이스의 cascading 메서드는 다음과 같다.

*from*: 쿼리 소스를 추가한다.

*innerJoin, join, leftJoin, fullJoin, on*: 조인 부분을 추가한다. 조인 메서드에서 첫 번째 인자는 조인 소스이고, 두 번재 인자는 대상(별칭)이다.

*where*: 쿼리 필터를 추가한다. 가변인자나 and/or 메서드를 이용해서 필터를 추가한다.

*groupBy*: 가변인자 형식의 인자를 기준으로 그룹을 추가한다.

*having*: Predicate 표현식을 이용해서 "group by" 그룹핑의 필터를 추가한다.

*orderBy*: 정렬 표현식을 이용해서 정렬 순서를 지정한다. 숫자나 문자열에 대해서는 asc()나 desc()를 사용하고, OrderSpecifier에 접근하기 위해 다른 비교 표현식을 사용한다.

*limit, offset, restrict*: 결과의 페이징을 설정한다. limit은 최대 결과 개수, offset은 결과의 시작 행, restrict는 limit과 offset을 함께 정의한다.

### 2.1.9 정렬

정렬을 위한 구문은 다음과 같다.

```
QCustomer customer = QCustomer.customer;
query.from(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.desc())
    .list(customer);
```

위 코드는 다음의 JPQL과 동일하다.

```
from Customer as customer
    order by customer.lastName asc, customer.firstName desc
```

### 2.1.10 그룹핑

그룹핑은 다음과 같은 코드로 처리한다.

```
query.from(customer)
    .groupBy(customer.lastName)
    .list(customer.lastName);
```

동등한 JPQL은 다음고 같다.
```
select customer.lastName
    from Customer as customer
    group by customer.lastName
```

### 2.1.11 DeleteClause

Querydsl JPA에서 DeleteClause는 간단한 delete-where-execute 형태를 취한다. 다음은 몇 가지 예다.

```
QCustomer customer = QCustomer.customer;
// delete all customers
new JPADeleteClause(entityManager, customer).execute();
// delete all customers with a level less than 3
new JPAeDeleteClause(entityManager, customer).where(customer.level.lt(3)).execute();
```
JPADeleteClause 생성자의 두 번째 파라미터는 삭제할 엔티티 대상이다. where는 필요에 따라 추가할 수 있으며, execute를 실행하면 삭제를 수행하고 삭제된 엔티티의 개수를 리턴한다.

Hibernate 이용시, HibernateDeleteClause를 사용하면 된다.

JPA의 DML 절은 JPA 레벨의 영속성 전파 규칙을 따르지 않고, 2차 레벨 캐시와의 연동되지 않는다.

### 2.1.12 UpdateClause

Querydsl JPA의 UpdateClause은 간단한 update-set/where-execute 형태를 취한다. 다음은 몇 가지 예다.

```
QCustomer customer = QCustomer.customer;
// rename customers named Bob to Bobby
new JPAUpdateClause(session, customer).where(customer.name.eq("Bob"))
    .set(customer.name, "Bobby")
    .execute();
```

JPAUpdateClause 생성자의 두 번째 파라미터는 수정할 엔티티 대상이다. set은 SQL의 update 스타일로 프로퍼티 수정을 정의하고, execute를 실행하면 수정을 실행하고 수정된 엔티티의 개수를 리턴한다.

Hibernate 이용시, HibernateUpdateClause를 사용한다.

JPA에서 DML 절은 JPA 레벨의 영속성 전파 규칙을 따르지 않고, 2차 레벨 캐시와 연동되지 않는다.

### 2.1.13 서브쿼리

서브쿼리를 만들려면 JPASubQuery를 사용하면 된다. 서브쿼리를 만들기 위해 from 메서드로 쿼리 파라미터를 정의하고, unique나 list를 이용한다. unique는 단일 결과를 위해 사용하고 list는 리스트 결과를 위해 사용한다. 서브쿼리도 쿼리처럼 타입에 안전한 Querydsl 표현식이다.

```
QDepartment department = QDepartment.department;
QDepartment d = new QDepartment("d");
query.from(department)
    .where(department.employees.size().eq(
        new JPASubQuery().from(d).unique(d.employees.size().max())
     )).list(department);
```

다른 예제

```
QEmployee employee = QEmployee.employee;
QEmployee e = new QEmployee("e");
query.from(employee)
    .where(employee.weeklyhours.gt(
        new JPASubQuery().from(employee.department.employees, e)
        .where(e.manager.eq(employee.manager))
        .unique(e.weeklyhours.avg())
    )).list(employee);
```

Hibernate를 사용할 경우, HibernateSubQuery를 사용하면 된다.

### 2.1.14 원래의 JPA Query 구하기

만약 쿼리를 실행하기 전에 JPA Query를 구하고 싶다면, 다음 코드를 사용한다.

```
JPAQuery query = new JPAQuery(entityManager);
Query jpaQuery = query.from(employee).createQuery(employee);
// ...
List results = jpaQuery.getResultList();
```

### 2.1.15 JPA 쿼리에서 네이티브 SQL 사용하기

JPASQLQuery 클래스를 사용하면 JPA의 네이티브 SQL을 Querydsl에서 사용할 수 있다.

이걸 사용하려면 SQL 스키마를 위한 Querydsl 쿼라 티입을 생성해야 한다. 다음은 이를 위한 Maven 설정 예를 보여주고 있다.

```
<plugin>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-maven-plugin</artifactId>
  <version>${project.version}</version>
  <executions>
    <execution>
      <goals>
        <goal>export</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <jdbcDriver>org.apache.derby.jdbc.EmbeddedDriver</jdbcDriver>
    <jdbcUrl>jdbc:derby:target/demoDB;create=true</jdbcUrl>
    <packageName>com.mycompany.mydomain</packageName>
    <targetFolder>${project.basedir}/target/generated-sources/java</targetFolder> 
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.apache.derby</groupId>
      <artifactId>derby</artifactId>
      <version>${derby.version}</version>
    </dependency>
  </dependencies>
</plugin>
```

지정한 위치에 쿼리 타입을 성공적으로 생성했다면, 쿼리에서 그 타입을 사용할 수 있다.

한 개 컬럼 쿼리:
```
// serialization templates
SQLTemplates templates = new DerbyTemplates();
// query types (S* for SQL, Q* for domain types)
SAnimal cat = new SAnimal("cat");
SAnimal mate = new SAnimal("mate");
QCat catEntity = QCat.cat;

JPASQLQuery query = new JPASQLQuery(entityManager, templates);
List<String> names = query.from(cat).list(cat.name);
```

한 쿼리에서 엔티티(예, QCat)와 테이블(예, SAnimal)에 대한 참조를 섞어 쓰고 싶다면, 같은 변수명을 갖도록 해야 한다. SAnimal.animal은 "animal"이란 변수명을 가지므로 새로운 인스턴스 (new SAnimal("cat"))을 대신 사용했다.

다음과 같이 할 수도 있다.

```
QCat catEntity = QCat.cat;
SAnimal cat = new SAnimal(catEntity.getMetadata().getName());
```

다중 컬럼 쿼리:
```
query = new JPASQLQuery(entityManager, templates);
List<Object[]> rows = query.from(cat).list(cat.id, cat.name);
```

모든 컬럼 쿼리:
```
List<Object[]> rows = query.from(cat).list(cat.all());
```

SQL로 쿼리를 하고, 결과는 엔티티로 구하기:
```
query = new JPASQLQuery(entityManager, templates);
List<Cat> cats = query.from(cat).orderBy(cat.name.asc()).list(catEntity);
```

조인을 이용한 쿼리:
```
query = new JPASQLQuery(entityManager, templates);
cats = query.from(cat)
    .innerJoin(mate).on(cat.mateId.eq(mate.id))
    .where(cat.dtype.eq("Cat"), mate.dtype.eq("Cat"))
    .list(catEntity);
```

쿼리 결과를 DTO로 구하기:
```
query = new JPASQLQuery(entityManager, templates);
List<CatDTO> catDTOs = query.from(cat)
    .orderBy(cat.name.asc())
    .list(ConstructorExpression.create(CatDTO.class, cat.id, cat.name));
```

## 2.2 JDO 쿼리

**[생략함]**

## 2.3 SQL 쿼리

본 절에서는 SQL 모듈의 쿼라 타입 생성과 쿼리 기능을 설명한다.

### 2.3.1 메이븐 통합

메이븐 프로젝트에 다음의 의존을 추가한다.

```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-sql</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-sql-codegen</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```

코드 생성을 메이븐이나 Ant에서 할 경우 querydsl-sql-codegen 의존은 생략할 수 있다.

### 2.3.2 메이븐을 통한 코드 생성

코드 생성은 주로 메이븐 플러그인을 통해서 수행한다. 다음은 설정 예다.

```
<plugin>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-maven-plugin</artifactId>
  <version>${querydsl.version}</version>
  <executions>
    <execution>
      <goals>
        <goal>export</goal>
      </goals>
    </execution>
  </executions>
  <configuration>
    <jdbcDriver>org.apache.derby.jdbc.EmbeddedDriver</jdbcDriver>
    <jdbcUrl>jdbc:derby:target/demoDB;create=true</jdbcUrl>
    <packageName>com.myproject.domain</packageName>
    <targetFolder>${project.basedir}/target/generated-sources/java</targetFolder>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>org.apache.derby</groupId>
      <artifactId>derby</artifactId>
      <version>${derby.version}</version>
    </dependency>
  </dependencies>
</plugin>
```

컴파일 소스 루트 대신에 테스트 컴파일 소스 루트로 targetFolder를 추가하려면 test-export 골을 사용하면 된다.

표 2.1 파라미터

| 이름 | 설명 |
|---|---|
| jdbcDriver | JDBC 드라이버 클래스 이름 |
| jdbcUrl | JDBC URL |
| jdbcUser | DB 사용자 |
| jdbcPassword | DB 사용자 암호 |
| namePrefix | 생성될 쿼리 클래스의 접두어 (기본: Q) |
| nameSuffix | 생성될 쿼리 클래스의 접미사 (기본: ) |
| beanPrefix | 생성될 빈Bean 클래스의 접두어 |
| beanSuffix | 생성될 빈 클래스의 접미사 |
| packageName | 생성될 소스 파일이 위치할 패키지 |
| beanPackageName | 빈 파일이 생성될 패키지 이름 (기본: packageName) |
| beanInterface | 빈 클래스에 추가할 인터페이스 목록 (기본: 없음) |
| beanAddToString | true로 지정하면 기본 toString() 구현을 생성 (기본: false) |
| beanAddFullConstructor | true로 지정하면 기본 생성자 외에 완전한 생성자를 생성 (기본: false) |
|beanPrintSupertype | true로 지정하면 상위 타입을 출력 (기본: false) |
|schemaPattern | 스키마 이름 패턴. 반드시 데이터베이스에 존재하는 스키마 이름과 일치해야 한다. (기본: null) |
| tableNamePattern | 테이블 이름 패턴. 반드시 데이터베이스에 존재하는 테이블 이름과 일치해야 하며, 콤마로 구분해서 두 개 이상 지정할 수 있다. (기본: null) |
| targetFolder | 소스 파일을 생성할 폴더를 지정 |
| namingStrategyClass | NamingStrategy로 사용할 클래스 이름을 입력 (기본: DefaultNamingStrategy) |
| beanSerializerClass | BeanSerializer로 사용할 클래스 이름 (기본: BeanSerializer) |
| serializerClass | Serializer로 사용할 클래스 이름 (기본: MetaDataSerializer) |
| exportBeans | true로 지정하면 빈을 함께 생성. 2.14.13 참고. (기본: false) |
| innerClassesForKeys | true로 지정하면 키를 내부 클래스로 생성 (기본: false) |
| validationAnnotations | true로 지정하면 Validation 어노테이션의 직렬화를 가능하게 함 (기본: false) |
| columnAnnotation | true로 지정하면 컬럼 어노테이션을 추출함 (기본: false) |
| createScalaSources | true로 지정하면 자바 소스 대신 Scala 소스로 추찰함 (기본: false) |
| schemaToPackage | true로 지정하면 스키마 이름을 패키지에 붙임 (기본: false) |
| lowerCase | true로 지정하면 이름을 소문자로 변환 (기본: false) |
| exportTables | true로 지정하면 테이블을 추출 (기본: true) |
| exportViews | true로 지정한 뷰를 추출 (기본: true) |
| exportPrimaryKeys | true로 지정하면 PK를 추출 (기본: true) |
| exportForeignKeys | true로 지정하면 외부키를 추출 (기본: true) |
| customTypes | 커스텀 사용자 타입 (기본: 없음) |
| typeMappings | 테이블.컬럼에서 자바 타입으로 매핑 (기본: 없음) |
| numericMapping | 크기/숫자에서 자바 타입으로 매핑 (기본: 없음) |
| imports | 생성된 쿼리 클래스에 추가할 자바 import 목록: 패키지의 경우 (.* 없이) 패키지 이름만(예, com.bar), 클래스의 경우 완전한 클래스 이름 (예, com.bar.Foo) 사용. (기본: 없음) |

추가로 타입 구현을 등록하고 싶을 때 customTypes를 사용한다.

```
<customTypes>
  <customType>com.mysema.query.sql.types.InputStreamType</customType>
</customTypes>
```

테이블.컬럼을 위한 자바 타입을 등록하고 싶을 때 typeMappings를 사용한다.

```
<typeMappings>
  <typeMapping>
    <table>IMAGE</table>
    <column>CONTENTS</column>
    <type>java.io.InputStream</type>
  </typeMapping>
</typeMappings>
```

숫자 매핑을 위한 기본 타입은 다음과 같다.

표 2.2 숫자 매핑

| 크기 | 자리(digits) | 타입 |
|---|---|---|
| > 18 | 0 | BigInteger |
| > 9 | 0 | Long |
| > 4 | 0 | Integer |
| > 2 | 0 | Short |
| > 0 | 0 | Byte |
| > 16 | > 0 | BigDecimal |
| > 0 | > 0 | Double |

특정 크기/자리에 대한 커스텀 타입은 다음과 같이 설정한다.

```
<numericMappings>
  <numericMapping>
    <size>1</size>
    <digits>0</digits>
    <javaType>java.lang.Byte</javaType>
  </numericMapping>
</numericMappings>
```

Imports can be used to add cross-schema foreign keys support.

### 2.3.3 ANT를 통한 코드 생성

Querydsl-sql 모듈이 제공하는 com.mysema.query.sql.ant.AntMetaDataExporter ANT 태스크는 ANT 태스크??와 같은 기능을 제공한다. 태스크의 설정 파라미터느는 메이븐 플러그인과 동일하다.

### 2.3.4 쿼리 타입 만들기

DB 스키마를 Querydsl의 쿼리 타입으로 만들려면 다음과 같이 하면 된다.

```
java.sql.Connection conn = ...;
MetaDataExporter exporter = new MetaDataExporter();
exporter.setPackageName("com.myproject.mydomain");
exporter.setTargetFolder(new File("target/generated-sources/java"));
exporter.export(conn.getMetaData());
```

위 코드를 실행하면 데이터베이스 스키마로부터 생성한 쿼리 타입 소스 코드(com.myproject.mydomain 패키지에 속함)를 target/generated-sources/java 디렉토리에 만든다.

생성된 타입의 클래스 이름은 변형된 테이블 이름이 되며, 쿼라 티입 프로퍼티 경로의 이름을 변형된 컬럼 이름이 된다.

추가로, 간략한 조인 설정을 위해 PK와 FK를 위한 필드가 추가된다.

### 2.3.5 설정

com.mysema.query.sql.Configuration 클래스를 이용해서 설정하며, Configuration 클래스는 생성자 인자로 Querydsl SQL Dialect를 취한다. 예를 들어, H2 DB 사용시 다음과 같이 생성한다.

```
SQLTemplates templates = new H2Templates();
Configuration configuration = new Configuration(templates);
```

Querydsl은 서로 다른 RDBMS를 위한 SQL 직렬화를 커스터마이징하기 위해 SQL Dialect를 사용한다. 사용가능한 Dialect는 다음과 같다.

- CUBRIDTemplates (tested with 8.4)
- DerbyTemplates (tested with 10.8.2.2)
- HSQLDBTemplates (tested with 2.2.4)
- H2Templates (tested with 1.3.164)
- MySQLTemplates (tested with MySQL 5.5)
- OracleTemplates (test with Oracle 10 and 11)
- PostgresTemplates (tested with 9.1)
- SQLiteTemplates (tested with xerial JDBC 3.7.2)
- SQLServerTemplates (tested with SQL Server)
- SQLServer2005Templates (for SQL Server 2005)
- SQLServer2008Templates (for SQL Server 2008)
- SQLServer2012Templates (for SQL Server 2012 and later)
- TeradataTemplates (tested with Teradata 14)

SQLTemplate 객체의 설정을 변경하려면 다음과 같이 빌더 패턴을 사용할 수 있다.

```
H2Templates.builder()
     .printSchema() // to include the schema in the output
     .quote()       // to quote names
     .newLineToSingleSpace() // to replace new lines with single space in the output
     .escape(ch)    // to set the escape char
     .build();      // to get the customized SQLTemplates instance
```

Configuration 클래스를 이용하면 setUseLiterals(true)를 통한 리터럴의 직접 직렬화 활성, 스키마와 테이블 재정의, 커스텀 타입을 등록할 수 있다. 완전한 내용은 javadoc의 Configuration를 참고한다.

### 2.3.6 쿼리

Querydsl SQL을 이용해서 쿼리하는 방법은 다음처럼 간단하다.

```
QCustomer customer = new QCustomer("c");

SQLQuery query = new SQLQuery(connection, configuration); 
List<String> lastNames = query.from(customer)
    .where(customer.firstName.eq("Bob"))
    .list(customer.lastName);
```
위 코드는 다음의 SQL로 변환되어 실행된다. (테이블 이름은 customer, 컬럼 이름은 first_name, last_name이라고 가정)
```
SELECT c.last_name
FROM customer c
WHERE c.first_name = 'Bob'
```

### 2.3.7 일반 용법

SQLQuery 클래스의 cascading 메서드는 다음과 같다.

*from*: 쿼리 소스를 추가한다.

*innerJoin, join, leftJoin, fullJoin, on*: 조인 부분을 추가한다. 조인 메서드에서 첫 번째 인자는 조인 소스이고, 두 번재 인자는 대상(별칭)이다.

*where*: 쿼리 필터를 추가한다. 가변인자나 and/or 메서드를 이용해서 필터를 추가한다.

*groupBy*: 가변인자 형식의 인자를 기준으로 그룹을 추가한다.

*having*: Predicate 표현식을 이용해서 "group by" 그룹핑의 필터를 추가한다.

*orderBy*: 정렬 표현식을 이용해서 정렬 순서를 지정한다. 숫자나 문자열에 대해서는 asc()나 desc()를 사용하고, OrderSpecifier에 접근하기 위해 다른 비교 표현식을 사용한다.

*limit, offset, restrict*: 결과의 페이징을 설정한다. limit은 최대 결과 개수, offset은 결과의 시작 행, restrict는 limit과 offset을 함께 정의한다.

### 2.3.8 조인
다음 구문을 이용해서 조인을 한다.
```
QCustomer customer = QCustomer.customer;
QCompany company = QCompany.company;
query.from(customer)
    .innerJoin(customer.company, company)
    .list(customer.firstName, customer.lastName, company.name);
```
레프트 조인은 다음과 같다.
```
query.from(customer)
    .leftJoin(customer.company, company)
    .list(customer.firstName, customer.lastName, company.name);
```
조인 조건을 쓸 수도 있다.
```
query.from(customer)
    .leftJoin(company).on(customer.company.eq(company.id))
    .list(customer.firstName, customer.lastName, company.name);
```

### 2.3.9 정렬
다음은 정렬 구문이다.
```
query.from(customer)
    .orderBy(customer.lastName.asc(), customer.firstName.asc())
    .list(customer.firstName, customer.lastName);
```
위 코드 아래 SQL 쿼리와 동등하다.
```
SELECT c.first_name, c.last_name
FROM customer c
ORDER BY c.last_name ASC, c.first_name ASC
```

### 2.3.10 그룹핑
다음 형식을 이용해서 그룹핑을 한다.
```
query.from(customer)
    .groupBy(customer.lastName)
    .list(customer.lastName);
```
다음은 위 코드에 해당하는 SQL 쿼리다.
```
SELECT c.last_name
FROM customer c
GROUP BY c.last_name
```

### 2.3.11 서브쿼리
서브쿼리를 만들려면 SQLSubQuery를 사용하면 된다. 서브쿼리를 만들려면 JPASubQuery를 사용하면 된다. 서브쿼리를 만들기 위해 from 메서드로 쿼리 파라미터를 정의하고, unique나 list를 이용한다. unique는 단일 결과를 위해 사용하고 list는 리스트 결과를 위해 사용한다. 서브쿼리도 쿼리처럼 타입에 안전한 Querydsl 표현식이다.

```
QCustomer customer = QCustomer.customer;
QCustomer customer2 = new QCustomer("customer2");
query.from(customer).where(
  customer.status.eq(new SQLSubQuery().from(customer2).unique(customer2.status.max()))
  .list(customer.all())
```

다른 예제
```
QStatus status = QStatus.status;
query.from(customer).where(
  customer.status.in(new SQLSubQuery().from(status).where(status.level.lt(3)).list(status.id))
  .list(customer.all())
```

### 2.3.12 리터럴 조회
리터럴을 조회하려면, 다음과 같이 constant 인스턴스를 생성해주면 된다.

```
query.list(Expressions.constant(1),
           Expressions.constant("abc"));
```
com.mysema.query.support.Expressions 클래스는 프로젝션, 오퍼레이션, 템플릿 생성을 위한 유용한 정적 메서드도 제공한다.

### 2.3.13 쿼리 확장 지원
엔진에 특화된 구문을 사용하려면 커스텀 쿼리 확장을 사용한다. 커스텀 쿼리 확장은 AbstractSQLQuery를 상속받아 구현할 수 있다. 다음은 MySQLQuery 클래스에서 플래그를 추가하는 예를 보여주고 있다.
```
public class MySQLQuery extends AbstractSQLQuery<MySQLQuery> {

    public MySQLQuery(Connection conn) {
        this(conn, new MySQLTemplates(), new DefaultQueryMetadata());
    }

    public MySQLQuery(Connection conn, SQLTemplates templates) {
        this(conn, templates, new DefaultQueryMetadata());
    }

    protected MySQLQuery(Connection conn, SQLTemplates templates, QueryMetadata metadata) {
        super(conn, new Configuration(templates), metadata);
    }

    public MySQLQuery bigResult(){
        return addFlag(Position.AFTER_SELECT, "SQL_BIG_RESULT ");
    }

    public MySQLQuery bufferResult(){
        return addFlag(Position.AFTER_SELECT, "SQL_BUFFER_RESULT ");
    }


    // ...
}
```
플래그는 직렬화 과정에서 특정 위치에 삽입될 수 있는 커스텀 SQL 부분 코드다.
com.mysema.query.QueryFlag.Position 열거 타입에 지원되는 위치가 정의되어 있다.

### 2.3.14 윈도우 함수
Querydsl은 SQLExpressions 클래스를 통해서 윈도우 함수를 지원한다.

다음은 사용 예다.

```
query.from(employee)
    .list(SQLExpressions.rowNumber()
        .over()
        .partitionBy(employee.name)
        .orderBy(employee.id));
```

### 2.3.15 다른 SQL 표현식
SQLExpressions 클래스의 정적 메서드를 이용해서 다른 SQL 표현식을 사용할 수 있다.

### 2.3.16 DML 명령 사용하기
Querydsl SQL 모듈의 모든 DMLClause 구현체는 Connection, 쿼리에 사용될 SQLTemplate, DMLClause와 엮일 메인 엔티티의 세 개 파라미터를 필요로 한다.

#### 2.3.16.1 삽입
컬럼 지정
```
QSurvey survey = QSurvey.survey;

new SQLInsertClause(conn, configuration, survey)
    .columns(survey.id, survey.name)
    .values(3, "Hello").execute();
```

컬럼 없이
```
new SQLInsertClause(conn, configuration, survey)
    .values(4, "Hello").execute();
```

서브쿼리 이용
```
new SQLInsertClause(conn, configuration, survey)
    .columns(survey.id, survey.name)
    .select(new SQLSubQuery().from(survey2).list(survey2.id.add(1), survey2.name))
    .execute();
```

서브쿼리 이용, 컬럼 없이
```
new SQLInsertClause(conn, configuration, survey)
    .select(new SQLSubQuery().from(survey2).list(survey2.id.add(10), survey2.name))
    .execute();
```
columns/values 메서드 사용하지 않고, set 메서드를 이용
```
QSurvey survey = QSurvey.survey;

new SQLInsertClause(conn, configuration, survey)
    .set(survey.id, 3)
    .set(survey.name, "Hello").execute();
```
위 코드는 첫 번째 예제와 동일하다. set 메서드를 사용하면 내부적으로 columns/values가 사용된다.

아래 형식의 코드에서는 컬럼과 쿼리 결과 집합을 매핑하는 것에 주의하자.

```
columns(...).select(...)
```

변경된 행 개수 대신 생성된 키를 구하고 싶다면 executeWithKey/s 메서드를 사용한다.

```
set(...)
```

위 코드는 한 개 컬럼을 매핑한다. 서브 쿼리 결과가 없으면 null을 사용한다.

빈의 데이터에 기반해서 clause 인스턴스를 생성하려면 다음의 코드를 사용한다.

```
new SQLInsertClause(conn, configuration, survey)
    .populate(surveyBean).execute();
```
위 코드는 빈의 데이터 중 null은 제외한다. null도 포함시키고 싶다면 아래 코드를 사용한다.
```
new SQLInsertClause(conn, configuration, survey)
    .populate(surveyBean, DefaultMapper.WITH_NULL_BINDINGS).execute();
```

#### 2.3.16.2 수정
where 절 포함
```
QSurvey survey = QSurvey.survey;

new SQLUpdateClause(conn, configuration, survey)
    .where(survey.name.eq("XXX"))
    .set(survey.name, "S")
    .execute();
```
where 없이
```
new SQLUpdateClause(conn, configuration, survey)
    .set(survey.name, "S")
    .execute();
```
빈을 이용
```
new SQLUpdateClause(conn, configuration, survey)
    .populate(surveyBean)
    .execute();
```

### 2.3.16.3 삭제
where 절 포함
```
QSurvey survey = QSurvey.survey;

new SQLDelecteClause(conn, configuration, survey)
    .where(survey.name.eq("XXX"))
    .execute();
```
where 없이
```
new SQLDelecteClause(conn, configuration, survey)
    .execute()
```

### 2.3.17 DMLClause의 배치 지원
Querydsl SQL은 DML API를 통해서 JDBC 배치 업데이터를 지원한다. 같은 구조를 갖는 DML을 연속해서 실행할 경우, addBatch() 메서드를 이용해서 한 DMLClause로 묶을 수 있다.

수정:
```
QSurvey survey = QSurvey.survey;

insert(survey).values(2, "A").execute();
insert(survey).values(3, "B").execute();

SQLUpdateClause update = update(survey);
update.set(survey.name, "AA").where(survey.name.eq("A")).addBatch();
update.set(survey.name, "BB").where(survey.name.eq("B")).addBatch();
```

삭제:
```
insert(survey).values(2, "A").execute();
insert(survey).values(3, "B").execute();

SQLDeleteClause delete = delete(survey);
delete.where(survey.name.eq("A")).addBatch();
delete.where(survey.name.eq("B")).addBatch();
assertEquals(2, delete.execute());
```

삽입:
```
SQLInsertClause insert = insert(survey);
insert.set(survey.id, 5).set(survey.name, "5").addBatch();
insert.set(survey.id, 6).set(survey.name, "6").addBatch();
assertEquals(2, insert.execute());
```

### 2.3.18 빈 클래스 생성
MetaDataExporter를 이용해서 테이블에 대한 자바빈 DTO 타입을 생성한다.
```
java.sql.Connection conn = ...;
MetaDataExporter exporter = new MetaDataExporter();
exporter.setPackageName("com.myproject.mydomain");
exporter.setTargetFolder(new File("src/main/java"))	;
exporter.setBeanSerializer(new BeanSerializer());
exporter.export(conn.getMetaData());
```
DMLClause의 populate 메서드의 인자로 빈 타입을 사용할 수 있으며, 쿼리에서 빈 타입을 직접 선택할 수 있다. 다음은 JUnit으로 작성한 간단한 예이다.

```
QEmployee e = new QEmployee("e");

// Insert
Employee employee = new Employee();
employee.setFirstname("John");
Integer id = insert(e).populate(employee).executeWithKey(e.id);
employee.setId(id);

// Update
employee.setLastname("Smith");
assertEquals(1l, update(e).populate(employee).where(e.id.eq(employee.getId())).execute());

// Query
Employee smith = query().from(e).where(e.lastname.eq("Smith")).uniqueResult(e);
assertEquals("John", smith.getFirstname());

// Delete
assertEquals(1l, delete(e).where(e.id.eq(employee.getId())).execute());
```
앞서 예제에서 사용한 팩토리 메서드는 다음과 같다.

```
protected SQLUpdateClause update(RelationalPath<?> e){
    return new SQLUpdateClause(Connections.getConnection(), templates, e);
}

protected SQLInsertClause insert(RelationalPath<?> e){
    return new SQLInsertClause(Connections.getConnection(), templates, e);
}

protected SQLDeleteClause delete(RelationalPath<?> e){
    return new SQLDeleteClause(Connections.getConnection(), templates, e);
}

protected SQLMergeClause merge(RelationalPath<?> e){
    return new SQLMergeClause(Connections.getConnection(), templates, e);
}

protected SQLQuery query() {
    return new SQLQuery(Connections.getConnection(), templates);
}
```

### 2.3.19 SQL 쿼리와 바인딩 추출하기

getSQL 메서드를 통해서 SQL 쿼리와 바인딩 값을 구할 수 있다.

```
SQLBindings bindings = query.getSQL(customer.id, customer.firstname, customer.lastname);
System.out.println(bindings.getSQL());
```
SQL 문자열에 포함된 모든 리터럴이 필요하다면, setUseLiterals(true)를 이용해서 쿼리의 리터럴 직렬화를 활성화하면 된다.

### 2.3.20 커스텀 타입

Querydsl SQL은 ResultSet/Statement에서 커스텀 타입 매핑을 지원한다. com.mysema.query.sql.Configuration 객체를 이용해서 커스텀 카입 매핑을 등록한다. Configuration 객체는 실제 쿼리의 생성자 인자로 제공된다.

```
Configuration configuration = new Configuration(new H2Templates());
// overrides the mapping for Types.DATE
configuration.register(new UtilDateType());
```

특정 테이블 컬럼을 위한 커스텀 타입 매핑 등록
```
Configuration configuration = new Configuration(new H2Templates());
// declares a maping for the gender column in the person table
configuration.register("person", "gender",  new EnumByNameType<Gender>(Gender.class));
```
숫자에 대한 커스텀 타입 매핑을 등록하려면 registerNumeric 메서드를 사용한다.

```
configuration.registerNumeric(5,2,Float.class);
```
이는 Float 타입을 NUMERIC(5,2) 타입으로 매핑한다.

### 2.3.21 Query와 Clause 리스닝

SQLListener는 쿼리와 DMLClause를 리스닝 할 때 사용되는 리스너 인터페이스이다. Configuration이나 Query, Clause의 addListener 메서드를 통해서 SQLListener 객체를 등록할 수 있다.

리스너의 적용 예로는 데이터 동기화, 로깅, 캐싱, 검증이 있다.

## 2.4 루신 쿼리

이 절에서는 루신 모듈의 쿼리 기능을 설명한다.

### 2.4.1 메이븐 통합

Querydsl 루신을 사용하려면 루신3은 querydsl-lucene3 모듈을 그리고 루신4는 querydsl-lucene4 모듈을 사용한다.

루신 3:
```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-lucene3</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```
루신 4:
```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-lucene4</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```
### 2.4.2 쿼리 타입 생성
다음과 같은 방식으로 year와 title 필드를 가진 쿼리 타입을 직접 작성할 수 있다.

```
public class QDocument extends EntityPathBase<Document>{
    private static final long serialVersionUID = -4872833626508344081L;

    public QDocument(String var) {
        super(Document.class, PathMetadataFactory.forVariable(var));
    }

    public final StringPath year = createString("year");

    public final StringPath title = createString("title");
}
```
QDocument는 year와 title 필드를 가진 루신 Document를 표현한다.

루신의 경우 스키마 데이터를 사용할 수 없기 때문에 코드 생성 기능을 사용할 수 없다.

### 2.4.3 쿼리

Querydsl 루신으로 쿼리하는 것은 간단하다.

```
QDocument doc = new QDocument("doc");

IndexSearcher searcher = new IndexSearcher(index);
LuceneQuery query = new LuceneQuery(true, searcher);
List<Document> documents = query
    .where(doc.year.between("1800", "2000").and(doc.title.startsWith("Huckle"))
    .list();
```

위 코드는 다음의 루신 쿼리로 바뀐다.

```
+year:[1800 TO 2000] +title:huckle*
```

### 2.4.4 일반 용법

LuceneQuery 클래스의 cascading 메서드는 다음과 같다.

*where*: 쿼리 필터를 추가한다. 가변인자나 and/or 메서드를 이용해서 필터를 추가한다. PStrings에 수행되는 오퍼레이션을 지원한다. (matches, indexOf, charAt은 제외). 현재 in은 지원되지 않으며, 향후 지원할 예정이다.

*orderBy*: 정렬 표현식을 이용해서 정렬 순서를 지정한다. 숫자나 문자열에 대해서는 asc()나 desc()를 사용하고, OrderSpecifier에 접근하기 위해 다른 비교 표현식을 사용한다.

*limit, offset, restrict*: 결과의 페이징을 설정한다. limit은 최대 결과 개수, offset은 결과의 시작 행, restrict는 limit과 offset을 함께 정의한다.

### 2.4.5 정렬

정렬 구문은 다음과 같다.
```
query
    .where(doc.title.like("*"))
    .orderBy(doc.title.asc(), doc.year.desc())
    .list();
```
위 코드는 다음 루신 쿼리와 동일하다.

```
title:*
```
title과 year의 오름차순으로 결과를 정렬한다.

sort 메서드와 Sort 인스턴스를 사용해서 정렬을 지정할 수 있다.

```
Sort sort = ...; 
query
    .where(doc.title.like("*"))
    .sort(sort)
    .list();
```

### 2.4.6 결과 개수 제한

결과 개수 제한은 다음과 같이 한다.

```
query
    .where(doc.title.like("*"))
    .limit(10)
    .list();
```

### 2.4.7 오프셋

오프셋은 다음과 같이 지정한다.
```
query
    .where(doc.title.like("*"))
    .offset(3)
    .list();
```

### 2.4.8 퍼지(fuzzy) 검색
com.mysema.query.lucene.LuceneExpressions 클래스에서 정의된 fuzzyLike 메서드를 이용해서 퍼지 검색을 할 수 있다.

```
query
    .where(LuceneExpressions.fuzzyLike(doc.title, "Hello"))
    .list();
```

### 2.4.9 루신 필터를 쿼리에 적용하기

단일 루신 필터를 쿼리에 적용할 수 있다.

```
query
    .where(doc.title.like("*"))
    .filter(filter)
    .list();
```
distinct 필터링을 위한 distinct(Path) 메서드를 제공한다.

```
query
    .where(doc.title.like("*"))
    .distinct(doc.title)
    .list();
```

## 2.5 Hibernate Search 쿼리
Hibernate Search 모듈의 쿼리 기능을 설명한다.

### 2.5.1 Querydsl 쿼리 타입 생성

쿼리 타입을 생성하는 방법은 본 문서의 JPA 쿼리 부분을 참고한다.

### 2.5.2 쿼리

Querydsl Hibernate Search를 이용한 쿼리는 다음과 같이 간단하다.

```
QUser user = QUser.user;
SearchQuery<User> query = new SearchQuery<User>(session, user);
List<User> list = query
	.where(user.firstName.eq("Bob"))
	.list();
```

### 2.5.3 일반 용법

일반 용법은 Querying Lucene의 일반 용법을 참고한다.

쿼리 직렬화 과정에서 Querydsl Lucene module과의 유일한 차이점은 경로를 다르게 처리한다는 것이다. org.hibernate.search.annotations.Field 어노테이션이 적용된 프로퍼티인 경우, 필드 이름에 대한 대체 방법으로 name 속성의 값을 사용한다.

### 2.6 Mongodb 쿼리

Mongodb 모듈의 조회 기능을 설명한다.

### 2.6.1 메이븐 통합

메이븐 프로젝트에 다음 의존을 추가한다.

```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-apt</artifactId>
  <version>${querydsl.version}</version>
  <scope>provided</scope>
</dependency>

<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-mongodb</artifactId>
  <version>${querydsl.version}</version>
</dependency>

<dependency>
  <groupId>org.slf4j</groupId>
  <artifactId>slf4j-log4j12</artifactId>
  <version>1.6.1</version>
</dependency>
```
Querydsl을 이용해서 쿼리 타입을 생성하기 위해 메이븐 APT 플러그인을 설정한다.
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
              <processor>com.mysema.query.apt.morphia.MorphiaAnnotationProcessor</processor>
            </configuration>
          </execution>
        </executions>
      </plugin>
    ...
    </plugins>
  </build>
</project>
```
MorphiaAnnotationProcessor는 com.google.code.morphia.annotations.Entity 어노테이션이 적용된 도메인 타입을 위한 Querydsl 쿼리 타입을 생성한다.

`mvn clean install`을 실행하면 target/generated-sources/java 폴더에 쿼라 타입 코드가 생성된다.

이클립스를 사용중이면, `mvn eclipse:eclipse'을 실행해서 target/generated-sources/java을 이클립스 프로젝트의 소스 폴더에 포함시킨다.

이제 쿼리 도메인 모델을 이용해서 Mongodb를 조회할 수 있다.

### 2.6.2 쿼리

Querydsl Mongodb를 이용하면 다음과 같이 간단하게 쿼리할 수 있다.
```
Morphia morphia;
Datastore datastore;
// ...
QUser user = new QUser("user");
MorphiaQuery<User> query = new MorphiaQuery<User>(morphia, datastore, user);
List<User> list = query
	.where(user.firstName.eq("Bob"))
	.list();
```

### 2.6.3 일반 용법

MongodbQuery 클래스의 cascading 메서드는 다음과 같다.

ere*: 쿼리 필터를 추가한다. 가변인자나 and/or 메서드를 이용해서 필터를 추가한다. PStrings에 수행되는 오퍼레이션을 지원한다. (matches, indexOf, charAt은 제외). 현재 in은 지원되지 않으며, 향후 지원할 예정이다.

*orderBy*: 정렬 표현식을 이용해서 정렬 순서를 지정한다. 숫자나 문자열에 대해서는 asc()나 desc()를 사용하고, OrderSpecifier에 접근하기 위해 다른 비교 표현식을 사용한다.

*limit, offset, restrict*: 결과의 페이징을 설정한다. limit은 최대 결과 개수, offset은 결과의 시작 행, restrict는 limit과 offset을 함께 정의한다.


### 2.6.4 정렬

정렬 구문은 다음과 같다.

```
query
    .where(doc.title.like("*"))
    .orderBy(doc.title.asc(), doc.year.desc())
    .list();
```
title과 year의 오름차순으로 결과를 정렬한다.

### 2.6.5 결과 개수 제한

다음과 같이 결과 개수를 제한한다.

```
query
    .where(doc.title.like("*"))
    .limit(10)
    .list();
```

### 2.6.6 오프셋
다음과 같이 오프셋을 지정한다.
```
query
    .where(doc.title.like("*"))
    .offset(3)
    .list();
```

### 2.6.7 공간 쿼리
near(Douyble[]) 메서드를 이용해서 공간 검색을 할 수 있다.
```
query
    .where(geoEntity.location.near(50.0, 50.0))
    .list();
```

### 2.6.8 관련 필드만 선택하기

관련 필드만 선택하고 싶다면, 선택 대상 목록을 갖는 list, iterate, uniqueResult, singleResult 메서드를 사용하면 된다.
```
query
    .where(doc.title.like("*"))
    .list(doc.title, doc.path);
```
이 쿼리는 문서의 title과 path 필드만 조회한다.

## 2.7 콜렉션 쿼리
**[생략함]**

## 2.8 Scala에서의 쿼리
querydsl-scala 모듈을 사용하면 Scala에서 Querydsl을 사용할 수 있다. 이를 사용하려면 메이븐에 다음의 의존을 추가해주면 된다.
```
<dependency>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-scala</artifactId>
  <version>${querydsl.version}</version>
</dependency>
```

### 2.8.1 Scala를 위한 DSL 표현식
Scala 용 Querydsl은 표현식 생성을 위한 다른 DSL을 제공한다. Scala용 DSL은 연산자 오버로딩, 함수 포인터, 임의 임포트 등의 언어 기능을 활용해서 가독성과 간결함을 높였다.

다음은 Scala 용 DSL의 개요를 정리한 것이다.

```
//표준                   Scala 대체

expr isNotNull          expr is not(null)
expr isNull             expr is null
expr eq "Ben"           expr === "Ben"
expr ne "Ben"           expr !== "Ben"
expr append "X"         expr + "X"
expr isEmpty            expr is empty
expr isNotEmpoty        expr not empty

// boolean
left and right          left && right
left or right           left || right
expr not                !expr

// comparison
expr lt 5               expr < 5
expr loe 5              expr <= 5
expr gt 5               expr > 5
expr goe 5              expr >= 5
expr notBetween(2,6)    expr not between (2,6)
expr negate             -expr
 
// numeric
expr add 3              expr + 3
expr subtract 3         expr - 3
expr divide 3           expr / 3
expr multiply 3         expr * 3
expr mod 5              expr % 5

// collection
list.get(0)             list(0)
map.get("X")            map("X")
```

### 2.8.2 향상된 프로젝션

Querydsl Scala 모듈은 Querydsl 쿼리 프로젝션을 좀더 Scala에 맞게 만들기 위해 임의 변환(Implicit conversion)을 제공한다.

Querydsl 쿼리에서 Scala 프로젝션을 쓰려면 RichProjectable와 RichSimpleProjectable 래퍼를 사용해야 한다. com.mysema.query.scala.Helpers의 컨텐트를 임포트하면 필요한 임의 변환이 사용 가능해진다.

예를 들어, 다음 쿼리는 표준 API를 사용하고 있으며 Object[] 타입의 java.util.List를 리턴한다.

```
query.from(person).list(person.firstName, person.lastName, person.age)
```

추가된 컨버전을 사용하면, ㅣist 대신에 Scala list 타입 결과를 리턴하는 select를 사용하거나 uniqueResult/singleResult 대신에 Option 타입 결과를 제공하는 unique/single을 사용할 수 있다.

앞서 쿼리를 임의 컨버전을 사용하면 다음과 같이 표현할 수 있다.
```
import com.mysema.query.scala.Helpers._

query.from(person).select(person.firstName, person.lastName, person.age)
```
이 경우 결과 타입은 List[(String,String,Integer)] 즉, Tuple3[String,String,Integer]의 List가 된다.

### 2.8.3 SQL을 이용한 쿼리
자바 용 Querydsl SQL처럼 쿼리를 생성할 때 사용할 Query 타입을 생성해야 한다. 다음은 생성 방법을 보여주는 예제 코드이다.

빈 타입 없이 생성:

```
val directory = new java.io.File("target/jdbcgen1")
val namingStrategy = new DefaultNamingStrategy()
val exporter = new MetaDataExporter()
exporter.setNamePrefix("Q")
exporter.setPackageName("com.mysema")
exporter.setSchemaPattern("PUBLIC")
exporter.setTargetFolder(directory)
exporter.setSerializerClass(classOf[ScalaMetaDataSerializer])
exporter.setCreateScalaSources(true)
exporter.setTypeMappings(ScalaTypeMappings.create)
exporter.export(connection.getMetaData)
```

빈 타입을 이용한 생성:
```
val directory = new java.io.File("target/jdbcgen2")
val namingStrategy = new DefaultNamingStrategy()
val exporter = new MetaDataExporter()
exporter.setNamePrefix("Q")
exporter.setPackageName("com.mysema")
exporter.setSchemaPattern("PUBLIC")
exporter.setTargetFolder(directory)
exporter.setSerializerClass(classOf[ScalaMetaDataSerializer])
exporter.setBeanSerializerClass(classOf[ScalaBeanSerializer])
exporter.setCreateScalaSources(true)
exporter.setTypeMappings(ScalaTypeMappings.create)
exporter.export(connection.getMetaData)
```

#### 2.8.3.1 간편 쿼리

Querydsl Scala는 Querydsl SQL을 위한 간편 쿼리 구문을 제공한다. 이 구문은 Rogue 프레임워크와 같은 도메인 지향 쿼리 구문에서 영감을 얻었다.

임의 변환을 이용해서 RelationalPath 인스턴스를 쿼리로 변환하는 방법으로 도메인 지향 쿼리를 구현했다.
서비스나 DAO 클래스가 com.mysema.query.scala.sql.SQLHelpers 트레잇을 구현해서 이 기능을 사용할 수 있다.

이 간편 구문을 사용하려면, 쿼리를 위한 시작점으로 테마 모델 클래스를 사용할 수 있다.

다음과 같은 일반 구문을 사용하는 대신

```
query().from(employee).select(employee.firstName, employee.lastName)
```

Employee나 QEmployee의 컴패니언 객체를 사용해서 다음과 같이 작성할 수 있다.

```
Employee.select(_.firstName, _.lastName)
```

표현식에 orderBy, where, select, single 그리고 unique를 제공하는 대신, 쿼리의 루트 표현식을 파라미터로 받고 다른 표현식을 리턴하는 함수를 제공할 수 있다. 다음은 위 코드를 확장한 것이다.

```
Employee.select({ e => e.firstName }, { e => e.lastName })
```
자세한 내용은 com.mysema.query.scala.sql.RichSimpleQuery 클래스 문서를 참고하자.

#### 2.8.3.2 코드 생성

querydsl-maven-plugin 플러그인을 사용해서 SQL 메타타입과 프로젝션을 위한 Scala 소스를 생성할 수 있다. 다음은 설정 예시이다.

```
<plugin>
  <groupId>com.mysema.querydsl</groupId>
  <artifactId>querydsl-maven-plugin</artifactId>
  <version>${querydsl.version}</version>
  <configuration>
    <jdbcDriver>com.mysql.jdbc.Driver</jdbcDriver>
    <jdbcUrl>jdbc:mysql://localhost:3306/test</jdbcUrl>
    <jdbcUser>matko</jdbcUser>
    <jdbcPassword>matko</jdbcPassword>
    <packageName>com.example.schema</packageName>
    <targetFolder>${project.basedir}/src/main/scala</targetFolder>
    <exportBeans>true</exportBeans>
    <createScalaSources>true</createScalaSources>
  </configuration>
  <dependencies>
    <dependency>
      <groupId>mysql</groupId>
	  <artifactId>mysql-connector-java</artifactId>
      <version>5.1.16</version>
	</dependency>
	  <dependency>
    	<groupId>com.mysema.querydsl</groupId>
    	<artifactId>querydsl-scala</artifactId>
    	<version>${querydsl.version}</version>
  	  </dependency>
  	  <dependency>
    	<groupId>org.scala-lang</groupId>
    	<artifactId>scala-library</artifactId>
    	<version>${scala.version}</version>
  	  </dependency>
  </dependencies>
</plugin>
```
querydsl:export 골을 실행하면 된다.

### 2.8.4 다른 백엔드를 위한 쿼리
다른 백엔드에 대해 쿼리를 할 때, Expresssion 모델을 수동으로 만들거나 별칭 기능을 사용해야 한다.

JPA를 이용한 예제:

```
@Entity
class User {
  @BeanProperty
  @Id
  var id: Integer = _;
  @BeanProperty
  var userName: String = _;
  @BeanProperty
  @ManyToOne
  var department: Department = _;
}

@Entity
class Department {
  @BeanProperty
  @Id
  var id: Integer = _;
  @BeanProperty
  var name: String = _;
}
```

다음은 몇 가지 쿼리 예제이다.

List
```
val person = Person as "person"

query.from(person).where(person.firstName like "Rob%").list(person)
```

고유 결과(Unique result)
```
query.from(person).where(person.firstName like "Rob%").unique(person)
```

Long where
```
query.from(person)
  .where(person.firstName like "Rob%", person.lastName like "An%")
  .list(person)
```

정렬
```
query.from(person).orderBy(person.firstName asc).list(person)
```

Not null
```
query.from(person) 
  .where(person.firstName isEmpty, person.lastName isNotNull) 
  .list(person)
```
위 코드에서 쿼리 생성을 위한 팩토리 메서드는 아래와 같다.
```
def query() = new JPAQuery(entityManager)
```
위 코드에서 다음과 같은 변수도 필요하다.
```
val person = Person as "person"
```
주의: XML 설정을 이용해서 Hibernate를 사용할 경우 Scala 지원을 사용할 수 없다. HibernateDomainExporter는 현재 자바 소스 파일에 대해서만 동작한다.
