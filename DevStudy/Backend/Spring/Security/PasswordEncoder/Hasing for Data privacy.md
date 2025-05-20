> Hassing is process where data is converted into a hash value or digest value 


### Feaures of Hashing 
- Data once hashed **is non-reversable** ➡ **No one can derive plain-text** from hash value
- No One can determine orginal data from hashed data
- Depending on what hash function i choose, I'll get different results.
- Same input + function ➡ Always get same hash value
- Always 1 way
		![[image/PDF 7.png]]

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


