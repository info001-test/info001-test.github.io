---
title: "Spring Security OAuth2 Authorization using Jdbc Token Store"
summary: "Spring Security OAuth2 Authorization using Jdbc Token Store"

sidebar: spring_security_sidebar

keywords: spring security,spring security example,spring security tutorial,spring security oauth2,spring security jdbc,bcrypt password encoder,spring boot,codeaches 

tags: [spring-security-tag]

permalink: spring-security/oauth2-authorization-jdbc-token-store/

gh-repo: spring-security-oauth2-authorization-jdbc-token-store
gh-badge: [star, watch, follow, download, issue]

folder: spring_security

date_published: 2016-02-26
date_modified: 2016-02-27

---

## Prerequisites {#prerequisites}

I have used Open JDK 11 and Spring Tool Suite 4 IDE for this tutorial. I have not tried with the other versions though.

 - [Open Source JDK 11](https://jdk.java.net/11){:target="_blank"}
 - [Spring Tool Suite 4 IDE](https://spring.io/tools){:target="_blank"}

## Build authorization server {#build_auth_server}

**Create a Spring Boot starter project using Spring Initializr**

Let's utilize [spring initializr web tool](https://start.spring.io/){:target="_blank"} and create a skeleton spring boot project for Authorization Server. I have updated Group field to **com.codeaches**, Artifact to **oauth2server** and selected `Web`,`Security`,`Cloud OAuth2`,`H2`,`JPA` dependencies. I have selected Java Version as **11**

Click on `Generate Project`. The project will be downloaded as `oauth2server.zip` file on your hard drive.

>Alternatively, you can also generate the project in a shell using cURL

```sh
curl https://start.spring.io/starter.zip  \
       -d dependencies=web,cloud-security,cloud-oauth2,h2,data-jpa \
       -d language=java \
       -d javaVersion=11 \
       -d type=maven-project \
       -d groupId=com.codeaches \
       -d artifactId=oauth2server \
       -d bootVersion=2.1.3.RELEASE \
       -o oauth2server.zip
```

**Import and build**

Import the project in STS as `Existing Maven project` and do Maven build.

> Add the jaxb-runtime dependancy if the build fails with an error "javax.xml.bind.JAXBException: Implementation of JAXB-API has not been found on module path or classpath"

```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

**Create tables for clients, users and groups**

Let's create tables to hold the OAuth2 client, access and refresh token details in embedded h2 db by providing the DDL scripts which runs during server startup.

`src/main/resources/schema.sql`

```sql
drop table oauth_client_details if exists;
create table oauth_client_details (
    client_id varchar(256) primary key,
    resource_ids varchar(256),
    client_secret varchar(256),
    scope varchar(256),
    authorized_grant_types varchar(256),
    web_server_redirect_uri varchar(256),
    authorities varchar(256),
    access_token_validity integer,
    refresh_token_validity integer,
    additional_information varchar(4096),
    autoapprove varchar(256)
);

drop table oauth_access_token if exists;
create table oauth_access_token (
  token_id VARCHAR(256),
  token LONGVARBINARY,
  authentication_id VARCHAR(256) PRIMARY KEY,
  user_name VARCHAR(256),
  client_id VARCHAR(256),
  authentication LONGVARBINARY,
  refresh_token VARCHAR(256)
);

drop table oauth_refresh_token if exists;
create table oauth_refresh_token (
  token_id VARCHAR(256),
  token LONGVARBINARY,
  authentication LONGVARBINARY
);
```

> `oauth_client_details` table is used to store client details.  
> `oauth_access_token` and `oauth_refresh_token` is used internally by OAuth2 server to store the access and refresh tokens.

**Create a client**

Let's insert a record in `oauth_client_details` table for a client named `appclient` with a password `appclient@123`.  
> `appclient` has access to the petstore resource with read and write `scope`.  
> The password needs to be saved to DB in Bcrypt format. I have used an online tool to Bcrypt the password with 8 rounds. 

`src/main/resources/data.sql`

```sql
INSERT INTO
  oauth_client_details (
    client_id,
    client_secret,
    resource_ids,
    scope,
    authorized_grant_types,
    authorities,
    access_token_validity,
    refresh_token_validity
  )
VALUES
  (
    'appclient',
    '$2a$08$ePUWmsLTqNezRk7MCUfg6.HU3RUO3N2M6H.Xj0gMvKiUsGgvg/Fve',
    'petstore',
    'read,write',
    'authorization_code,check_token,refresh_token,password',
    'ROLE_CLIENT',
    2500,
    250000
  );
```

**Create tables for users, groups, group authorities and group members**

Let's create tables to hold the users and groups details in embedded h2 db by providing the DDL scripts which runs during server startup.

`src/main/resources/schema.sql`

```sql
drop table users if exists;
create table users(
    username varchar_ignorecase(256) not null primary key,
    password varchar_ignorecase(256) not null,
    enabled boolean not null
);

drop table groups if exists;
create table groups (
    id bigint generated by default as identity(start with 0) primary key,
    group_name varchar_ignorecase(50) not null
);

drop table group_authorities if exists;
create table group_authorities (
    group_id bigint not null,
    authority varchar(50) not null,
    constraint fk_group_authorities_group foreign key(group_id) references groups(id)
);

drop table group_members if exists;
create table group_members (
    id bigint generated by default as identity(start with 0) primary key,
    username varchar(50) not null,
    group_id bigint not null,
    constraint fk_group_members_group foreign key(group_id) references groups(id)
);
```

**Add users, groups, group authorities and group members**

1. Let's create users named `john` with a password `john@123` and `kelly` with a password `kelly@123`.  
2. Create a group `PETSTORE_USER_AND_ADMIN_GROUP` and assign the roles `AUTHORIZED_PETSTORE_USER` and `AUTHORIZED_PETSTORE_ADMIN`.  
3. Similarly create a group `PETSTORE_USER_ONLY_GROUP` with role `AUTHORIZED_PETSTORE_USER`.  
4. Add `john` to group `PETSTORE_USER_AND_ADMIN_GROUP` and `kelly` to group `PETSTORE_USER_ONLY_GROUP`.

> The password needs to be saved to DB in Bcrypt format. I have used an online tool to Bcrypt the password with 4 rounds.  

`src/main/resources/data.sql`

```sql
INSERT INTO users (username,password,enabled) 
    VALUES ('john', '$2a$04$Ts1ry6sOr1BXXie5Eez.j.bsvqC0u3x7xAwOInn2qrItwsUUIC9li', TRUE);
INSERT INTO users (username,password,enabled) 
    VALUES ('kelly','$2a$04$qkCGgz.e5dkTiZogvzxla.KXbIvWXrQzyf8wTPJOOJBKjtHAQhoBa', TRUE);
  
INSERT INTO groups (id, group_name) VALUES (1, 'PETSTORE_USER_AND_ADMIN_GROUP');
INSERT INTO groups (id, group_name) VALUES (2, 'PETSTORE_USER_ONLY_GROUP');

INSERT INTO group_authorities (group_id, authority) VALUES (1, 'AUTHORIZED_PETSTORE_USER');
INSERT INTO group_authorities (group_id, authority) VALUES (1, 'AUTHORIZED_PETSTORE_ADMIN');

INSERT INTO group_authorities (group_id, authority) VALUES (2, 'AUTHORIZED_PETSTORE_USER');

INSERT INTO group_members (username, group_id) VALUES ('john', 1);
INSERT INTO group_members (username, group_id) VALUES ('kelly', 2);
```

**Configure OAuth2 Server**

Let's create a class `AuthServerConfig.java` and annotate with `@EnableAuthorizationServer`. This annotation is used by spring internally to configure the OAuth 2.0 Authorization Server mechanism

1. `JdbcTokenStore` implements token services that stores tokens in a database.    
2. `BCryptPasswordEncoder` implements PasswordEncoder that uses the BCrypt strong hashing function. Clients can optionally supply a "strength" (a.k.a. log rounds in BCrypt) and a SecureRandom instance. The larger the strength parameter the more work will have to be done (exponentially) to hash the passwords. The value used in this example is 8 for `client secret`.    
3. `AuthorizationServerEndpointsConfigurer` configures the non-security features of the Authorization Server endpoints, like token store, token customizations, user approvals and grant types.    
5. `AuthorizationServerSecurityConfigurer` configures the security of the Authorization Server, which means in practical terms the /oauth/token endpoint.    
6. `ClientDetailsServiceConfigurer` configures the ClientDetailsService, e.g. declaring individual clients and their properties.

`com.codeaches.oauth2server.AuthServerConfig.java`

```java
@Configuration
@EnableAuthorizationServer
public class AuthServerConfig extends AuthorizationServerConfigurerAdapter {

    @Autowired
    DataSource ds;

    @Autowired
    AuthenticationManager authMgr;

    @Autowired
    private UserDetailsService usrSvc;

    @Bean
    public TokenStore tokenStore() {
        return new JdbcTokenStore(ds);
    }

    @Bean("clientPasswordEncoder")
    PasswordEncoder clientPasswordEncoder() {
        return new BCryptPasswordEncoder(8);
    }

    @Override
    public void configure(AuthorizationServerSecurityConfigurer cfg) throws Exception {

        // This will enable /oauth/check_token access
        cfg.checkTokenAccess("permitAll");

        // BCryptPasswordEncoder(8) is used for oauth_client_details.user_secret
        cfg.passwordEncoder(clientPasswordEncoder());
    }

    @Override
    public void configure(ClientDetailsServiceConfigurer clients) throws Exception {
        clients.jdbc(ds);
    }

    @Override
    public void configure(AuthorizationServerEndpointsConfigurer endpoints) throws Exception {

        endpoints.tokenStore(tokenStore());
        endpoints.authenticationManager(authMgr);
        endpoints.userDetailsService(usrSvc);
    }
}
```

**Configure User Security Authentication**

Let's create a class `UserSecurityConfig.java` to handle user authentication.

1. `setEnableAuthorities(false)` disables the usage of authorities table and `setEnableGroups(true)` enables the usage of groups, group authorities and group members tables.  
2. `BCryptPasswordEncoder` implements PasswordEncoder that uses the BCrypt strong hashing function. Clients can optionally supply a "strength" (a.k.a. log rounds in BCrypt) and a SecureRandom instance. The larger the strength parameter the more work will have to be done (exponentially) to hash the passwords. The value used in this example is 4 for `user's password`   

`com.codeaches.oauth2server.UserSecurityConfig.java`

```java
@Configuration
public class UserSecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    DataSource ds;

    @Override
    @Bean(BeanIds.USER_DETAILS_SERVICE)
    public UserDetailsService userDetailsServiceBean() throws Exception {
        return super.userDetailsServiceBean();
    }

    @Override
    @Bean(name = BeanIds.AUTHENTICATION_MANAGER)
    public AuthenticationManager authenticationManagerBean() throws Exception {
        return super.authenticationManagerBean();
    }

    @Bean("userPasswordEncoder")
    PasswordEncoder userPasswordEncoder() {
        return new BCryptPasswordEncoder(4);
    }

    @Override
    protected void configure(AuthenticationManagerBuilder auth) throws Exception {

        // BCryptPasswordEncoder(4) is used for users.password column
        JdbcUserDetailsManagerConfigurer<AuthenticationManagerBuilder> cfg = auth.jdbcAuthentication()
                .passwordEncoder(userPasswordEncoder()).dataSource(ds);

        cfg.getUserDetailsService().setEnableGroups(true);
        cfg.getUserDetailsService().setEnableAuthorities(false);
    }
}

```

**Configure oauth2server project to run on port 9050**

`src/main/resources/application.properties`

```properties
server.port=9050
```

**Start the OAuth2 Server**

Run the `oauth2server` application as `Spring Boot App` and make sure the server has started successfully on port `9050`

```java
TomcatWebServer  : Tomcat started on port(s): 9050 (http) with context path ''
DemoApplication  : Started DemoApplication in 12.233 seconds (JVM running for 14.419)
```

## Test authorization server {#test_auth_server}

Now that we have the `oauth2server` application up and running, let's test the application by submitting few POST calls.

**Get a token**

Let's get a token from OAuth2 Server for `kelly` using the URI `/oauth/token` and `grant_type=password`

*Request*

```sh
curl -X POST http://localhost:9050/oauth/token \
    --header "Authorization:Basic YXBwY2xpZW50OmFwcGNsaWVudEAxMjM=" \
    -d "grant_type=password" \
    -d "username=kelly" \
    -d "password=kelly@123"
```

> `YXBwY2xpZW50OmFwcGNsaWVudEAxMjM=` is the Base 64 authorization version of client_id and client_secret.  

*Response*

```json
{
  "access_token": "13df4f18-7763-4772-9960-895ca905dd56",
  "token_type": "bearer",
  "refresh_token": "6d49fd10-b92e-4bb2-b58d-b83212d70bcb",
  "expires_in": 24999,
  "scope": "read write"
}
```

**Validate the access_token**

Let's validate the above retrieved **access_token** `13df4f18-7763-4772-9960-895ca905dd56` by making a call to OAuth2 Server using the URI `/oauth/check_token`

*Request*

```sh
curl -X POST http://localhost:9050/oauth/check_token \
    -d "token=13df4f18-7763-4772-9960-895ca905dd56"
```

*Response*

```json
{
  "aud": [
    "petstore"
  ],
  "user_name": "kelly",
  "scope": [
    "read",
    "write"
  ],
  "active": true,
  "exp": 1545401270,
  "authorities": [
    "AUTHORIZED_PETSTORE_USER"
  ],
  "client_id": "appclient"
}
```

**Get a new token by using earlier obtained refresh token**

Let's get a new token from OAuth2 Server by using the earlier obtained **refresh_token** `6d49fd10-b92e-4bb2-b58d-b83212d70bcb` using the URI `/oauth/token` and `grant_type=refresh_token`

*Request*

```sh
curl -X POST http://localhost:9050/oauth/token \
    --header "Authorization:Basic YXBwY2xpZW50OmFwcGNsaWVudEAxMjM=" \
    -d "grant_type=refresh_token" \
    -d "refresh_token=6d49fd10-b92e-4bb2-b58d-b83212d70bcb"
```
> `YXBwY2xpZW50OmFwcGNsaWVudEAxMjM=` is the Base 64 authorization version of client_id and client_secret.  

*Response*

```json
{
  "access_token": "807d4eda-ed9e-48d7-bc1a-29e78987376a",
  "token_type": "bearer",
  "refresh_token": "6d49fd10-b92e-4bb2-b58d-b83212d70bcb",
  "expires_in": 24999,
  "scope": "read write"
}
```
## Build resource server {#build_resource_server}

Let's create a Spring Boot REST Service named petstore and expose couple of end points. This will be our resource server.

**Create a Spring Boot starter project using Spring Initializr**

Let's utilize [spring initializr web tool](https://start.spring.io/){:target="_blank"} and create a skeleton spring boot project for PetStore Resource Server. I have updated Group field to **com.codeaches**, Artifact to **petstore** and selected `Web`,`Security`,`Cloud OAuth2` dependencies. I have selected Java Version as **11**

Click on `Generate Project`. The project will be downloaded as `petstore.zip` file on your hard drive.

>Alternatively, you can also generate the project in a shell using cURL

```sh
curl https://start.spring.io/starter.zip  \
       -d dependencies=web,cloud-security,cloud-oauth2 \
       -d language=java \
       -d javaVersion=11 \
       -d type=maven-project \
       -d groupId=com.codeaches \
       -d artifactId=petstore \
       -d bootVersion=2.1.3.RELEASE \
       -o petstore.zip
```

**Import and build**

Import the project in STS as `Existing Maven project` and do Maven build.

> Add the jaxb-runtime dependancy if the build fails with an error "javax.xml.bind.JAXBException: Implementation of JAXB-API has not been found on module path or classpath"

```xml
<dependency>
    <groupId>org.glassfish.jaxb</groupId>
    <artifactId>jaxb-runtime</artifactId>
</dependency>
```

**Configure petstore project to run on port 8010**

`src/main/resources/application.properties`

```properties
server.port=8010
```

**Enable Resource Server Mechanism on petstore application**

Let's annotate DemoApplication.java with `@EnableResourceServer`. This annotation is used by spring internally to configure the Resource Server mechanism

`com.codeaches.petstore.DemoApplication.java`

```java
@SpringBootApplication
@EnableResourceServer
public class DemoApplication {

    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

**Add REST methods to PetStore Applications**

Let's create a class `PetstoreController.java` and configure REST methods pet() and favouritePet()

> `/pet` can be acessed by user who belongs to `AUTHORIZED_PETSTORE_USER`.  
> `/favouritePet` can be acessed by user who belongs to `AUTHORIZED_PETSTORE_ADMIN`.

`com.codeaches.petstore.PetstoreController.java`

```java
@RestController
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class PetstoreController {

    @GetMapping("pet")
    @PreAuthorize("hasAuthority('AUTHORIZED_PETSTORE_USER')")
    public String pet(Principal principal) {
        return "Hi " + principal.getName() + ". My pet is dog";
    }

    @GetMapping("favouritePet")
    @PreAuthorize("hasAuthority('AUTHORIZED_PETSTORE_ADMIN')")
    public String favouritePet(Principal principal) {
        return "Hi " + principal.getName() + ". My favourite pet is cat";
    }
}
```
> Make sure to add `@EnableGlobalMethodSecurity(prePostEnabled = true)` to enable `@PreAuthorize` checks.

**Configure petstore application with OAuth2 Server token info URI and Client Credentials**

Let's update petstore application with client credentials for `appclient` and the `/oauth/check_token` URL of OAuth2 Authorization Server.
> Note that the client `appclient` is authorized to access petstore resource in oauth_client_details table. 


`src/main/resources/application.properties`

```properties
security.oauth2.client.client-id=appclient
security.oauth2.client.client-secret=appclient@123

security.oauth2.resource.id=petstore

security.oauth2.resource.token-info-uri=http://localhost:9050/oauth/check_token
```

**Start the petstore Resource Server**

Run the `petstore` application as `Spring Boot App` and make sure the server has started successfully on port `8010`

```java
TomcatWebServer  : Tomcat started on port(s): 8010 (http) with context path ''
DemoApplication  : Started DemoApplication in 12.233 seconds (JVM running for 14.419)
```

## Test resource server {#test_resource_server}

**Test `/pet` for a user who belongs to `AUTHORIZED_PETSTORE_USER`**

> Both john and kelly has access to `/pet`

*Request*

```sh
curl -X GET http://localhost:8010/pet \
    --header "Authorization:Bearer 807d4eda-ed9e-48d7-bc1a-29e78987376a"
```
> `807d4eda-ed9e-48d7-bc1a-29e78987376a` is the access_token obtained from OAuth2 Server for the user `kelly`.

*Response*

```
Hi kelly. My pet is dog
```

**Test `/favouritePet` for a user user who belongs to `AUTHORIZED_PETSTORE_ADMIN`**

> Only `john` has access to `/favouritePet`

*Request*

```sh
curl -X GET http://localhost:8010/favouritePet \
    --header "Authorization:Bearer 6092fa20-f191-4637-aa91-524472a99ac3"
```
> `6092fa20-f191-4637-aa91-524472a99ac3` is the access_token obtained from OAuth2 Server for the user `john`. 

*Response*

```
Hi john. My favourite pet is cat
```

**Test `/favouritePet` for a user user who `does NOT` belong to `AUTHORIZED_PETSTORE_ADMIN`**

> kelly does not belong to `AUTHORIZED_PETSTORE_ADMIN` and hence does not have access to `/favouritePet`

*Request*

```sh
curl -X GET http://localhost:8010/favouritePet \
    --header "Authorization:Bearer 807d4eda-ed9e-48d7-bc1a-29e78987376a"
```

*Response*

```json
{
    "error": "access_denied",
    "error_description": "Access is denied"
}
```

## Summary {#summary}

Congratulations! You just created a Spring Boot OAuth2 Authorization and Resource Servers with Jdbc Token Store and BCrypt Password Encoder.
