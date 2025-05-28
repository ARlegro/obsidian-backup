```bash
sudo apt-get update
sudo apt-get install -y openjdk-21-jdk

java --version

#JAVA_HOME 환경변수 설정 
echo 'export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64' >> ~/.zshrc
echo 'export PATH=$JAVA_HOME/bin:$PATH'      >> ~/.zshrc
source ~/.zshrc

# gradlew 실행 권한 주기 
chmod +x ./gradlew    
```