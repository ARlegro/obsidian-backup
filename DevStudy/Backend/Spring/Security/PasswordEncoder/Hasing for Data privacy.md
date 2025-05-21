> Hassing is process where data is converted into a hash value or digest value 


### Feaures of Hashing 
- Data once hashed **is non-reversable** ➡ **No one can derive plain-text** from hash value
- No One can determine orginal data from hashed data
- Depending on what hash function i choose, I'll get different results.,
- Same input + function ➡ Always get same hash value
- Always 1 way
		![[PDF 7.png]]

Test Hashing in terminal Using openssl 
```shell
echo -n "EazyBytes@12345" | openssl dgst -sha256
SHA2-256(stdin)= 177b845bf42225b6da04ad14df88e4f27b6d872c3c4143f1f37c22431bb9dc1e
echo -n "EazyBytes@12345" | openssl dgst -sha256
SHA2-256(stdin)= 177b845bf42225b6da04ad14df88e4f27b6d872c3c4143f1f37c22431bb9dc1e
```
- Same "Input + Funtion" ➡ Same Result 
- note : sha256 is well-known hash-function 


### Drawback 

#### 💢 Same Hash Value ➡ Same Input 
- **Week1 (on Brute Force Attack)** 
	- Hacker will **cacluate the hash value** By taking commonly used PW 
	- And **match** user's hash value 
	#Attack #simple-pw  #used-pw

- **Week2 (on Dictionary or Rainbow Attack)**
	- **Not calculate hash value** unlike Week1
	- Instead, **prepare a table** with the most commonly used pw 
	- and **Match** hashed Data to hacker's table 
     > this method, don't have to calculate hash value. just take hash value 
  
#### 💢 Too Fast 
> Hashing is so fast that it's good for hackers to utilize the hacking techniques.



### How to overcome Drawback 
> Overcome hashing's drawback by following 2 solutions 

#### 1. Use 'Salts' during hashing

❓What is Salts
- **Random string** that is added to a password before hashing it 
	- Salts value is randomly generated on every data 
- it makes hash value unique even if 2 users have same password 

✅Benefit of Salts 
- **Prevent identical passwords from producing the same hash**
	- Without salt:
	    - `hash("password123") → abc123`
	    - `hash("password123") → abc123`
	        
	- With salt:
	    - `hash("password123 + saltA") → a1f9e3...`
	    - `hash("password123 + saltB") → 9c7d8b...`

- **Protect** against rainbow table attacks
	- rainbow table's precomputed list is no longer useful
	- Cuz Data is stored combination of `salt + passsword` hashing 
		- Password : pwd123  
		- Salt: aX4z9!K2  
		- Input for hashing : pwd123 + aX4z9!K2  
		- result of hashing :  hash(pwd123aX4z9!K2)

> That is, Purpose of Salts is To make hash values unique and prevent hash-based attacks

Example of hashing based salts ➡ Bcrypt, PBKDF2

> [!WARNING]
> **Only with salts, can't protect** haker's DB attack. Cuz **DB has salt+ hasedpw**. and If hacker steal these info, they can guess pw easily 

#### 2. Slowing hashing function  

❓Why make hashing function slow 
- thanks to hashing function's **speed**, hacker can try guessing in short time 
- so, Even if datas are stolen, We need to make it harder for hackers to guess pw By slowing (Require a lot of CPU, RAM )

✅Recommend hash algorithms
- Bcrypt, Script, Argon2 
- these are going to intentially slow
- Hacker will take reallly a lot of time and money to guess PW 

❓Won't the Slowing affect Business
- it won't affect the organization the same way
- Cuz every user is not going to login every sec, ms, min
	- **Login is infrequent** request 


Next : [[PasswordEncodder]]
