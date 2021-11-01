# SpringBoot集成Shiro

[TOC]

## 一、Shiro简介

​        Apache Shiro 是一个功能强大、灵活的，开源的安全框架。它可以干净利落地处理身份验证、授权、企业会话管理和加密。Shiro属于轻量级框架，相对于SpringSecurity简单的多。

### ​1. Shiro的用途 

Shiro 能做什么呢？

- 验证用户身份
- 用户访问权限控制，比如：1. 判断用户是否分配了一定的安全角色；2. 判断用户是否被授予完成某个操作的权限；
- 在非 Web 或 EJB 容器的环境下可以任意使用 Session API；
- 可以响应认证、访问控制，或者 Session 生命周期中发生的事件；
- 可将一个或以上用户安全数据源数据组合成一个复合的用户 "view"(视图)；
- 支持单点登录(SSO)功能；
- 支持提供“Remember Me”服务，获取用户关联信息而无需登录；
  …

### 2. 三大核心组件

三个核心组件：Subject, SecurityManager 和 Realms.

1. **Subject：**即“当前操作用户”。但是，在Shiro中，Subject这一概念并不仅仅指人，也可以是第三方进程、后台帐户（Daemon Account）等，每个subject实例都会被绑定到SercurityManger上。

2. **SecurityManager：**SecurityManager是Shiro核心，主要协调Shiro内部的各种安全组件安全管理器，管理所有Subject，可以配合内部安全组件。(类似于SpringMVC中的DispatcherServlet)。

3. **Realms：**用户数据和Shiro数据交互的桥梁。比如需要用户身份认证、权限认证。一般需要自己实现。

### 3. 特性

Authentication（认证）, Authorization（授权）, Session Management（会话管理）, Cryptography（加密）被 Shiro 框架的开发团队称之为应用安全的四大基石：

- **Authentication（认证）：**用户身份识别，通常被称为用户“登录”
- **Authorization（授权）：**访问控制。比如某个用户是否具有某个操作的使用权限。
- **Session Management（会话管理）：**特定于用户的会话管理，甚至在非web 或 EJB 应用程序。
- **Cryptography（加密）：**在对数据源使用加密算法加密的同时，保证易于使用。

还有其他的功能来支持和加强这些不同应用环境下安全领域的关注点。特别是对以下的功能支持：

- **Web Support（Web支持）**：Shiro 提供的 Web 支持 api ，可以很轻松的保护 Web 应用程序的安全。
- **Caching（缓存）**：缓存是 Apache Shiro 保证安全操作快速、高效的重要手段。
- **Concurrency（并发）**：Apache Shiro 支持多线程应用程序的并发特性。
- **Testing（测试）**：支持单元测试和集成测试，确保代码和预想的一样安全。
- **Run As**：这个功能允许用户假设另一个用户的身份（在许可的前提下）进行访问。
- **Remember Me（记住我）**：跨 session 记录用户的身份，只有在强制需要时才需要登录。

## 二、SpringBoot集成Shiro

用SpringBoot集成shiro，简单实现登录验证，权限管理。

### 1. Maven配置

在pom.xml中添加如下依赖，引入shiro：

```xml
        <dependency>
            <groupId>org.apache.shiro</groupId>
            <artifactId>shiro-spring</artifactId>
            <version>1.3.2</version>
        </dependency>
```

### 2. 基础类（User，Role，Permission）：

为了测试，我新建了简单的用户、角色、权限类；具体代码如下：

User类：

```java
@Data
public class User {
    private String id;
    private String userName;
    private String password;
    /**
     * 用户对应的角色集合
     */
    private Set<Role> roles;
}
```

Role类：

```java
@Data
public class Role {
    private String id;
    private String roleName;
    /**
     * 角色对应权限集合
     */
    private Set<Permissions> permissions;
}
```

Permission类：

```java
@Data
public class Permissions {
    private String id;
    private String permissionsName;
}
```

### 3. 配置ShiroConfig类

ShiroConfig配置是集成shiro核心的部分，先上代码：

```java
@Configuration
public class ShiroConfig {
    
    /**
     * Filter工厂，设置对应的过滤条件和跳转条件
     * @param securityManager
     * @return
     */
    @Bean(name = "shiroFilter")
    public ShiroFilterFactoryBean shiroFilter(SecurityManager securityManager) {
        ShiroFilterFactoryBean shiroFilterFactoryBean = new ShiroFilterFactoryBean();
        shiroFilterFactoryBean.setSecurityManager(securityManager);
        //自定义拦截器
        Map<String, Filter> filtersMap = new LinkedHashMap<String, Filter>();
        LogoutFilter logoutFilter = new LogoutFilter();
        logoutFilter.setRedirectUrl("/logout");
        filtersMap.put("logout", logoutFilter);
        shiroFilterFactoryBean.setFilters(filtersMap);
        //登录页面
        shiroFilterFactoryBean.setLoginUrl("/login");
        Map<String, String> filterChainDefinitionMap = new LinkedHashMap<>();
        // <!-- authc:所有url都必须认证通过才可以访问; anon:所有url都都可以匿名访问-->
        filterChainDefinitionMap.put("/login", "anon");
        filterChainDefinitionMap.put("/add", "authc");
        filterChainDefinitionMap.put("/admin", "authc");
        filterChainDefinitionMap.put("/index", "authc");
        // 其他路径均需要身份认证，这行代码必须放在所有权限设置的最后，不然会导致所有 url 都被拦截，一般位于最下面，优先级最低，剩余的都需要认证
        filterChainDefinitionMap.put("/**", "authc");
        shiroFilterFactoryBean.setFilterChainDefinitionMap(filterChainDefinitionMap);
        return shiroFilterFactoryBean;

    }

    /**
     * 将自己的验证方式加入容器
     * @return
     */
    @Bean
    public CustomRealm myShiroRealm() {
        return new CustomRealm();
    }


    /**
     * 权限管理，配置主要是Realm的管理认证
     * @return
     */
    @Bean
    public SecurityManager securityManager() {
        DefaultWebSecurityManager securityManager = new DefaultWebSecurityManager();
        securityManager.setRealm(myShiroRealm());
        return securityManager;
    }
    
    /**
     * 开启注解
     * @return
     */
    @Bean
    public AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor() {
        AuthorizationAttributeSourceAdvisor authorizationAttributeSourceAdvisor = new AuthorizationAttributeSourceAdvisor();
        authorizationAttributeSourceAdvisor.setSecurityManager(securityManager());
        return authorizationAttributeSourceAdvisor;
    }
```

1. **shiroFilter**：Filter工厂，设置对应的过滤条件和跳转条件。

   **常用认证**：

   - anon: 所有 url 都都可以匿名访问
   - authc: 需要认证才能进行访问
   - user: 配置记住我或认证通过可以访问

   **注意点**：认证配置的时候一般位于最下面的，优先级最低。如果要色泽其他路径均需要身份认证，必须放在所有权限设置的最后，不然会导致所有 url 都被拦截。

2. **myShiroRealm**：将自己的验证方法放进容器，CustomRealm需要自己实现，下面会贴我的代码。

3. **securityManager**：权限管理，配置主要是Realm的管理认证。

4. **authorizationAttributeSourceAdvisor**：开启Shiro的注解(如@RequiresRoles，@RequiresPermissions)，需借助SpringAOP扫描使用Shiro注解的类，并在必要时进行安全逻辑验证。

###  4. 创建CustomRealm类

CustomRealm需要继承AuthorizingRealm类，并实现方法，对用户进行验证。注解比较详细，我就不重复赘述了。

```java
public class CustomRealm extends AuthorizingRealm {
    @Autowired
    private UserService userService;

    /**
     * 权限配置类
     */
    @Override
    protected AuthorizationInfo doGetAuthorizationInfo(PrincipalCollection principalCollection) {
        //获取登录用户名
        String userName = (String) principalCollection.getPrimaryPrincipal();
        //查询用户
        User user = userService.getUserByUserName(userName);
        //添加角色和权限
        SimpleAuthorizationInfo simpleAuthorizationInfo = new SimpleAuthorizationInfo();
        for (Role role : user.getRoles()) {
            //添加角色
            simpleAuthorizationInfo.addRole(role.getRoleName());
            //添加权限
            for (Permissions permissions : role.getPermissions()) {
                simpleAuthorizationInfo.addStringPermission(permissions.getPermissionsName());
            }
        }
        return simpleAuthorizationInfo;
    }

    /**
     * 认证配置类
     */
    @Override
    protected AuthenticationInfo doGetAuthenticationInfo(AuthenticationToken authenticationToken) throws AuthenticationException {
        if (StringUtils.isEmpty((String)authenticationToken.getPrincipal())) {
            return null;
        }
        //获取用户信息
        String userName = authenticationToken.getPrincipal().toString();
        User user = userService.getUserByUserName(userName);
        if (user == null) {
            //这里返回后会报出对应异常
            return null;
        } else {
            //这里验证authenticationToken和simpleAuthenticationInfo的信息
            return new SimpleAuthenticationInfo(userName, user.getPassword(), getName());
        }
    }
}
```

### 5. 自定义Controllr演示

我写了4个方法：

- 登录（/login）：用户名、密码登录，可以捕获异常，返回对应的信息
- 管理员的方法（/admin）：admin角色可以访问，用**@RequiresRoles**这个注解控制
- 主页（/index）：有query权限的人可以访问，用**@RequiresPermissions**这个注解控制
- 新增方法（/add）：有add权限的人可以访问，用**@RequiresPermissions**这个注解控制

```java
@RestController
@Slf4j
public class LoginController extends BaseController {

    @GetMapping("/login")
    public String login(String userName, String password) {
        if (StringUtils.isEmpty(userName) || StringUtils.isEmpty(password)) {
            return "用户名或密码为空！";
        }
        //用户认证信息
        Subject subject = SecurityUtils.getSubject();
        UsernamePasswordToken usernamePasswordToken = new UsernamePasswordToken(
                userName,
                password
        );
        try {
            //进行验证，这里可以捕获异常，然后返回对应信息
            subject.login(usernamePasswordToken);
        } catch (UnknownAccountException uae) {
            return "用户不存在";
        } catch (IncorrectCredentialsException ice) {
            return "密码不正确";
        } catch (LockedAccountException lae) {
            return "账户已锁定";
        } catch (ExcessiveAttemptsException eae) {
            return "用户名或密码错误次数过多";
        } catch (AuthenticationException ae) {
            return "用户名或密码不正确！";
        }
        if (subject.isAuthenticated()) {
            return "登录成功";
        } else {
            return "登录失败";
        }
    }

    @RequiresRoles("admin")
    @GetMapping("/admin")
    public String admin() {
        return "管理员界面";
    }

    @RequiresPermissions("query")
    @GetMapping("/index")
    public String index() {
        return "主页！！！";
    }

    @RequiresPermissions("add")
    @GetMapping("/add")
    public String add() {
        return "新增成功！";
    }

```

**一些常见的shiro异常**：

1. **AuthenticationException 认证异常**：Shiro在登录认证过程中，认证失败需要抛出的异常。
   - **CredentitalsException 凭证异常**
     - IncorrectCredentialsException 不正确的凭证
     - ExpiredCredentialsException 凭证过期
   - **AccountException 账号异常**
     - ConcurrentAccessException: 并发访问异常（多个用户同时登录时抛出）
     - UnknownAccountException: 未知的账号
     - ExcessiveAttemptsException: 认证次数超过限制
     - DisabledAccountException: 禁用的账号
     - LockedAccountException: 账号被锁定
     - UnsupportedTokenException: 使用了不支持的Token
2. **AuthorizationException: 授权异常**，Shiro在登录认证过程中，授权失败需要抛出的异常。
   - **UnauthorizedException**：抛出以指示请求的操作或对请求的资源的访问是不允许的。
   - **UnanthenticatedException**：当尚未完成成功认证时，尝试执行授权操作时引发异常。

### 6. 对错误进行处理

我这边是用BaseController的方式，也可以用全局异常处理，我这边单独对权限不通过错误进行处理，方便演示：

```java
public class BaseController {
    //定义exceptionHandler解决未被controller层吸收的exception
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.OK)
    @ResponseBody
    public Object handlerException(HttpServletRequest request, Exception ex) {
        if (ex instanceof BusinessException) {
            BusinessException businessException = (BusinessException) ex;
            return CommonReturnType.isFail(businessException.getErrCode(), businessException.getErrMsg());
        }else if (ex instanceof AuthorizationException){
            //此处对没有权限做了处理
            return CommonReturnType.isFail(EmBusinessError.USER_NOT_PERMISSION.getErrCode(), EmBusinessError.USER_NOT_PERMISSION.getErrMsg());
        } else {
            ex.printStackTrace();
            return CommonReturnType.isFail(EmBusinessError.SERVER_ERROE.getErrCode(), EmBusinessError.SERVER_ERROE.getErrMsg());
        }
    }
}
```

### 7. 演示

​	这边我创建了两个账号，一个是admin，角色为admin，有add和query权限；另一个是sxl，角色为user，只有query权限。密码均为123456.

- 测试登录：

  - 先进入首页试试：localhost:8090/index

    此时会直接跳转到登录页面；

  ![image-20210113125014048](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113125014048.png)

  - 普通账号sxl登录：localhost:8090/login?userName=sxl&password=123456

    显示登录成功

    ![image-20210113125150144](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113125150144.png)

  - 密码输错登录：localhost:8090/login?userName=sxl&password=12345

    显示密码不正确

    ![image-20210113125359530](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113125359530.png)

  - 账号输错登录：localhost:8090/login?userName=sx&password=123456

    显示用户不存在

    ![image-20210113125453629](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113125453629.png)

- 测试权限

  - 先用普通账号登录：localhost:8090/login?userName=sxl&password=123456

    打开首页：进入成功

    ![image-20210113134913769](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113134913769.png)

    打开新增：显示没有权限

    ![image-20210113134946758](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113134946758.png)

    打开管理员页面：显示没有权限

    ![image-20210113135034408](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113135034408.png)

  - 用管理员账号登录：localhost:8090/login?userName=admin&password=123456

    打开首页：进入成功

    ![image-20210113134913769](/Users/chenmingliang/Library/Application%20Support/typora-user-images/image-20210113134913769.png)

    打开新增：新增成功

    ![image-20210113135130707](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113135130707.png)

    打开管理员页面：进入成功

    ![image-20210113135151662](/Users/chenmingliang/Library/Application Support/typora-user-images/image-20210113135151662.png)

## 三、总结

​	本次简单的学习和使用了下shiro，感觉如果要使用，自己搭建一下还是很方便的，只需要角色和权限功能的话，已经能够满足基本开发需要了。

​	但是shiro远不止以上的内容，这仅仅是完成了登录认证和权限管理这两个功能，更多内容以后有时间再做探讨。