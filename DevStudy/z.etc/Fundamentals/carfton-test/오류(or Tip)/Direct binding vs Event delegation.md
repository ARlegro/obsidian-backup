> JS에서 직접 바인딩 VS 이벤트 위임의 차이

시험을 준비하면서 HTML테그에 JQuery를 사용해서 직접적으로 click기능을 바인딩하려고 했다.
하지만 제대로 작동하지 않아 큰 문제가 발생 

MovieCard는 Ajax의 응답을 받아 나중에 DOM에 삽입하는 구조이다.
로드 시점에는 존재하지 않으므로 직접 

### 직접 바인딩 
```javascript
$(".trash-btn").click(function () {
  // 클릭 이벤트 처리
});

$("#sorter-likes, #sorter-viewers, #sorter-date").click(function (e) {
  ...
});
```
- 특정 요소에 직접 이벤트 리스너를 붙이는 방식이다.
- 💙장점
	- 코드 구조가 단순하고 직관적이다.
	
- 💢단점 
	- 현재 DOM에 있는 요소만 해당된다. 즉, 동적으로 추가된 요소에는 작동하지 않는 단점이 있다.
	- 

#### 직접 바인딩 실패 예시 
```js
$(".trash-btn").click(() => alert("삭제"));
```
- 위의 코드는 HTML 로드 직후 존재하는 `.trash-btn`에만 작동한다.
- 💢추후, append()로 추가된 카드에는 버튼 클릭해도 작동이 안된다 


### 이벤트 위임 
> 가능한 이것 쓰기 
```javascript
$(부모요소).on("이벤트타입", "자식선택자", "핸들러 함수")

$(document).on("click", ".trash-btn", function(){

})
```
- documnet에 클릭 이벤트를 등록한다
- 이벤트가 `.trash-btn`요소에 발생해 버블링이되면 조건에 맞는지 판단하고 핸들러를 실행한다.
- 💙장점
	- 버블링을 이용해 동적으로 추가되는 요소도 자동으로 처리된다.
	- 리스너 하나로 수많은 자식을 관리할 수 있다 (성능)
- 구조 
	- 
