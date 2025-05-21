
I'll use 3 Lower Environment for conditionally using security
- Dev
- SIT : System Integration Testing
- UAT : User Acceptant Testing

> If success def, sit, uate ➡ Can Product 




In prod
- There is no need to show-sql, format-sql
- log level : TRACE ➡ ERROR



### @Profile :  conditionally provider

> I want to have different provider depends on profile
> - prod : password checku
> - other : not check 

```JAVA 
@Profile("prod")
@Profile("!prod")
```

