
### How to Valdiate PW 
![[image/PDF 1.png]]
Don't use plain-text Password 

--- 
### Data Privach 

> There is 3 ways for Data privacy 
1. Encoding
2. Encryption
3. Hashing 

![[image/PDF 3.png]]


> Which approach to choose depends on situation

#### 1. Encoding 
- Process of converting data from one form to another 
- There is **no need** to do with **cryptography** ⭐
	- Anyone can decode 
- **Not** **involved any secrets** and completely **reversable** ⭐ 
	- this is weak point  💢 ➡ Not Safe Process!! 
	- so Encoding is **not recommended for securing data** like password
- **When used❓**
	- **BinaryData** ex. image, video
	- ASCII, BASE64, UNICODE

>[!tip] Encoding Decoding Test

Encoding file By openssl 
```shell
openssl base64 -in plain.txt -out encode.txt
```

Decoding file 
```shell
openssl base64 -d -in encode.txt -out decode.txt
```
-d : decode


#### 2. Encryption
> Very safe, but not practical 

**☑Concept** 
- Process of trasforming data in such a way that **guarantes confidentaility**
- **How guarantes❓**
	- using **encryption algorithm**
- Ruslt of encrption is called 'key'
- For Decryption
	- need to 1) **same algorithm** and 2) **same security key** 

**☑Downside**
- It's not practical 
- Cuz 
	1. Hassle 
		- Has to generate 
		- Need key during encryption && decryption
	2. Need to take care of key. can't do anything 

![[image/PDF 4.png]]

##### **Deep Dive of Encryption** 
>[!SUCCESS]  There are 2 types of key in encryption algorithm 
>1. Symmetric 
>2. Asymmetric 

**1‍⃣ Symmetric Encryption**
![[image/PDF 5.png]]
**☑Concept**
- **use same secret key** during the encryption and decryption
	- It is important to manage secret key
- Very fast and efficient ➡ apt to encrypting large volumes of data 
- Key sharing is simple
	- Anyone with access to the key can both encrypt and decrypt the data.

✅When use it
- Best suited for **data at rest** (stored data), such as:
	- S3 buckets, database files, backups, etc.
- For example (S3 Bucket shared by a team)
	- Multiple users or systems can access the same encrypted data 
	- With symmetric encryption, the **same key** can be used to allow all authorized users to access the data.
	- There's **no need to encrypt the same file multiple times** with different keys.
	- Even If someone who has key lose key, it'no problem
			![[Pasted image 20250520172404.png]]

💢Downside 
- Key Distribution Problem ⭐
	- The **biggest weekness**
	- If the key is intercepted during transmission, the data can be compromised.

- Difficult to grand/revoke 
- **If Key is Leaked, All data is compromised** 
  > So, it's not secure process for password encryption

**2‍⃣ Asymmetric** 
![[image/PDF 6.png]]
☑Concept 
- There is two different key
	1. Public Key
	2. Private Key
 - The owner
	 - **keep** the **private key** themself
	 - **distribute** the **public key**
 - To Encryption ➡ Publcic key is needed
 - To Decryption ➡ Private Key is needed owned by owner
 - 🧰Only with public key, Can't decryption

✅When is useful ❓
- Data in transit 
	- When sending encrypted Data to client, even if hacker hijack data, hacker can't read data Cuz that data is needed decryption only by private key

💢Downside 
- Slow than symmetric
- Key management is complex
- One-way communication : this makes shared access complex 
	- So organization where works with many people is not used well 
	 - Cuz There is **no guarantee that it won't be stolen** if shared by multiple people.
  >That,s why encryption process is not for password proccess 
  
Next  : [[Hasing for Data privacy]]

