
인덱싱뿐만 아니라 파니셔닝, 샤딩, 동시성 등을 테스트하기 위해서는 수많은 rows를 생성하는 법을 알아야한다.


일단 테이블 생성 
```SQL 
CREATE TABLE temp
(
	value INTEGER NOT NULL
)
```


100만 데이터 삽입 
- INSERT시 VALUES 부분에 서브쿼리를 넣는 것이다.
```SQL 
INSERT INTO temp(value) 
SELECT random() * 100 
FROM generate_series(0, 1000000);


postgres=# select count(*) from temp;
  count  
---------
 1000001   // 0부터라 100만 1개임 
(1 row)
```

- `generate_series` = postgreSQL의 내장된 함수 
- `random()` 
	- 0~1의 랜덤 벨류 (소수점 2자리)
	- 소수점 없애려고 x 100  (INTEGER이기 때문)
> 위의 간단한 예시를 보면 1~100까지의 value를 가진 데이터들이 100만개 생긴다.


더미데이터 추가 예시 : [[Dummy Data]]
