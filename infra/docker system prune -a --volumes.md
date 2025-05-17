```
> [my-server internal] load build context: ------ failed to solve: ResourceExhausted: write /var/lib/docker/overlay2/9j51mu4nt0wejvsrwmix7j9ll/diff/.venv/lib/python3.12/site-packages/pip/_vendor/rich/__pycache__/_emoji_codes.cpython-312.pyc: no space left on device ubuntu@ip-172-31-10-225:~/craftonExam$
```

**no space left on device**
- Docker가 빌드하는 과정에서 디스크(특히 `/var/lib/docker`가 위치한 파티션)에 쓸 공간이 모자라서 중단된 것

디스크 사용 현황 `df -h`

도커가 사용중인 용량 확인 : docker system df 

**⭐불필요한 Docker 자원 정리** 
>[!tip] docker system prune -a --volumes
- 미사용 이미지(`IMAGE`), 중지된 컨테이너, 남은 볼륨을 한 번에 제거
- 볼륨까지 포함하면 상당한 공간 확보 가능


