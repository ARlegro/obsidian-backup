
> WSL2에서 Angular가 뜨지 않는 가장 흔한 이유
### `localhost`가 Windows 기준인지 WSL 기준인지 꼬임
- `ng serve`는 기본적으로 `localhost`에서 4200 포트를 연다.
- 근데 브라우저가 접근하는 `localhost`는 **Windows 호스트**
- Angular dev 서버는 **WSL 내부**에서 열려있기 때문에 접근이 안 될 수 있음.


### 해결 법 

방법 1. 터미널 이용 
```shell
ng serve --host 0.0.0.0
```


방법2. angular.json 수정 
> 항상 0.0.0.0으로 열기
```json
"serve": {
  "options": {
    "host": "0.0.0.0",
    ...
  }
}
```
`angular.json`의 `"serve"` 설정에 `host`를 지정해놓으면 매번 안 써도 됨:

