#springboot oauth2 授权码模式与密码模式

## oauth2简介

OAuth是一个关于授权（authorization）的开放网络标准，在全世界得到广泛应用，目前的版本是2.0版。

oauth2支持授权的方式有四种：授权码模式(authorization_code)、密码模式(password)、隐式模式(implicit)、客户端模式(client_credentials)。其中，比较常见的就是授权码模式和密码模式。

**授权码模式过程**：

1. 封装参数，访问授权服务器登录与授权接口
      接口：http://localhost:8080/oauth/authorize
      参数：response_type client_id scope redirect_uri state
      返回值：code
2. 拿到code，获取token
      接口：http://localhost:8080/oauth/token
      参数：client_id client_secret grant_type code redirect_uri state
      返回值：access_token
3. 根据token，访问资源
      接口：http://localhost:8080/api/test/hello
      参数：access_token

**密码模式的过程**：

1. 根据用户名密码等参数直接获取token
      接口：http://localhost:8080/oauth/token
      参数：username password grant_type client_id client_secret redirect_uri
      返回值：access_token

2. 根据token，访问资源
      接口：http://localhost:8080/api/test/hello
      参数：access_token 

   可以看出，授权码模式和密码模式有些区别，授权码模式多了一步就是登陆。密码模式直接把用户名和密码交给授权服务器了，所以不用再人为登陆，这也要求用户非常信任该应用。

##具体过程

项目依赖主要是security和oauth2：

```xml
<parent>
	<groupId>org.springframework.boot</groupId>
	<artifactId>spring-boot-starter-parent</artifactId>
	<version>2.1.4.RELEASE</version>
</parent>
<dependencies>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-web</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-security</artifactId>
	</dependency>
	<dependency>
		<groupId>org.springframework.security.oauth</groupId>
		<artifactId>spring-security-oauth2</artifactId>
		<version>2.3.4.RELEASE</version>
	</dependency>
</dependencies>
```

主要代码：

AuthServerConfiguration.java

```java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfiguration extends AuthorizationServerConfigurerAdapter{
	@Autowired
	private AuthenticationManager authenticationManager;
	@Autowired
	private PasswordEncoder passwordEncoder;
	
	@Override
	public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
		endpoints.authenticationManager(authenticationManager);
		endpoints.allowedTokenEndpointRequestMethods(HttpMethod.GET,HttpMethod.POST);
	}
	
	@Override
	public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
		security.realm("oauth2-resources")
				.tokenKeyAccess("permitAll()")
				.checkTokenAccess("isAuthenticated()")
				.allowFormAuthenticationForClients();
	}
	//授权配置，这里指定了四种授权方式，外加一个刷新令牌(refresh_token)的方式，这个是在令牌(token)失效的情况下无需重新走一遍全部流程，只需要做一次刷新请求即可获得新的access_token。 
	@Override
	public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
			   .withClient("client")
			   .secret(passwordEncoder.encode("secret"))
			   .redirectUris("http://example.com")
			   .authorizedGrantTypes("authorization_code","password","refresh_token","implicit","client_credentials")
			   .scopes("all")
			   .autoApprove(true)
			   .resourceIds("oauth2-resource")
			   .accessTokenValiditySeconds(1200)
			   .refreshTokenValiditySeconds(50000);
	}
}
```

ResourceServerConfiguration.java

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfiguration extends ResourceServerConfigurerAdapter{
	@Override
	public void configure(HttpSecurity http) throws Exception {
		http.authorizeRequests()
			.antMatchers("/api/**").hasRole("ADMIN")
			.antMatchers("/test/**").authenticated()
			.anyRequest().authenticated();
	}
}
//资源服务器设置了/api/**下的接口访问需要用户登录授权，还需要ADMIN角色，而/test/**仅仅需要用户登录授权即可。
```


SecurityConfiguration.java

```java
@Configuration
@EnableWebSecurity
//@Order(1)
public class SecurityConfiguration extends WebSecurityConfigurerAdapter{
	@Bean
	public PasswordEncoder passwordEncoder() {
		return new BCryptPasswordEncoder();
	}
	
	@Override
	@Bean
	public AuthenticationManager authenticationManagerBean() throws Exception {
		return super.authenticationManagerBean();
	}
	
    //Security配置了两个用户，一个ADMIN角色，另外一个USER角色，稍作区分，为后面的测试做准备。
	@Override
	protected UserDetailsService userDetailsService() {
		InMemoryUserDetailsManager manager = new InMemoryUserDetailsManager();
		manager.createUser(User.withUsername("admin").password(passwordEncoder().encode("admin")).roles("ADMIN").build());
		manager.createUser(User.withUsername("user").password(passwordEncoder().encode("123456")).roles("USER").build());
		return manager;
	}
	
	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(userDetailsService()).passwordEncoder(passwordEncoder());
	}
	
	
	@Override
	protected void configure(HttpSecurity http) throws Exception {
		http.httpBasic().and()
			.authorizeRequests()
			.antMatchers("/oauth/**","/login")
			.permitAll()
			.anyRequest()
			.authenticated()
			//.and()
			//.formLogin()
			.and()
			.csrf().disable();
	}
	
}
```

​	这里两个测试接口，分别对应资源服务中设置的权限/api/hello不仅需要用户登录授权，还需要ADMIN角色，而/test/hello只需要用户登录授权即可，不需要角色。 

HelloController.java

```java
@RestController
@RequestMapping("/api")
public class HelloController {
	@GetMapping("/hello")
	public String hello() {
		return "hello,authenticated() with role ADMIN.";
	}
}
```

TestController.java

```java
@RestController
@RequestMapping("/test")
public class TestController {
	@GetMapping("/hello")
	public String hello() {
		return "hello,authenticated().";
	}
}
```



##测试

直接访问接口：http://localhost:8080/test/hello或者http://localhost:8080/api/test/hello

显示没有授权，因为都需要access_token参数才能访问

###授权码模式：

访问接口：<http://localhost:8080/oauth/authorize?response_type=code&state=123456&client_id=client&scope=all&redirect_uri=http://example.com>

注：这里有个参数state，他并不是一个必须的参数，有的文章说是用来防止攻击的，他可以是任意值。 

​	通过浏览器访问以上地址，第一次没有登录，会弹出登录提示框，输入用户名密码(admin/admin)，然后会跳到redirect_uri参数指定的页面，这里是http://example.com，地址栏会携带参数code。如下所示：

<http://example.com/?code=8QzGxv&state=123456>     

​	拿到code,我们在postman中发送post请求到http://localhost:8080/oauth/token接口，并携带如下参数：client_id,client_secret,grant_type,code,state,redirect_uri参数。便可以获取到access_token。（其中grant_type=authorization_code）

​	现在再将access_token=参数带上，分别访问接口/api/hello和/test/hello，因为是ADMIN，两个接口都可访问。

​	再通过user/123456用户登录授权，获取的access_token来看看访问的效果，访问/api/hello报访问拒绝错误，访问/test/hello是OK的

###密码模式验证

​	首先通过postman访问接口http://localhost:8080/oauth/token，参数就是grant_type,username,password,client_id,client_secret,redirect_uri。与authorization_code授权方式不同的是，这里的grant_type=password，而且增加了参数client_secret和username,password。用admin/admin登陆，访问之后可以获得access_token。

​	现在再将access_token=参数带上，分别访问接口/api/hello和/test/hello，因为是ADMIN，两个接口都可访问。

​	再使用user/123456用户密码获取access_token，访问/api/hello报访问拒绝错误，访问/test/hello是OK的。