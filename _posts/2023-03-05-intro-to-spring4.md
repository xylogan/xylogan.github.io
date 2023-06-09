---
title: 인프런 스프링 입문 김영한 강의 정리 4
categories: [Computer engineering, Backend engineering]
tags: [백엔드, 스프링, 자바, backend, spring, java]
---

Backend engineering을 공부하는 학생이라면 한번쯤 들어봤을 [김영한 님의 인프런 스프링 입문 강의](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%9E%85%EB%AC%B8-%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8/dashboard)를 들으며 기억해두면 도움될 만한 내용을 정리할 예정입니다. 강의를 모두 수강할 때까지 본 강의 정리 포스팅은 버전을 업데이트하며 주기적으로 업로드 할 예정입니다. 하지만, 주로 기록 목적의 포스팅이다보니 강의 목차 흐름에 따라 내용이 순차적으로 정리되어 다소 본문 내용의 일관성은 떨어질 수 있음을 미리 참고바랍니다. 

## H2 데이터베이스 설치 및 연결
> H2 데이터베이스를 설치해준 뒤 압축을 풀고 "/h2/bin"에 터미널로 들어가 "chmod 755 h2.sh"로 권한을 준 뒤 "./h2.sh"를 실행하면 웹 UI가 하나 나타납니다. 이때 WEB URL을 보면 임의의 IP주소가 들어가 있는데 이 부분을 "localhost"로 치환해주면 됩니다. 그 후 JDBC URL을 "jdbc:h2:tcp://localhost/~/test"로 바꿔주면 소켓을 통해 어플리케이션이랑 웹 컨트롤러 등에서 동시에 충돌 없이 DB에 잘 접근할 수 있게 됩니다. 데이터베이스 관련된 파일은 사용자의 홈 디렉터리에 "test.mv.db"에 저장되어 있는데 연결에 문제가 있다면 "rm test.mv.db"로 한번 지워준 뒤 실행시켰던 "h2.sh"도 중지시켰다가 재실행하고 처음에는 JDBC URL에 "jdbc:h2:~/test"를 입력하여 연결해줬다가 그 이후부터는 아까처럼 충돌 없이 소켓으로 연결하기 위해 "jdbc:h2:tcp://localhost/~/test"로 연결해주면 됩니다. "jdbc:h2:~/test"는 H2 데이터베이스에 파일 모드로 접근하는 것입니다. 이렇게 접속하는 이유는 ~/test 경로에 DB를 파일로 생성한다는 의미인데 파일을 생성하려면 권한 문제가 있어서 이렇게 세션 키를 가진채로 파일 모드로 먼저 한번 접근해주어 DB 파일을 생성해준 뒤 여러 사용자끼리 충돌 없이 사용하기 위해 네트워크 모드인 tcp로 연결해주는 것입니다.

## 스프링 통합 테스트
> 스프링 통합 테스트란 테스트 코드를 실행할 때 실제로 스프링과 데이터베이스 등도 함께 띄워 테스트를 진행하는 것을 말합니다. 이를 위해 테스트 코드 클래스에 "@SpringBootTest" 어노테이션을 달아주면 됩니다. 이렇게 되면 실제로 DB에 테스트할 때 사용된 데이터들이 반영될 수 있는데 "@Transactional" 어노테이션을 테스트 케이스에 달면 테스트 시작 전에 트랜잭션을 모두 실행하고 테스트가 끝나면 항상 롤백해주기 때문에 테스트를 아무리 하여도 데이터베이스에 반영시키지 않을 수 있어 테스트 코드 간의 충돌을 막아줄 수 있습니다. 예제 코드는 아래와 같습니다. 
   
```java
package hello.hellospring.service;

import hello.hellospring.domain.Member;
import hello.hellospring.repository.MemberRepository;
import hello.hellospring.repository.MemoryMemberRepository;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.junit.jupiter.api.Assertions.assertThrows;

@SpringBootTest
@Transactional //이걸 테스트 케이스에 달면 테스트 실행할 때 트랜잭셔널 실행하고 DB에 데이터를 모두 넣은 다음에 롤백을 해줘서 DB에 테스트한 데이터들이 반영되지 않음
class MemberServiceIntegrationTest {

    @Autowired
    MemberService memberService;
    @Autowired
    MemberRepository memberRepository;

    @Test
    void 회원가입() {
        //given
        Member member = new Member();
        member.setName("spring");

        //when
        Long saveId = memberService.join(member);

        //then
        Member findMember = memberService.findOne(saveId).get();
        assertThat(member.getName()).isEqualTo(findMember.getName());
    }

    @Test
    public void 중복_회원_예외() {
        //given
        Member member1 = new Member();
        member1.setName("spring");

        Member member2 = new Member();
        member2.setName("spring");
        //when
        memberService.join(member1);
        IllegalStateException e = assertThrows(IllegalStateException.class, () -> memberService.join(member2));
        assertThat(e.getMessage()).isEqualTo("이미 존재하는 회원입니다.");

    }

}
```

## 스프링JdbcTemplate
> 스프링 JdbcTemplate는 순수 Jdbc 코드에서의 대부분의 반복적인 코드를 제거해주어 조금 더 효율적으로 사용할 수 있게 해줍니다. 순수 Jdbc와 동일하게 Implementation 등을 추가해주고 동일한 환경 설정을 해주면 됩니다. 예제 코드는 아래와 같습니다. jdbcTemplate 객체에서 쿼리를 사용하는데 쿼리를 DB에 전달하고 처리한 다음 해당 쿼리의 결과값을 백엔드 자체에서도 활용할 수 있도록 memberRowMapper()를 사용하여 여기서 객체를 만들어줍니다. 
   
```java
package hello.hellospring.repository;

import hello.hellospring.domain.Member;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.jdbc.core.namedparam.MapSqlParameterSource;
import org.springframework.jdbc.core.simple.SimpleJdbcInsert;

import javax.sql.DataSource;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.Optional;

public class JdbcTemplateMemberRepository implements MemberRepository {
    private final JdbcTemplate jdbcTemplate;

    public JdbcTemplateMemberRepository(DataSource dataSource) {
        jdbcTemplate = new JdbcTemplate(dataSource);
    }

    @Override
    public Member save(Member member) {
        SimpleJdbcInsert jdbcInsert = new SimpleJdbcInsert(jdbcTemplate);
        jdbcInsert.withTableName("member").usingGeneratedKeyColumns("id");
        Map<String, Object> parameters = new HashMap<>();
        parameters.put("name", member.getName());
        Number key = jdbcInsert.executeAndReturnKey(new
                MapSqlParameterSource(parameters));
        member.setId(key.longValue());
        return member;
    }

    @Override
    public Optional<Member> findById(Long id) {
        List<Member> result = jdbcTemplate.query("select * from member where id = ?", memberRowMapper(), id);
        return result.stream().findAny();
    }

    @Override
    public Optional<Member> findByName(String name) {
        List<Member> result = jdbcTemplate.query("select * from member where name = ?", memberRowMapper(), name);
        return result.stream().findAny();
    }

    @Override
    public List<Member> findAll() {
        return jdbcTemplate.query("select * from member", memberRowMapper());
    }

    private RowMapper<Member> memberRowMapper() {
        return (rs, rowNum) -> {
            Member member = new Member();
            member.setId(rs.getLong("id"));
            member.setName(rs.getString("name"));
            return member;
        };
    }
}
```

## JPA
> JPA란 Java Persistence API입니다. JPA는 자바 진영에서 ORM(Object-Relational Mapping) 기술 표준으로 사용되는 인터페이스의 모음입니다. JPA는 인터페이스 역할로, JPA를 구현한 대표적인 오픈소스로는 Hibernate가 있습니다. ORM은 Object-Relational Mapping으로 Object와 RDB의 테이블을 매핑하여 관리해주는 것으로 생각하면 됩니다. JPA를 사용하기 위해 dependencies에 "implementation 'org.springframework.boot:spring-boot-starter-data-jpa'"를 추가해주면 됩니다. 이 내부 자체적으로 jdbc를 담고 있으므로 이와 관련된건 지워줘도 됩니다. 예제로 사용했던 Member를 테이블로 매핑하기 위해 JPA가 관리하는 객체로 만들어줘야 합니다. 해당 클래스에 @Entity 어노테이션을 붙여주면 됩니다. Member 클래스 내 id는 PK(Primary Key)이므로 @Id 어노테이션을 붙여주고, @GeneratedValue(strategy = GenerationType.IDENTITY)도 붙여줍니다. IDENTITY 생성 전략이란 DB에서 알아서 Id값을 생성해주는 전략을 말합니다.   
아래는 JpaMemberRepository 코드입니다. JPA는 EntityManager로 모두 동작하기 때문에 해당 객체를 만들어 줘야합니다. 이때 "data-jpa"를 Implementation 하기만 하면 스프링 부트가 알아서 데이터 베이스와 연결해서 EntityManager를 만들어주므로 주입해서 사용만하면 됩니다.   
em.persist()를 사용하면 DataBase에 쿼리를 날려줌과 동시에 별로도 RowMapper 등을 사용할 필요 없이 해당 객체에 값 setting도 다 해줍니다. 마찬가지로 em.find(entity.class, PK)를 사용하면 해당 Entity 테이블에서 해당 PK 값을 통해 조회 쿼리를 날려줍니다.   
여기서 주의할 점이 findById()와 같이 1 건의 결과가 나오는 PK를 활용한 쿼리들은 간단히 JPA 메소드로 처리가 가능한데, PK 기반이 아닌 것들은 JPQL을 별도로 작성해줘야 합니다. JPQL이란 쿼리를 테이블 대상이 아니고 객체를 대상으로 쿼리를 날리는 것입니다. 즉 여기서는 Member @Entity를 대상으로 쿼리를 날리는 것입니다.   
    
```java
package hello.hellospring.repository;

import hello.hellospring.domain.Member;

import javax.persistence.EntityManager;
import java.util.List;
import java.util.Optional;

public class JpaMemberRepository implements MemberRepository{
    private final EntityManager em; //JPA는 EntityManager로 모두 동작함

    public JpaMemberRepository(EntityManager em) {  //data-jpa를 Implementation하면 스프링 부트가 알아서 데이터 베이스와 연결해서 EntityManager를 만들어주므로 주입만 하면 됌
        this.em = em;
    }

    @Override
    public Member save(Member member) {
        em.persist(member);
        return member;
    }

    @Override
    public Optional<Member> findById(Long id) {
        Member member = em.find(Member.class, id);
        return Optional.ofNullable(member);
    }

    @Override   //JPA 특징이 findById와 같이 단 건의 결과가 나오는 PK를 활용한 건 간단히 JPA 메소드로 해결이 가능한데, PK 기반이 아닌 것들은 JPQL을 작성해 줘야함.
    public Optional<Member> findByName(String name) {
        List<Member> result = em.createQuery("select m from Member m where m.name = :name", Member.class)
                .setParameter("name", name)
                .getResultList();

        return result.stream().findAny();


    }

    @Override
    public List<Member> findAll() {
        return em.createQuery("select m from Member m", Member.class)
                .getResultList();   //JPQL이란 쿼리를 테이블 대상이 아니고 객체를 대상으로 쿼리를 날리는 것, Member @Entity를 대상으로 쿼리를 날리는 것 m은 Alias
    }
}
```

## 스프링 데이터 JPA
> 스프링 데이터 JPA는 기본 JPA보다 더욱 간단하게 리포지토리에 구현 클래스 없이 인터페이스만으로 개발을 끝낼 수 있습니다. 그리고 반복적으로 개발해온 CRUD 기능도 알아서 제공해줍니다. SpringDataJpaMemberRepository 인터페이스 예제 코드는 아래와 같습니다. 스프링 데이터 JPA를 사용하기 위한 인터페이스는 JpaRepository 인터페이스를 상속 받아야 합니다. 인터페이스가 인터페이스를 받을 때는 Implements가 아니고 extends를 사용해야 합니다. 인터페이스가 JpaRepository를 상속 받고 있으면 해당 인터페이스의 구현체를 스프링이 자동으로 만들어주고 스프링 빈에 등록까지 해줍니다. 즉 저희는 이따가 SpringConfig에서 의존성 주입하여 사용해주기만 하면 됩니다. 이렇게 스프링 데이터 JPA를 사용하면 기본적인 CRUD 메소드는 다 제공해주는데 findByName()과 같이 각자 서비스의 비즈니스 로직마다 다른 변수명을 가지는 메소드들은 @Override를 활용하여 만들어 줘야합니다. 이렇게 만들어주면 스프링 데이터 JPA가 분석하여 JPQL로 변환하여 만들어줍니다.   
   
```java
package hello.hellospring.repository;

import hello.hellospring.domain.Member;
import org.springframework.data.jpa.repository.JpaRepository;

import java.util.Optional;

public interface SpringDataJpaMemberRepository extends JpaRepository<Member, Long>, MemberRepository {  //인터페이스가 인터페이스를 받을 때는 Implements가 아니고 extends이다.
    //JpaRepository<>를 받고 있으면 구현체를 자동으로 만들어준다. 그 후 자동으로 스프링 빈에 등록해줌.
    @Override
    Optional<Member> findByName(String name);   //이렇게 일반적이지 않은 비즈니스 로직에 대한 메소드들은 오버라이드해줘야함.
    //이렇게 하면 JPQL로 "select m from Member m where m.name = ?"로 짜줌.
}
```

## AOP
> AOP란 Aspect Oriented Programming라는 뜻으로 공통 관심 사항(cross-cutting concern)와 핵심 관심 사항(core concern) 분리하는 것을 말합니다. 예를들어 현업에서 각 메소드별 호출 시간을 구하라고 과제를 받는다면 모든 메소드의 시작과 끝에 시간을 측정하는 메소드를 넣어주는 것이 간단한 해결책이 될 수 있겠지만 수많은 메소드가 있다면 구현하는데 굉장히 시간도 오래걸리고 비효율 적일 것입니다. 이렇게 중복되는 로직을 공통 관심 사항으로 분리하는 것을 말합니다. 분리 한 뒤에 필요한 핵심 관심 사항을 지정하여 일부에만 적용할 수 있도록 사용하는 것인데 스프링에서는 이를 위해 프록시 방식을 활용합니다. 예를들어 memberService에 AOP를 적용한다면 프록시를 활용하여 가짜 memberService 객체를 생성하여 공통 관심 사항 AOP 로직을 실행시키고 그 후 joint.Proceed() 메소드를 활용하여 진짜 memberService로 이동합니다. AOP를 적용하기 위해서 aop 패키지 밑에 TimeTraceAop 클래스를 만들어 주었고 @Component 어노테이션을 통해 스프링 빈에 등록해줬습니다. 예제 코드는 아래와 같습니다.   

```java
package hello.hellospring.aop;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class TimeTraceAop {
    @Around("execution(* hello.hellospring..*(..))")    //main package 하위에 AOP를 다 적용하라는 뜻
    public Object execute(ProceedingJoinPoint joinPoint) throws Throwable { //메소드 호출할 때마다 인터셉트가 딱딱 걸리는 것
        long start = System.currentTimeMillis();
        System.out.println("START: " + joinPoint.toString());
        try {
            return joinPoint.proceed();
        } finally {
            long finish = System.currentTimeMillis();
            long timeMs = finish - start;
            System.out.println("END: " + joinPoint.toString() + " " + timeMs + "ms");
        }

    }
}
``` 

## References
> https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EC%9E%85%EB%AC%B8-%EC%8A%A4%ED%94%84%EB%A7%81%EB%B6%80%ED%8A%B8/dashboard