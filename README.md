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

***Create a client***

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



##  spring-boot-rest-data-jpa project run

1. `git clone https://github.com/ahasanhabibsumon/spring-boot-rest-data-jpa.git`
2. `project import any IDE`
3. `Go to application.properties and make sure databasename, username, password`
4. `Run spring boot project`
5. `open postman and import in postman REST-API-CRUD.postman_collection file `

