

### Row_ID 
- 대부분의 DB는 자체적으로 시스템이 유지 관리하는 Row_ID를 생성한다.
	- Postgres는 이를 tuple_id라고도 부름 
	- 반면, mysql은 따로, row_id 가 없고 그냥 primary key 사실상 유사 Row_ID이다.
- 이는, 특정 Row를 공유하게 식별하는 Row 자체의 위치를 알려주는 역할을 한다.

### Page


고정된 크기 

한 테이블은 수천 개의 row가 있을 수 있으므로, 하나의 파일로 보관되더라도 **물리적으로 여러 페이지에 나뉘어 있음**

DB는 단일 row를 읽는 것이 아니라 I/O를 통해 Page를 읽어서 그 많은 row를 읽는 것 
Page들은 disk에 저장되어 있다.

가령 한 테이블에 1000개의 row가 있을 때, 각 page마다 5개의 row를 저장하고 있다면 200page가 있다는 것이다.
이 page들은 물리적으로 시작과 끝점을 가지고 있다.
> 페이지를 잘 알아야 성능을 높일 수 있다 ➡ Page 하나당 I/O작업이 하나라 잘 조절해야한다. 


### I/O

**디스크에 대한 읽기 요청을 수행하는 작업**
당연하지만, I/O를 적게 할수록 쿼리는 더 빨라진다.
I/O를 통해 DB에 있는 Page를 읽어 많은 Row를 가져올 수 있다.

#### I/O가 비싼 내부적 이유
1. **Page 1개  = I/O 작업 1개** 
	- Page하나당 I/O작업이 하나이다.
	- 즉, Page가 여러개면 I/O작업이 여러 번 일어난다는 것 
	> [!WARNING] 물론, partition같은 것을 통해 하나의 I/O 작업이 여러 Page접근하게 할 수 있음
	  
2. **조건이 있어도 전부 가져온다.**
	- `WEHRE`같은 조건을 적용해도 이는 DB에서 하는 것이기 때문에, 결국 Page를 읽는 것은 조건이 반영되지않는다.
	  
3. **'Byte ➡ 메모리'** 전환 비용 
	- Disk에는 데이터가 Byte형태로 되어있으며, 이를 DB가 읽기 위해서는 역직렬화를 해서 메모리 구조로 되돌려야한다.
	- 이 비용은 비싸다.


>[!tip] 결국 얼마나 Page를 Pulling할지 I/O작업을 일으킬지 아런 이유들로 성능이 차이남 


>[!WARNING] OS 캐시 - 반드시 I/O가 disk에 접근하는 것은 아니다. 
>-  I/O로 OS 캐시에  접근도 가능 
>- 일부 I/O는 OS 캐시로 이동한다.
>- 특히 PostgreSQL은 OS 캐시에 크게 의존한다 


### 힙(Heap)

데이터 테이블을 가리키는 **Page들의 모음** 
테이블 자체에 대한 **모든 것**을 가지고 있으며, 테이블의 모든 내용은 힙 안에 있다.

**💢문제 : 힙을 탐색하는 것은 비싸다 ❗**
- 모든 것을 가지고 있고,
- **줄때도 모든 것을 주기 때문**에 비용이 든다.

**✅해결 - Index** 
- 힙의 비싼 문제를 해결해주는 것이 INDEX이다.
- INDEX는 힙의 어떤 부분을 읽어야 할지 정확히 알려주는 데 도움이 된다.
- 따라서, 힙의 모든 페이지를 읽을 필요가 없어진다.


### 인덱스(INDEX)
>데이터 구조 

#### Concept 
- 힙으로 정확히 어디로 가야 하는지 알려주어 **효율적인 방식으로 정보를 가져오는 데 도움**이 되는 또 다른 **데이터 구조**
- 힙에 대한 포인터(Pointer)를 갖고 있다.
	- 포인터 = row_id를 가리키는 숫자
	- Row_id는 
		- HEAP 테이블의 페이지 번호 +  Offset 으로 구성된다.
		- 가져와야 할 데이터 힙 페이지에 대한 더 많은 메타데이터 가짐 
- 해당 poiter가 있는 페이지 정보도 담고 있음 

데이터의 일부를 가지고 있으며, 무언가를 빠르게 검색하는 데 사용

인덱스 또한 **페이지 형태로 저장**되며, 인덱스의 항목을 가져오기 위한 **I/O 비용**이 발생
데이터 구조의 묶음(대개 B-트리)이며, 이 B-트리를 읽어서 내용을 파싱해야 한다.
그리고 이거는 Disk에 존재한다? 따라서, 데이터베이스를 시작할 때, 인덱스를 디스크에서 읽어 메모리로 가져와야 한다.

일부 인덱스는 메모리에 들어갈 수 있지만, 다른 인덱스는 그렇지 않다.
힙에 원하는 페이지 들어가서 원하는 row만 가져올 수 있다???


####  INDEX 유무에 따른 FLOW

다음 쿼리를 INDEX유무에 따라 
```SQL
SELECT * FROM employee 
WHERE employee_id = 10000;
```

1. **INDEX 없는 경우❌**
	- Heap에 있는 Page들을 뒤져야 하는데, **데이터가 어디 Page에 있는지 모른 상태로 뒤진다.**
	- 최악의 경우 N번 뒤짐 
	- 물론 어떤 DB는 이 상황에서 Page를 뒤질때 Multiple Thread를 쓰기도 한다.
	  
2. **INDEX 있는 경우 ✅**
	- 우선, INDEX로부터 찾고자하는 데이터의 row_id와 페이지 정보를 얻는다.
	- Heap에 접근하는데, 페이지도 알고 row_ID도 알고 있으므로 바로 데이터를 가져올 수 있다.


>[!WARNING] UUID를 사용 시 성능 저하 및 구조 정리
>- UUID는 완전 랜덤값이라, 정렬되지 않은 값이다.
>- 인덱스 유지작업 문제 
>	-  UUID를 INDEX 컬럼으로 사용할 경우 유지 작업에 문제가 생긴다.
>	- 예를 들어, PostgreSQL은 해당 UUID값을 B-Ttree 인덱스로 정렬 관리하려고 시도한다. 
>	- 문제는 랜덤한 UUI값이 인덱스에 삽이될 때마다 노드 분할이라는 작업이 빈번하게 일어나고 디스크 접근 패턴도 비연속적이라 성능이 저하되는 문제가 있다
>- **인덱스를 자주 건드리는 구조라면 성능이 많이 저하** 
>> 정리 : UUID는 **비정렬된 특성 때문에 B-Tree 인덱스 성능이 저하**될 수 있다.

