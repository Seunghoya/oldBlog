---
layout: single
title: "[인증/보안] Token Sprint Revew"
tag: [HTTP, HTTPS, TIL]
---

# Token Sprint Revew
이번에 진행한 토큰기반 인증 스프린트를 진행하면서 새로 공부한 내용들을 정리해보자.

## Server

### login.js
<script src="https://gist.github.com/Seunghoya/767450822d9b6143bfefd577b8d234c3.js"></script>
로그인이 작동하기 위해서는 데이터베이스에 저장된 정보가 요청한 정보와 일치하는지 먼저 찾아야 한다. sequelize에서 제공하는 메서드 중 findOne를 사용하면 요청받은 유저 아이디와 패스워드를 데이터베이스에 일치하는지 찾아낼 수 있다.<br>
이를 바탕으로 토큰생성할 때 필요한 정보들을 페이로드에 담아주고(10번 라인) 페이로드를 다음 토큰을 생성한 다음(17~18라인) 유저가 일치했을 때 200상태코드와 함께 쿠키에는 리프레쉬 토큰을, 응답에는 엑세스 토큰을 전달한다.

### accessTokenRequest.js
<script src="https://gist.github.com/Seunghoya/0c41f7bc264613dbbb904f2145a63d39.js"></script>
요청 헤더에 담겨있는 정보가 토큰정보를 이용해 검증 가능한 지 여부를 먼저 판단한 다음, 데이터베이스에 일치하는 유저 정보가 있는지 판단해, 일치여부에 따른 응답을 전송해준다.

### refreshTokenRequest.js
<script src="https://gist.github.com/Seunghoya/f33fcada0ae6889ec6427107f25ec79c.js"></script>
기본적으로 리프레쉬 토큰이 담겨있는 위치가 쿠키라는걸 보여준다. 
먼저 쿠키에 리프레쉬 토큰이 정상적으로 담겨있는지(9번라인), 있다면 정상적인 토큰인지(13번 라인) 제대로 정상적인 리프레쉬 토큰이 담겨있다면 이를 검증해 유저정보를 찾고, 이를 다시 페이로드에 담아서 새로운 엑세스 토큰을 발급해준다.

### jwt모듈/verift.js
```js
  if (!jwtString){
    return done(new JsonWebTokenError('jwt must be provided'));
  }

  if (typeof jwtString !== 'string') {
    return done(new JsonWebTokenError('jwt must be a string'));
  }

  var parts = jwtString.split('.');

  if (parts.length !== 3){
    return done(new JsonWebTokenError('jwt malformed'));
  }
```
서버파트를 다 진행했을 때에도 문제가 발생했는데, jwt토큰의 형태가 '.'으로 구분된 3블록으로 구성되어 있어야 한다는 것이다.<br>
jwt 공식문서 제일 첫 페이지에 나와있는 형태 그대로의 모습을 가지고 있어야만 모듈에서 토큰을 인식한다는 점에서 굉장히 놀라웠다. 

## Client

### Client/src/components/Login.js
<script src="https://gist.github.com/Seunghoya/a030453f2262440c36c61a4580ed3f97.js"></script>
로그인 파트에선 컴포넌트가 가진 상태를 이용해 서버에 로그인 요청을 보내고, 요청이 성공했을 때 로그인 상태를 바꾸면서 엑세스 토큰을 상태에 담아주는 로직을 구현했다.

### Client/src/components/Mypage.js
<script src="https://gist.github.com/Seunghoya/2fe104db6c9a7440d6df6e63cc12f0ba.js"></script>
마이페이지에서 요구하는 건 엑세스토큰과 리프레쉬토큰을 이용해 각각의 엔드포인트에 요청을 보내 엑세스토큰으로 유저정보를 받아오거나, 새로운 엑세스 토큰을 발급받는 과정을 구현하는 것이다. 
새로 발급받은 엑세스 토큰 정보를 상태에 저장하는 과정까지 필요했다.

### 후기
물론 임시적으로 제한된 스프린트 내에서 구현한 로직이지만 이를 바탕으로 토큰 인증에 대한 이해도를 넓힐 수 있는 기회였고, 특히 엑세스 토큰과 리프레쉬 토큰이 각각 담긴 위치와 사용처에 대해 확실하게 알 수 있었다. 끝