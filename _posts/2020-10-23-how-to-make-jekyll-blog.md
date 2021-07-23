---
title: Jekyll을 이용하여 Github 블로그 만들기(with windows, docker)
date: 2020-10-23 12:56:00 +0900
categories: [Github blog]
tags: [github blog, jekyll, windows, docker]    # TAG names should always be lowercase
---
## 개요
- 전반적 구성
	- Jekyll을 이용하여 웹 사이트에 필요한 정적(static) 파일을 생성한다(ex. html, css, js, img 등)
	- Github에 정적 파일 호스팅을 시도한다.
	- 매번 Github에 올려서 확인하기 어려움으로 로컬 환경을 구축하여 로컬에서 확인 후 Github에 올린다.
- 정적 웹 사이트 구조
	- 오직 정적 파일로만 이루어진 웹 사이트
	- 정적 파일을 호스팅 서버에 올리고, 웹 서버(ex. 아파치 웹 서버, nginx) 설정 및 실행시키면,  
		외부에서 호스팅 서버의 웹 서버로 접근하여 웹 사이트를 확인할 수 있게 된다. 	
	- Github이 호스팅 서버 역할을 해줌으로 우리는 Jekyll 파일만 생성하여 업로드하면  
		자동으로 Github 서버에서 자동으로 빌드하여 호스팅해준다.
- Jekyll 이란?
	- Ruby 언어로 만들어진 정적 사이트 생성기
	- > __*Jkyll 공식 도큐먼트에서의 Jekyll 정의*__  
	  > Jekyll 은 정적 사이트 생성기입니다.  
	  > 당신이 즐겨 사용하는 마크업 언어로 작성된 텍스트를 Jekyll 에 넘겨주면  
	  > 레이아웃을 사용해 정적 웹사이트를 생성합니다.  
	  > 사이트 URL 의 형식이나 어떤 데이터를 사이트에 표시할 것인지 등,  
	  > 여러 동작을 조정할 수 있습니다.  
	- [Jekyll 공식 도큐먼트](https://jekyllrb-ko.github.io/docs/){:target="_blank"}

## 대상 독자
- Github 계정이 있고 사이트에 로그인한 상태, [github 회원가입](https://github.com/join?ref_cta=Sign+up&ref_loc=header+logged+out&ref_page=%2F&source=header-home){:target="_blank"}
- docker 설치 되어있는 상태, [docker for windows 설치](https://hub.docker.com/editions/community/docker-ce-desktop-windows/){:target="_blank"}
- git for windows 설치 되어있는 상태, [git for windows 설치](https://git-scm.com/download/win){:target="_blank"}
- vscode가 설치되어 있는 상태(다른 에디터도 상관없다), [vscode 설치](https://code.visualstudio.com/){:target="_blank"}

## Jekyll 테마 선택 및 포크
- 설명
	- Jekyll 공식 도큐먼트를 보고 내가 원하는 웹 사이트를 만들 수 있지만,  
		다른 훌륭한 개발자가 만들어 둔 테마를 사용할 수도 있다.
- 실행방법
	- 웹 브라우저를 켜고 [Jekyll 테마 검색 사이트](http://jekyllthemes.org/){:target="_blank"}로 이동한다.
	- 원하는 테마를 검색하고 선택한다(나는 Chirpy를 선택했다).
	- 여기서 홈페이지를 누른다.  
		<a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/00-chirpy-jekyll-theme-site.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/00-chirpy-jekyll-theme-site.jpg" width="100%" height="100%"></a>
	- Jekyll theme의 github repository가 뜨면 fork를 누른다.   
	    <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/01-chirpy-github-fork.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/01-chirpy-github-fork.jpg" width="100%" height="100%"></a>
	- fork 뜬 repository에서 settings를 누른다.  
	    <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/02-forked-repo-settings.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/02-forked-repo-settings.jpg" width="100%" height="100%"></a>
	- reposiory 이름을 [github user name].github.io로 바꾼다.  
	   <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/03-rename-forked-repository.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/03-rename-forked-repository.jpg" width="100%" height="100%"></a>

- 확인
	- 5분 정도 기다린 후 [github user name].github.io로 접속해보면 테마 데모와 동일한 화면을 볼 수 있다.

## 파일 수정 및 원격 repository에 push
- 설명
	- 이제 fork 뜬 원격 repository의 내용을 변경만 하면 웹 사이트에 바로 적용된다.  
	- 파일 수정 사항을 repository에 적용하는 작업은 github 명령어로 이뤄지므로   
		명령어로 실행하는 환경(CLI)에 익숙하지 않는 유저라면 조금 어렵게 느낄 수도 있다.
- 실행방법
	- git bash를 실행한다.  
	  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/06-run-git-bash.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/06-run-git-bash.jpg" width="50%" height="50%"></a>  
	- 원하는 위치로 이동하고 디렉토리를 생성한다(여기서는 바탕화면에 git 디렉토리 생성)
		```
		> cd Deskop
		> mkdir git
		```
	- 여기서는 https 인증 방식으로 사용하도록 한다.
	- fork 뜬 repository로 가서 https 링크를 복사한다.
	- 클론 뜬다.  
		<a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/04-forked-repo-clone.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/04-forked-repo-clone.jpg" width="100%" height="100%"></a>  
		```
		> git clone [복사한 https 링크]
		```
	- 변경사항을 확인할 수 있게 _config.yml 파일을 수정한다.  
		[기존 블로그 제목] 부분을 원하는 제목으로 변경한다.  
		```
		# in config.yml
		...
		title: [기존 블로그 제목]
		...
		```
	- 모든 변경 사항을 fork 뜬 원격 repository에 적용한다.
		```
		# git DB에 추적할 변경 사항 추가
		$ git add .
		
		# 추가된 변경 사항 commit(커밋 메세지는 상황에 따라 다르게 작성하면 된다.)
		$ git commit -m "블로그 제목 수정"
		
		# repository remote branch에 push
		$ git push origin master
		```
	- repository를 확인한다.
		- fork 뜬 repository로 가서 commit란을 확인하면 적용된 것을 확인할 수 있다.  
		  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/05-check-new-commits.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/05-check-new-commits.jpg" width="100%" height="100%"></a>  
		- 블로그 url로 접속하여 title이 변경된 것으로도 확인이 가능하다.

## 로컬 환경 구축
- 설명
	- 매번 파일을 수정할 때마다 확인하기 위해 github에 적용하면 너무 번거롭다.
	- 그래서 로컬 환경을 구축하여 변경사항을 로컬에서 확인 가능하도록 한다.
	- 여러 방법이 있겠지만 내가 선택한 Chirpy는 bash shell script로 미리 짜둔 코드가 많아서  
		리눅스하고 맥만 지원한다.
	- 때문에 docker를 사용하여 리눅스 환경을 구축하고 그 위에서 local dev server를 실행하기로 하였다.
	- !알림 - docker에 익숙하지 않은 유저는 매우 복잡할 수 있으므로  윈도우를 지원하는    
	          다른 테마를 사용하여 docker 없이 로컬 윈도우에 ruby를 설치하여 처리할수도 있다.
- 실행방법
	- docker 실행
		- docker를 켠다.  
		  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/07-run-docker.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/07-run-docker.jpg" width="80%" height="80%"></a>  
		- cmd 창을 연다.  
		  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/08-run-cmd.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/08-run-cmd.jpg" width="80%" height="80%"></a>  
		- docker 실행 확인  
		  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/09-docker-run-check.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/09-docker-run-check.jpg" width="100%" height="100%"></a>  
		  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/10-docker-run-check-2.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/10-docker-run-check-2.jpg" width="100%" height="100%"></a>  
		- 내 블로그 디렉토리로 이동  
		  ```
		  > cd [내 블로그 경로]
		  ```
	- jekyll 컨테이너 생성
		- 설명
			- 많은 예시에서 매 실행마다 docker container를 생성하고 종료 시 container를  
			  삭제하는 방식을 사용하고 있었다.
			- 나의 경우 최초 container 생성 시 jekyll 및 bundle 명령어 실행이 너무 느려서  
			  container를 유지하는 방법을 선택했다.
		- jekyll docker image를 dockerhub에서 다운로드 받는다.
			```
			> docker pull jekyll/jekyll:4.1
			```
		- docker container 생성 및 bash shell 진입
			```
			> docker run --name my-blog -it -v "%cd%:.." -v "%cd%/vendor/bundle:/usr/local/bundle" -p 4000:4000 jekyll/jekyll /bin/bash
			```
	- vendor 디렉토리 캐싱
		- 설명
			- vendor 디렉토리를 캐싱하지 않으면 매 container 생성마다 의존성 패캐지를 다운로드하게 된다.
			- 이를 막기 위해 container와 공유하는 내 컴퓨터 경로에 vendor/bundle 디렉토리를  
			  다운로드 받아 캐싱한다.
		- bundle config 수정
			```
			$ bundle config set path "/vendor/bundle"
			``` 
		- Gemfile에 적힌 의존성 패키지 다운로드(최초 실행 시 오래 걸림)
			```
			$ bundle install
			```
	- 윈도우 개행문자 수정
		- 설명
			- bash shell script 파일의 윈도우 개행(new line)을 리눅스 개행으로 바꿔줘야 한다.
			- Gemfile도 수정해준다.
		-  윈도우 개행문자 수정
			```
			$ find / -name *.sh -exec sed -i 's/\r$//g' {} \;
			$ find / -name "Gemfile" -exec sed -i 's/\r$//g' {} \;
			```
	- jekyll 로컬 서버 실행 및 종료
		- 설명
			- Chirpy 테마 제작자가 로컬 서버 실행 커맨드를 정리하여 run.sh 만들어 두었기 때문에    
			  run.sh를 실행시켜 로컬 서버를 띄울 수 있다.  
			- 하지만 다른 대부분의 테마는 jekyll serve -H 0.0.0.0 명령어로    
			  로컬 서버를 띄울 수 있을 것이다.  
			- 블로그 파일의 일부를 수정할 때마다 로컬 서버를 재실행해야 변경사항을 확인할 수 있다.  
				(수정할 때마다 로컬 서버를 재시작 시키는 방법도 있겠지만 여기서는 다루지 않는다.)  
		- jekyl 로컬 서버 실행(최초 실행 시 오래 걸림)
			```
			$ bash tools/run.sh --docker
			```
		- jekyl 로컬 서버 실행 확인
			- 아래 화면이 나오면 로컬 서버가 실행 중인 것이다.  
			  <a href="/assets/img/2020-10-23-how-to-make-jekyll-blog/11-run-local-server.jpg" target="_blank"><img src="/assets/img/2020-10-23-how-to-make-jekyll-blog/11-run-local-server.jpg" width="80%" height="80%"></a>  
			- 웹 브라우저를 켜고 http://localhost:4000 로 들어가면 로컬 서버에서 빌드된  
			  내 블로그를 확인할 수 있다.
		- jekyll 로컬 서버 종료
			- Ctrl + c를 누르면 종료된다.
	- container 종료 및 재실행
		- 설명
			- 모든 작업이 종료되면 컨테이너를 종료하면 된다.
			- 실행 중인 docker 프로세스를 종료하거나, 컴퓨터를 종료하면 컨테이너는 자동 종료된다.
			- 종료된 컨테이너는 재실행 및 bash shell 진입이 가능하다.
		- 종료  
			```
			$ exit
			```
		- 재시작  
			```
			# my-blog 컨테이너 재시작
			> docker start my-blog
			# my-blog bash shell 접근
			> docker exec -it my-blog /bin/bash
			```
					
## 참고
- [jekyll-theme-chirpy git repository](https://github.com/cotes2020/jekyll-theme-chirpy/tree/master){:target="_blank"}
- [jekyll docker README](https://github.com/envygeeks/jekyll-docker/blob/master/README.md){:target="_blank"}
- [Compile a Jekyll project without installing Jekyll or Ruby by using Docker](https://dev.to/michael/compile-a-jekyll-project-without-installing-jekyll-or-ruby-by-using-docker-4184){:target="_blank"}
- [Improving Jekyll build time](https://carlosbecker.com/posts/jekyll-build-time/){:target="_blank"}
			