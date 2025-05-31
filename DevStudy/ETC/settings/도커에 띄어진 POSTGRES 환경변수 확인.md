
```bash
docker exec -it postgres-db env | grep POSTGRES_
```

추가 : docker-compse에서 환경 변수 시 `-` 사용 필수 
```

POSTGRES_PASSWORD: ${DATABASE_PASSWORD:-pwd1234}
```
			
