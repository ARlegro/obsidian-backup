
```bash
# 설치된 Java 대안(alternative) 목록을 보고 정확한 경로 파악
sudo update-alternatives --config java

cd ~
nano ~/.bashrc   or  ~/.zshrc

파일편집 추가 
export JAVA_HOME=/usr/lib/jvm/java-21-openjdk-amd64/bin/java
export PATH=$JAVA_HOME/bin:$PATH

반영 
source ~/.bashrc

확인 
echo $JAVA_HOME
```


