---
dg-publish: true
dg-home: false
---

>SSH : 다른 컴퓨터에 접속할 때 쓰는 프로그램

>[!SUCCESS]  목표 : WSL 환경에서 AWS EC2인스턴스에 접속하기

>[!tip] EC2 접속 시 SSH를  쓰는 이유 
>- EC2는 서버이기 때문에, 보안상 비밀번호 없이 키파일로만 접속이 가능하다 
>- 배울 것
>	- 키 파일 저장 
>	- 윈도우의 파일 WSL환경으로 복사 
>	- 보안 권한 설정
>	- SSH 방법
>	- 키파일 적용

#### 1. 키 파일 다운로드
> 다운로드 위치 가정 :  C:\ec2key\jungle.pem 

#### 2. WSL 내 .ssh 폴더 생성 
```bash
mkdir -p ~/.ssh
```
- `-p` : 디렉토리가 없으면 생성, 있으면 무시 


#### 3. WSL로 키 파일 복사 - cp 
- cp `[윈도우상 위치] ~/.ssh/`
```bash
cp /mnt/c/ec2key/jungle.pem ~/.ssh/
```
- `/mnt/c/...` : Window 디스크를 wsl에서 접근하는 경로 

#### 4. 키 파일 권한 제한 
- ec2에 보내기 전 키 파일 권한을 제한한다.

> [!INFO] 권한 제한 이유
> - SSH는 보안을 매우 중요하게 여겨서, 키파일이 너무 개방적이면 위험 요소로 간주하고 사용을 거부한다.
> - 따라서 400 모드로 권한을 주면된다.
> - **400 의미** : 나혼자만 읽을 수 있는 상태 (수정도 불가)
```bash
chmod 400 ~/.ssh/jungle.pem
```
<br>

**✔번외 : 권한 제한 확인 방법**
```bash
ls -l ~/.ssh

[정상 예시]
-r-------- 1 root root 1678 May 13 17:36 jungle.pem
```

#### 5. EC2 접속 
```bash
ssh -i [키페어] ubuntu@ec2pulicIp

ssh -i ~/.ssh/jungle.pem ubuntu@13.125.109.128
```

