

Filtering authorization란???


Annotaition 종류 
- @PreFilter : 파라미터로 뭘 받을 수 있을지 제한하는 것 

### 간단 소캐 
#### @PreFilter

![[supporter/image/PDF 26.png]]
filterObject = data type
contactName = Contact 객체의 필드명 

> Contact 객체의 필드명이 'TEST'가 아닐 경우에만 이 메서드의 파라미터로 받을 수 있다.
> ❓만약, `List<Contact>` 중에 하나만 그렇다면??

PreFilterAuthorizationMethodInterceptor에 의해 처리됨 


#### @PostFilter 
> 목적 : 

![[supporter/image/PDF 27.png]]

반환한 Contact중에 contactName 필드가 Test인게 있다면, 오류 터트린다.

PostFilterAuthorizationMethodInterceptor에 의해 처리됨 


## Demo

### @PreFilter
contact 컨트롤러 수정 
```java 



```


프론트 코드 변경 Cuz 컨트롤러의 param을 List로 변경했으므로 
(dashboard.service.ts ➡ saveMessag)
```typescript
[기존]
saveMessage(contact : Contact){  
  return this.http.post(environment.rooturl + AppConstants.CONTACT_API_URL,contact,{ observe: 'response'});  
}

[변경]
saveMessage(contact : Contact){  
  var contacts = [];  
  contacts.push(contact);  
  return this.http.post(environment.rooturl + AppConstants.CONTACT_API_URL,contacts,{ observe: 'response'});  
}
```


#### 테스트 해보기 ("/contact")
UI버전, Postman 버전 둘 다 

실행 - debug 모드 
- 분명 front에서 List형식으로 전달했는데 Debug모드로 보면 param의 List size = 0 이다.
- 이는, @PreFilter가 필터링했기 때문 (ContactName이 TEST이면 필터링되게)




### @PostFilter
> `@PreFilter`에서 했던 메서드에 annotation바꿔서 동일하게 적용해볼 것 

#### 컨틀롤러 메서드에 적용 
```java 
public List<Contact> saveContactInquiryDetails
```


ContactName = Test 인 Contact를 전달해도 이 메서드는 진행된다. Cuz @PostFilter
하지만, Return 타입에 그런 조건이 적용되기 때문에 filtering 
(오류인가?? 아니면 그냥 빈객체???)

#### 프론트 코드 변경 - conact.components.ts
> List형태로 전달해야 하므로 

 **✔기존**
```typescript

export class ContactComponent implements OnInit {  
  model = new Contact();

	saveMessage(contactForm: NgForm) {  
	  this.dashboardService.saveMessage(this.model).subscribe(  
	    responseData => {  
	      this.model = <any> responseData.body;  
	      contactForm.resetForm();  
	    });
```

기존 : model = new Contact();
변경 : contacts = new Array(); 

✅변경 
```typescript

```

saveMessage 전체적 변경 
마지막 요소만 `model`에 남깁
contactForm.resetForm() 으로 폼 초기화 

