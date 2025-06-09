
세팅 
- 테이블 명 : test 
- 컬럼명 : a, b, c (전부 INTEGER 타입)
- 데이터 수 : 12million


준비된 쿼리 
```SQL
SELECT c FROM test WHERE a = 70
SELECT c FROM test WHERE b = 100

SELECT c FROM test WHERE a = 73 AND b = 123;
SELECT c FROM test WHERE a = 86 AND b = 65;
```

준비된 인덱스 
```SQL 
CREATE INDEX ON test(a)
CREATE INDEX ON test(b)
```

### SELECT c FROM test WHERE a = 70


비트맵 인덱스 스캔 발동 
- C는 인덱스가 아니기에 C를 가져오기 위해 Heap에 가야한다.
- Heap의 테이블에 접근하기 전에 모든 value의 map을 building 한다.


심화 : 여기서 `LIMIT = 10`을 주면 ❓
```SQL 


```
- 이번에는 Bitmap Index Scan이 아닌 Index Scan을 썼다.
- 이유 : 굳이 10개 찾는데 bimap만들면서 오버헤드 일으킬 필요 없다고 PostgreSQL이 판단한 것 
