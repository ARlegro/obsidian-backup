
###  Overview

Setting yaml for situation that exist only for 1 user
```yaml
spring:  
  security:  
    user:  
      name: ${SECURITY_USERNAME:admin}  
      password: ${SECURITY_PASSWORD:1234}
```

But In reality, there's **not just one user**!
So, Need to **setup that allosws us to create a number of users** 

> [!SUCCESS] Learning Objectives 
> - How to create Multiple user accounts
> - and, How to create their credentials iside the memory



### Related implementations and interfaces

`UserDetailsService` 
- Interface which is **helpu to deal with the user account**
- ex. create user, fetch the existing users 
- That is, This interfaces **support all user-related operation** 

> [!WARNING] Interface only has one methods(loadUserByUsername)

`UserDetailsManager`
- It is also Interface extends UserDetailsService
- UserDetailsManager has CRUD related user 
- Note : [[Flow 3. AuthenticationProvider and Manager, encoder]]
- Representative Implementations
	1. **InMemoryUserDetailsManager** : used to store user data inside a InMemory
	2. **JdbcUserDetailsManager** : used to store user data inside a DB 


### Config multiple users 

>Need to Register UserDetailsService 

☑Register Bean 
```java
@Bean  
UserDetailsService userDetailsService() {  
    UserDetails user = User
						    .withUsername("user1").password("pwd1234")
						    .authorities("read").build();  
    UserDetails admin = User
						    .withUsername("admin1").password("pwd4321")
						    .authorities("admin").build();  
    return new InMemoryUserDetailsManager(user, admin);  
}
```

`build()` : **change encodedPassword** 
```java
public UserDetails build() {  
    String encodedPassword = (String)this.passwordEncoder.apply(this.password);  
    return new User(this.username, encodedPassword, !this.disabled, !this.accountExpired, !this.credentialsExpired, !this.accountLocked, this.authorities);  
}
```





☑Internal of InMemoryUserDetailsManager Constructor
#create #store #InMemory
```java
private final Map<String, MutableUserDetails> users = new HashMap();

public InMemoryUserDetailsManager(UserDetails... users) {  
    for(UserDetails user : users) {  
        this.createUser(user);   // create : put user into HashMap context
    }
public void createUser(UserDetails user) {  
    Assert.isTrue(!this.userExists(user.getUsername()), "user should not exist");  
    if (user instanceof MutableUserDetails mutable) {  
        this.users.put(user.getUsername().toLowerCase(Locale.ROOT), mutable);  
    } else {  
        this.users.put(user.getUsername().toLowerCase(Locale.ROOT), new MutableUser(user));  
    }    
```
- Consturctor ➡ createUser ➡ put


and when we fetch users inMemory : loadUserByUsername()
```java
public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {  
    UserDetails user = (UserDetails)this.users.get(username.toLowerCase(Locale.ROOT));  
    if (user == null) {  
        throw new UsernameNotFoundEx
```


### noop 

#### 💔Error Cuz missing paasword encoding 
When coufigure userDetails like this correctly, I'll get 500 Error
```java
UserDetails admin = User  
        .withUsername("admin1").password("pwd4321")   ❌ 여기가 문제 
        .authorities("admin").build();  
return new InMemoryUserDetailsManager(user, admin);
```

![[Pasted image 20250519141042.png]]
#### Reason 

>[!EXAMPLE] Background : Spring Sercurity requires pw encoding 
>- Since 5.x, all pw **must be encoded** 
>- so **plain text pw like `pwd1234` can't be stored** 
>- Basically Without any prefix, spring security **assumes it's an encoded pw.**
>- Then, it attempt to match using DelegatingPasswordEncoder 
>- DelegatingPasswordEncoder try to figure out the encoder But it's null Cuz no prefix

❓Why do I get an error without the prefix even though the encoding defaults to `bcrypt`?
- Even though Spring Security uses `DelegatingPasswordEncoder` by default, it **requires a prefix to determine which encoder to use**. ⭐
- **`DelegatingPasswordEncoder.matches()`** flow 
```java 
private class UnmappedIdPasswordEncoder implements PasswordEncoder {
		
		public boolean matches(CharSequence rawPassword, String prefixEncodedPassword) {
				int start = prefixEncodedPassword.indexOf(idPrefix);  // '{'
				int end   = prefixEncodedPassword.indexOf(idSuffix, start); // '}'
				if (start < 0 && end < 0) {
				    throw new IllegalArgumentException(
				        "..."
```
no prefix ➡ `start < 0 && end < 0`  ➡ `IllegalArgumentException`
- if i configure, `.password("{noop}pwd4321)`, DelegatingPasswordEncoder doesn't throw this error 

> [!WARNING]  Encoder를 Custom해도 자동 인코딩은 절대 일어나지 않는다. 
> - 대신 평문 pw도 오류를 안내는 것일 뿐 
> - if custome PWEncoder 
> 	- userdetails에서 pw를 db or memory에 저장 시 기본적으로는 평문으로 저장됨
> 	- 근데 {} prefix 붙이면 암호화되면서 저장 됨
> - Encoding해서 저장하고 싶으면 여러가지 방법이 있는데 그건 나중에 배울 것 


####  One of solution 

{noop}
```java 
UserDetails user = User  
        .withUsername("user1")
	      .password("{noop}pwd1234")  // tells Spring to treat it as plain text
        .authorities("read").build();
```
- tells Spring to **use NoOpPasswordEncoder**, which means:
	- No hashing
	- Just compare plain strings: `input.equals(storedPassword)`


### Problem of {noop}
> ❓What is problem of beolow code 
```java 
    UserDetails user = User  
            .withUsername("user1").password("{noop}pwd1234")  
            .authorities("read").build();  

```



1. **Hard-coding** the credentials 
	- it is vulnerable to security

2. PW is **being stored as a plain text**
	- If hacker has access to application heap, the hacker can easily get pw

> So, we need to use PWencoder in a encrypted or hashing format

### Solution1. PasswordEncoder 
> There are a lot of encdoer. 
> PasswordEncoder is one of them 


#### How to Register 1. PasswordEncoderFactories
```java
@Bean  
PasswordEncoder passwordEncoder(){  
    return PasswordEncoderFactories.createDelegatingPasswordEncoder();  
}
```
- this's default PWencoder : `BCryptPasswordEncoder`
- result 
	- when setting userDetails pw, there is no need to add prefix ex. {noop}
	- Cuz automatically pw will be stored in Bcrypt hash formate
- So, Even if hackers access data, they can't interpret it 

> Of coures, Even if custom PWEncoder, It's okay to use a prefix when setting userdetails.
> Cuz the encoder will be different depending on what the prefix is 
```java 
    public static PasswordEncoder createDelegatingPasswordEncoder() {  
        String encodingId = "bcrypt";  ✅
        Map<String, PasswordEncoder> encoders = new HashMap();  
        encoders.put(encodingId, new BCryptPasswordEncoder()); //✅ 기본값 
        encoders.put("ldap", new LdapShaPasswordEncoder());  
        encoders.put("MD4", new Md4PasswordEncoder());  
        encoders.put("MD5", new MessageDigestPasswordEncoder("MD5"));  
        encoders.put("noop", NoOpPasswordEncoder.getInstance());  
        encoders.put("pbkdf2", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_5());  
        encoders.put("pbkdf2@SpringSecurity_v5_8", Pbkdf2PasswordEncoder.defaultsForSpringSecurity_v5_8());  
        encoders.put("scrypt", SCryptPasswordEncoder.defaultsForSpringSecurity_v4_1());  
        encoders.put("scrypt@SpringSecurity_v5_8", SCryptPasswordEncoder.defaultsForSpringSecurity_v5_8());  
        encoders.put("SHA-1", new MessageDigestPasswordEncoder("SHA-1"));  
        encoders.put("SHA-256", new MessageDigestPasswordEncoder("SHA-256"));  
        encoders.put("sha256", new StandardPasswordEncoder());  
        encoders.put("argon2", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_2());  
        encoders.put("argon2@SpringSecurity_v5_8", Argon2PasswordEncoder.defaultsForSpringSecurity_v5_8());  
        return new DelegatingPasswordEncoder(encodingId, encoders);  
    }  
}
```


#### How to Register 2. Specify encoder 

> If i wany to use only BCryptPasswordEncoder. follow below code
```java
@Bean  
PasswordEncoder passwordEncoder(){  
    return new BCryptPasswordEncoder();
```

❌But it ins't flexiable 
- if i go to company using legacy programming, 
- maybe i will face prefix like {noop}
- In this situation, PasswordEncoderFactories is better choice 


> [!WARNING]  Evne if register a custom PasswordEncoder bean, **automatic encoding will never occur** when storing passwords.
> - The password will still be stored **in plain text**, unless you explicitly encode it
> - if custome PWEncoder 
> 	- userdetails에서 pw를 db or memory에 저장 시 기본적으로는 평문으로 저장됨
> 	- 근데 {} prefix 붙이면 암호화되면서 저장 됨
> - Later, Learning Mehtod of storing in PWD Encoding [[3. REST API that support user-registration]]

Below code is one of that solution
```java
UserDetails admin = User  
        .withUsername("admin1").password("{bcrpt}pwd4321")  
        .authorities("admin").build();
```


❓그럼 왜 평문 저장해놓고 평문 입력하면 로그인 성공하냐???
- 평문과 평문이 만나면 자동으로 fallback비교 있어서 true가 됨 (암호화 효과만 없는 것일 뿐)



