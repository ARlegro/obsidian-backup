
```shell
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash

#환경 변수 바로 적용 
export NVM_DIR="$HOME/.nvm"
source "$NVM_DIR/nvm.sh"

#LTS 버전 Node.js 설치 및 상용
nvm install --lts
nvm use --lts

#Angular CLI 설치 (WSL 내부용)
npm install -g @angular/cli
```


```shell
nvm install --lts
nvm use --lts
npm install -g @angular/cli
cd ~/security/bank-app-ui
rm -rf node_modules package-lock.json
npm cache clean --force
npm install
ng serve
```


wsl 에 인텔리제이 launcher comman-line 추가 
```terminal
export PATH="$PATH:/mnt/c/Program Files/JetBrains/IntelliJ IDEA 2024.3.1/bin"
alias idea='idea64.exe'
```



``

