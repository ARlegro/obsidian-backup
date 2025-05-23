
```terminal
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
➜  eazybank-start git:(3.3.0) ✗ export PATH="$PATH:/mnt/c/Program Files/JetBrains/IntelliJ IDEA 2024.3.1/bin"

➜  eazybank-start git:(3.3.0) ✗ alias idea='idea64.exe'

```