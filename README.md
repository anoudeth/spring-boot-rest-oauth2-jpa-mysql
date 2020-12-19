## Secure RESTful Web Service using spring-boot, spring-cloud-oauth2, JPA, MySQL and Hibernate Validator
![1](https://user-images.githubusercontent.com/31319842/99893591-9d003b80-2cab-11eb-9f06-1d5a785c775b.png)

## Oauth2

In this Spring security oauth2 tutorial, learn to build an authorization server to authenticate your identity to provide access_token, which you can use to request data from resource server.

***Introduction to OAuth 2*** OAuth 2 is an authorization method to provide access to protected resources over the HTTP protocol. Primarily, oauth2 enables a third-party application to obtain limited access to an HTTP service –

- either on behalf of a resource owner by orchestrating an approval interaction between the resource owner and the HTTP service
- or by allowing the third-party application to obtain access on its own behalf.

**OAuth2 Roles:** There are four roles that can be applied on OAuth2:

- `Resource Owner`: The owner of the resource — this is pretty self-explanatory.
- `Resource Server`: This serves resources that are protected by the OAuth2 token.
- `Client`: The application accessing the resource server.
- `Authorization Server`: This is the server issuing access tokens to the client after successfully authenticating the resource owner and obtaining authorization.

**OAuth2 Tokens:** Tokens are implementation specific random strings, generated by the authorization server.

- `Access Token`: Sent with each request, usually valid for about an hour only.
- `Refresh Token`: It is used to get a 00new access token, not sent with each request, usually lives longer than access token.

### Quick Start a Cloud Security App

Let's start by configuring spring cloud oauth2 in a Spring Boot application for microservice security.

First, we need to add the `spring-cloud-starter-oauth2` dependency:

```
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-oauth2</artifactId>
    <version>2.2.2.RELEASE</version>
</dependency>
```

This will also bring in the `spring-cloud-starter-security`dependency.

```
<dependency>
     <groupId>org.springframework.cloud</groupId>
     <artifactId>spring-cloud-starter-security</artifactId>
</dependency>
```

And we need to add spring cloud `dependency `  in`dependencyManagement` 

```
<dependencyManagement>
	<dependencies>
		<dependency>
			<groupId>org.springframework.cloud</groupId>
			<artifactId>spring-cloud-dependencies</artifactId>
			<version>${spring-cloud.version}</version>
			<type>pom</type>
			<scope>import</scope>
		</dependency>
	</dependencies>
</dependencyManagement>
```

Another think is now we will add spring cloud version into the `properties` tag:

````
<properties>
  <spring-cloud.version>Greenwich.RELEASE</spring-cloud.version>
</properties>
````

**Create tables for users, groups, group authorities and group members**

For Spring OAuth2 mechanism to work, we need to create tables to hold users, groups, group authorities and group members. We can create these tables as part of application start up by providing the table definations in `schema.sql` file as shown below. This setup is good enough for POC code. `src/main/resources/schema.sql`

```
create table if not exists  oauth_client_details (
  client_id varchar(255) not null,
  client_secret varchar(255) not null,
  web_server_redirect_uri varchar(2048) default null,
  scope varchar(255) default null,
  access_token_validity int(11) default null,
  refresh_token_validity int(11) default null,
  resource_ids varchar(1024) default null,
  authorized_grant_types varchar(1024) default null,
  authorities varchar(1024) default null,
  additional_information varchar(4096) default null,
  autoapprove varchar(255) default null,
  primary key (client_id)
);

create table if not exists  permission (
  id int(11) not null auto_increment,
  name varchar(512) default null,
  primary key (id),
  unique key name (name)
) ;

create table if not exists role (
  id int(11) not null auto_increment,
  name varchar(255) default null,
  primary key (id),
  unique key name (name)
) ;

create table if not exists  user (
  id int(11) not null auto_increment,
  username varchar(100) not null,
  password varchar(1024) not null,
  email varchar(1024) not null,
  enabled tinyint(4) not null,
  accountNonExpired tinyint(4) not null,
  credentialsNonExpired tinyint(4) not null,
  accountNonLocked tinyint(4) not null,
  primary key (id),
  unique key username (username)
) ;

create table  if not exists permission_role (
  permission_id int(11) default null,
  role_id int(11) default null,
  key permission_id (permission_id),
  key role_id (role_id),
  constraint permission_role_ibfk_1 foreign key (permission_id) references permission (id),
  constraint permission_role_ibfk_2 foreign key (role_id) references role (id)
);

create table if not exists role_user (
  role_id int(11) default null,
  user_id int(11) default null,
  key role_id (role_id),
  key user_id (user_id),
  constraint role_user_ibfk_1 foreign key (role_id) references role (id),
  constraint role_user_ibfk_2 foreign key (user_id) references user (id)
);
```

- `oauth_client_details table` is used to store client details.
- `oauth_access_token` and `oauth_refresh_token` is used internally by OAuth2 server to store the user tokens.

**Create a client**

Let’s insert a record in `oauth_client_details` table for a client named appclient with a password `appclient`.

Here, `appclient` is the ID has access to the `product-server` and `sales-server` resource.

I have used `CodeachesBCryptPasswordEncoder.java` available [here](https://github.com/habibsumoncse/spring-boot-microservice-auth-zuul-eureka-hystrix/blob/master/micro-auth-service/src/main/resources/schema.sql) to get the Bcrypt encrypted password.

`src/main/resources/data.sql`

```
INSERT INTO oauth_client_details (client_id, client_secret, web_server_redirect_uri, scope, access_token_validity, refresh_token_validity, resource_ids, authorized_grant_types, additional_information) 
VALUES ('mobile', '{bcrypt}$2a$10$gPhlXZfms0EpNHX0.HHptOhoFD1AoxSr/yUIdTqA8vtjeP4zi0DDu', 'http://localhost:8080/code', 'READ,WRITE', '3600', '10000', 'inventory,payment', 'authorization_code,password,refresh_token,implicit', '{}');

/*client_id - client_secret*/
/* mobile - pin* /


 INSERT INTO PERMISSION (NAME) VALUES
 ('create_profile'),
 ('read_profile'),
 ('update_profile'),
 ('delete_profile');

 INSERT INTO role (NAME) VALUES ('ROLE_admin'),('ROLE_editor'),('ROLE_operator');

 INSERT INTO PERMISSION_ROLE (PERMISSION_ID, ROLE_ID) VALUES
     (1,1), /*create-> admin */
     (2,1), /* read admin */
     (3,1), /* update admin */
     (4,1), /* delete admin */
     (2,2),  /* read Editor */
     (3,2),  /* update Editor */
     (2,3);  /* read operator */


 insert into user (id, username,password, email, enabled, accountNonExpired, credentialsNonExpired, accountNonLocked) VALUES ('1', 'admin','{bcrypt}$2a$12$xVEzhL3RTFP1WCYhS4cv5ecNZIf89EnOW4XQczWHNB/Zi4zQAnkuS', 'habibsumoncse2@gmail.com', '1', '1', '1', '1');
 insert into  user (id, username,password, email, enabled, accountNonExpired, credentialsNonExpired, accountNonLocked) VALUES ('2', 'ahasan', '{bcrypt}$2a$12$DGs/1IptlFg0szj.3PttmeC8swHZs/pZ6YEKng4Cl1l2woMtkNhvi','habibsumoncse2@gmail.com', '1', '1', '1', '1');
 insert into  user (id, username,password, email, enabled, accountNonExpired, credentialsNonExpired, accountNonLocked) VALUES ('3', 'user', '{bcrypt}$2a$12$udISUXbLy9ng5wuFsrCMPeQIYzaKtAEXNJqzeprSuaty86N4m6emW','habibsumoncse2@gmail.com', '1', '1', '1', '1');
 /*
 username - passowrds:
 admin - admin
 ahasan - ahasan
 user - user
 */


INSERT INTO ROLE_USER (ROLE_ID, USER_ID)
    VALUES
    (1, 1), /* admin-admin */,
    (2, 2), /* ahasan-editor */ ,
    (3, 3); /* user-operatorr */ ;
```

### Configure Authorization Server

Annotate the `Oauth2AuthorizationServerApplication.java` with `@EnableAuthorizationServer`. This enables the Spring to consider this service as authorization Server.

Let’s create a class `AuthorizationServerConfiguration.java` with below details.

- **JdbcTokenStore** implements token services that stores tokens in a database.
- **BCryptPasswordEncoder** implements PasswordEncoder that uses the BCrypt strong hashing function. Clients can optionally supply a “strength” (a.k.a. log rounds in BCrypt) and a SecureRandom instance. The larger the strength parameter the more work will have to be done (exponentially) to hash the passwords. The value used in this example is 8 for client secret.
- **AuthorizationServerEndpointsConfigurer** configures the non-security features of the Authorization Server endpoints, like token store, token customizations, user approvals and grant types.
- **AuthorizationServerSecurityConfigurer** configures the security of the Authorization Server, which means in practical terms the /oauth/token endpoint.
- **ClientDetailsServiceConfigurer** configures the ClientDetailsService, e.g. declaring individual clients and their properties.

```
@EnableAuthorizationServer
@Configuration
public class AuthorizationServerConfiguration implements AuthorizationServerConfigurer {

    @Autowired
    private PasswordEncoder passwordEncoder;
    
    @Autowired
    private DataSource dataSource;
    
    @Autowired
    private AuthenticationManager authenticationManager;

    @Bean
    TokenStore jdbcTokenStore() {
        return new JdbcTokenStore(dataSource);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer security) throws Exception {
    security.checkTokenAccess("isAuthenticated()").tokenKeyAccess("isAuthenticated()");

    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(dataSource).passwordEncoder(passwordEncoder);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {
        endpoints.tokenStore(jdbcTokenStore());
        endpoints.authenticationManager(authenticationManager);
    }
    
    @Bean
    public FilterRegistrationBean<CorsFilter> corsFilter() {
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowCredentials(true);
        config.addAllowedOrigin("*");
        config.addAllowedHeader("*");
        config.addAllowedMethod("*");
        source.registerCorsConfiguration("/**", config);
        FilterRegistrationBean<CorsFilter> bean = new FilterRegistrationBean<CorsFilter>(new CorsFilter(source));
        bean.setOrder(Ordered.HIGHEST_PRECEDENCE);
        return bean;
    }
}
```

**Configure User Security Authentication** Let’s create a class `UserSecurityConfig.java` to handle user authentication.

- **PasswordEncoder** implements PasswordEncoder that uses the BCrypt strong hashing function. Clients can optionally supply a “strength” (a.k.a. log rounds in BCrypt) and a SecureRandom instance. The larger the strength parameter the more work will have to be done (exponentially) to hash the passwords. The value used in this example is 4 for user’s password.

```
@Configuration
@EnableGlobalMethodSecurity(prePostEnabled = true, securedEnabled = true)
public class WebSecurityConfiguration extends WebSecurityConfigurerAdapter {

	@Autowired
	private UserDetailsService userDetailsService;

	@Bean
	protected AuthenticationManager getAuthenticationManager() throws Exception {
		return super.authenticationManagerBean();
	}

	@Bean
	PasswordEncoder passwordEncoder() {
		return PasswordEncoderFactories.createDelegatingPasswordEncoder();
	}

	@Override
	protected void configure(AuthenticationManagerBuilder auth) throws Exception {
		auth.userDetailsService(userDetailsService).passwordEncoder(passwordEncoder());
	}
}

```

### Test Authorization Service

***Get Access Token***

Let’s get the access token for `admin` by passing his credentials as part of header along with authorization details of appclient by sending `client_id` `client_pass` `username` `userpsssword`

Now hit the POST method URL via POSTMAN to get the OAUTH2 token.

**`http://localhost:8080/oauth/token`**

Now, add the Request Headers as follows −

*`Authorization` − Basic Auth with your Client Id and Client secret.

*`Content Type` − application/x-www-form-urlencoded [![1](https://user-images.githubusercontent.com/31319842/95816138-2e40d180-0d40-11eb-99c7-403cdf7ef070.png)](https://user-images.githubusercontent.com/31319842/95816138-2e40d180-0d40-11eb-99c7-403cdf7ef070.png)

Now, add the Request Parameters as follows −

- `grant_type` = password
- `username` = your username
- `password` = your password [![2](https://user-images.githubusercontent.com/31319842/95816163-3bf65700-0d40-11eb-9c87-7b721e0a268f.png)](https://user-images.githubusercontent.com/31319842/95816163-3bf65700-0d40-11eb-9c87-7b721e0a268f.png)

**HTTP POST Response**

```
{ 
  "access_token":"000ff762-414c-4605-858a-0ed7bee6f68e",
  "token_type":"bearer",
  "refresh_token":"79aabc70-f310-4c49-bf7e-516208b3bef4",
  "expires_in":999999,
  "scope":"read write"
}
```



### Configure Resource Server

The **resource server** is the OAuth 2.0 term for your API **server**. The **resource server** handles authenticated requests after the application has obtained an access token. ... Each of these **resource servers** are distinctly separate, but they all share the same authorization **server**.

#### Enable oauth2 on sales service

Now add the `@EnableResourceServer` and `@Configuration` annotation on Spring boot application class present in src folder. With this annotation, this artifact will act like a resource service. With this `@EnableResourceServer` annotation, this artifact will act like a resource service.

```
@Configuration
@EnableResourceServer
public class ResourceServerConfiguration extends ResourceServerConfigurerAdapter {

	private static final String RESOURCE_ID = "microservice";
	private static final String SECURED_READ_SCOPE = "#oauth2.hasScope('READ')";
	private static final String SECURED_WRITE_SCOPE = "#oauth2.hasScope('WRITE')";
	private static final String SECURED_PATTERN = "/**";

	@Override
	public void configure(ResourceServerSecurityConfigurer resources) {
		resources.resourceId(RESOURCE_ID);
	}

	@Override
	public void configure(HttpSecurity http) throws Exception {
			http.csrf().disable()
				.sessionManagement().disable()
				.authorizeRequests()
				.antMatchers("/employee/list").permitAll()
				.and()
				.requestMatchers().antMatchers(SECURED_PATTERN)
				.and()
				.authorizeRequests().antMatchers(HttpMethod.POST, SECURED_PATTERN)
				.access(SECURED_WRITE_SCOPE).anyRequest()
				.access(SECURED_READ_SCOPE);
	}
}
```



## How to run spring-boot-secure-rest application?

### Build Project

Now, you can create an executable JAR file, and run the Spring Boot application by using the Maven shown below − For Maven, use the command as shown below −

**Project import in sts4 IDE** `File > import > maven > Existing maven project > Root Directory-Browse > Select project form root folder > Finish`

### Run project

After “BUILD SUCCESSFUL”, you can find the JAR file under the build/libs directory. Now, run the JAR file by using the following command −

```
java –jar <JARFILE> `Run on sts IDE `click right button on the project >Run As >Spring Boot App
```

Server Running on: `8082` port

**Test HTTP GET Request to resource service**

```
curl --request GET 'localhost:8082/employee/find' --header 'Authorization: Bearer 8b22d6b0-bd2c-44b3-9934-d20268ebe886'
```

![Screenshot from 2020-12-07 09-11-07](https://user-images.githubusercontent.com/31319842/101305239-6bbb6a00-386c-11eb-8252-9bfd4d9d092d.png)

- Here `[localhost:8082/employee/find]` on the `http` means protocol, `localhost` for hostaddress 8082 are service port and `/employee/find` is method URL.
- Here `[Authorization: Bearer 62e2545c-d865-4206-9e23-f64a34309787']` `Bearer` is toiken type and `62e2545c-d865-4206-9e23-f64a34309787` is auth service provided token

### For getting All API Information

On this repository we will see `secure-microservice-architecture.postman_collection.json` file, this file have to `import` on postman then we will ses all API information for testing api.

