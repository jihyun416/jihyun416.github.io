---
layout: post
title:  Springboot - Spring Security JWT
author: Jihyun
category: springboot
tags:
- springboot
- spring security
- jwt
date: 2021-08-10 09:25 +0900
---

### Spring Security란?

Spring Security는 강력하고 사용자 정의 가능한 인증 및 액세스 제어 프레임워크로, Spring 기반 애플리케이션 보안을 위한 사실상의 표준이다.

Spring Security는 사용자의 인증 도메인에 맞춰 커스텀 할 수 있으며(사용자 아이디, 권한 등), 세션을 활용한 폼로그인 방식, 세션을 이용하지 않는 STATELESS 방식 등으로 구현이 가능하다.

요새 개발의 추세가 백앤드와 프론트앤드를 분리하여 REST API 형태를 활용하는 경우가 많기 때문에, 이 포스팅에서는 JWT를 활용하여 STATELESS 방식으로 인증, 인가를 하는 방법을 기술할 것이다.



### JWT란?

![JWT](https://jihyun416.github.io/assets/springboot_7_1.png)

Json Web Token의 약자로, JSON 객체를 사용하여 가볍고 자가수용적인 (self-contained) 방식으로 정보를 안전성 있게 전달해주기 위한 토큰이다.

header.payload.signature 의 삼단 구조로 구성되며,

header에는 알고리즘과 토큰타입을

payload에는 데이터를

signature에는 secret을 이용하여 만든 서명이 들어있어 토큰의 위변조 여부를 판단할 수 있다.

데이터는 서명의 위변조 여부와 관계 없이 decode가 가능하기 때문에, 노출되었을때 위험한 정보(비밀번호)를 담아서는 안된다.

일반적으로 사용자 식별 정보와 권한을 담아 서명이 검증되었을 경우 따로 DB에 엑세스 할 필요 없이 JWT에 있는 정보를 꺼내서 활용해서 사용한다.



### STATELESS 인증/인가 특징 및 구현 시 주안점

1. 무상태이기 때문에 매번 요청시 마다 인증 시 받았던 토큰을 들고와서 본인이 누구인지 알려야한다.
2. 토큰에는 사용자를 식별할 수 있는 정보와 권한에 대한 정보가 있어야 한다.
3. 서버에서는 발급한 토큰을 별도로 저장하지 않는다.
4. 토큰이 노출될 수 있으므로 payload에 만료시간을 담아 일정 시간만 사용 가능하도록 한다.
5. 매번 토큰을 요청할 수 없으므로 유효시간이 짧은 access token과 refresh token을 쌍으로 발급하고, 매 요청시에는 access token을 사용하고, 만료 시 refresh token을 통해 토큰을 재발급 받는다. 클라이언트는 refresh token은 안전한 저장소에 보관해야 한다.



## 1. build.gradle dependency 추가 : spring security, jsonwebtoken

```yaml
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-security'
    implementation 'io.jsonwebtoken:jjwt:0.9.1'
}
```

- Spring security를 위한 의존성 추가

  - 의존성을 추가하는 것 만으로도 프로젝트 전역에 시큐리티가 설정되면서 시큐리티 폼로그인 창이 설정될 것이다. 

    *시큐리티를 바로 구현 안할건데 추가하면 이게 뭐지 할 수 있으니 주의할 것*

- JWT를 위한 의존성 추가



## 2. application.yml에 토큰 생성에 필요한 Value 정의

```yaml
jwt:
  secret: [your secret]
  access:
    name: access
    valid: 3600000
  refresh:
    name: refresh
    valid: 2592000000
```

- jwt.secret : jwt 생성 시 서명을 만들기 위한 secret 정의, 남들이 알 수 없는 문자로 구성한다.
- jwt.access.name, jwt.refresh.name : 토큰 구분을 위한 값 정의
- jwt.access.valid, jwt.refresh.valid : 토큰 만료 시간 정의 (밀리초 단위)



## 3. SecurityUser

```java
public class SecurityUser extends User {
    private static final String ROLE_PREFIX = "ROLE_";

    public SecurityUser(String userId, List<String> roles) {
        super(userId, userId, makeGrantedAuthority(roles));
    }

    private static List<GrantedAuthority> makeGrantedAuthority(List<String> roles) {
        List<GrantedAuthority> list = new ArrayList<>();
        for (String role : roles) {
            list.add(new SimpleGrantedAuthority(ROLE_PREFIX + role));
        }
        return list;
    }
}
```

- Security에서 인가 사용자 정보를 다룰 때 User 객체를 사용하는데, 사용자의 환경에 맞게 커스텀 하려면 User를 상속받아 커스텀 해야한다.
- User : org.springframework.security.core.userdetails.User *(내가 만든 유저 도메인 아님 주의!!)*

- JWT 토큰에서 파싱한 아이디(userId)와 권한(roles)을 파라미터로 받아 생성할 계획
- makeGrantedAuthority를 통해 일반 문자열을 security가 취급하는 권한 클래스 리스트로 변환한다.



## 4. JwtTokenProvider 구현

```java
@Component
@NoArgsConstructor
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String secretKey;
    @Value("${jwt.access.name}")
    public String ACCESS_TOKEN;
    @Value("${jwt.refresh.name}")
    public String REFRESH_TOKEN;
    @Value("${jwt.access.valid}")
    public Long ACCESS_VALID;
    @Value("${jwt.refresh.valid}")
    public Long REFRESH_VALID;

    @PostConstruct
    protected void init() {
        secretKey = Base64.getEncoder().encodeToString(secretKey.getBytes());
    }

    // Access Token 생성
    public String createAccessToken(String subject, List<String> roles) {
        Claims claims = Jwts.claims().setSubject(subject);
        claims.put("type", ACCESS_TOKEN);
        claims.put("roles", roles);
        Date now = new Date();
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(new Date(now.getTime() + ACCESS_VALID))
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .compact();
    }

    // Refresh Token 생성
    public String createRefreshToken(String subject, List<String> roles) {
        Claims claims = Jwts.claims().setSubject(subject);
        claims.put("type", REFRESH_TOKEN);
        claims.put("roles", roles);
        Date now = new Date();
        return Jwts.builder()
                .setClaims(claims)
                .setIssuedAt(now)
                .setExpiration(new Date(now.getTime() + REFRESH_VALID))
                .signWith(SignatureAlgorithm.HS256, secretKey)
                .compact();
    }

    // Request의 Header에서 token 가져오기
    public String resolveToken(HttpServletRequest request) {
        String bearer = request.getHeader(HttpHeaders.AUTHORIZATION);
        if(!CollectionUtil.isEmpty(bearer)) return bearer.replace("Bearer ", "");
        else return null;
    }

    // subject 추출 (userId)
    public String getSubject(String token) {
        return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().getSubject();
    }

    // role 추출 (권한 리스트)
    public List<String> getRoles(String token) {
        return Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token).getBody().get("roles", ArrayList.class);
    }
  
    // Jwt 토큰 만료 정보
    public String getExp(String token) {
        Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
        return String.valueOf(claims.getBody().getExpiration().getTime());
    }

    // Jwt 토큰으로 인증 정보 반환
    public Authentication getAuthentication(String token) {
        String userId = this.getSubject(token);
        List<String> roles = this.getRoles(token);
        UserDetails userDetails = new SecurityUser(userId, roles);
        return new UsernamePasswordAuthenticationToken(userDetails, "", userDetails.getAuthorities());
    }

    // Access token 유효성 체크
    public void validateAccessToken(String token) throws Exception {
        Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
        if(!claims.getBody().get("type", String.class).equals(ACCESS_TOKEN)) {
            throw new InvalidAccessTokenException("Access Token이 아닙니다.");
        }
    }

    // Refresh token 유효성 체크
    public void validateRefreshToken(String token) throws Exception {
        Jws<Claims> claims = Jwts.parser().setSigningKey(secretKey).parseClaimsJws(token);
        if(!claims.getBody().get("type", String.class).equals(REFRESH_TOKEN)) {
            throw new InvalidRefreshTokenException("Refresh Token이 아닙니다.");
        }
    }
}
```

- @PostConstruct init() : value에서 가져온 값은 평문이기 때문에 토큰 생성시에는 Base64로 인코딩 된 secret을 사용할 것이라서 빈 생성 이후 인코딩 처리하여 다시 저장한다.
- createAccessToken : Access Token을 생성한다.

  - subject : 식별자 (User Id) 저장
  - type : 토큰 타입 저장 (엑세스 토큰임을 알림, 리프래시 토큰으로 데이터 접근 못하도록 하기 위함)
  - roles : 권한 리스트
  - issueAt : 토큰 발행일시 (iat), 현재 시간
  - expiration : 토큰 만료일시 (exp), 현재 시간에서 지정한 유효시간만큼 더한다.
  - sigWith : 암호화 알고리즘, secret 값 세팅
- createRefreshToken : Refresh Token을 생성한다.

  - Access Token과 동일한 방식으로 토큰 타입을 리프래시로, 유효일시를 리프래시 유효시간을 더해 만든다.
- resolveToken : Header에서 토큰 정보를 가져온다. Bearer 방식을 이용할 것이므로,  Authorization 헤더를 가져온 뒤 "bearer " 부분을 제거해준다.
- getSubject : 사용자 식별자(sub)를 가져온다.
- getRoles : 권한 리스트(roles)를 가져온다.
- getExp : 만료일시 (exp)를 가져온다.
- getAuthentication : 토큰에서 정보를 파싱하여 SecurityUser를 생성 후 Authentication 객체를 반환한다.
- validateAccessToken : 엑세스 토큰의 유효성을 체크한다.

  - JWT format 체크 : header.payload.signature 의 구조가 아니면 MalformedJwtException 이 발생한다.
  - 유효기간 체크 : 유효기간이 지났으면 ExpiredJwtException 이 발생한다.
  - 토큰 변조 여부 체크 : 서명 부분이 변조되었거나 다른 secret으로 생성되었다면 SignatureException 이 발생한다.
  - 엑세스 토큰 여부 체크 : 엑세스 토큰이 아닐 경우 InvalidAccessTokenException 를 발생시켰다.

- validateRefreshToken : 리프레시 토큰의 유효성을 체크한다.



## 5. JwtAuthenticationFilter 구현

```java
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends GenericFilterBean {
    private final JwtTokenProvider jwtTokenProvider;

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain filterChain) throws IOException, ServletException, InvalidAccessTokenException {
        String token = jwtTokenProvider.resolveToken((HttpServletRequest) request);
        if (!CollectionUtil.isEmpty(token)) {
            try {
                jwtTokenProvider.validateAccessToken(token);
                Authentication auth = jwtTokenProvider.getAuthentication(token);
                SecurityContextHolder.getContext().setAuthentication(auth);
            } catch (Exception e) {
                request.setAttribute("exception", e);
            }
        }
        filterChain.doFilter(request, response);
    }
}
```

- Security Config 설정 시 태울 필터를 구현한다.
- resolveToken으로 토큰을 가져온 뒤 validateAccessToken으로 유효성을 체크한다.
- 유효성 체크 시 이상이 없으면 Authentication을 생성하여 SecurityContextHolder에 넣는다.
- 유효성 체크에 이상이 있을 경우 인가가 필요한 경우에만 에러가 발생하도록 익셉션을 request attribute에 넣고 넘어가도록 처리한다.
  - 익셉션 정보를 넘겨주지 않을 경우 단순히 인가 컨텍스트가 없어서 401 에러가 난다는 정보만 볼 수 있기 때문에, 토큰의 문제일 경우 문제를 정확히 알리기 위해 다음과 같은 처리를 했다.
  - 그냥 401만 있으면 되면 별도로 처리하지 않아도 괜찮다



## 6. AuthenticationEntryPoint 구현체 생성

```java
@Component
public class CustomAuthenticationEntryPoint implements AuthenticationEntryPoint {
    @Override
    public void commence(HttpServletRequest request, HttpServletResponse response, AuthenticationException ex) throws IOException {
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpStatus.UNAUTHORIZED.value());
        try (OutputStream os = response.getOutputStream()) {
            String message = ex.getClass().getName();
            String data = ex.getMessage();
            if(request.getAttribute("exception")!=null) {
                Exception e = (Exception) request.getAttribute("exception");
                message = e.getClass().getName();
                data = e.getMessage();
            }
            ObjectMapper objectMapper = new ObjectMapper();
            objectMapper.writeValue(os, ResponseDTO.builder()
                    .result(false)
                    .status(HttpStatus.UNAUTHORIZED.value())
                    .message(message)
                    .data(data)
                    .build());
            os.flush();
        }
    }
}
```

- 인가가 필요한 경로에 인가 없이 접근 시에 타는 handler이다.
- 이 처리는 Controller에 가기도 전에 되는 부분이기 때문에, @ControllerAdvice에서 정의한 ExceptionHandler로는 처리가 불가하다. 따라서 response에 강제로 원하는 형태의 응답을 write, flush 한다.



## 7. AccessDeniedHandler 구현체 생성

```java
@Slf4j
public class CustomAccessDeniedHandler implements AccessDeniedHandler {
    @Override
    public void handle(HttpServletRequest request, HttpServletResponse response, AccessDeniedException accessDeniedException) throws IOException, ServletException {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null) {
            log.info("User: " + auth.getName() + " attempted to access the protected URL: " + request.getRequestURI());
        }
        response.setContentType(MediaType.APPLICATION_JSON_VALUE);
        response.setStatus(HttpStatus.FORBIDDEN.value());
        try (OutputStream os = response.getOutputStream()) {
            ObjectMapper objectMapper = new ObjectMapper();
            objectMapper.writeValue(os, ResponseDTO.builder()
                    .result(false)
                    .status(HttpStatus.FORBIDDEN.value())
                    .message(accessDeniedException.getClass().getName())
                    .data(accessDeniedException.getMessage())
                    .build());
            os.flush();
        }
    }
}
```

- 특정 권한이 필요한 경로에 권한 없이 접근 시에 타는 handler이다.
- 마찬가지로 이 처리는 Controller에 가기도 전에 되는 부분이기 때문에, @ControllerAdvice에서 정의한 ExceptionHandler로는 처리가 불가하다. 따라서 response에 강제로 원하는 형태의 응답을 write, flush 한다.



## 8. SecurityConfig 정의 - Security 설정

```java
@Configuration
@RequiredArgsConstructor
@EnableGlobalMethodSecurity(prePostEnabled = true)
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {
    private final JwtTokenProvider jwtTokenProvider;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.addAllowedOrigin(CorsConfiguration.ALL);
        configuration.addAllowedMethod("GET");
        configuration.addAllowedMethod("POST");
        configuration.addAllowedMethod("PUT");
        configuration.addAllowedMethod("DELETE");
        configuration.addAllowedHeader(CorsConfiguration.ALL);
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        http
                .httpBasic().disable()
                .csrf().disable()
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
                .and()
                .authorizeRequests()
                .antMatchers("/exception/**",
                        "/sign/**"
                ).permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
                .and().cors().configurationSource(source)
                .and().exceptionHandling().authenticationEntryPoint(new CustomAuthenticationEntryPoint())
                .and().exceptionHandling().accessDeniedHandler(new CustomAccessDeniedHandler())
                .and().addFilterBefore(new JwtAuthenticationFilter(jwtTokenProvider), BasicAuthenticationFilter.class)
                .headers().frameOptions().disable();
    }

    @Override // ignore check swagger resource
    public void configure(WebSecurity web) {
        web.ignoring().antMatchers("/swagger", "/swagger-ui/**", "/v3/api-docs/**");
    }
}
```

- Cors : 모두 허용, Csrf 사용 안함, frameOption 사용 안함
- Http Method : GET, POST, PUT, DELETE 허용

- STATELESS 으로 선언하여 세션을 사용하지 않는다.
- permitAll : 앞의 antMatcher에서 기술한 패스는 인가 없이 모두 접근 허가 *(로그인, 회원가입, 토큰재요청 등은 모두 허용 해야 함)*
- hasRole : 앞의 antMatcher에서 기술한 패스는 특정 권한이 있을때만 접근 허가
- authenticated : 인가 되어야 접근 허가
- authenticationEntryPoint : 인가가 필요한 경우에 인가정보가 없을 경우 처리해줄 handler (앞서 정의한 CustomAuthenticationEntryPoint 사용)
- accessDeniedHandler : 특정 권한이 필요한 경우 권한이 없을 경우 처리해줄 handler (앞서 정의한 CustomAccessDeniedHandler 사용)
- 정의한 JwtAuthenticationFilter 를 BasicAuthenticationFilter 전에 타도록 등록



## 9. 가입 / 로그인 / 토큰리프레시

```java
public class SignService {
    private final UserRepository userRepository;
    private final UserAuthorityRepository userAuthorityRepository;
    private final AuthorityRepository authorityRepository;
    private final JwtTokenProvider jwtTokenProvider;

    @Transactional
    public ResponseDTO signUp(SignUpDTO signUpDTO) {
        Optional<User> mightUser = userRepository.findByUserIdEquals(signUpDTO.getUserId());
        if(mightUser.isPresent()) {
            return ResponseDTO.builder()
                    .status(HttpStatus.OK.value())
                    .result(false)
                    .message("이미 가입된 아이디 입니다.")
                    .build();
        } else {
            BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
            User savedUser = userRepository.save(User.builder()
                    .userId(signUpDTO.getUserId())
                    .userName(signUpDTO.getUserName())
                    .password(passwordEncoder.encode(signUpDTO.getPassword()))
                    .build());
            if(signUpDTO.getAuthority()!=null && !signUpDTO.getAuthority().isEmpty()) {
                Optional<Authority> mightAuthority = authorityRepository.findById(signUpDTO.getAuthority());
                mightAuthority.ifPresent(auth -> {
                  userAuthorityRepository.save(UserAuthority.builder().authority(auth).user(savedUser).build());
                });
            }
            return ResponseDTO.builder()
                    .status(HttpStatus.OK.value())
                    .result(true)
                    .message("가입이 완료되었습니다.")
                    .build();
        }
    }

    @Transactional(readOnly = true)
    public TokenDTO signIn(SignInDTO signInDTO) {
        BCryptPasswordEncoder passwordEncoder = new BCryptPasswordEncoder();
        Optional<User> mightUser = userRepository.findByUserIdEquals(signInDTO.getUserId());
        if(mightUser.isPresent()) {
            User user = mightUser.get();
            if(passwordEncoder.matches(signInDTO.getPassword(), user.getPassword())) {
                ArrayList<String> roles = new ArrayList<>();
                for (UserAuthority auth : user.getAuthorities()) {
                    roles.add(auth.getAuthority().getAuthorityId());
                }
                //jwt 발급
                String accessToken = jwtTokenProvider.createAccessToken(user.getUserId(), roles);
                String refreshToken = jwtTokenProvider.createRefreshToken(user.getUserId(), roles);
                return TokenDTO.builder()
                        .result(true)
                        .message("로그인에 성공했습니다.")
                        .accessToken(accessToken)
                        .refreshToken(refreshToken)
                        .accessTokenExp(jwtTokenProvider.getExp(accessToken))
                        .refreshTokenExp(jwtTokenProvider.getExp(refreshToken))
                        .build();
            } else {
                return TokenDTO.builder().result(false).message("가입하지 않은 이메일이거나, 잘못된 비밀번호입니다.").build();
            }
        } else {
            return TokenDTO.builder().result(false).message("가입하지 않은 이메일이거나, 잘못된 비밀번호입니다.").build();
        }
    }

    @Transactional(readOnly = true)
    public TokenDTO refreshToken(HttpServletRequest request) throws Exception {
        String token = jwtTokenProvider.resolveToken(request);
        if(!CollectionUtil.isEmpty(token)) {
            jwtTokenProvider.validateRefreshToken(token);
            Optional<User> mightUser = userRepository.findByUserIdEquals(jwtTokenProvider.getSubject(token));
            if(mightUser.isPresent()) {
                User user = mightUser.get();
                ArrayList<String> roles = new ArrayList<>();
                for (UserAuthority auth : user.getAuthorities()) {
                    roles.add(auth.getAuthority().getAuthorityId());
                }
                //jwt 발급
                String accessToken = jwtTokenProvider.createAccessToken(user.getUserId(), roles);
                String refreshToken = jwtTokenProvider.createRefreshToken(user.getUserId(), roles);
                return TokenDTO.builder()
                        .result(true)
                        .message("로그인에 성공했습니다.")
                        .accessToken(accessToken)
                        .refreshToken(refreshToken)
                        .accessTokenExp(jwtTokenProvider.getExp(accessToken))
                        .refreshTokenExp(jwtTokenProvider.getExp(refreshToken))
                        .build();
            } else {
                throw new InvalidRefreshTokenException("유효하지 않은 아이디 입니다.");
            }
        } else {
            throw new InvalidRefreshTokenException("Refresh Token이 필요합니다.");
        }
    }
}
```

- Security를 활용한 Form Login 방식이 아니기 때문에 로그인을 별도로 구현해주어야 함
- Security에서 제공하는 **BCryptPasswordEncoder**를 활용하여 가입 시 비밀번호를 **encode**하여 저장하고, 로그인 시 **matches**를 이용하여 일치 여부를 확인한다. *(복호화는 불가능하다! matches를 이용하여 평문으로 받은 암호와 저장된 암호를 비교할 수 있다.)*
- Refresh의 경우 Authorization에 access token 대신 refresh token을 받아 해당 값으로 refresh token 발급 절차를 밟도록 구현하였다. 이 안에서 발생하는 Exception은 컨트롤러에서 발생하는 Exception이므로 ControllerAdvice에서 적절한 헤더와 함께 처리되도록 추가 구현이 필요하다.

```java
@RequiredArgsConstructor
@RequestMapping("/sign")
@RestController
public class SignController {
    private final SignService signService;

    @PostMapping("/up")
    public ResponseDTO signUp(@RequestBody SignUpDTO signUpDTO) {
        return signService.signUp(signUpDTO);
    }

    @PostMapping("/in")
    public TokenDTO signIn(@RequestBody SignInDTO signInDTO) {
        return signService.signIn(signInDTO);
    }
  
    @PostMapping(value="/refreshToken")
    public TokenDTO refreshToken(HttpServletRequest request) throws Exception {
        return signService.refreshToken(request);
    }
}
```



### +Extra : Swagger에 인증 헤더 추가

```java
@Configuration
public class SwaggerConfig {
    @Bean
    public OpenAPI userAPI() {
        SecurityScheme securityScheme = new SecurityScheme()
                .type(SecurityScheme.Type.HTTP).scheme("bearer").bearerFormat("JWT")
                .in(SecurityScheme.In.HEADER).name(HttpHeaders.AUTHORIZATION);
        SecurityRequirement schemaRequirement = new SecurityRequirement().addList("bearerAuth");
        return new OpenAPI()
                .components(new Components().addSecuritySchemes("bearerAuth", securityScheme))
                .security(Arrays.asList(schemaRequirement))
                .info(new Info()
                        .title("User API")
                        .description("User application")
                        .version("v1"));
    }
}
```

- bearer 인증 방식 추가해주면 [Authorize] 버튼이 생긴다.
- 버튼을 눌러 얻은 토큰을 넣어주면 로그인이 되며 Swagger 이용 시 Authorization 헤더에 해당 bearer 정보가 따라다닌다. *(매번 넣을 필요 없이 편하다!)*





## 10. 테스트

#### 1) permitAll 로 지정한 로그인이 토큰 없이 잘 되는지 확인

![permitAll](https://jihyun416.github.io/assets/springboot_7_2.png)

#### 2) authenticticated 로 지정한 조회가 토큰 없이 안되는지 확인 (401)

![401](https://jihyun416.github.io/assets/springboot_7_3.png)

#### 3) authenticticated 로 지정한 조회가 토큰이 있을때 되는지 확인

![200](https://jihyun416.github.io/assets/springboot_7_4.png)

#### 4) hasRole로 지정한 조회가 권한이 없을때 안되는지 확인 (403)

![403](https://jihyun416.github.io/assets/springboot_7_5.png)

#### 5) hasRole로 지정한 조회가 권한이 있을때 되는지 확인

![200](https://jihyun416.github.io/assets/springboot_7_6.png)

#### 6) 형식에 맞지 않는 JWT 를 넣었을 때 오류가 나는지 확인

![malformat](https://jihyun416.github.io/assets/springboot_7_7.png)

#### 7) 서명부분을 변조했을 때 오류가 나는지 확인

![signature](https://jihyun416.github.io/assets/springboot_7_8.png)

#### 8) 유효기간 지난 토큰을 넣었을 때 오류가 나는지 확인

![expire](https://jihyun416.github.io/assets/springboot_7_9.png)

#### 9) Refresh Token을 넣고 조회를 했을 때 오류가 나는지 확인

![invalid](https://jihyun416.github.io/assets/springboot_7_10.png)

#### 10) Refresh Token을 넣고 토큰 재발급 요청을 했을 시 정상 처리 되는지 확인

![refresh](https://jihyun416.github.io/assets/springboot_7_11.png)

#### 11) Access Token을 넣고 토큰 재발급 요청을 했을 시 오류가 나는지 확인

![invalid](https://jihyun416.github.io/assets/springboot_7_12.png)

##### 

#### 참고

> [Spring.io Spring Security](https://spring.io/projects/spring-security)
>
> [geunwoobaek JWT란](https://velog.io/@geunwoobaek/JWT%EB%9E%80)

