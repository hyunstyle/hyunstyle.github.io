---
layout: post
title: 로그인 REST API와 회원가입 기능을 구현하면서
description: "very good."
tags: [NHN Ent., Developer, Toast, Rookie]
image:
  background: triangular.png
---

어느덧 기술교육도 중반으로 접어들었다. 이번 포스트에서는 기술교육 기간동안 서비스를 만들면서 특히 중점적으로 담당했던 로그인 REST API 부분과, 추가 기능으로 회원가입을 구현하게 된 이유와 
배운 점 등에 대해 기술하려고 한다.

결론부터 이야기하면, 우리 TF에서는 로그인 REST API와 회원가입 구현을 위해 Spring Security를 사용하였다. Spring Security를 선택한 이유는 몇 가지가 있다. 기획 및 요구사항을 바탕으로,
우리 TF는 아래의 조건을 만족시켜야 한다고 생각하였다.

1. 데이터베이스에 사용자의 비밀번호는 `암호화`된 상태로 저장되어야 한다.
2. REST API 호출시 `비정상적인 URL 및 parameter로 접근을 시도`할 때 이를 막아야 한다.
3. 추후에 사용자에게 주어진 `권한`을 바탕으로 세션을 관리해야 할 수도 있을 것 같다는 생각이 들었다.


크게 위 3가지 이유를 바탕으로 적절한 REST API 구축 방법을 찾아보다가, Spring Security를 프로젝트에 적용하게 되었다.
Spring boot에서 Spring Security를 사용하려면 아래와 같은 dependency를 pom.xml에 추가하여야 한다.

```yaml
<!-- https://mvnrepository.com/artifact/org.springframework.boot/spring-boot-starter-security -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

이후, Security Config 파일을 추가한다.

```yaml

@Configuration
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired 
    MemberService memberService;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.NEVER)
            .and()
            .authorizeRequests()
            .antMatchers("/rest/login").anonymous() // 해당 경로의 페이지들은 접근할 수 있도록(anonymous) 허용
            .antMatchers("/rest/members").anonymous()
            .antMatchers("/rest/join/isValid").anonymous()
            .antMatchers("/rest/join").anonymous()
            .antMatchers("/rest/matches").anonymous()
            .antMatchers(HttpMethod.OPTIONS, "/**").permitAll() // Spring은 Options 요청을 막고있기 때문에 이것을 허용해줌. 실제 GET, POST 요청 전 preflight
            .anyRequest().fullyAuthenticated() // 그 외의 요청은 거부 (인증된 사용자 + isRememberMe(false) 인 경우만 허용)
            .and()
            .logout();
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.userDetailsService(memberService)
            .passwordEncoder(memberService.passwordEncoder()); // Password 암호화 사용 (Bcrypt 알고리즘)
    }
}
```

MemberService 클래스는 로그인과 회원가입 등 member 관련 서비스를 처리하는 빈이다.
`configure(HttpSecurity http)` 함수 부분에서, 
csrf (Cross-site Request Forgery) 공격은 아직 사용자 정보를 update 하는 등의 쿼리문 자체가 존재하지 않기 때문에, disable 시켜놓은 상태이다.

위 처리를 통해 매번 request에 대해, 인증되지 않은 사용자는 .antMatchers 로 허용된 경로에만 접근을 시도할 수 있다. 허용되지 않은 경로에 대해서는, 403(Access Denied) 오류가 발생한다.
request에 대한 접근 제어 형식은 아래와 같다.

anonymous() : 인증되지 않은 사용자도 접근 가능
authenticated() : 인증된 사용자만 접근 가능
fullyAuthenticated() : 인증된 사용자 + isRememberMe(false) 인 경우에만 접근 가능


또한, `configure(AhtuenticationManagerBuilder auth` 함수에서 인증에 사용할 서비스를 @Autowired 된 memberService 로 설정한 후,
인증에 .passwordEncoder 함수를 추가하면 간편하게 사용자 인증에 난독화된 암호를 사용할 수 있다. 이 때 패스워드 인코딩 알고리즘은 Bcrypt를 사용하였다. 패스워드 암호화 기법 중,
Bcrypt는 현재 가장 널리 쓰이고 있는 방식 중 하나이고, 현재 Spring에서도 가장 권고하는 방식으로 알고 있다. Spring에서 Bcrypt 알고리즘을 사용하는 간단한 방식은 아래와 같다.

`Bcrypt 알고리즘 참조`
link: https://d2.naver.com/helloworld/318732


```yaml
    String id = "Hello"; // 사용자가 입력한 아이디
    String passwd = "testPassword"; // 사용자가 입력한 패스워드

    String encodedPasswd = new BCryptPasswordEncoder().encode(passwd); // bcrypt 알고리즘으로 인코딩된 패스워드

```

위의 자바 코드를 작성 후 로그를 찍어보면, encodedPasswd 에서는 복잡하게 암호화된 스트링이 출력되는 것을 확인할 수 있다.

이후,

```yaml

    Member member = memberService.getMemberByLocalpart(id); // id를 통해 memberService -> mapper -> MyBatis를 통해 member 정보 조회 -> 리턴

    if(member != null) {
        LOGGER.info("raw passwd: " + passwd);
        LOGGER.info("encoded input: " + encodedPasswd);
        LOGGER.info("user passwd: " + member.getPasswd());

        PasswordEncoder passwordEncoder = memberService.passwordEncoder();

        if(passwordEncoder.matches(passwd, member.getPasswd())) {
            // 사용자가 입력한 패스워드가 DB에 저장된 member의 패스워드와 일치하는 경우
        }
        else {
            // 비밀번호가 일치하지 않는 경우
        }
    }
```

구체적인 mapper 부분이나 service 부분 등은 생략되었지만 대략적으로 위 과정을 통해, 간단히 사용자가 입력한 아이디와 비밀번호가 올바른지 판단할 수 있다. 

주의할 점은 Spring Security 사용시 사용자의 Authority (권한) 를 명시적으로 override해서 부여해야 한다. 기본적인 권한은 ROLE_USER, ROLE_ADMIN 두 가지로, 다른 설정이 없다면 꼭
위 두가지 권한 중 하나 이상을 DTO에 부여해야만 한다.


위와 같은 방식으로 암호화를 구현하였더니, 불편한 점이 생겼다. 새로운 Member를 만들어서 테스트를 할 때 DB에 비밀번호를 추가하는 것이 너무 불편하였다.
프로그램에서 아이디와 비밀번호를 입력하고, 로그를 통해 Bcrypt 알고리즘이 적용된 output을 복사하여 DB에 비밀번호를 붙여넣기하고 update 쿼리를 실행하여야 제대로 새로운 Member로
로그인을 할 수 있었기 때문이다. 이러한 불편을 해소하기 위해, 회원가입을 구현하게 되었다. 


```yaml

    public void createMember(JoinMember member) {
            String rawPassword = member.getPasswd();
            String encodedPassword = new BCryptPasswordEncoder().encode(rawPassword);
            member.setPasswd(encodedPassword);
            member.setAuthority("ROLE_USER");
            member.setDoorayLink("");

            memberMapper.createMember(member);
        }
```

회원가입 역시, 디테일한 부분을 생략하면 위와 같은 메소드로 간단히 구현할 수 있다. 사용자에게 웹에서 입력받은 정보를 form으로 전송받으면, 유효성 검사 후 위 service 함수를 호출하여 DB에 member를 추가하게 된다.
memberMapper.createMember 메소드를 통해, MyBatis mapper 내부에서는 Insert 쿼리가 실행되게 된다. 유심히 볼 부분은 최초 form 으로 전송한 사용자의 비밀번호를
BcryptPasswordEncoder().encode 메소드를 통해 암호화하여 setter로 재지정하는 모습이다. 이를 통해 DB에 추가될 때 사용자의 암호는 Bcrypt 알고리즘을 통과한 output로 저장되게 된다.
위와 같은 과정을 통해 Bcrypt 적용된 output 로그를 복사하여 DB에 붙여넣는 등의 귀찮은 짓을 하지 않아도 새로운 Member를 만들어 테스트를 할 수 있게 되었다.





짧은 기간이라 Spring Security를 심도있게 이해하고 적용한 코드는 아닐 수 있지만 위 과정을 통해 기본적인 REST 방식의 로그인과 회원가입을 모두 처리할 수 있게 되었다.
Spring으로 REST API를 개발할 때 Spring Security를 활용하는 것도 괜찮지 않나 하는 생각이 들었다.


결론은 이제 테스트 코드를 짜야한다.