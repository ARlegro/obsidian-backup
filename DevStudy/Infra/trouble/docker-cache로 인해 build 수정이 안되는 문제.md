

1. Docker 이미지 새로 빌드
```bash
docker-compose build --no-cache server
```

2.  기존 컨테이너 & 네트워크 모두 제거(필요 시)
	```BASH
	docker-compose down --rmi all
	```
3. Dev 프로파일로 컨테이너 다시 올리기


