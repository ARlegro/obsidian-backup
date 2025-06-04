만약 Java를 직접 설치하고 싶지 않고 Gradle이 자동 설치하도록 하고 싶다면, 다음을 `settings.gradle`에 추가:
```java 
pluginManagement {
    repositories {
        gradlePluginPortal()
        mavenCentral()
        google()
    }
}
```





참고 : wsl 에서 gradle 버전 깔기
```bash
sudo apt update
sudo apt install -y openjdk-17-jdk
```