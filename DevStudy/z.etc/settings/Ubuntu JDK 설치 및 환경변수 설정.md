```bash
sudo apt update
sudo apt install -y openjdk-21-jdk

java --version

#JAVA_HOME 환경변수 설정 
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH'      >> ~/.zshrc
source ~/.zshrc

# gradlew 실행 권한 주기 
chmod +x ./gradlew    
```


## 다른 방법 
> 17, 21 계속 바꾸기 힘들다면 

https://sdkman.io/install
Java 버전을 명령어 한 줄로 전환할 수 있음.

### zip 패키지 설치(있으면 안해도 됨)

> sdkman 설치하는데 zip을 풀 수 있어야 됨 
```java
sudo apt update
sudo apt install zip unzip -y
```


### sdkman 설치
```bash
curl -s "https://get.sdkman.io" | bash 

// sdk 명령어 활성화
source "$HOME/.sdkman/bin/sdkman-init.sh"`
```


### Java 설치

```bash
sdk install java 17.0.10-tem
sdk install java 21.0.2-tem
sdk use java 17.0.10-tem
```


### 전환

```bash
`sdk use java 17.0.10-tem # 또는 sdk use java 21.0.2-tem`
```





👉 이걸 `.bashrc`나 `.zshrc`에 적으면 로그인 시 자동으로 적용할 수도 있고, 디렉토리별 `.sdkmanrc` 파일로 **프로젝트별 버전 고정**도 가능함.