
아래와 같은 도커파일로 실행을 하던 중 
`xargs is not available`오류가 발생했다.


```dockerfile
FROM openjdk:21-jdk AS build  
WORKDIR /app  
COPY . .  
RUN chmod +x gradlew && ./gradlew clean build -x test;  
  
FROM openjdk:21-jdk  
WORKDIR /app  
COPY --from=build app/build/libs/*SNAPSHOT.jar app.jar  
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

### 원인 분석 
빌드 도구인 Gradle Wrapper는 내부에서 `xargs`를 호출한다.
`openjdk:21-jdk` 태그가 가리키는 이미지엔 `xargs`가 빠져 있던 것이다.
`openjdk`의 14버전 이상의 도커 이미지는 `xargs`를 포함하지 않는데 그 이유는 deprecated되었기 때문


### 해결방법 
https://github.com/gradle/gradle/issues/19682#issuecomment-1256202437
위의 링크에서 `xargs`를 지원해주는 것들을 이미지로 바꾸면 된다.
```dockerfile
FROM openjdk:21-jdk AS build

# xargs가 들어 있는 findutils 설치
RUN apt-get update && \
		apt-get install -y --no-install-recommends findutils && \
		rm -rf /var/lib/apt/lists/*

WORKDIR /app
COPY . .

RUN chmod +x gradlew \
 && ./gradlew clean build -x test
```



### 캐시 제거

이전에 실행했던 잘못된 명령어가 남아있어서 빌드가 안될 수 있기에 
캐시를제거해야 한다.
```bash
docker buildx prune -a
```

