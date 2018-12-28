# 使用Spring进行OAuth2授权

## OAuth2简介

简单地来说，OAuth2协议就是为了使第三方能够在不获取资源所有者凭证（如明文密钥）的情况下，能够通过令牌来访问受限资源的授权协议。其协议的流程如下：

```tex
     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+
```

如果想要更深入了解，可以看一下[RFC6749](https://tools.ietf.org/html/rfc6749)。

## Spring（Spring Boot）中的OAuth2

spring中的security模块提供了完整的OAuth2协议支持，通过一些简单的配置，就可以快速的搭建起一个简单的OAuth2完整服务（包括授权界面，资源服务器和授权服务器）。

### 授权界面配置

授权界面的配置可以通过@EnableWebSecurity注解来开启，配置类需要继承_WebSecurityConfigurerAdapter_类，并按需求进行配置。下面的例子展示了一个简单的授权界面，为用户admin（密码admin）授权管理员权限。

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {

    protected void configure(HttpSecurity httpSecurity) throws Exception {
        httpSecurity
                .csrf().disable()
                .formLogin().permitAll()
                .and()
                .anonymous().disable()
                .httpBasic()
                .and()
                .authorizeRequests().anyRequest().authenticated()
        ;
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {
        auth.inMemoryAuthentication()
                .withUser("admin").password("admin").roles("ADMIN");
    }

}
```

### 授权服务器配置

授权服务器可以通过@EnableAuthorizationServer注解来开启，配置类需要继承_AuthorizationServerConfigurerAdapter_类，并按需求进行配置。下面的例子展示了一个简单的授权服务器，其配置的意思是为client这个客户端（密钥为clientpassword）分配read和write的权限，授权方式是password。关于其他的授权方式，可以查看OAuth2文档。生成的token的有效期是1小时，token保存在内存中（也可以通过JDBC方式保存在数据库中）。

```java
@Configuration
@EnableAuthorizationServer
public class AuthorizationServerConfig extends AuthorizationServerConfigurerAdapter {

	@Autowired
    private AuthenticationManager authenticationManager;
	
	@Override
    public void configure(final AuthorizationServerEndpointsConfigurer endpoints) {
		endpoints.tokenStore(tokenStore())
			.authenticationManager(authenticationManager);
    }
	
	@Override
    public void configure(final ClientDetailsServiceConfigurer clients) throws Exception {
		clients.inMemory()
	        .withClient("client")
	        .secret("clientpassword")
	        .scopes("read", "write")
	        .authorizedGrantTypes("password")
	        .accessTokenValiditySeconds(3600);
    }

	@Bean
	public TokenStore tokenStore() {
		return new InMemoryTokenStore();
	}

}
```

### 资源服务器配置

资源服务器的配置与授权服务器类似，需要打开@EnableResourceServer注解，并继承_ResourceServerConfigurerAdapter_类。下面是一个简单的资源服务器配置，将/products路径下的请求资源设置为products，并只允许ADMIN角色的用户访问。通常来说资源服务器可以部署在不同的服务器上，也可以部署在同一个服务内。

```java
@Configuration
@EnableResourceServer
public class ResourceServerConfig extends ResourceServerConfigurerAdapter {

    @Override
    public void configure(ResourceServerSecurityConfigurer resources) {
        resources.resourceId("products");
    }

	@Override
	public void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
                .authorizeRequests()
                .antMatchers("/products")
                .hasRole("ADMIN")
                .and().authorizeRequests()
                .anyRequest().permitAll();
	}
	
}
```

### 配置一个资源

简单的controller即可，路径需要与我们在资源服务器中配置的路径一致。

```java
@RestController
@RequestMapping("/products")
public class ProductResource {

	@GetMapping
	public List<String> list() {
		return Arrays.asList("apple", "banana", "coffe");
	}
}
```

### how to use

1. 启动Springboot，访问链接http://localhost:8080/products
2. 由于未经过授权，会跳转到http://localhost:8080/login
3. 输入用户名密码（admin/admin）后，跳转回 http://localhost:8080/products ，并打印出结果。

