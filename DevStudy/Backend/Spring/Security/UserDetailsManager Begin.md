
> In this time, I will use JdbcUserDetailsManager
> So, need to add 3 dependencies
> - JDBC API
> - MYSQL
> - SPRING DATA JPA 


## Without JPA 
### Script by SQL Client

> SCL Cient ex. datagrip, Sqlectron

#### Save Schema 
```SQL
create table users(username varchar(50) not null primary key,password varchar(500) not null,enabled boolean not null);
create table authorities (username varchar(50) not null,authority varchar(50) not null,constraint fk_authorities_users foreign key(username) references users(username));
create unique index ix_auth_username on authorities (username,authority);
```

#### Save userDetails 
```SQL

INSERT IGNORE INTO `users` VALUES ('user', '{noop}EazyBytes@12345', '1');
INSERT IGNORE INTO `authorities` VALUES ('user', 'read');

INSERT IGNORE INTO `users` VALUES ('admin', '{bcrypt}$2a$12$88.f6upbBvy0okEa7OfHFuorV29qeK.sVbB9VQ6J6dWM1bW6Qef8m', '1');
INSERT IGNORE INTO `authorities` VALUES ('admin', 'admin');
```

--- 
### Change InMemoryUserDetailsManager ➡  JdbcUserDetailsManager
```java 
@Bean
UserDetailsService userDetailsService(DataSource dataSource) {
    return new JdbcUserDetailsManager(dataSource); 
}
```
- Since we scripted sql, there is no need to create UserDetails 


>[!tip] Tip : Managing import 
>- Solution1.
>	- ctrl + alt + o ➡ Remove unnessary import 
>- Solution2. 
>	- setting ➡ editor ➡ general ➡ Auto Import ➡ check Optimize imports on the fly


- 


--- 

> [!WARNING] 💢 Problem of process from now 
> - i should ask my clients or manager to create SQL Script


> Solution is using Spring Data Jpa Framework 


--- 
## Use Spring Data Jpa Framework 

```java
@SpringBootApplication  
@EnableWebSecurity   // optional 
public class EasyBackendApplication {
```
> `@EnableWebSecurity` is Optional in Spring boot

```java
public interface CustomerRepository extends JpaRepository<Customer, Long> {
```

```java
@Entity  
@Table(name = "customers")  
@Getter @Setter  
public class Customer {  
  
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)  
    private long id;  
  
    private String name;  
    private String email;  
    private String password;  
    private String role;  
}
```







 