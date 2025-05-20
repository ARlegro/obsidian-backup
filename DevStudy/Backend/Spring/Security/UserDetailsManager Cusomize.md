> [!WARNING] Downside of Using Data Jpa
> - shoul use a predefined Schema, Table Structure  
> - it's **not flexiable**  

### Introduction
☑**Not use UserDetailsManager** 
- Not Many project are going to implement the UserDetaislManager 
- Cuz the disadvantages mentioned above (Instead, they use **UserDetailsService**)

**☑Not use UserDetailsService as is**
- Also, don't use UserDetailsService as is. 
- Instead, they use **Customize**)

☑**Customize userDetailsService**
- If i **customize userDetailsService**
	- Authentication Provider will use my cusomized UserDetailsServcie (Not use `JdbcUserDetailsService` or `InMemoryUser~~~`)
- How to customize?
	- Override loadUserByUsername() 

### Customize 
