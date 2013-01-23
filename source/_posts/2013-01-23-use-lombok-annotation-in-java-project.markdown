---
layout: post
title: "Java 코드의 가독성을 높일 수 있는 Lombok Annotation을 추가해보자."
date: 2013-01-23 18:29
comments: true
categories: [java, programming, lombok]
---
Lombok을 사용해봅시다.
====================

자바에서 DTO, VO, Domain Object 만들다보면, 멤버필드에 대한 Getter/Setter 메소드, Equals, hashCode, ToString과 멤버필드에 주입하는 생성자를 만드는 코드 등으로 불필요하게 코드가 길어지는 경우를 볼 수가 있다. 불필요하지만 생성해야 하는 코드들을 줄일 수 있는 방법이 있다면, 얼마나 좋을까?

### Project Lombok 소개
* 사이트 : [Project lombok](http://projectlombok.org/)
* Lombok Feature : [http://projectlombok.org/features/index.html](http://projectlombok.org/features/index.html)
* Java Source를 컴파일할 때, Lombok의 Annotation을 확인해서 그에 적합한 메소들을 생성해주는 방식이라고 이해를 하면 되겠다.

### 적용사례
Lombok annotation을 적용하지 않은 현재 스터디에서 사용하는 User domain 코드
``` java User.java
package net.slipp.domain.user;

public class User {
	private String userId;
	private String password;
	private String name;
	private String email;

	public User(String userId, String password, String name, String email) {
		this.userId = userId;
		this.password = password;
		this.name = name;
		this.email = email;
	}

	public String getUserId() {
		return userId;
	}

	public String getPassword() {
		return password;
	}

	public String getName() {
		return name;
	}

	public String getEmail() {
		return email;
	}
	
	public boolean matchPassword(String loginPassword) {
		if (loginPassword == null) {
			return false;
		}
		
		return loginPassword.equals(password);
	}
	
	public void update(User user) {
		this.userId = user.getUserId();
		this.password = user.getPassword();
		this.name = user.getName();
		this.email = user.getEmail();
	}
	
	@Override
	public int hashCode() {
		final int prime = 31;
		int result = 1;
		result = prime * result + ((email == null) ? 0 : email.hashCode());
		result = prime * result + ((name == null) ? 0 : name.hashCode());
		result = prime * result + ((password == null) ? 0 : password.hashCode());
		result = prime * result + ((userId == null) ? 0 : userId.hashCode());
		return result;
	}

	@Override
	public boolean equals(Object obj) {
		if (this == obj)
			return true;
		if (obj == null)
			return false;
		if (getClass() != obj.getClass())
			return false;
		User other = (User) obj;
		if (email == null) {
			if (other.email != null)
				return false;
		} else if (!email.equals(other.email))
			return false;
		if (name == null) {
			if (other.name != null)
				return false;
		} else if (!name.equals(other.name))
			return false;
		if (password == null) {
			if (other.password != null)
				return false;
		} else if (!password.equals(other.password))
			return false;
		if (userId == null) {
			if (other.userId != null)
				return false;
		} else if (!userId.equals(other.userId))
			return false;
		return true;
	}

	@Override
	public String toString() {
		return "User [userId=" + userId + ", password=" + password + ", name=" + name + ", email=" + email + "]";
	}
}
```

* Lombok Annotation 적용코드
``` java User.java
package net.slipp.domain.user;

import lombok.*;

@AllArgsConstructor
@EqualsAndHashCode
@ToString
public class User {
	
	@Getter
	private String userId;
	@Getter
	private String password;
	@Getter
	private String name;
	@Getter
	private String email;

	public boolean matchPassword(String loginPassword) {
		if (loginPassword == null) {
			return false;
		}
		
		return loginPassword.equals(password);
	}
	
	public void update(User user) {
		this.userId = user.getUserId();
		this.password = user.getPassword();
		this.name = user.getName();
		this.email = user.getEmail();
	}
}
```

> 90라인이 넘는 코드를 30여줄로 1/3 줄일 수가 있다.

### 설치방법
1. lombok.jar 다운로드
	1. lombok.jar 직접 다운로드
		- [http://projectlombok.org/download.html]
	2. pom.xml dependency 추가
``` xml pom.xml
<dependencies>
	...

	<!-- Project lombok -->
	<dependency>
	    <groupId>org.projectlombok</groupId>
	    <artifactId>lombok</artifactId>
	    <version>0.11.6</version>
	</dependency>
	...
</dependencies>
```

2. lombok.jar 설치하기
	1. lombok.jar 직접 다운로드의 경우 : 
		- 다운로드 위치 이동
		- java -jar lombok.jar
	2. Maven pom.xml 에 추가한 경우 :
		- cd ~/.m2/repository/org/projectlombok/lombok/<version>
		- java -jar lombok-<version>.jar
	3. IDE 설치 위치[Specify locaion...]를 검색해서 경로에 추가한다.
		![lombok 실행 1](../images/lombok/lombok_01.png)
		![lombok 실행 2](../images/lombok/lombok_02.png)
	4. eclipse.ini or sts.ini 파일 변경되고 동일한 경로에 lombok.jar 추가됨
		![.ini 에 lombok 설정 추가](../images/lombok/lombok_03.png)
	- 별도로 프로젝트 내에 라이브러리를 추가해서 사용할 수도 있겠다.

### 소스코드에서 Lombok Annotation을 추가해보자
* [@Data](http://projectlombok.org/features/Data.html)
* [@Getter/@Setter](http://projectlombok.org/features/GetterSetter.html)
	* 접근제어 : AccessLevel[PUBLIC, PROTECTED, PACKAGE, PRIVATE]을 통해서 접근레벨을 제한할 수 있다.
		- 예 : @Getter(AccessLevel.PACKAGE), @Setter(AccessLevel.PRIVATE)
	* getter/setter 관례에 따라서 get필드명, set필드명 메소드가 생성됨
	* Getter
	* Setter
* [@EqualsAndHashCode](http://projectlombok.org/features/EqualsAndHashCode.html)
	* 코드에서 객체의 비교 등의 용도로 사용되는 equals(), hashCode() 메소드의 코드를 절감할 수가 있다.
	* @EqualsAndHashCode(exclude={"field1", "field2"}) 처럼 필요에 따라서 특정 필드를 제외할 수가 있다.
* [@ToString](http://projectlombok.org/features/ToString.html)
	* 로그Log에서 객체의 내용을 확인하는 등의 용도로 쓰이는 toString() 메소드를 대신할 수 있다.
	* @ToString(exclude="field1") 처럼 필요에 따라서 특정 필드를 제외할 수 있다.  
* [@Log](http://projectlombok.org/features/Log.html)
	* 최근에 알게된 기선님의 글() 녀석인데, Logger와 관련된 코드들을 줄일 수 있다. 
	* 추가하면 자동으로 필드에 private static final Logger log 가 추가된다. 이후 로그를 찍으려는 곳에서는 log.error(), log.warn(), log.debug(), log.info() 형태로 사용하면 된다. 

### 정리 
이전에 있던 회사에의 프로젝트에 적용해서 사용했던 녀석인데, 자바 코드를 작성하다보면 어쩔 수 없이 작성해야하는 메소드들을 애노테이션으로 클래스에 정의를 하는 것만으로 대신할 수 있다는 것은 가독성과 생산성을 높일 수 있는 좋은 수단이 된다. 
사실, 이 글은 한참 전에 작성(이걸 처음 사용해보기 시작했을 때)이 되었어야하는게 맞는데, 이제서야 작성한다. 이런이런. 
