---
layout: post
title: "octopress를 이용한 첫번째 포스팅"
date: 2013-01-11 00:54
comments: true
categories: etc 
---
Github에 블로그 개설하기(http://ihoneymon.github.com)
==============

### 사전준비
* git 설치
* [rbenv](http://octopress.org/docs/setup/rbenv/) 혹은 [RVM](http://octopress.org/docs/setup/rvm/)을 이용해서 ruby 1.9.3 설치 하기
* github에 username.github.com 으로 repository를 생성한다.
	ex) ihoneymon.github.com

### 참고사이트
* Octopress [http://octopress.org/](http://octopress.org/)
* [Installing Ruby with RVM](http://octopress.org/docs/setup/rvm/) - octopress
* [Installing Ruby With Rbenv](http://octopress.org/docs/setup/rbenv/) - octopress
* [Deploying to Github Pages](http://octopress.org/docs/deploying/github/)

### 기타사항
* Ruby, rbenv와 RVM이 설치되어 있어야 한다.
* Ruby 버전 확인 : 1.9.3 이상이어야 함
	```ruby --version``` or ```ruby -v```
	* 내 맥북 설치버전 : 1.8.7
* ruby 업데이트 하기
	* 참고사이트 : http://octopress.org/docs/setup/rvm/
	* install RVM
		```
		curl -L https://get.rvm.io | bash -s stable --ruby
		```
		- 자동으로 설치되어 있는 ruby를 업데이트 시켜준다.
	* install Ruby 1.9.3
		1. rvm install 1.9.3
		2. rvm use 1.9.3
		3. rvm rubygems latest

* Setup Octopress (Octopress 설정)
	1. git clone git://github.com/imathis/octopress.git octopress
	2. cd octopress
	3. ruby -v #ruby 1.9.3 이상이어야 함

	Next, install dependencies(의존성 있는 것들을 설치)

	4. gem install bundler
	5. rbenv rehash
	6. bundle install

	다음단계로 의존성 있는 라이브러리 설치
	
* Deploying(배포)
