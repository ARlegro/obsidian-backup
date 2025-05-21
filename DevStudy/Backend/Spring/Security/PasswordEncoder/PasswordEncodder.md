
> [!INFO] Previous [[Hasing for Data privacy]]
> - There is hashing's drawbacks
> - To solve them we need to
> 	- salts for hashing 
> 	- slow hashing function
> - Examples of Argolithms for solution ➡ Bcrypt, scrypt, Argon2, PBKDF2 


### PasswordEncoder Interface

> In spring security, there are a lot of implementaions of PasswordEncdoer 
>Before study implementaions, Let's go Interface 

```java
/**  
 * Service interface for encoding passwords. 
 * The preferred implementation is {@code BCryptPasswordEncoder}. **/

public interface PasswordEncoder {  
  
    /**  
     * Generally, a good encoding algorithm applies a SHA-1 or     
		 * greater hash combined with an 8-byte or greater randomly generated salt.  
		 **/    
		String encode(CharSequence rawPassword);  
  
    /**  
     * Verify the encoded password obtained from storage matches the submitted raw     
     * The storeed password itself is never decoded
     **/
     boolean matches(CharSequence rawPassword, String encodedPassword);  
     
     default boolean upgradeEncoding(String encodedPassword) {  
       return false;  
    }
```

- Preferred implementation : `BCyptPasswordEncoder`⭐
	- It is default encoder which is recommended by spring security
- The storeed password itself is never decoded ⭐
	- That is, can't deriver original value from DB 
- UpgradeEncoding is default false 
	- It is encoding more than 1 
	- It is usend only for super-critical applications 
	- Most of application use single time hashing. so Returns of UpgradingEncoding is false
	- If i want to multitime encoding, override it 



> Spring security's Default encoder is **DelegatingPasswordEncoder** 
> It's baed on **prefix** pw 


### Implementation

1. NoOpPasswordEncoder
	- It is for legacy 
	- no hashing, encoding ➡ just use equals 
	- If i want to store pw in the plain-text, use it 
	  
2. Pbkdf2PasswordEncoder
	- It it **deprecated** 
	- uses PBKDF2 that generate random salt value
	- Don't use it Cuz there are more advanced hashing argolithm
	  
3. **BcryptPasswordEncoder**  ⭐⭐
	- Preferred 
	- Using Bcrypt hashing function
	- Intentially Slowing down hashing function (By multiple iterating)
		- it demand lot of CPU ➡ can make hacker's life tough 
	- ⭐It's **easy for config** **for developer** Cuz provide a lot of constructor 
```java 
public class BCryptPasswordEncoder implements PasswordEncoder {  
	  ... 
    public BCryptPasswordEncoder() 
	  public BCryptPasswordEncoder(int strength) 
  
    public BCryptPasswordEncoder(BCryptVersion version) 
  
    public BCryptPasswordEncoder(BCryptVersion version, SecureRandom random)
  
    public BCryptPasswordEncoder(int strength, SecureRandom random) {  
  
    public BCryptPasswordEncoder(BCryptVersion version, int strength) {  
		
		public BCryptPasswordEncoder(BCryptVersion version, int strength, SecureRandom random) {  
			    ...
    }
```
		  
4. **ScyptPasswordEncoder**
	- Using scypt hashing fucntion 
	- It' is also requre a lot of CPU, memory like BcyptPasswordEncoder
	- Considered more stronger than BcyptPasswordEncoder
	- 💢But it's required tough configuration. Below is constructor
```java 
public class SCryptPasswordEncoder implements PasswordEncoder {

		public SCryptPasswordEncoder(int cpuCost, int memoryCost, int parallelization, int keyLength, int saltLength) {

```
	  
5. Argon2PasswordEncdoer
	- latest hashing argorithm
	- Advanced version of Scypt argorithm (Argon2 > Scypt)
	- Handles Scypt's drawback
	- 💢But it's also required tough config.
```java
public class Argon2PasswordEncoder implements PasswordEncoder {

		public Argon2PasswordEncoder(int saltLength, int hashLength, int parallelism, int memory, int iterations) {  
		    this.hashLength = hashLength;  
		    this.parallelism = parallelism;  
		    this.memory = memory;  
		    this.iterations = iterations;  
		    this.saltGenerator = KeyGenerators.secureRandom(saltLength);  
}
```


>[!QUESTION] Why spring security recommend BcyptPasswordEncoder
> #simple #simple-config #hard-coding
>- As mentioned, Argon2, Scypt is considered strong than Bcypt
>- ❓Then, Why recommend BcyptPasswordEncoder
>- ✅ **Scypt, Argon2 requires tough configuration** to developer 
>	- As you can see from the constructor, need to right-tuned CPU cost, memory cost etc 
>	- It's difficult to balance
>	- Not only makes things tough for hackers ➡ But also makse tough for developer



### DelegatingPasswordEncoder 

> When i config passwordEncoder, Never hard-coding
> Just. Delegate `DelegatingPasswordEncoder` 
![[supporter/image/PDF 9.png]]

DelegatingPasswordEncoder
- manage multiple hash types 
- So, can **delegate the correct implementation** of PasswordEncoder (**based on prefix** of the password)





### Summary : Which type of data privacy choose 

1. Password ➡ Hashing
2. Data at rest(저장된 데이터) ➡ Symmetric Encryption  ex. AES, DES 
3. Data in transit(전송중 데이터) ➡ Asymmetric Encrption   ex. RSA, ECC 
4. Binary data ➡ Encoding


