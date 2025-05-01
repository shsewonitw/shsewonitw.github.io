---
layout: post
title: "[Linux] 알파인 리눅스 설치"
subtitle: "Alpine Linux"
category: devlog
tags: linux
---

![image](https://user-images.githubusercontent.com/50475160/91658696-7ba50e80-eb05-11ea-9e2f-20e65b58720d.png)


알파인 리눅스( Alpine Linux ) 는 가벼움과 보안성으로 Docker의 컨테이너로 많이 쓰인다.

먼저 [알파인 리눅스 홈페이지](https://alpinelinux.org)에 들어가서 본인 컴퓨터 아키텍쳐에 맞는 iso 파일을 다운받는다.

그리고 알파인 리눅스를 설치하기 위한 vm을 설치한다. ( 어떤 제품이든 상관없다 )

여기선 가벼운 [virtualbox](https://www.virtualbox.org)를 사용했는데, vmware 보단 확실히 사용성이 떨어진다.

윈도우에서는 문제없이 설치가 잘 될텐데, High Sierra 이상의 mac os에서는 설치가 안될수도 있다. 

그땐 [시스템 환경설정] - [보안 및 개인 정보 보호] - [일반] 에서 

<img width="550" alt="image" src="https://user-images.githubusercontent.com/50475160/91650395-2db6e900-eaba-11ea-98c5-6cbdbbebffcf.png">

소프트웨어를 허용해준 후 다시 설치하면 된다.

설치된 virtualbox 를 실행하고 새로만들기를 누른 후 아래의 과정을 거쳐 

vm 환경을 구성한다. 

<img width="889" alt="image" src="https://user-images.githubusercontent.com/50475160/91650472-00b70600-eabb-11ea-890d-c8902f7ff72b.png">

<img width="889" alt="image" src="https://user-images.githubusercontent.com/50475160/91650486-217f5b80-eabb-11ea-9dec-b7a5bb10f30e.png">

<img width="889" alt="image" src="https://user-images.githubusercontent.com/50475160/91650488-2f34e100-eabb-11ea-90c8-0183d98928b7.png">

여기까지 하면 os를 설치하기 위한 가상머신이 만들어진다. 




아래의 과정은 virtualbox 에서 터미널 명령을 작성하기는 

상당히 불편하기 때문에 ssh툴에서 쉽게 붙기위한 네트워크 세팅이다.

<img width="327" alt="image" src="https://user-images.githubusercontent.com/50475160/91650508-70c58c00-eabb-11ea-9b13-ddde6f16f8b3.png">

<img width="784" alt="image" src="https://user-images.githubusercontent.com/50475160/91650515-9783c280-eabb-11ea-80f8-26f82a6ae88c.png">

위처럼 네트워크 규칙을 만든 뒤, 아까 만든 가상환경의 [설정] - [네트워크] 에 들어가서 

아래처럼 셋팅을 해준다.

<img width="798" alt="image" src="https://user-images.githubusercontent.com/50475160/91650522-b41ffa80-eabb-11ea-9551-539f3eee481b.png">

네트워크 셋팅을 완료한 후 맨 처음 받았던 알파인 리눅스 iso파일을 가상 드라이브에 넣어준다.

<img width="932" alt="image" src="https://user-images.githubusercontent.com/50475160/91650551-04975800-eabc-11ea-83b3-80008c367b49.png">



모든 사전 셋팅을 완료하고 부팅을 하면 터미널 창이 나온다.

처음 접속 시 root 로 바로 들어가면 비밀번호 입력없이 접속이 가능하다.

```sh
setup-alpine
```

을 입력하여 알파인리눅스 세팅을 한다.

대부분 default 값으로 진행하면 되는데 , 

keyboard layout은 `us`

disk 선택 시 `sda`, `lvm`, `sys` 순으로 입력 해준다.

모든 과정 진행 후, 

아래처럼 iso파일을 드라이브에서 꼭 제거해준다. 
( 제거하지 않으면 재부팅 시 다시 설치 됨 )

<img width="530" alt="image" src="https://user-images.githubusercontent.com/50475160/91650777-a324b880-eabe-11ea-8e6a-92ab1ca886fc.png">

이미지 제거까지 완료 하였으면 재부팅을 해준다.
```sh
reboot
```



아래처럼 로그인 화면이 나온다면 Alpine Linux 설치와 셋팅이 완료 된것이다.

<img width="832" alt="image" src="https://user-images.githubusercontent.com/50475160/91650798-1e866a00-eabf-11ea-9add-72b072fe5a75.png">
