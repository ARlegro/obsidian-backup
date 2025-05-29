
Most of request is using HTTPS protocol Cuz safe
So, most of organization use HTTPS server


> Default : Application is accepting only the plain HTTP request.
> Then, no one is going to use my website

>[!SUCCESS] Spring security help setting HTTPS 
>- In this page, will handle '**how to force to send the reuqest only using HTTPS**'
>- Default is acception both of HTTPS +HTTP 


### Configure

Spring Security's is originally accepting both of traffic(HTTPS + HTTP)
> I'll confgure accepting only HTTPS when prod environment 


#### Origin Config 
```java
@Configuration  
@Profile("prod")  
public class ProjectSecurityProdConfig {  
  
    @Bean  
    SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {  
        http.csrf(csrfConfig -> csrfConfig.disable())  
            .authorizeHttpRequests((requests) -> requests  
                    .requestMatchers("/myAccount", "/myBalance", "/myLoans", "/myCards").authenticated()  
                    .requestMatchers("/notices", "/contact", "/error", "/register", "/invalidSession").permitAll());  
        http.formLogin(withDefaults());  
        http.httpBasic(withDefaults());  
        return http.build();  
    }  
  
    @Bean  
    public PasswordEncoder passwordEncoder() {  
        return PasswordEncoderFactories.createDelegatingPasswordEncoder();  
    }
```

#### Change : depreciated 

```java 
@Bean  
SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {  
    http.requiresChannel(rcc -> rcc.anyRequest().requiresSecure())  // Only HTTPS
        .csrf(csrfConfig -> csrfConfig.disable())  
        ...
    http.httpBasic(withDefaults());  
    return http.build();
```

✅ Step1. requiresChannel( rcc -> ... ) 
- If someone is trying to send a request using HTTP protocol, 
- Then, this cofiguration **will stop a request** 
- 💢But it's not work in local env ➡ Cuz 
- note : rcc = require channel config 

This is Only HTTP setting for local env
```java 
http.requiresChannel(rcc -> rcc.anyRequest().requiresInsecure())
```

#### ⭐최신 cofig : RedirectToHttps 
> 현재 requiresChannel은 deprecated 
https://docs.spring.io/spring-security/reference/migration-7/web.html#use-redirect-to-https

requiresChannel ➡ redirectToHttps 
```java 
/** @deprecated */  
@Deprecated  
public HttpSecurity requiresChannel(Customizer<ChannelSecurityConfigurer<HttpSecurity>.ChannelRequestMatcherRegistry> requiresChannelCustomizer) throws Exception {  
  ApplicationContext context = this.getContext();  
  requiresChannelCustomizer.customize(((ChannelSecurityConfigurer)this.getOrApply(new ChannelSecurityConfigurer(context))).getRegistry());  
  return this;  
}  

[변경]
public HttpSecurity redirectToHttps(Customizer<HttpsRedirectConfigurer<HttpSecurity>> httpsRedirectConfigurerCustomizer) throws Exception {  
  httpsRedirectConfigurerCustomizer.customize((HttpsRedirectConfigurer)this.getOrApply(new HttpsRedirectConfigurer()));  
  return this;  
}
```
