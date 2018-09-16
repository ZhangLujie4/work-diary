# 缘由

最近的项目里遇到了spring security的内容。

通过查找资料，敲demo练习，做出了如下归纳总结。

# 实现spring-security-jwt

项目地址：https://github.com/ZhangLujie4/spring-security-jwt-demo

首先理一下大致思路（权限控制如何实现？）

+ 用户登录验证账号密码，根据用户生成jwt
+ 每次rest请求时会有filter对jwt进行解析验证，成功后保存这个Authentication（这一步就是权限控制）
+ 可以根据这个Authentication获得用户的标识信息

### 核心类

### 1.security的配置类（继承WebSecurityConfigurerAdapter）

在security的配置类里面主要实现三个configure方法

+ configure(AuthenticationManagerBuilder auth) **user-detail相关配置**
+ configure(WebSecurity web) **一般用来设置不需要权限过滤的文件**
+ configure(HttpSecurity http) **核心，用来进行权限控制，过滤设置等重要的配置**

#### 1.1	configure(AuthenticationManagerBuilder auth)

常用的userdetails的相关配置有:

```java
/** planA **/:
@Autowired
public void configureGlobal( AuthenticationManagerBuilder auth ) throws Exception {
    auth.userDetailsService( jwtUserDetailsService ) //jwtUserDetailsService为自定义UserDetailsService(继承于UserDetailsService)
        .passwordEncoder( passwordEncoder() );
}

/** planB **/
@Override
public void configure(AuthenticationManagerBuilder auth) throws Exception {
    // 使用自定义身份验证组件
    auth.authenticationProvider(new CustomAuthenticationProvider(userDetailsService, bCryptPasswordEncoder));
}
```

#### 1.2	configure(WebSecurity web) 

在这里可以举个例子，在demo中并未用到：

```java
@Override
    public void configure(WebSecurity web) throws Exception {
        // TokenAuthenticationFilter will ignore the below paths
        web.ignoring().antMatchers(
                HttpMethod.POST,
                "/auth/login"
        );
        web.ignoring().antMatchers(
                HttpMethod.GET,
                "/",
                "/webjars/**",
                "/*.html",
                "/favicon.ico",
                "/**/*.html",
                "/**/*.css",
                "/**/*.js"
            );

    }
```

这里可以提以下web.ignoring()和http的permitAll()的区别：

WebSecurity主要是配置跟web资源相关的，比如css、js、images等等，但是这个还不是本质的区别，关键的区别如下：

+ 前者设置后不经过spring security的过滤器
+ 后者还是需要经过spring security的过滤器(其中包含登录的和匿名的)。

#### 1.3	configure(HttpSecurity http) 	

我的demo中实现了第三个configure，具体如下：

```java
@Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .csrf().disable()
            .headers().frameOptions().disable().and()
            .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .authorizeRequests()
            .antMatchers("/api/user/**").permitAll()
            .antMatchers("/api/admin/**").hasAuthority("ROLE_ADMIN")
            .and()
            .addFilterBefore(new JwtFilter(tokenProvider), UsernamePasswordAuthenticationFilter.class);
    }
```

addFilterBefore主要用来实现自定义jwtfilter的设置。

当然也可以用apply()方法添加一些配置（其中包含jwtfilter）。

### 2.自定义的Filter

#### 2.1

demo中的filter实现具体如下：

```java
/**
 * @author tori
 * 2018/7/30 下午2:06
 */
public class JwtFilter extends GenericFilterBean {

    public static final String AUTH_HEADER = "Authentication";
    public static final String AUTH_PREFIX = "Bearer ";

    private TokenProvider tokenProvider;

    public JwtFilter(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }

    @Override
    public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain filterChain) throws IOException, ServletException {
        HttpServletRequest httpServletRequest = (HttpServletRequest)servletRequest;
        String jwt = getAuthHeader(httpServletRequest);
        if (StringUtils.hasText(jwt)) {
            if (tokenProvider.validateToken(jwt)) {
                Authentication authentication = tokenProvider.getAuthentication(jwt);
                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }
        filterChain.doFilter(servletRequest, servletResponse);
    }

    private String getAuthHeader(HttpServletRequest request) {
        String bearerToken = request.getHeader(AUTH_HEADER);
        String token = null;
        if (bearerToken != null && bearerToken.startsWith(AUTH_PREFIX)) {
            token = bearerToken.substring(7);
        }
        return token;
    }
}
```

filter的作用：

+ 读取到请求头中的授权的token

+ 从token解析出Authentication对象，将其保存到上下文里

  `SecurityContextHolder.getContext().setAuthentication(authentication)`

+ Authentication作为之后判断用户权限的依据。

一般情况下我们自定义一个类实现Authentication接口，这样我们就可以通过解析Authentication获取到我们需要的成员属性值了。(getPrincipal)

**当然，你也可以选择用其他方式实现自定义的Authentication**

**步骤一**

**一个UserDetails的实现类**

```JAVA
public class User implements UserDetails {
    ......
}
```

**步骤二 **

**一个AbstractAuthenticationToken的继承类**

```java
public class TokenBasedAuthentication extends AbstractAuthenticationToken {

    private String token;
    private final UserDetails principle;
    ......
}
```

因为AbstractAuthenticationToken也是实现了Authentication，所以这个类就可以当做Authentication作为jwt的解析对象。

**补充**

```java
@Autowired
public void configureGlobal( AuthenticationManagerBuilder auth ) throws Exception {
    auth.userDetailsService( jwtUserDetailsService )
        .passwordEncoder( passwordEncoder() );
}
```

这里的jwtUserDetailsService是一个实现了UserDetailsService的类，可以在里面做权限对象相关的操作(修改密码，通过username查询数据库等)

```java
@Autowired
private PasswordEncoder passwordEncoder;

@Autowired
private AuthenticationManager authenticationManager;
```

这两个是比较常用的自动装载的bean

### 3.TokenProvider

这个当然是最重要的了，用来提供生成和解析token方法的类

#### 生成token

```java
public String createToken(UserAuthentication userAuthentication) {
        String authorities = userAuthentication.getAuthorities().stream()
                .map(auth -> auth.getAuthority()).collect(Collectors.joining(","));

    long now = (new Date()).getTime();

    return JwtFilter.AUTH_PREFIX
        + Jwts.builder().setId(String.valueOf(userAuthentication.getUserId()))
        .setSubject(JSON.toJSONString(userAuthentication.getPrincipal()))//在这里可以存入序列化后的对象
        .claim(AUTHORITY_KEY, authorities)//这里可以存入key和value，value代表权限role的list
        .signWith(SignatureAlgorithm.HS512, SECRET_KEY) //秘钥和加密算法
        .setExpiration(new Date(now + EXPIRES_IN))//设置过期时间
        .compact();
}
```

#### 解析token获得Authentication

```java
public Authentication getAuthentication(String jwt) {
    Claims claims = Jwts.parser().setSigningKey(SECRET_KEY).parseClaimsJws(jwt).getBody();
    Collection<? extends GrantedAuthority> authorities = Arrays.asList(claims.get(AUTHORITY_KEY).toString().split(","))
        .stream().map(authority -> new SimpleGrantedAuthority(authority)).collect(Collectors.toList());
    SecurityUser user = JSON.parseObject(claims.get(Claims.SUBJECT).toString(), SecurityUser.class);
    user.setAuthorities(authorities);
    UserAuthentication authentication = new UserAuthentication(user);

    return authentication;
}
```

我这里是token可以解析出整个对象，你也可以用token仅仅解析出username(唯一标识)，用于后续的授权处理操作。



以上。
