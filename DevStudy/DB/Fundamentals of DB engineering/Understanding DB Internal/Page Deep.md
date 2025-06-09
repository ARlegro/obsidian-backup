
페이지 = 데이터 저장의 기본 단위

데이터를 저장하고 관리하는 방식은 물리적 디스크 공간을 효율적으로 조직하는 데 있다.
이 핵심 원리가 '페이지'이며 이를 이해해야 데이터베이스 성능 최적화와 문제 해결을 할 수 있다.

### Page 기본 개념 
- 데이터베이스 내의 **데이터 파일에 할당된 디스크 공간은** 논리적으로 0부터 n까지 연속적으로 번호가 매겨진 **페이지로 나뉘어진다**. 
- 페이지는 파일의 일부로 디스크에 저장된다.
- 디스크 I/O 작업은 페이지 수준에서 수행된다.
- SQL Server는 모든 데이터 행(row)을 페이지에 작성
- 모든 페이지는 DB별 정한 크기를 갖고 있다.

페이지 공간 유형 
1. 실제 데이터를 담는 곳 
2. 메타데이터를 저장하는 곳 

### 페이지 풀 (A pool of Pages)
>[!tip] 데이터베이스는 **공유 풀(shared pool)** 또는 **버퍼 풀(buffer pool)** 이라고 불리는 메모리 공간을 할당한다.

#### 읽기 작업에서의 풀 
- 디스크에서 읽어온 페이지들은 이 버퍼 풀에 저장된다.
- 일단 페이지가 버퍼 풀에 로드되면, 요청된 행뿐만 아니라 해당 페이지 내의 다른 행들(행의 길이에 따라)에도 접근할 수 있게 된다. 
- 이는 특히 인덱스 범위 스캔(index range scans)과 같은 읽기 작업의 효율성을 높여준다. 
- 행의 크기가 작을수록 한 페이지에 더 많은 행이 들어가므로, 한 번의 I/O 작업으로 더 많은 유용한 데이터를 얻을 수 있다.

#### 쓰기 작업에서의 풀 
>사용자가 행을 업데이트하면
1. 데이터베이스는 해당 행이 있는 페이지를 찾아 버퍼 풀(메모리)로 가져온다.
	- 업데이트 할 행만 가져오는게 아니라, 그 행이 속한 페이지를 전체를 가져온다.
	  
2. 메모리에서 해당 행을 업데이트한다.
3. 변경 사항에 대한 **저널 항목(Journal Entry)** (흔히 **WAL, Write-Ahead Log**라고 불림)을 디스크에 기록한다. 
4. **해당 페이지는 메모리에 남아** 있을 수 있어, 디스크에 최종적으로 플러시되기 전에 더 많은 쓰기 작업을 받을 수 있으므로 I/O 횟수를 최소화할 수 있다.
### 페이지 크기 

> 페이지 크기를 어떻게 조절하느냐에 따라 장단점이 있다.

1. 작은 페이지
	- 읽고 쓰는 속도가 빠르다.
	- 
2. 큰 페이지 


## 페이지가 디스크에 저장되는 방식

>[!tip] 
>데이터 베이스의 데이터는 row, 인덱스, 시퀀스 등 최종적으로 '페이지'에 저장된다.

### 하나의 예시 (DB마다 다름)

페이지를 디스크에서 읽고 쓰는 방법은 다양하다.
이번에는 하나의 방법에 대해 소개할 것 

방법 : 각 테이블 또는 컬렉션당 하나의 파일을 **고정 크기 페이지의 배열**로 만드는 것

이러한 설계 방식에서는, 디스크에서 무언가를 읽기 위해서는 1)파일이름 2) 오프셋 3)길이 
라는 3가지 정보가 필요하다.
- 어느 파일에서 (파일 이름)
- 어느 페이지부터 (오프셋)
- 어느 페이지까지 (길이)

### PostgreSQL 페이지 레이아웃 
> PostgreSQL의 기본 페이지 크기 = 8KB 
https://www.postgresql.org/docs/current/storage-page-layout.html

| Item           | Description                                                                                                      |
| -------------- | ---------------------------------------------------------------------------------------------------------------- |
| PageHeaderData | 24 bytes long. Contains general information about the page, including free space pointers.                       |
| ItemIdData     | Array of item identifiers pointing to the actual items. Each entry is an (offset,length) pair. 4 bytes per item. |
| Free space     | The unallocated space. New item identifiers are allocated from the start of this area, new items from the end.   |
| Items          | The actual items themselves.                                                                                     |
| Special space  | Index access method specific data. Different methods store different data. Empty in ordinary tables.             |
 페이지 헤더
 - 페이지 내의 내용을 설명하는 메타데이터공간
 - 또한 사용가능한 여유 공간으로도 쓰인다.

ItemId(항목 ID)
- item piointers의 배열 
- **항목이 페이지 내 어디에 있고 크기가 얼마나 되는지**를 가리킨다.
- HOT(Heap Only Tuple)최적화??를 가능하게 해준다.
	- 이전 항목 ID 포인터를 새로운 튜플을 가리키도록 변경합니다. 
	- 이런 식으로 인덱스와 다른 데이터 구조는 여전히 이전 튜플 ID를 가리킬 수 있어 매우 강력?

  > [!QUESTION] Row  vs  Tuple(item)
   >- Row : 사용자가 보는 것 
   >- Tuple(item) : 페이지 내의 행의 물리적 인스턴스 
   >- 하나의 Row가 여러 Tuple을 가질 수 있다.

**Items(항목)**
- **항목들** 자체가 페이지 내에 차례로 **저장되는 곳**
