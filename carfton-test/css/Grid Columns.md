


column들을 다룰떄 미리 정의해야하는 것이 columns이다
```html
<div class="columns">
	~~~ column 여러개 
</div>
```
이 안에서 column들이 사용?

3개의 content를 넣는다고 가정할 때 각 content는 자신만의 div 요소를 갖고, 각자 column클래스를 갖는다.


하나의 div는 12개의 column grid를 갖는다.
지정안하면 자동으로 나눠지지만
어던 요소에 몇 grid를 넣을지는 지정할 수 있따.

column 주는 법
```html
<div class="column is-2">

<div class="column is-5">
```



card안에 card-content로 감싼다음에 media 구조로 감싸기 


### 가로 정렬 카드 만들기

>[!tip] Card로 감싸기 ➡ Card-Content로 감싸기 ➡ Columns로 감싸고 Column마다 is-? 부여

![[Pasted image 20250515174717.png]]

```html
  <section class="section">
    <div class="container">
      <h3 class="title has-text-centered is-size4">Related Products</h3>
      <div class="card">
        <div class="card-content">
          <div class="columns">
            <div class="column is-4">
              <img
                alt="Product Image"
              />
            </div>
            <div class="column is-8">
              <p>$14</p>
              <p class="title is-size-5">Cortard Cup</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
```
