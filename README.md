# jpa-querydsl

# Study

# JPA

- 설정 (JPA + QueryDSL)
    
    Gradle
    
    ```groovy
    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
      
        // Querydsl
        implementation 'com.querydsl:querydsl-jpa'
        // Querydsl JPAAnnotationProcessor 사용 지정
        annotationProcessor "com.querydsl:querydsl-apt:${dependencyManagement.importedProperties['querydsl.version']}:jpa"
        // java.lang.NoClassDefFoundError(javax.annotation.Entity) 발생 대응
        annotationProcessor "jakarta.persistence:jakarta.persistence-api"
        // java.lang.NoClassDefFoundError(javax.annotation.Generated) 발생 대응
        annotationProcessor "jakarta.annotation:jakarta.annotation-api"
    }
    
    // clean task 실행시 QClass 삭제
    clean {
        delete file('src/main/generated')
    }
    ```
    
    Yaml
    
    ```yaml
    spring:
      datasource:
        url: 
        username: 
        password:
        driver-class-name:
    
      jpa:
        hibernate:
          ddl-auto: validate
        properties:
          hibernate:
          # show_sql: true  # System.out.print() SQL 로깅
            format_sql: true
    
    logging.level:
      org.hibernate.SQL: debug  # log.debug() SQL 로깅 
      org.hibernate.type: trace # ? Binding parameter의 값을 확인 할 수 있다.
                                # p6spy-spring-boot-starter를 추가하면 더 보기 편하다
    ```
    
    Java
    
    ```java
    package com.example.jpa.common.db.config;
    
    import com.example.jpa.common.db.base.SqlSessionTemplateDatabaseFactory;
    import com.zaxxer.hikari.HikariConfig;
    import com.zaxxer.hikari.HikariDataSource;
    import lombok.RequiredArgsConstructor;
    import lombok.extern.slf4j.Slf4j;
    import org.apache.ibatis.session.SqlSessionFactory;
    import org.mybatis.spring.SqlSessionFactoryBean;
    import org.mybatis.spring.SqlSessionTemplate;
    import org.springframework.beans.factory.annotation.Autowired;
    import org.springframework.beans.factory.annotation.Qualifier;
    import org.springframework.beans.factory.config.ServiceLocatorFactoryBean;
    import org.springframework.boot.context.properties.ConfigurationProperties;
    import org.springframework.context.ApplicationContext;
    import org.springframework.context.annotation.*;
    import org.springframework.core.env.Environment;
    import org.springframework.dao.annotation.PersistenceExceptionTranslationPostProcessor;
    import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
    import org.springframework.orm.jpa.JpaTransactionManager;
    import org.springframework.orm.jpa.LocalContainerEntityManagerFactoryBean;
    import org.springframework.orm.jpa.vendor.HibernateJpaVendorAdapter;
    import org.springframework.transaction.PlatformTransactionManager;
    import org.springframework.transaction.annotation.EnableTransactionManagement;
    
    import javax.sql.DataSource;
    import java.util.Properties;
    
    @Slf4j
    @Configuration
    @RequiredArgsConstructor
    @EnableTransactionManagement // Transation Manager를 활성화하기 위한 애노테이션
    @EnableJpaRepositories( // JPA Repository들을 활성화하기 위한 애노테이션
    	entityManagerFactoryRef = "masterEntityManager"
    	transactionManagerRef = "masterTransactionManager",
    	basePackages = "com.example.jpa.*.repository.command"
    )
    @PropertySource("classpath:/application-${spring.profiles.active}.yml")
    public class MasterDbConfiguration {
    
    	private final ApplicationContext applicationContext;
    	private final Environment env;
    
    	@Bean(name = "masterDbConfig")
    	@ConfigurationProperties(prefix = "spring.datasource")
    	public HikariConfig masterDbConfig() {
    		return new HikariConfig();
    	}
    
    	@Primary
    	@Bean(name = "masterDataSource", destroyMethod = "close")
    	public HikariDataSource masterDataSource() {
    		HikariDataSource dataSource = new HikariDataSource(masterDbConfig());
    		return dataSource;
    	}
    
    	@Primary
    	@Bean(name = "masterSqlSessionFactory")
    	public SqlSessionFactory masterSqlSessionFactory(@Autowired @Qualifier("masterDataSource") DataSource dataSource) throws Exception {
    		SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
    		sqlSessionFactoryBean.setDataSource(dataSource);
    		sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:/sqlmap/**/*.xml"));
    		return sqlSessionFactoryBean.getObject();
    	}
    
    	@Primary
    	@Bean(name = "masterSqlSessionTemplate", destroyMethod = "clearCache")
    	public SqlSessionTemplate masterSqlSessionTemplate(@Autowired @Qualifier("masterSqlSessionFactory") SqlSessionFactory sqlSessionFactory) {
    		return new SqlSessionTemplate(sqlSessionFactory);
    	}
    
    	@Primary
    	@Bean
    	public LocalContainerEntityManagerFactoryBean masterEntityManager() {
    		LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
    		em.setDataSource(masterDataSource());
    		em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
    		em.setJpaProperties(additionalJpaProperties());
    		em.setPackagesToScan(new String[] {"com.example.jpa.*.domain"});
    		return em;
    	}
    
    	@Primary
    	@Bean
    	public PlatformTransactionManager masterTransactionManager() {
    		JpaTransactionManager transactionManager = new JpaTransactionManager();
    		transactionManager.setEntityManagerFactory(masterEntityManager().getObject());
    		return transactionManager;
    	}
    
    	@Primary
    	@Bean
    	public ServiceLocatorFactoryBean serviceLocatorFactoryBean() {
    		ServiceLocatorFactoryBean serviceLocatorFactoryBean = new ServiceLocatorFactoryBean();
    		serviceLocatorFactoryBean.setServiceLocatorInterface(SqlSessionTemplateDatabaseFactory.class);
    		return serviceLocatorFactoryBean;
    	}
    
    	@Primary
    	@Bean
    	public PersistenceExceptionTranslationPostProcessor masterExceptionTranslation(){
    		return new PersistenceExceptionTranslationPostProcessor();
    	}
    
    	Properties additionalJpaProperties() {
    		Properties properties = new Properties();
    		properties.setProperty("hibernate.dialect", env.getProperty("spring.jpa.hibernate.dialect"));
    		properties.setProperty("hibernate.hbm2ddl.auto", env.getProperty("spring.jpa.hibernate.ddl-auto"));
    		properties.setProperty("hibernate.default_batch_fetch_size", env.getProperty("spring.jpa.hibernate.default_batch_fetch_size"));
    		properties.setProperty("hibernate.format_sql", env.getProperty("spring.jpa.hibernate.format_sql"));
    
    		return properties;
    	}
    }
    ```
    

## ORM(Object-Relational Mapping)

객체와 관계형 데이터베이스를 매핑하는 방법

## 영속성 컨텍스트(Persistence Context)

어플리케이션과 데이터베이스 사이에 존재하는 논리적인 개념으로 엔티티를 저장하는 환경을 의미한다.

최초로 엔티티의 상태를 저장해 두는데 이것을 스냅샷이라고 한다. 

플러시 시점에 스냅샷과 엔티티를 비교해서 인서트, 업데이트 쿼리를 날린다.

- 1차 캐시
- 동일성 보장 (REPEATABLE READ 등급의 트랙잭션 격리 수준)
- 쓰기 지연
- 변경 감지 (Dirty Checking)
- 지연 로딩

엔티티의 생명주기

- 비영속 (new/transient)
    
    영속성 컨텍스트와 전혀 관계가 없는 새로운 상태
    
- 영속 (managed)
    
    영속성 컨텍스트에 관리되는 상태
    
- 준영속 (detached)
    
    영속성 컨텍스트에 저장되었다가 분리된 상태
    
- 삭제 (removed)
    
    삭제된 상태
    

## 엔티티 매핑 (Entity Mapping)

- 객체와 테이블 : @Entity, @Table
- 필드와 컬럼 : @Column
- 기본 키 : @Id
- 연관 관계 : @JoinColumn, @ManyToOne, @OneToMany, @OneToOne

@Entity

> @Entity가 붙은 클래스는 JPA가 관리

**기본 생성자 필수**(public or protected)
enum, interface, final Class, inner Class 사용 X
저장할 필드에 final 사용 X
> 

@Column

| Attribute | Description | Default Value |
| --- | --- | --- |
| name | 필드와 매핑할 테이블의 컬럼 이름 |  |
| insertable | 등록 가능 여부 | true |
| updateable | 변경 가능 여부 | true |
| nullable(DDL) | null 값의 허용 여부 | true |
| unique(DDL) | unique 제약 조건 | false |
| columnDefinition(DDL) | 데이터 베이스 컬럼 정보를 직접 정의<br/> ex) varchar(100) default ‘EMPTY’ |  |
| length(DDL) | 문자 길이 제약 조건, String에만 사용 | 255 |
| precision(DDL)<br/>scale(DDL) | 소수점을 포함한 전체 자릿수 설정<br/>소수의 자릿수 설정 | 0<br/>0 |

@Enumerated

| Attribute | Description | Default |
| --- | --- | --- |
| ORDINAL | Enum Type의 순서를 데이터베이스에 저장 (java.lang.Enum#ordinal) | ✅ |
| STRING | Enum Type의 이름을 데이터베이스에 저장 (java.lang.Enum#name) |  |

@Temporal

> LocalDate, LocalDateTime 타입 사용시에는 생략 가능 (최신 하이버네이트 지원)
> 

@Lob

> 매핑하는 필드 타입이 문자면 CLOB, 그 외에는 BLOB

CLOB : String, char[], java.sql.CLOB
BLOB : byte[], java.sql.BLOB
> 

@Transient

> 영속성 컨텍스트에서 관리하지 않는 속성을 지정
> 

## 기본키 매핑

@Id (직접 할당)

@GeneratedValue (자동 생성)

| Strategy | Description |
| --- | --- |
| IDENTITY | 데이터베이스에 위임, MYSQL |
| SEQUENCE | 데이터베이스 시퀀스 오브젝트 사용, ORACLE<br/>@SequenceGenerator 필요 |
| TABLE | 키 생성용 테이블 사용, 모든 DB에서 사용<br/>@TableGenerator 필요 |
| AUTO | 방언에 따라 자동 지정, 기본값 |

## 연관관계 매핑

외래키의 위치를 기준으로 연관관계 주인을 설정 (주인이 아닌경우 읽기만 가능)

> ⚠️ 단방향 매핑만으로도 이미 연관관계 매핑은 완료
양방향 매핑은 반대방향으로 조회 기능이 추가된 것일 뿐으로 단방향 매핑을 잘 하고 양방향은 필요할때 추가
> 

## 영속성 전이 (CASCADE)

특정 엔티티를 영속 상태로 만들 때 연관된 엔티티도 함께 영속상태로 만든다.

| Cascade 				| Description							  |
| -------------------	| --------------------------------------- |
| CascadeType.PERSIST	| 부모를 영속화할 때 연관된 자식들도 함께 영속화 한다. |
| CascadeType.MERGE 	| 트랜잭션이 종료되고 detach 상태에서 연관 엔티티를 추가하거나 변경된 이후에 부모 엔티티가 merge()를 수행하게 되면 변경사항이 적용된다. <br/>연관 엔티티의 추가 및 수정 모두 반영됨 |
| CascadeType.REMOVE	| 삭제 시 연관된 엔티티도 같이 삭제됨<br/>orphanRemoval 옵션과의 차이점은 연관된 엔티티 중 하나가 삭제되는 경우이고 REMOVE 옵션은 부모 엔티티가 삭제된 경우 연관된 엔티티인 자식 엔티티까지 전부 삭제한다는 옵션이다. collections에서 제거되더라도 해당 로우를 삭제한다. 			|
| CascadeType.DETACH	| 부모 엔티티가 detach()를 수행하게 되면, 연관된 엔티티도 detach() 상태가 되어 변경사항이 반영되지 않는다. |
| CascadeType.ALL 		| 모든 Cascade 적용 						|
| CascadeType.REFRESH 	| 엔티티를 새로 고칠 때, 이 필드에 보유 된 엔티티도 새로 고친다. |

```java
@OneToMany(mappedBy="parent", cascade=CascadeType.PERSIST)
private List<Child> childList = new ArrayList<>();
```

⚠️ 단일 소유자 일때만 사용, 다른 연관관계가 있을경우는 사용 X

## 고아 객체

참조 객체와 연관관계가 끊어지면 삭제

```java
@OneToMany(mappedBy="parent", cascade=CascadeType.PERSIST, orphanRemoval = true)
private List<Child> childList = new ArrayList<>();
```

cascade + orphanRemoval = 부모 객체와의 연관관계에 따라 자식 객체의 라이프사이클을 관리

## ****Jpa Mysql column 자료형****

| JAVA | DB |
| --- | --- |
| Byte | tinyint(4) |
| Short | smallint(6) |
| Integer | int(11) |
| Long | bigint(20) |
| BigDecimal | decimal(19,2) |
| Float | float |
| Double | double |
| Boolean | bit(1) |
| Date<br/>LocalDate | date |
| Timestamp<br/>LocalDateTime | datetime |
| Time | time |
| String | varchar(255) |
| Clob | longtext |
| Byte[] | tinyblob |
| Blob | longblob |

## 변경 감지(dirty checking)와 병합(Merge)

준영속 엔티티

영속성 컨텍스트(Persistence Context)가 더 이상 관리하지 않는 Entity

(DB에 한번 저장 되어서 식별자가 존재하는 Entity)

준영속 엔티티를 수정하는 2가지 방법

- 변경 감지
- 병합

변경 감지

Commit 되는 시점에 변경이 이뤄진다.

```java
User user = new User();
user.setId(userUpdateRequest.getId());
user.setName(userUpdateRequest.getName());
user.setGender(userUpdateRequest.getGender());
user.setEmail(userUpdateRequest.getEmail());

@Transactional
public void updateUser(Long userId, UserUpdateRequest request) {
	User findUser = userRepository.findUserByUserId(userId);
	user.setName(userUpdateRequest.getName());
	user.setGender(userUpdateRequest.getGender());
	user.setEmail(userUpdateRequest.getEmail());
}
```

## 지연 로딩(Lazy Loading)과 조회 성능 최적화

양방향 연관관계는 둘 중의 한쪽은 @JsonIgnore, @JsonBackReference을 선언해야 한다.

@JsonManagedReference : 양방향 관계에서 정방향 참조할 변수에 추가하면 직렬화에 포함된다.

@JsonBackReference : 양방향 관계에서 역방향 참조로, 추가하면 직렬화에서 제외된다.

- 지연 로딩 시 실제 객체가 아닌 ByteBuddyInterceptor를 이용해 Proxy 객체를 생성하게 된다.

실제 객체에 접근하려 할 때 실제 Query를 실행해 객체를 주입한다. (Proxy 초기화)

항상 지연 로딩을 기본으로 하고, 성능 최적화가 필요한 경우에는 페치 조인(fetch join)을 사용

## 지연 로딩 (Lazy Loading)

❗1 + N 문제

```java
@Data
public class UserDto {
	private int userId;
	private String userName;
	private String userEmail;
	private int teamId;
	private String teamName;
	
	public UserDto(User user) {
		this.id = user.getUserId();
		this.userName = user.getName();
		this.userEamil = user.getEmail();
		this.teamId = user.getTeam().getTeamId();
		this.teamName = user.getTeam().getName();
	}
}
```

- 쿼리가 총 1 + N 번 실행된다.
    - `user` 조회 1번 (AppVersion 조회 결과수가 1이 된다.)
    - `user → team` 지연 로딩 (Select Team query) N번
    - ex) `user`의 결과가 N개일 경우 1 + N 번 실행
    
    ❗ 지연 로딩으로 조회할 entity가 추가될 경우 1 + N + N .... 번 실행
    

## 페치 조인 (Fetch Join)

- 1 + N 이 아닌 Query가 1번만 실행

- Entity로 받아와서 가공

```java
List<User> users = em.createQuery(
      "select u from User u" +
         " join fetch u.team u", User.class)
   .getResultList();

return users.stream().map(UserListDto::new)
   .collect(Collectors.toList());
```

- Dto에 종속적으로 최적화

```java
List<User> users = em.createQuery(
      "select new com.jpa.example.user.domain.UserListDto(u.userId, u.name, u.email, t.teamId, t.name)" +
				 " from User u" +
         " join fetch u.team u", User.class)
   .getResultList();
```

⭐ Query 방식 선택 권장 순서

1. Entity를 DTO로 변환하는 방법
2. 필요시 페치 조인으로 성능 최적화
3. DTO로 직접 조회하는 방법으로 최적화
4. JPA가 제공하는 Native SQL이나 JDBC Template나 Mybatis를 사용해 SQL 직접 사용

## 컬렉션 조회 최적화

> 컬렉션 페치 조인을 사용하면 페이징 불가능
❗1:N 조인이 발생하므로 데이터가 예측 할 수 없이 증가한다.
컬렉션 페치 조인은 1개만 사용
둘 이상 페치 조인 사용시 데이터가 부정합하게 조회 될 수 있다.
> 

실행 쿼리 수 : 4

```java
return queryFactory.select(new QVersionDto(appVersion))
			.from(appVersion)
			.where(appVersion.id.eq(id))
			.fetchOne();
```

실행 쿼리 수 : 2

```java
AppVersion result = em.createQuery(
		"select distinct v from AppVersion v" +
			" join fetch v.appUrls u" +
			" join fetch v.appInformation i" +
			" where v.id = :id", AppVersion.class)
	.setParameter("id", id)
	.getSingleResult();

return new VersionDto(result);
```

1. 먼저 ToOne(OneToOne, ManyToOne) 관계를 모두 페치 조인 한다. ToOne 관계는 Row수를 증가시키지 않으므로 페이징 쿼리에 영향을 주지 않는다.
2. 컬렉션은 지연 로딩으로 조회한다.
3. 지연 로딩 성능 최적화를 위해 `hibernate.default_batch_fetch_size`, `@BatchSize`를 적용한다.

📕 100~1000 사이의 사이즈 설정 권장 → 애플리케이션은 100이든 1000이든 결국 전체 데이터를 로딩해야 하므로 메모리 사용량 측면에서는 동일하다. (순간 부하를 견딜 수 있는 사이즈로 설정)

⭐ 일대다 관계인 컬렉션은 IN 절을 활용해서 메모리에 미리 조회해서 최적화

## OSIV (Open Session In View)

> 영속성 컨텍스트의 생존 범위를 지정
true일 경우 Client에게 Response 되기까지 영속성 컨텍스트를 유지
false일 경우 트랜잭션이 이뤄지는 범위에서만 영속성 컨텍스트를 유지
> 

`spring.jpa.open-in-view`: true

![osiv](https://github.com/crongcm/jpa-querydsl/assets/113030711/8e3bc4ce-26c9-4a1d-a0b1-4c873224b6f8)

`spring.jpa.open-in-view`: false

![osiv-2](https://github.com/crongcm/jpa-querydsl/assets/113030711/d1a2681e-eec6-46f2-86b9-5504f3f3d6b1)


# QueryDSL

## Q-Type 활용

```java
QUser u = new QUser('u');  // Alias 설정
QUser qUser = QUser.user;      // 기본 인스턴스 사용

import static com.jpa.example.entity.QUser.*;
user
```

## 검색 조건

| Method | Description |
| --- | --- |
| eq | = |
| ne | ≠ |
| not | ! |
| isNotNull | IS NOT NULL |
| in | IN |
| notIn | NOT IN |
| between | BETWEEN AND  |
| goe | ≥ |
| gt | > |
| loe | ≤ |
| lt | < |
| like | LIKE |
| contains | LIKE ‘&&’ |
| startsWith | LIKE ‘&’ |
| and | AND |
| , | AND |
| or | OR |

```java
.eq("달구")       // = '달구'
.ne("달구")       // != '달구'
.eq("달구").not() // != '달구'

.isNotNull()     // IS NOT NULL

.in(10, 20)       // IN (10, 20)
.notIn(10, 20)    // NOT IN (10, 20)
.between(10, 30)  // BETWEEN 10 AND 30

.goe(30)  // >= 30
.gt(30)   // > 30
.loe(30)  // <= 30
.lt(30)   // < 30

.like("달구%")      // LIKE '달구%'
.contains("달구")   // LIKE '%달구%' 
.startsWith("달구") // LIKE '달구%'

.and()  // AND
,       // AND
.or()   // OR
```

## 결과 조회

| Method | Description |
| --- | --- |
| fetch() | 리스트 조회<br/>❗데이터 없으면 빈 리스트 반환 |
| fetchOne() | 단 건 조회<br/>❗결과가 없으면 ‘null’<br/>❗둘 이상이면 com.querydsl.core.NonUniqueResultException |
| fetchFirst() | limit(1).fetchOne() |
| fetchResults() | 페이징 정보 포함, total count 쿼리 추가 실행 |
| fetchCount() | count 쿼리로 변경해서 count 조회 |

## 정렬

| Method | Description |
| --- | --- |
| desc() | desc |
| asc() | asc |
| nullsLast() | 없으면 마지막에 출력 |

## 페치 조인

```java
join(member.team, team).fetchJoin()
```

## 서브 쿼리

❗ JPA JPQL 서브쿼리의 한계점으로 from절의 서브쿼리(인라인 뷰)는 지원하지 않는다.

```java
QUser userSub = new QUser("userSub");

List<User> result = queryFactory
                            .selectFrom(user)
                            .where(user.age.eq(
                              JPAExpressions
                                .select(userSub.age.max())
                                .from(userSub)
                            ))
                            .fetch();
```

from절의 서브쿼리 해결방안

1. 서브쿼리를 join으로 변경한다. (가능한 상황도 있고, 불가능한 상황도 있다.)
2. 애플리케이션에서 쿼리를 2번 분리해서 실행한다.
3. nativeSQL을 사용한다.

⚠️ 한방 쿼리 보다는 여러 쿼리로 나눠서 처리하는걸 권장

## CASE

단순한 CASE

```java
select(user.age
            .when().then()
            .when().then()
            .otherwise()
)
```

`new CaseBuilder()` : 복잡한 CASE

```java
new CaseBuilder()
            .when(user.age.between(0, 20)).then()
            .when(user.age.between(21, 30)).then()
            .otherwise
```

⚠️ CASE문을 쓰기 보다는 애플리케이션이나 프리젠테이션 단에서 처리하는걸 권장

## 상수

```java
queryFactory
      .select(Expressions.constant(”A”))
      .from(user)
      .fetch();
```

## 문자 더하기

```java
queryFactory
      .select(user.name.concat("_").concat(user.age.stringValue()))
      .from(user)
      .fetch();
```

⚠️ `user.age.stringValue()` 문자가 아닌 다른 타입들은 `stringValue()`로 문자 변환할 수 있다. (ENUM을 처리할 때 자주 사용)

## QueryDSL 결과 반환

기본

```java
List<Tuple> result = queryFactory.
                        .select(user.name, user.age)
                        .from(user)
                        .fetch();

for (Tuple tuple : result) {
	String name = tuple.get(user.name);
	Integer age = tuple.get(user.age);
}
```

⚠️ Repository에서 tuple → DTO 로 변환하여 사용 권장 (QueryDSL에 종속적)

DTO

```java
// Setter, 기본생성자 필수
select(Projections.bean(UserDto.class, user.name, user.age))
// 필드에 주입 (필드 매칭)
//.as() , ExperssionUtils.as()
select(Projections.fields(UserDto.class, user.name, user.age))
// Type이 일치해야 한다
select(Projections.constructor(UserDto.class, user.name, user.age))
```

```java
@QueryProjection
UserDto(String name, int age) {
  this.name = name;
  this.age = age;
}

//QUserDto 생성
queryFactory
    	.select(new QUserDto(user.name, user.age))
    	.from(user)
    	.fetch();
```

⚠️ @QueryProjection은 QueryDSL에 종속적이게 된다.

## 동적 쿼리

BooleanBuilder

```java
BooleanBuilder builder = new BooleanBuilder();
if (name != null) {
	builder.and(user.name.eq(name))
}
if (age != null) {
	builder.and(user.age.eq(age))
}

return queryFactory
              .selectFrom(user)
              .where(builder)
              .fetch();
```

Where

```java
private List<User> searchUser(String name, Integer age) {
  return queryFactory
                .selectFrom(user)
                .where(nameEq(name), ageEq(age)) // null 일 경우 무시된다.
                .fetch();
}

private BooleanExpression nameEq(String name) {
	return SpringUtils.hasText(name) ? user.name.eq(name) : null;
}

private BooleanExpression ageEq(Integer age) {
	return age != null ? user.age.eq(age) : null;
}

private BooleanExpression allEq(String name, Integer age) {
	return nameEq(name).and(ageEq(age));
}
```

Bonus

```java
private BooleanBuilder ageEq(Integer age) {
    return nullSafeBuilder(() -> member.age.eq(age));
}

private BooleanBuilder roleEq(String roleName) {
    return nullSafeBuilder(() -> member.roleName.eq(roleName));
}

public static BooleanBuilder nullSafeBuilder(Supplier<BooleanExpression> f) {
    try {
        return new BooleanBuilder(f.get());
    } catch (IllegalArgumentException e) {
        return new BooleanBuilder();
    }
}
```

## SQL Function

```java
queryFactory
          .select(Expressions.stringTemplate(
            "function('replace', {0}, {1}, {2}",
            user.name, "user", "U"))
          .from(user)
          .fetch();
```

## Paging

```java
public Page<Dto> searchPageSimple(SearchCondition condition, Pageable pageable) {
	List<Dto> content = queryFactory
                              .select()
                              .from()
                              .where()
                              .offset(pageable.getOffset())
                              .limit(pageable.getPageSize())
                              .fetch();

	JPAQuery<> countQuery = queryFactory
                                  .select()
                                  .from()
                                  .where();

	// 첫 번째 페이지인데 사이즈가 작을 경우 또는 마지막 페이질 경우는 페이징 카운트 쿼리 실행 X
	return PageableExecutionUtils.getPage(content, pageable, countQuery::fetchCount);
}
```

```java
SearchCondition condition = new SearchCondition();
PageRequest pageRequest = PageRequest.of(0, 3); // page, size / page 0부터 시작(0==1)

Page<Dto> result = repository.searchPageSimple(condition, pageRequest);
```
