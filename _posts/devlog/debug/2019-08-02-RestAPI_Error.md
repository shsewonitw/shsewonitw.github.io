---
layout: post
title: "[Debug] 공공데이터 Rest API 사용 시 500번 에러"
subtitle: "Rest API 500 Error"
category: devlog
tags: debug
---

![image](https://user-images.githubusercontent.com/50475160/91658822-975ce480-eb06-11ea-8a5c-d9f913b7ddbb.png)


> :pencil:

Spring 웹 프로젝트 중, 공항의 정보를 유저에게 실시간으로 제공하기 위해 공공데이터를 받아와야 할 일이 있었다.

데이터는 [공공데이터포털](https://www.data.go.kr) 에서 Rest 방식으로 제공받을 수 있다.

다음은 공공데이터포털에서 제공해주는 샘플코드의 일부

```java
StringBuilder urlBuilder = new StringBuilder("http://openapi.airport.kr/openapi/service/StatusOfDepartures/getDeparturesCongestion"); /*URL*/
        urlBuilder.append("?" + URLEncoder.encode("ServiceKey","UTF-8") + "=서비스키"); /*Service Key*/
        urlBuilder.append("&" + URLEncoder.encode("terno","UTF-8") + "=" + URLEncoder.encode("1", "UTF-8")); /*터미널 구분 1: 1터미널 2: 2터미널 */
        URL url = new URL(urlBuilder.toString());
        HttpURLConnection conn = (HttpURLConnection) url.openConnection();
        conn.setRequestMethod("GET");
        conn.setRequestProperty("Content-type", "application/json");
System.out.println("Response code: " + conn.getResponseCode());
```

먼저 테스트로 웹 브라우저에서 직접 요청을 보냈을때 응답을 보면
![img1](https://user-images.githubusercontent.com/50475160/65506451-83782e00-df06-11e9-95d1-50dd9ae10cb5.PNG)

XML 형식으로 필요한 데이터가 잘 넘어오는 것을 볼 수 있다.

이제 서버단에서 데이터를 받아와 파싱 후 입맛에 맞게 사용하면 되겠다고 생각했지만 문제가 발생

> :beetle:

![error500](https://user-images.githubusercontent.com/50475160/65510447-ed490580-df0f-11e9-83b7-a2cc92940c0a.PNG)

`Error 500`이라 하면 위의 메세지에서도 보이듯이 내부 서버의 문제일 가능성이 가장 크다

그리고 `Error 500`은 어떠한 자세한 정보를 표시하거나 담고있지 않기 때문에 오류를 보고 정확한 원인을 파악하는것이 어렵기도 하다.



[stackoverflow](http://stackoverflow.com)에서 `rest api 500 error` 키워드로 뒤적이다 보니 크게 3가지 정도의 원인이 있었는데

:one: 말 그대로 '서버'오류

(여기서 말하는 '서버'는 내가 구현해놓은 서버가 아닌 rest 요청을 받아 응답을 해주는 api 제공자를 의미)

가장 대다수의 의견은 "현재 api 제공자의 서버에 무언가 문제가 있어서 응답을 제대로 못해주는 것" 이었다.

하지만 이번 경우는 위에서 브라우저로 직접 요청을 보냈을때 응답이 정상적으로 오는 것을 확인했다.

:two: 요청주소의 인코딩 문제

그 다음 많았던 의견이 요청주소의 인코딩 문제였다. 하지만 위의 샘플코드에서 보이듯이 URLEncoder를 통해 url 을 utf-8로 인코딩 해주고 있는것을 볼 수 있다. 

혹시나 해서 

```java
StringBuilder("http://openapi.airport.kr/openapi/service/StatusOfDepartures/getDeparturesCongestion");
```

이 부분까지 인코딩을 해봤지만 결과는 `Error 500` 으로 동일.

:three: 로직 자체의 문제 or 런타임 에러

마지막은 server-side 에서 작성된 코드의 로직 자체가 잘못됐거나 런타임 에러가 있을 것이라는 의견이였다.

하지만 내 IDE 에서는 어떠한 에러도 발생하지 않았고 단순히

```java
conn.getResponseCode()
```

에서 500이라는 숫자를 뱉을 뿐이였다.
그 말인 즉슨 Connection 자체가 내부 서버 문제로 제대로 이뤄지지 못하고 있다는 것인데,

똑같은 URL을 브라우저에서 요청보낼 땐 정상적인 Response가 오고

Java 코드로 요청을 보낼 땐 500번 에러가 발생한다.

> :bulb:

결국 해결은 헤더에 response data 타입을 명시적으로 지정해주면서 되었다.

그리고 api문서에 응답 데이터 표준은 xml 이지만 json 도 지원한다고 명시가 되어 있어서

json으로 받아보기로 했다.

```java
conn.setRequestProperty("Accept", "application/json");
```

그러자 결과는

![result](https://user-images.githubusercontent.com/50475160/65570659-d7cded00-df9c-11e9-9320-1bf868826f64.PNG)



500번 에러는 나오지 않고 응답이 잘 넘어온다.

물론 xml 로 지정해줘도 잘 넘어오는걸 볼 수 있었다.

브라우저상에서 직접 요청 보냈을때 표준데이터인 xml 로 알아서 넘어오는 것을 보고 

서버에서 똑같이 요청을 보내면 응답이 잘 올것이라고 생각했던 것이 착오였다.

> :end: 결론은 response data 타입을 명시적으로 지정해야 한다.
