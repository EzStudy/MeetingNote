# Spring Boot Security + OAuth2 적용 일기

### 1. 구글 서비스 등록

 먼저 구글 서비스에 신규 서비스를 생성합니다. 여기서 발급된 인증 정보(clientId와 clientSecret)를 통해서 로그인 기능과 소셜 서비스 기능을 사용 할 수 있으니 무조건 발급받고 시작해야 합니다.

구글 클라우드 플랫폼 주소([https://console.cloud.google.com](https://console.cloud.google.com))로 이동합니다. 프로젝트 선택에서 새 프로젝트를 선택합니다.

![스크린샷 2021-12-08 오후 10 51 08](https://user-images.githubusercontent.com/54675591/145219716-43cc9097-3040-48ed-848c-befbad0c3eea.png)

프로젝트 이름이 최소 4개 이상이여야 해서 이름을 CSQSSO 로 지어줬습니다.

생성이 완료된 프로젝트를 선택하고 왼쪽 메뉴 탭을 클릭해서 API 및 서비스 카테고리로 이동합니다.

![스크린샷 2021-12-08 오후 10 53 13](https://user-images.githubusercontent.com/54675591/145219999-27d49e54-c359-4537-a080-5d106ac457ae.png)

맨 위의 사용자 인증 정보 만들기 에서 OAuth 클라이언트 ID 를 클릭합니다.

![스크린샷 2021-12-08 오후 10 55 35](https://user-images.githubusercontent.com/54675591/145220342-2b81e8ac-46b2-447f-b530-cc07dd89e91d.png)

애플리케이션 유형, 이름, 승인된 리디렉션URI를 입력해줍니다.

![스크린샷 2021-12-08 오후 11 07 56](https://user-images.githubusercontent.com/54675591/145222253-560ea895-057c-4fd9-80af-3cb1dbdcb5fd.png)

> 승인된 리디렉션 URI 설명
>
> * 서비스에서 파라미터로 인증 정보를 주었을 때 인증이 성공하면 구글에서 리다이렉트할 URL dlqslek.
> * 스프링 부트 2 버전의 시큐리티에서는 기본적으로 {도메인}/login/oauth2/code/{소셜서비스코드} 로 리다이렉트 URL을 지원하고 있습니다.
> * 사용자가 별도록 리다이렉트 URL을 지원하는 Controller를 만들 필요가 없습니다. 시큐리티에서 이미 구현해 놓은 상태입니다.
> * 현재는 개발단계이므로 http://localhost:8080/login/oauth2/code/google로만 등록합니다.
> * 배포를 하게되면 localhost외에 추가로 주소를 추가해야하며, 이건 이후 단계에서 진행

이후 만들기를 누르면 OAuth 클라이언트가 생성됩니다.

![스크린샷 2021-12-08 오후 11 12 03](https://user-images.githubusercontent.com/54675591/145223048-dc6a9d08-092c-466f-b100-865d8c74ab80.png)

### 2. Application-auth 등록

Src/main/resources에 application-oath.yml 파일을 생성합니다.

위에서 생성된 클라이언트 ID와 클라이언트 보안 비밀번호를 다음과 같이 등록합니다.

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: 등록한 클라이언트 ID
            client-secret: 등록한 클라이언트 SECRET 키
            scope: profile, email
```

스프링 부트에서는 properties또는 yml의 이름을 application-xxx.yml로 만들면 xxx라는 이름의 profile이 생성되어 이를 통해 관리할 수 있습니다. 즉, profile=xxx라는 식으로 호출하면 해당 yml의 설정들을 가져올 수 있습니다. profile=oauth를 dev에 포함 시켜야 하므로 application-dev.yml에 다음을 추가해 줍니다.

```yaml
spring:
  #oauth
  profiles:
    include: oauth
```

### 3. .gitignore 등록

구글 로그인을 위한 클라리언트 id와 클라이언트 보안 비밀은 보안이 중요한 정보들입니다. 이들이 외부에 노출될 경우 언제든 개인정보를 가져갈 수 있는 취약점이 될 수 있습니다. 실제로 github에 올라갈시 x 될 수 있으니 .gitignore에 다음과 같이 한 줄의 코드를 등록합니다.

```yaml
### oauth ###
application-oauth.yml
```

### 4. 구글 로그인 연동

User 클래스 생성

```java
package io.ezstudy.open.csq.domain.user.domain;

import io.ezstudy.open.csq.domain.model.BaseTimeEntity;
import javax.persistence.Column;
import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.Id;
import javax.validation.constraints.NotNull;
import lombok.Builder;
import lombok.Getter;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.GenericGenerator;

@Entity
@Getter
@NoArgsConstructor
public class User extends BaseTimeEntity {
    @Id
    @GeneratedValue(generator = "uuid")
    @GenericGenerator(name = "uuid", strategy = "uuid2")
    @Column(columnDefinition = "VARCHAR(36)", insertable = false, updatable = false, nullable = false)
    private String id;

    @NotNull
    @Column(length = 20, nullable = false, unique = true)
    private String name;

    @NotNull
    @Column(length = 100, nullable = false, unique = true)
    private String email;

    @Column(length = 20)
    private String gender;

    @NotNull
    @Column(length = 20, nullable = false)
    private String provider;

    @NotNull
    @Column(length = 10)
    private String role;

    @Builder
    public User(String name, String email, String gneder, String provider,
        String role, String createdAt, String updatedAt, String deletedAt){
        super(createdAt, updatedAt, deletedAt);
        this.name = name;
        this.email = email;
        this.gender = gneder;
        this.provider = provider;
        this.role = role;
    }
}

```

User의 CRUD를 책임질 UserRepository 도 생성

```java
package io.ezstudy.open.csq.domain.user.dao;

import io.ezstudy.open.csq.domain.user.domain.User;
import java.util.Optional;
import org.springframework.data.jpa.repository.JpaRepository;

public interface UserRepository extends JpaRepository<User, String> {
    Optional<User> findByEmail(String email);
}
```

이후 bundle.gralde에 의존성을 추가합니다.

```
// security
    implementation 'org.springframework.boot:spring-boot-starter-security'
    //oauth
    implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'
```

> Spring-boot-starter-oauth2-client
>
> * 소셜 로그인 등 클라이언트 입장에서 소셜 기능 구현 시 필요한 의존성입니다.

이후 시큐리티 관련 패키지를 생성합니다. Src/main/java/io.ezstudy.open.csq/global/config/auth 를 생성하고 이곳 에 클래스를 만들어주겠습니다.

* SecurityConfig

* CustomOAuth2UserService

  * 이 클래스에서는 구글 로그인 이후 가져온 사용자의 정보(email,name, picture 등)들을 기반으로 가입 및 정보수정, 세션 저장 등의 기능을 지원합니다.
  * 구글 사용자 정보가 업데이트 되었을 때를 대비하여 update 기능도 같이 구현되었습니다.

  > * registrationId (변수)
  >   * 현재 로그인 진행 중인 서비스를 구분하는 코드입니다.
  >   * 여기서는 구글만 사용하는 불피요한 값이지만 이후 네이버, 카카오, 페북 로그인 연동시 네이버로 로그인인지, 구글 로그인인지 구분하기 위해 사용합니다.
  > * userNameAttributeName (변수)
  >   * OAuth2 로그인 진행 시 키가 되는 필드 값을 이야기합니다. Primary Key와 같은 의미입니다.
  >   * 구글의 경우 기본적으로 코드를 지원하지만, 네이버 카카오 등은 기본 지원하지 않습니다. 구글의 기본 코드는 `sub`입니다.
  >   * 이후 카카오 로그인과 구글 로그인을 동시 지원할 때 사용됩니다.
  > * OAuthAttributes (클래스)
  >   * OAuth2UserService를 통해 가져온 OAuth2User의 attribute를 담을 클래스입니다.
  > * SessionUser (클래스)
  >   * 세션에 사용자 정보를 저장하기 위한 DTO 클래스입니다.
  >   * 왜 User클래스를 쓰지 않고 새로 만들어서 쓰는 지 뒤이어서 상세하게 설명하겠습니다.

* OAuthAttributes

  > * Of()
  >   * OAuth2User에서 반환하는 사용자 정보는 Map이기 때문에 값 하나하나를 변홚ㅐ야 합니다.
  > * toEntity()
  >   * User엔티티를 생성합니다.
  >   * OAuthAttributes에서 엔티티를 생성하는 시점은 처음 가입할때 입니다.
  >   * 가입할 때의 기본 권한을 ADMIN으로 주기 위해서 role 빌더값에는 Role.ADMIN으로 줍니다.

* SessionUser

  * SessionUser에는 인증된 사용자 정보만 필요합니다. 그 외에 필요한 정보들은 없으니 name, email 만 필드로 선언합니다.
