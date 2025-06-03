
세팅 
```bash
docker run --name pgacid -d -e POSTGRES_PASSWORD=postgres postgres:latest

docker exec -it 8f75 psql -U postgres


```


### Automicity 테스트 
터미널로 아래꺼 직접 실행 
```SQL
create table products 
(
	pid serial primary key, 
	name text,
	price float,
	inventory integer
);	

create table sales
(
	saleid serial primary key,
	pid integer,
	price float,
	quantity integer
);

INSERT into products (name, price, inventory)
VALUES('phone', '999.99', 100);
```

products에 INSERT한 데이터의 inventory필드 10개 빼볼 것 

```SQL
BEGIN transaction;

select * from products (확인용)

UPDATE products SET inventory = inventory - 10;

%% 여기서 Crash 발생시키면?? %%
exit;

[재접속 후 확인]
SELECT * FROM products
-> 변경이 안됐다 ❗❗

```

exit ➡ crash같은 것 

수정 및 sales에 데이터 INSERT한 뒤, COMMIT 안한채로 
다른 터미널로 접속해서 데이터 접근해보기 



>트랜잭션 시작한 순간인거처럼  모든 쿼리들이 적용되는 것 
>이를, Repeatable Read라 부름 

Postgres에서 trascation 시작 시 Isolation level 부여하기
```SQL
BEGIN transaction isolation level repeatable read;
```
- Postgres의 default isolation level = Read Committed
- 이렇게 하면 첫 트랜잭션이 실행중일 때, 다른 트랜잭션이 Modify해도 첫 트랜잭션은 그거에 반영받지 않고 Reapeatable Read가 가능하다.


