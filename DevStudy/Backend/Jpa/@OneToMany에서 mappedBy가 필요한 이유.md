
### 요점 정리

| 항목                | 설명                                                         |
| ----------------- | ---------------------------------------------------------- |
| **mappedBy**      | `@OneToMany`에서 사용되는 속성으로, **반대쪽 엔티티에서 이 관계를 관리하고 있음을 명시**함 |
| **외래 키(FK)**      | 실제 DB에서 관계를 유지하는 핵심. → **관계의 주인이 갖고 있어야 함**                |
| **관계의 주인(owner)** | 외래 키를 관리하며 DB에 실제 영향을 주는 쪽 → 일반적으로 `@ManyToOne` 쪽          |

```JAVA
@Entity
public class Member {
    @ManyToOne
    @JoinColumn(name = "team_id")
    private Team team;
}

@Entity
public class Team {
    @OneToMany(mappedBy = "team")
    private List<Member> members = new ArrayList<>();
}
```


### 명시하지 않으면 어떻게 되나?

`@OneToMany`에 `mappedBy`를 명시하지 않으면, JPA는 해당 컬렉션을 **관계의 주인이라고 착각**하여 **중간 테이블 또는 잘못된 외래 키를 생성**하려고 합니다.
```JAVA 
`@OneToMany 
private List<Member> members;`
```
- 이렇게 하면 JPA는 `Team` 테이블에 외래 키가 있어야 한다고 잘못 판단하고,
- **연결 테이블(team_members)** 을 만들어버리거나, DB 설계를 위반하는 결과를 낳습니다.


> 따라서 
> - `@OneToMany(mappedBy = "...")` 는 **양방향 연관관계 설정 시 필수**
> - `mappedBy` 는 **관계의 주인이 어디인지** JPA에게 알려주는 역할