
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





