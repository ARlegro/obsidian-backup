
일단 간단한 문제

### 학급 회장(해쉬)

#### 문제 
```
학급 회장을 뽑는데 후보로 기호 A, B, C, D, E 후보가 등록을 했습니다.
투표용지에는 반 학생들이 자기가 선택한 후보의 기호(알파벳)가 쓰여져 있으며 선생님은 그 기호를 발표하고 있습니다.
선생님의 발표가 끝난 후 어떤 기호의 후보가 학급 회장이 되었는지 출력하는 프로그램을 작성하세요.
반드시 한 명의 학급회장이 선출되도록 투표결과가 나왔다고 가정합니다.

15
BACBACCACCBDEDE

C
```


#### 내가 푼 정답 
```JAVA
Scanner sc = new Scanner(System.in);  
int n = sc.nextInt();  
String line = sc.next();  
  
char[] choices = line.toCharArray();  
Map<Character, Integer> choiceMap = new HashMap<>();  
  
for (int i = 0; i < choices.length; i++) {  
    char c = choices[i];  
    Integer isFirst = choiceMap.putIfAbsent(c, 1);  
    if (isFirst != null && isFirst == 1) {  
        continue;  
    }  
  
    Integer count = choiceMap.get(c);  
    count++;  
    choiceMap.put(c, count);  
}  
  
int max = 0;  
Character result = null;  
  
for (Character c : choiceMap.keySet()) {  
    Integer count = choiceMap.get(c);  
    if (count > max) {  
        max = count;  
        result = c;  
    }  
}  
  
System.out.println(result);
```
내가 푼 답이 정답처리가 되긴했지만 가독성도 떨어지고 복잡하다 
Map 알고리즘은 처음 풀어서 낯설었는데, 좀 더 Map의 API에 익숙해져야되는 것 같다.



#### 모범 답안 
```java
Scanner sc = new Scanner(System.in);  
int n = sc.nextInt();  
String line = sc.next();  
  
HashMap<Character, Integer> map = new HashMap<>();  
for (char c : line.toCharArray()){  
    map.put(c, map.getOrDefault(c, 0) + 1);  
}  
  
int max = 0;  
Character result = null;  
for (Character c : map.keySet()){  
    Integer count = map.get(c);  
    if (count > max){  
        max = count;  
        result = c;  
    }  
}  
System.out.println(result);
```


getOrDefault라는 API❗❗





### 아나그램 (여러 방법 풀어보기)

> 개인적으로 이 문제는 보자마자 바로 풀었는데, 그래도 더 배울점이 없을까??? 에 대해서 GPT와 토론 후 Map방법이 아니라 다른 방법으로 코드를 줄이거나 시간 및 공간복잡도를 줄이는 방법을 위주로 적어 볼 것 

문제 
```
Anagram이란 두 문자열이 알파벳의 나열 순서를 다르지만 그 구성이 일치하면 두 단어는 아나그램이라고 합니다.

예를 들면 AbaAeCe 와 baeeACA 는 알파벳을 나열 순서는 다르지만 그 구성을 살펴보면 A(2), a(1), b(1), C(1), e(2)로

알파벳과 그 개수가 모두 일치합니다. 즉 어느 한 단어를 재 배열하면 상대편 단어가 될 수 있는 것을 아나그램이라 합니다.

길이가 같은 두 개의 단어가 주어지면 두 단어가 아나그램인지 판별하는 프로그램을 작성하세요. 아나그램 판별시 대소문자가 구분됩니다.

AbaAeCe
baeeACA

YES
```

내가 푼 답 
```java
Scanner sc = new Scanner(System.in);  
String x = sc.next();  
String y = sc.next();  
  
char[] xCharArray = x.toCharArray();  
char[] yCharArray = y.toCharArray();  
  
HashMap<Character, Integer> mapX = new HashMap<>();  
HashMap<Character, Integer> mapY = new HashMap<>();  
  
for(int i=0; i < xCharArray.length; i++){  
    char xc = xCharArray[i];  
    char yc = yCharArray[i];  
    mapX.put(xc, mapX.getOrDefault(xc, 0) + 1);  
    mapY.put(yc, mapY.getOrDefault(yc, 0) + 1);  
}  
  
String isAnargram = "YES";  
for(Character c : mapX.keySet()) {  
    if (mapX.get(c) != mapY.get(c)){  
        isAnargram = "NO";  
    }  
}  
System.out.println(isAnargram);
```

#### 개선 V1. 코드 줄이는 방법 (속도는 더 느림)
Arrays의 sort 메서드를 활용해서 각 입력값의 char 배열을 정렬하고 비교 
장점 : 정말 간단한 코드 
단점 : 시간 복잡도가 N log N 으로 더 느려짐 

>[!tip] Arrays.sort() : **Dual-Pivot Quicksort** 알고리즘을 사용 ➡ 평균적으로 O(n log n)
```java
        Scanner sc = new Scanner(System.in);  
        String x = sc.next();  
        String y = sc.next();  
  
        char[] xCharArray = x.toCharArray();  
        char[] yCharArray = y.toCharArray();  
  
        Arrays.sort(xCharArray);  
        Arrays.sort(yCharArray);  

        String isAnargram = "YES";  
        for (int i = 0; i < xCharArray.length; i++) {  
            if (xCharArray[i] != yCharArray[i]) isAnargram = "NO";  
        }  
  
        System.out.println(isAnargram);
```

#### 개선 V2. Map을 한개만 쓰기 
> 이거는 HashMap을 하나만 이용해서 푸는 것으로 공간을 좀 더 절약하는 방법이다.

2개의 문자가 아나그램이라면 어쨌든 각 문자열의 구성은 같다는 거니까 시소타기마냥 한쪽에서 더하고 한쪽에서는 빼고 하다보면 map의 value가 0이되야하는 원리를 이용 
```java 
Scanner sc = new Scanner(System.in);
String x = sc.next();
String y = sc.next();

char[] xCharArray = x.toCharArray();
char[] yCharArray = y.toCharArray();

HashMap<Character, Integer> frequentMap = new HashMap<>();

for(int i=0; i < xCharArray.length; i++){
		frequentMap.put(xCharArray[i], frequentMap.getOrDefault(xCharArray[i], 0) + 1);
		frequentMap.put(yCharArray[i], frequentMap.getOrDefault(yCharArray[i], 0) - 1);
}

String isAnargram = "YES";
for(Character c : frequentMap.keySet()){
		if (frequentMap.get(c) != 0) isAnargram = "NO";
}
System.out.println(isAnargram);
```


### 매출액의 종류 




#### 내가 쓴 답 : 타임 초과 오류 💢
```java
Scanner sc = new Scanner(System.in);  
int n = sc.nextInt();  
int k = sc.nextInt();  
  
int[] arr = new int[n];  
for (int i = 0; i < n; i++) {  
    arr[i] = sc.nextInt();  
}  
  
HashMap<Integer, Integer> map = new HashMap<>();  
  
int rt = 0;  
int[] resultArr = new int[n-k + 1];  
for (int lt = 0; lt < n - k +1 ; lt++) {  
    while (rt < k) {  
        int i = arr[lt + rt];  
        map.put(i, 0);  
        rt++;  
    }  
    resultArr[lt] = map.keySet().size();  
    rt = 0;  
    map.clear();  
}  
  
for (int i : resultArr) {  
    System.out.print(i + " ");  
}
```