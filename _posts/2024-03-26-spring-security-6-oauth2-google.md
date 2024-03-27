---
title: spring security 6에서 oauth2.0 구현하기(google)
date: 2024-03-26 20:37:39 +0900
categories: [Spring]
tags: [spring, spring-security]    # TAG names should always be lowercase
---

## 개요
- spring security 6을 이용하여 oauth2.0 인증을 구현해본다.  
- 예시에서는 oauth 2.0 인증 프로바이더로 google을 사용한다.  

## Oauth 2.0 이란
- 표준화된 인증 방식을 의미한다.  
- 한 곳에서 Ouath 인증을 하면 이 인증을 공유하는 애플리케이션끼리  
  별도의 인증 없이 인증할 수 있다[[참고1](https://ko.wikipedia.org/wiki/OAuth){:target="_blank"}].  

## 일반적인 구글 Oauth 2.0 구현 과정
- 구글 콘솔에서 oauth 2.0 사용자를 등록한다.  
  이떄 구글 로그인 성공 시 access_token을 받을   
  redirect_uri를 입력해야 한다.  
- redirect_uri는 로그인 처리를 할 백엔드 uri를 입력하면 된다.  
- 등록에 성공하면  구글 oauth 2.0의 client-id와 client-secret을 얻을 수 있다.  
- [구글] 사용자가 버튼을 클릭하면 구글 로그인 페이지로 이동한다.  
  로그인에 성공하면 미리 지정한 redirect_uri로 이동한다.  
- [내 앱 프론트엔드]  구글 로그인 url으로 리다이렉트 [[참고2](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow?hl=ko){:target="_blank"}].  
    - url  
      ```bash  
      curl -XGET "https://accounts.google.com/o/oauth2/v2/auth?  
       scope=https://www.googleapis.com/auth/userinfo.profile&  
       include_granted_scopes=true&  
       response_type=code&  
       state=state_parameter_passthrough_value&  
       redirect_uri=https://myapp-backend.com/google/callback&  
       client_id=my-client_id"  
      ```  
        - scope  
            - 구글 로그인 성공 시 접근 가능한 정보 범위이다.  
              최초 구글 콘솔에서 등록한 접근 가능 정보만 허용된다.  
            - [scope list](https://developers.google.com/identity/protocols/oauth2/scopes?hl=ko#oauth2){:target="_blank"}  
        - response_type  
            - request token이 담길 파라미터 이름  
            - request token을 이용하여 access_token을 발행할 수 있다.  
        - redirect_uri  
            - 구글 로그인 성공 시 리다이렉트 할 uri이다.  
            - 최초 구글 콘솔에서 등록한 uri로만 허용된다.  
        - client_id  
            - 구글 콘솔에서 발급 받은 client_id이다.  
- [내 앱 백엔드]  request_token 수신  
    - 구글 로그인에 성공하여 redirect_uri 응답을 받으면  
      request_token이 RequestParam의 code에 담겨 오게 된다.  
      ```bash  
      // 응답 예시  
      https://myapp-backend.com/google/callback?code=4/P7q7W91a-oMsCeLvIaQm6bTrgtp7  
      ```  
- [내 앱 백엔드]  access_token 발급  
    - request_token과 client-id, client-secret을 이용하여  
      access_token을 발급한다[[참고3](https://notspoon.tistory.com/47){:target="_blank"}].  
      ```bash  
      curl -XPOST "https://oauth2.googleapis.com/token?code=my-code&client_id=my-client-id&client_secret=my-client-secret&redirect_uri=https://myapp-backend.com/google/callback&grant_type=authorization_code"  \  
      --header "Content-Type: application/x-www-form-urlencoded"  
      ```  
- [내 앱 백엔드] acces_token 수신  
    - 요청에 성공하였다면 아래 정보가 body에 담겨 응답을 받는다.  
      ```json  
      {  
          "access_token": "my_access_token",  
          "expires_tn": "my_expires_tn",  
          "scope": "my_scope",  
          "token_type": "my_token_type",  
          "id_token": "my_id_token"  
      }  
      ```  
- [내 앱 백엔드] 유저 정보 조회  
    - access_token을 이용하여 유저 정보를 요청한다.  
      ```bash  
      curl -XGET "https://www.googleapis.com/oauth2/v1/userinfo" \  
      --header "Authorization: Bearer my-access-token"  
      ```  
                
- [내 앱 백엔드] 유저 정보 수신  
    - 요청에 성공하였다면 아래 정보가 body에 담겨 응답을 받는다.  
      ```json  
       {  
          "id": "my-id",  
          "email": "my-email",  
          "name": "my-name",  
          "locale": "my-locale"  
      }  
      ```  
    - 받은  응답으로 내 앱 유저 등록 및 로그인 처리를 진행하면 된다.  

## spring security 6에서의 구글 Oauth2.0
- 개요  
    - spring security 6을 사용한다면  
      위의 과정을 직접 구현할 필요가 없다.  
    - 구글 콘솔에서 oauth 2.0 사용자를 등록 후  
      관련 정보를 spring boot에 설정값으로 넣어주면  
      알아서 처리된다.  
- 환경  
    - java 21  
    - spring boot 3.2.3  
- 설치  
  ```bash  
  implementation 'org.springframework.boot:spring-boot-starter-security'  
  implementation 'org.thymeleaf.extras:thymeleaf-extras-springsecurity6'  
  implementation 'org.springframework.boot:spring-boot-starter-webflux'  
  implementation 'org.springframework.boot:spring-boot-starter-oauth2-client'  
  ```  
- application.properties  
  ```bash  
  # Spring Security  
  spring.security.oauth2.client.registration.google.client-id=my-client  
  spring.security.oauth2.client.registration.google.client-secret=my-client-secret  
  spring.security.oauth2.client.registration.google.authorization-grant-type=authorization_code  
  spring.security.oauth2.client.registration.google.scope=profile, email  
  ```  
- [내 앱 프론트엔드] 구글 로그인 url으로 리다이렉트   
    - 기본 구글 로그인 url  
        - https://myapp-backend.com/oauth2/authorization/google  
    - 구글 로그인 url 변경  
      ```bash  
      # application.properties에 추가  
      spring.security.oauth2.client.provider.google.authorization-uri=https://myapp-backend.com/oauth2/authorization/google-custom  
      ```  
    - 예시  
      ```html  
      <a href="https://myapp-backend.com/oauth2/authorization/google">  
        구글로그인  
      </a>  
      ```  
- [구글] 구글 로그인 화면으로 리다이렉트  
  로그인에 성공하면 지정된 redirect_uri로 이동  
    - 기본 redirect_uri  
        - https://myapp-backend.com/login/oauth2/code/google  
    - redirect_uri 변경  
      ```bash  
      spring.security.oauth2.client.registration.google.redirect-uri=https://myapp-backend.com/login/oauth2/code/custom-google  
      ```  
    - 주의사항  
        - 구글 콘솔에서 OAuth 2.0 사용자 등록 시에  
          위의 redirect_uri도 등록해야 구글 로그인이 가능하다.  
- 이후 과정  
    - 나머지는 알아서 처리해준다.  
      로그인에 성공했다면 바로 세션 쿠키를 얻게 된다.  

## spring security 6 설정
- 개요  
    - 인증이 필요한 url과 인증이 필요한 url을 설정한다.  
    - 허가되지 않은 접근 시 401 에러를 리턴한다.  
    - 로그인에 성공하면 특정 url로 리다이렉트 한다.  
      [[참고4](https://growth-coder.tistory.com/135){:target="_blank"}][[참고5](https://spring.io/guides/tutorials/spring-boot-oauth2#_social_login_simple){:target="_blank"}][[참고6](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html#oauth2login-sample-boot){:target="_blank"}][[참고7](https://docs.spring.io/spring-security/reference/servlet/oauth2/index.html#oauth2-client){:target="_blank"}]  
- 설정  
    - 설명  
        - SecurityGroup Configuration을 생성하고  
          하위에 SecurityFilterChain  빈을 생성한다.  
    - 코드  
        - global/config/SecurityGroup.java  
          ```java  
          @Configuration  
          @RequiredArgsConstructor  
          @EnableWebSecurity  
          public class SecurityGroup {  
              @Bean  
              SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
                    
                  // @formatter:off  
                  http  
                      .csrf(AbstractHttpConfigurer::disable)  
                      .authorizeHttpRequests((a) -> a  
                                      // 인증이 필요한 페이지  
                                      .requestMatchers("/private").authenticated()  
                                      // 그 외 나머지 페이지는 인증 없이 접근 가능  
                                      .anyRequest().permitAll()  
                      )  
                       // 허가되지 않은 접근 시 401 에러 리턴  
                      .exceptionHandling(e -> e  
                                      .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))  
                      )  
                      .oauth2Login(l -> l  
                               // 로그인에 성공하면 /success 페이지로 이동  
                              .defaultSuccessUrl("/success")  
                      )  
                      ;  
                  // @formatter:on  
                    
                  return http.build();  
              }  
          }  
          ```  
        - oauth/AuthController.java  
          ```java  
          @RestController  
          public class AuthController {  
              @GetMapping("/auth/success")  
              public String authSucessful() {  
                  return "hello";  
              }  
          }  
          ```  
        - expt/ExptController.java  
          ```java  
          @RestController  
          public class ExptController {  
              @GetMapping("/private")  
              public String private() {  
                  return "private";  
              }  
                    
              @GetMapping("/normal")  
              public String normal() {  
                  return "normal";  
              }  
          }  
          ```  

## 인증(Authentication) 처리
- 개요  
    - OAuth 2.0 로그인에 성공하면 회원정보를 받게 된다.  
        - 예시  
          ```json  
          {  
              "email": "my-email",  
              "sub": "my-openId",  
              "name": "my-name",  
              "locale": "my-locale"  
          }  
          ```  
        - sub  
            - sub은 OpenId로 고유값이다.  
    - OAuth 2.0 회원정보를 바탕으로  
      유저의 DB 등록여부를 판단한다.  
    - 등록되지 않은 유저인 경우, DB에 회원정보를 기록한다.  
    - 위 부분은 OAuth2UserService의 loadUser를   
      Override하여 구현할 수 있다[[참고8](https://codediary21.tistory.com/73){:target="_blank"}].  
- user/User.java  
  ```java  
  @Document(collection = MongodbCollectionName.USER)  
  @Getter  
  @Setter  
  public class User {  
      @Id  
      public String _id;  
            
      @Indexed(unique = true)  
      public String email;  
      public String openId;  
      public String name;  
      public String locale;  
      public String provider;  
  }  
  ```  
- user/UserRepository.java  
  ```java  
  @Repository  
  public interface UserRepository extends MongoRepository<User, String> {  
      public User findByEmail(String email);  
  }  
  ```  
- oauth/OAuth2UserService.java  
  ```java  
  @RequiredArgsConstructor  
  @Service  
  public class OAuth2UserService extends DefaultOAuth2UserService {  
      public final UserRepository userRepository;  
            
      @Override  
      public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {  
          OAuth2User oAuth2User = super.loadUser(userRequest);  
          System.out.println("oAuth2User = " + oAuth2User.getAttributes());  
          Map<String, Object> attributes = oAuth2User.getAttributes();  
            
          // OAuth 2.0 프로바이더 명  
          String provider = userRequest.getClientRegistration().getRegistrationId();  
            
          // 유저 획득  
          User user = saveNewUser(attributes, provider);  
            
          // authorities 설정  
          Set<GrantedAuthority> authorities = new LinkedHashSet<>();  
          authorities.add(new OAuth2UserAuthority(userAttributes));  
          OAuth2AccessToken token = userRequest.getAccessToken();  
          for (String authority : token.getScopes()) {  
              authorities.add(new SimpleGrantedAuthority("SCOPE_" + authority));  
          }  
            
          // 고유값 키 할당; default = "sub"  
          String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint()  
                  .getUserNameAttributeName();  
            
          return new DefaultOAuth2User(authorities, attributes, userNameAttributeName);  
      }  
            
      private User saveNewUser(Map<String, Object> attributes, String provider) {  
          String email = (String) attributes.get("email");  
          String openId = (String) attributes.getOrDefault("sub", "");  
          String name = (String) attributes.getOrDefault("name", "");  
          String locale = (String) attributes.getOrDefault("locale", "");  
            
          User user = userRepository.findByEmail(email);  
          if (user == null) {  
              User newUser = new User();  
              newUser.setOpenId(openId);  
              newUser.setEmail(email);  
              newUser.setName(name);  
              newUser.setLocale(locale);  
              newUser.setProvider(provider);  
              User createdUser = userRepository.save(newUser);  
            
              return createdUser;  
          }  
            
          return user;  
      }  
  }  
  ```  

## Role 기반 인가(Authorization) 처리 
- 개요  
    - authority는 세분화된 접근 권한을 의미한다[[참고9](https://jake-seo-dev.tistory.com/669){:target="_blank"}].  
      ex) 쓰기 권한, 읽기 권한  
    - role은 권한 보다 높은 수준의 권한 그룹이다.  
      ex) 일반유저, 관리자  
    - spring security 6에서는 Role을 체크하여  
      페이지 접근을 제어하는 기능을 제공한다[[참고10](https://jaykaybaek.tistory.com/30){:target="_blank"}].  
    - spring에서 role은 보통 ROLE_로 시작한다.  
- 목표  
    - role은 ROLE_NORMAL, ROLE_ADMIN을 허용한다.  
    - 기본 유저는 role = ROLE_NORMAL 값을 갖는다.  
    - /admin-info 페이지는 role = ROLE_ADMIN 유저만 접근할 수 있다.  
- user/UserRole.java  
  ```java  
  public enum UserRole {  
      ADMIN, NORMAL;  
  }  
  ```  
- user/User.java  
  ```java  
  @Document(collection = MongodbCollectionName.USER)  
  @Getter  
  @Setter  
  public class User {  
      @Id  
      public String _id;  
            
      @Indexed(unique = true)  
      public String email;  
      public String openId;  
      public String name;  
      public String locale;  
      public String provider;  
      // >>>>> 추가 시작 >>>>>  
      public UserRole role;  
      // <<<<< 추가 끝 <<<<<  
  }  
  ```  
- global/config/SecurityGroup.java  
  ```java  
  @Configuration  
  @RequiredArgsConstructor  
  @EnableWebSecurity  
  public class SecurityGroup {  
      @Bean  
      SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
            
          // @formatter:off  
          http  
              .csrf(AbstractHttpConfigurer::disable)  
              .addFilterAfter(new AuthActiveFilter(), AuthorizationFilter.class)  
              .authorizeHttpRequests((a) -> a  
                              .requestMatchers("/private").authenticated()  
                              // >>>>> 추가 시작 >>>>>  
                              .requestMatchers("/admin-info").hasRole(UserRole.ADMIN.toString())  
                              // <<<<< 추가 끝 <<<<<  
                              .anyRequest().permitAll()  
              )  
              ...  
          // @formatter:on  
            
          return http.build();  
      }  
  }  
  ```  
- oauth/OAuth2UserService.java  
  ```java  
  @RequiredArgsConstructor  
  @Service  
  public class OAuth2UserService extends DefaultOAuth2UserService {  
      public final UserRepository userRepository;  
            
      @Override  
      public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {  
          OAuth2User oAuth2User = super.loadUser(userRequest);  
          // System.out.println("oAuth2User = " + oAuth2User.getAttributes());  
          Map<String, Object> attributes = oAuth2User.getAttributes();  
            
          String provider = userRequest.getClientRegistration().getRegistrationId();  
            
          // 유효성 검사  
          validateAttributes(attributes);  
            
          // 유저 획득  
          User user = saveNewUser(attributes, provider);  
            
          // >>>>> 추가 시작 >>>>>  
          // Role 할당  
          Set<GrantedAuthority> authorities = new LinkedHashSet<>();  
          String authority = user.getRole().toString();  
          authorities.add(new SimpleGrantedAuthority("ROLE_" + authority));  
          // <<<<< 추가 끝 <<<<<  
            
          // 고유값 키 할당; default = "sub"  
          String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint()  
                  .getUserNameAttributeName();  
            
          return new DefaultOAuth2User(authorities, attributes, userNameAttributeName);  
      }  
            
      private User saveNewUser(Map<String, Object> attributes, String provider) {  
          String email = (String) attributes.get("email");  
          String openId = (String) attributes.getOrDefault("sub", "");  
          String name = (String) attributes.getOrDefault("name", "");  
          String locale = (String) attributes.getOrDefault("locale", "");  
            
          User user = userRepository.findByEmail(email);  
          if (user == null) {  
              User newUser = new User();  
              newUser.setOpenId(openId);  
              newUser.setEmail(email);  
              newUser.setName(name);  
              newUser.setLocale(locale);  
              newUser.setProvider(provider);  
              // >>>>> 추가 시작 >>>>>  
              newUser.setRole(UserRole.NORMAL);  
              // <<<<< 추가 끝 <<<<<  
              User createdUser = userRepository.save(newUser);  
            
              return createdUser;  
          }  
            
          return user;  
      }  
  }  
  ```  

## Custom 인가 처리
- 개요  
    - User에 isActive 속성을 추가한다.  
    - isActive 속성이 true이면 로그인을 허가하고  
      false 이면 로그인을 불허한다.  
    - servlet filter를 이용하여 구현한다[[참고11](https://curiousjinan.tistory.com/entry/spring-filterchain-dofilter){:target="_blank"}].  
- user/User.java  
  ```java  
   // >>>>> 추가 시작 >>>>>  
  @FieldNameConstants  
  // <<<<< 추가 끝 <<<<<  
  @Document(collection = MongodbCollectionName.USER)  
  @Getter  
  @Setter  
  public class User {  
      @Id  
      public String _id;  
            
      @Indexed(unique = true)  
      public String email;  
      public String openId;  
      public String name;  
      public String locale;  
      public String provider;  
      public UserRole role;  
      // >>>>> 추가 시작 >>>>>  
      public Boolean isActive;  
      // <<<<< 추가 끝 <<<<<  
  }  
  ```  
- oauth/OAuth2UserKey.java  
  ```java  
  public enum OAuth2UserKey {  
      IS_ACTIVE(User.Fields.isActive);  
            
      private String value;  
            
      private OAuth2UserKey(String value) {  
          this.value = value;  
      }  
            
      public String getValue() {  
          return value;  
      }  
  }  
  ```  
- oauth/OAuth2UserService.java  
  ```java  
  @RequiredArgsConstructor  
  @Service  
  public class OAuth2UserService extends DefaultOAuth2UserService {  
      public final UserRepository userRepository;  
            
      @Override  
      public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {  
          OAuth2User oAuth2User = super.loadUser(userRequest);  
          // System.out.println("oAuth2User = " + oAuth2User.getAttributes());  
          Map<String, Object> attributes = oAuth2User.getAttributes();  
            
          String provider = userRequest.getClientRegistration().getRegistrationId();  
            
          // 유효성 검사  
          validateAttributes(attributes);  
            
          // 유저 획득  
          User user = saveNewUser(attributes, provider);  
            
          // Role 할당  
          Set<GrantedAuthority> authorities = new LinkedHashSet<>();  
          String authority = user.getRole().toString();  
          authorities.add(new SimpleGrantedAuthority("ROLE_" + authority));  
            
          // 고유값 키 할당; default = "sub"  
          String userNameAttributeName = userRequest.getClientRegistration().getProviderDetails().getUserInfoEndpoint()  
                  .getUserNameAttributeName();  
            
          // >>>>> 추가 시작 >>>>>  
          attributes.put(OAuth2UserKey.IS_ACTIVE.getValue(), user.getIsActive());  
          // <<<<< 추가 끝 <<<<<  
            
          return new DefaultOAuth2User(authorities, newAttributes, userNameAttributeName);  
      }  
            
      private User saveNewUser(Map<String, Object> attributes, String provider) {  
          String email = (String) attributes.get("email");  
          String openId = (String) attributes.getOrDefault("sub", "");  
          String name = (String) attributes.getOrDefault("name", "");  
          String locale = (String) attributes.getOrDefault("locale", "");  
            
          User user = userRepository.findByEmail(email);  
          if (user == null) {  
              User newUser = new User();  
              newUser.setOpenId(openId);  
              newUser.setEmail(email);  
              newUser.setName(name);  
              newUser.setLocale(locale);  
              newUser.setProvider(provider);  
              newUser.setRole(UserRole.NORMAL);  
              // >>>>> 추가 시작 >>>>>  
              newUser.setIsActive(true);  
              // <<<<< 추가 끝 <<<<<  
              User createdUser = userRepository.save(newUser);  
            
              return createdUser;  
          }  
            
          return user;  
      }  
  }  
  ```  
- oauth/AuthActiveFilter.java  
  ```java  
  public class AuthActiveFilter implements Filter {  
      @Override  
      public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain)  
              throws IOException, ServletException {  
          HttpServletResponse resp = (HttpServletResponse) servletResponse;  
          HttpServletRequest req = (HttpServletRequest) servletRequest;  
          OAuth2AuthenticationToken authToken = (OAuth2AuthenticationToken) req.getUserPrincipal();  
          if (authToken != null) {  
              boolean isActive = authToken.getPrincipal().getAttribute(OAuth2UserKey.IS_ACTIVE.getValue());  
              boolean isAuthenticated = authToken.isAuthenticated();  
            
              // 로그인은 성공, isActivate = false  
              if (isAuthenticated && !isActive) {  
                  resp.sendError(HttpServletResponse.SC_UNAUTHORIZED, "your account has been banned");  
                  return;  
              }  
            
          }  
            
          filterChain.doFilter(servletRequest, servletResponse);  
      }  
  }  
  ```  
- global/config/SecurityGroup.java  
  ```java  
  @Configuration  
  @RequiredArgsConstructor  
  @EnableWebSecurity  
  public class SecurityGroup {  
      @Bean  
      SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
            
          // @formatter:off  
          http  
              .csrf(AbstractHttpConfigurer::disable)  
              // >>>>> 추가 시작 >>>>>  
              .addFilterAfter(new AuthActiveFilter(), AuthorizationFilter.class)  
              // <<<<< 추가 끝 <<<<<  
              .authorizeHttpRequests((a) ->   
                              ...  
              )  
              ...  
          // @formatter:on  
            
          return http.build();  
  ```  
    - AuthActivateFilter는 인증 및 인가 과정 중 가장 마지막에 실행되어야 한다.  
    - @EnableWebSecurity(debug = true) 설정을 추가하면  
      Spring security 6에서 추가한 필터 목록을 볼 수 있다[[참고12](https://www.baeldung.com/spring-security-registered-filters){:target="_blank"}].  
    - 가장 마지막에 추가된 필터(예시에서는 AuthorizationFilter) 실행 후에  
      AuthActivateFilter를 추가하면 된다.  

## 로그아웃
- 개요  
    - 기본 로그아웃 url  
        - /logout  
    - 로그아웃 성공 시 redirect_uri를 지정한다[[참고13](https://docs.spring.io/spring-security/reference/reactive/oauth2/login/logout.html#oauth2login-advanced-oidc-logout){:target="_blank"}].  
    - 해당 페이지에서 세션 쿠키를 제거한다.  
- oauth/AuthController.java  
  ```java  
  @RestController  
  public class AuthController {  
      ....  
       // >>>>> 추가 시작 >>>>>  
      @GetMapping("/auth/logout")  
      public String logoutSucessful() {  
          HttpSession session = request.getSession(false);  
          if (session != null) {  
              session.invalidate();  
          }  
            
          return "logout";  
      }  
       // <<<<< 추가 끝 <<<<<  
  }  
  ```  
- global/config/SecurityGroup.java  
  ```java  
  @Configuration  
  @RequiredArgsConstructor  
  @EnableWebSecurity  
  public class SecurityGroup {  
      @Bean  
      SecurityFilterChain filterChain(HttpSecurity http) throws Exception {  
            
          // @formatter:off  
          http  
              ...  
               // >>>>> 추가 시작 >>>>>  
              .logout(l -> l  
                      .logoutSuccessUrl("/auth/logout")  
              )  
              ;  
               // <<<<< 추가 끝 <<<<<  
          // @formatter:on  
            
          return http.build();  
      }  
  }  
  ```  

## 참고
- [참고1 - OAuth](https://ko.wikipedia.org/wiki/OAuth){:target="_blank"}  
- [참고2 - 클라이언트 측 웹 애플리케이션용 OAuth 2.0 ](https://developers.google.com/identity/protocols/oauth2/javascript-implicit-flow?hl=ko){:target="_blank"}  
- [참고3 - 구글 로그인 쉽게 구현하기 3편 - 로그인 구현하기 (SpringBoot + Vue.js)](https://notspoon.tistory.com/47){:target="_blank"}  
- [참고4 - [Spring] 스프링 Oauth2 구글 로그인과 jpa 사용하여 유저 정보 데이터베이스에 저장 (OAuth2 스프링 1편)](https://growth-coder.tistory.com/135){:target="_blank"}  
- [참고5 - Single Sign On With GitHub](https://spring.io/guides/tutorials/spring-boot-oauth2#_social_login_simple){:target="_blank"}  
- [참고6 - Core Configuration](https://docs.spring.io/spring-security/reference/servlet/oauth2/login/core.html#oauth2login-sample-boot){:target="_blank"}  
- [참고7 - OAuth2 Client](https://docs.spring.io/spring-security/reference/servlet/oauth2/index.html#oauth2-client){:target="_blank"}  
- [참고8 - (Spring Security) OAuth2 서비스 구현 정리](https://codediary21.tistory.com/73){:target="_blank"}  
- [참고9 - 스프링 시큐리티에서 Authority 와 Role 의 차이는?](https://jake-seo-dev.tistory.com/669){:target="_blank"}  
- [참고10 - [Spring Security] 인증(Authentication), 인가(Authorization), 권한(Authority), 역할(Role) 알아보기](https://jaykaybaek.tistory.com/30){:target="_blank"}  
- [참고11 - Spring Boot: 필터에서 doFilter와 FilterChain이란?](https://curiousjinan.tistory.com/entry/spring-filterchain-dofilter){:target="_blank"}  
- [참고12 - Find the Registered Spring Security Filters](https://www.baeldung.com/spring-security-registered-filters){:target="_blank"}  
- [참고13 - OpenID Connect 1.0 Client-Initiated Logout](https://docs.spring.io/spring-security/reference/reactive/oauth2/login/logout.html#oauth2login-advanced-oidc-logout){:target="_blank"}  
