---
title: spring-security对OAuth2的集成（数据库的方式）
comments: true
date: 2019-04-28 09:06:42
tags:
    - spring-security-oauth2
    - oauth2
---

>##### 教程由来：项目需要为第三方客户端提供授权和资源访问，无疑OAuth2现在是最好的方式，如果OAuth2相关知识大家还不够了解，请移步到阮一峰的[理解OAuth2.0](https://www.ruanyifeng.com/blog/2014/05/oauth_2_0.html)，本文实战为主，理论方面请自行查阅相关资料。

## 1. OAuth2的四种模式
- 授权码模式（authorization code）(最正统的方式，也是目前绝大多数系统所采用的)(支持refresh token) (用在服务端应用之间)
- 密码模式（resource owner password credentials）(为遗留系统设计) (支持refresh token)
- 简化模式（implicit）(为web浏览器应用设计)(不支持refresh token) (用在移动app或者web app，这些app是在用户的设备上的，如在手机上调起微信来进行认证授权)
- 客户端模式（client credentials）(为后台api服务消费者设计) (不支持refresh token) (为后台api服务消费者设计)
>**本文采用数据库的方式对上述四种模式进行配置，网上绝大多数都是配置在内存中的demo，学习尚可，真实的开发环境却是还远远不够。** 
<!-- more -->
废话也就不多说了，咱进入正题吧。
## 2. 所需依赖

```	
		//Springboot版本为2.0.2.RELEASE
		 <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
		<!-- spring-security-->
 		<dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
         <!-- OAuth2-->
        <dependency>
            <groupId>org.springframework.security.oauth</groupId>
            <artifactId>spring-security-oauth2</artifactId>
            <version>2.3.0.RELEASE</version>
        </dependency>
```

## 3. 授权服务器
> 授权服务器是OAuth2的两大核心之一，它将根据不同的授权类型为客户端提供不同的获取令牌的方式。

```
    @Configuration
    @EnableAuthorizationServer  // 授权服务器核心注解
    protected static class AuthorizationServerConfiguration extends AuthorizationServerConfigurerAdapter {

        @Autowired
        AuthenticationManager authenticationManager; // 注入manager
        @Autowired
        private DataSource dataSource;  // 注入数据源
        @Autowired
        SecurityUserService userDetailsService;  //
        @Autowired
        ClientDetailsService clientDetailsService; 
        @Autowired
        private AuthorizationCodeServices authorizationCodeServices;
        // redis 的相关配置已注释，若需启用，在tokenStore中注入即可。
        // @Autowired
        // private RedisConnectionFactory redisConnectionFactory;
        // @Bean
        // public TokenStore redisTokenStore() {
        //     return new RedisTokenStore(redisConnectionFactory);
        // }
        /**
         * 密码加密
         */
        @Bean
        public PasswordEncoder passwordEncoder() {
            return new BCryptPasswordEncoder();
        }
        /**
         * ClientDetails实现
         * @return
         */
        @Bean
        public ClientDetailsService clientDetails() {
            return new JdbcClientDetailsService(dataSource);
        }

        @Bean
        public TokenStore tokenStore() {
            return new JdbcTokenStore(dataSource);
        }
        /**
         * 加入对授权码模式的支持
         * @param dataSource
         * @return
         */
        @Bean
        public AuthorizationCodeServices authorizationCodeServices(DataSource dataSource) {
            return new JdbcAuthorizationCodeServices(dataSource);
        }
        @Override
        public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
            // 1. 数据库的方式
            clients.withClientDetails(clientDetails());
			// 2. 在内存中配置，这种方式不够灵活，学习倒是没有问题
			// //配置两个客户端,一个用于password认证一个用于client认证
            // clients.inMemory().withClient("client_1")
            //         .resourceIds(DEMO_RESOURCE_ID)
            //         .authorizedGrantTypes("client_credentials", "refresh_token")
            //         .scopes("select")
            //         .authorities("client")
            //         .secret("123456")
            //         .and().withClient("client_2")
            //         .resourceIds(DEMO_RESOURCE_ID)
            //         .authorizedGrantTypes("password", "refresh_token")
            //         .scopes("select")
            //         .authorities("client")
            //         .secret("123456");
        }

        /**
         * 声明授权和token的端点以及token的服务的一些配置信息，
         * 比如采用什么存储方式、token的有效期等
         * @param endpoints
         */
        @Override
        public void configure(AuthorizationServerEndpointsConfigurer endpoints) {

            endpoints
           			// 使用redis的配置
            		// .tokenStore(new RedisTokenStore(redisConnectionFactory))
            		.tokenStore(tokenStore())
                    .authenticationManager(authenticationManager)
                    .userDetailsService(userDetailsService)
                    .authorizationCodeServices(authorizationCodeServices)
                    .setClientDetailsService(clientDetailsService);
        }
        
        /**
         * 声明安全约束，哪些允许访问，哪些不允许访问
         * @param oauthServer
         */
        @Override
        public void configure(AuthorizationServerSecurityConfigurer oauthServer) {
            // 允许表单认证
            oauthServer.allowFormAuthenticationForClients();
            // 配置BCrypt加密
            oauthServer.passwordEncoder(passwordEncoder());
            // 对于CheckEndpoint控制器[框架自带的校验]的/oauth/check端点允许所有客户端发送器请求而不会被Spring-security拦截
            oauthServer.tokenKeyAccess("permitAll()").checkTokenAccess("isAuthenticated()");
            // 此处可添加自定义过滤器，对oauth相关的请求做进一步处理
            // oauthServer.addTokenEndpointAuthenticationFilter(new Oauth2Filter());
        }
    }
```

## 4. 资源服务器
> **1. 资源服务器主要配置拦截的路径，以及访问拦截的URL所需要的权限。**
> **2. 本文，主系统、资源服务器和授权服务器放在一起的，很多人说这样配置security的主过滤器和资源服务器的过滤器会冲突，其实是不会的，资源服务器会优先于security主过滤器拦截你访问的URL，当然你也没必要在类上配置Order去改变他们的优先级，你只需要注意不要让你的资源服务器把security主过滤器放行的资源给拦截了就行。**

下面直接上代码

```
 	@Configuration
    @EnableResourceServer
    protected static class ResourceServerConfiguration extends ResourceServerConfigurerAdapter {
		private static final String RESOURCE_ID = "oauth2";
        @Override
        public void configure(ResourceServerSecurityConfigurer resources) {
            // 如果关闭 stateless，则 accessToken 使用时的 session id 会被记录，后续请求不携带 accessToken 也可以正常响应
            resources.resourceId(RESOURCE_ID).stateless(false);
        }

        /**
         * 为oauth2单独创建角色，这些角色只具有访问受限资源的权限，可解决token失效的问题
         * @param http
         * @throws Exception
         */
        @Override
        public void configure(HttpSecurity http) throws Exception {
            http
                // 获取登录用户的 session
                .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .and()
                    // 资源服务器拦截的路径 注意此路径不要拦截主过滤器放行的URL
                    .requestMatchers().antMatchers("/authmenu/**");
            http
                .authorizeRequests()
                     // 配置资源服务器已拦截的路径才有效
                    .antMatchers("/authmenu/**").authenticated();
                    // .access(" #oauth2.hasScope('select') or hasAnyRole('ROLE_超级管理员', 'ROLE_设备管理员')");
                    
            http
                .exceptionHandling().accessDeniedHandler(new OAuth2AccessDeniedHandler())
                .and()
                .authorizeRequests()
                    .anyRequest()
                    .authenticated();
        }
    }
```
>##### 关于资源拦截做几点说明：
>**1. 通过系统A用户申请的token除了能够访问到A用户被拦截的资源，还能够访问到A用户未被系统拦截的资源，所以最好为申请token的用户创建特定的角色（此类角色只能访问被资源服务器拦截的路径）。** 
>**2. 上述代码中` .antMatchers("/authmenu/**").authenticated();`这段配置可以保证资源服务器不对本系统登录的用户做访问限制。**

## 5. 主过滤器的配置
>**关于主过滤器，每个人的配置可能不太一样，都是有一点是必须的，就是必须把authenticationManagerBean方法注册为bean，本文是基于前后端分离而开发，下文只出示了与OAuth2相关的配置，详细信息请参考文末的GitHub地址。**

```
 	@Override
    @Bean
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }
    
 	@Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .anyRequest()
                .authenticated()
                .withObjectPostProcessor(urlObjectPostProcessor())
                .and()
            .formLogin()
                .loginPage("/login")
                .loginProcessingUrl("/login")
                .usernameParameter("username")
                .passwordParameter("password")
                .permitAll()
                .failureHandler(securityAuthenticationFailureHandler)
                .successHandler(userLoginSuccessHandler)
                .and()
                .exceptionHandling()
                .authenticationEntryPoint(securityAuthenticationEntryPoint)
                .and()
            .logout()
                .deleteCookies("remove")
                .invalidateHttpSession(false)
                .logoutUrl("/logout")
                .logoutSuccessHandler(securityLogoutSuccessHandler)
                .permitAll()
                .and()
            .csrf().requireCsrfProtectionMatcher(new AntPathRequestMatcher("/oauth/authorize"))
                .disable();

        http
            .sessionManagement()
                 // 无效session跳转
                 .invalidSessionUrl("/login")
                 .maximumSessions(1)
                 // session过期跳转
                 .expiredUrl("/login")
                 .sessionRegistry(sessionRegistry());
    }
```
## 6. OAuth2所需数据表

```
--
--  Oauth2 sql  -- MYSQL
--

Drop table  if exists oauth_client_details;
create table oauth_client_details (
  client_id VARCHAR(255) PRIMARY KEY,
  resource_ids VARCHAR(255),
  client_secret VARCHAR(255),
  scope VARCHAR(255),
  authorized_grant_types VARCHAR(255),
  web_server_redirect_uri VARCHAR(255),
  authorities VARCHAR(255),
  access_token_validity INTEGER,
  refresh_token_validity INTEGER,
  additional_information TEXT,
  create_time timestamp default now(),
  archived tinyint(1) default '0',
  trusted tinyint(1) default '0',
  autoapprove VARCHAR (255) default 'false'
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Drop table  if exists oauth_access_token;
create table oauth_access_token (
  create_time timestamp default now(),
  token_id VARCHAR(255),
  token BLOB,
  authentication_id VARCHAR(255),
  user_name VARCHAR(255),
  client_id VARCHAR(255),
  authentication BLOB,
  refresh_token VARCHAR(255)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Drop table  if exists oauth_refresh_token;
create table oauth_refresh_token (
  create_time timestamp default now(),
  token_id VARCHAR(255),
  token BLOB,
  authentication BLOB
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

Drop table  if exists oauth_code;
create table oauth_code (
  create_time timestamp default now(),
  code VARCHAR(255),
  authentication BLOB
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

-- Add indexes
create index token_id_index on oauth_access_token (token_id);
create index authentication_id_index on oauth_access_token (authentication_id);
create index user_name_index on oauth_access_token (user_name);
create index client_id_index on oauth_access_token (client_id);
create index refresh_token_index on oauth_access_token (refresh_token);

create index token_id_index on oauth_refresh_token (token_id);

create index code_index on oauth_code (code);
```
> [OAuth2表中所涉及字段的详细说明](https://blog.csdn.net/qq_34997906/article/details/89609297) 

最后附上我数据库的配置截图，嗯，差不多了。
截图：
![数据库oauth表](https://img-blog.csdnimg.cn/201904281408106.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM0OTk3OTA2,size_16,color_FFFFFF,t_70)
测试就不用写了吧，网上一大堆。
项目Github地址：https://github.com/Janche/springboot-security-project.git （你的星星是对我最大的支持）

>##### oauth2相关资料参考
>1. [程序猿DD-从零开始的Spring Security Oauth2（一）](http://blog.didispace.com/spring-security-oauth2-xjf-1/)\
>2. [Spring Security & OAuth2](https://github.com/monkeyk/spring-oauth-server)

