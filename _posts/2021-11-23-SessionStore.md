---
layout: single
title: "[인증/보안] Session Store 확인"
tag: [HTTP, HTTPS, TIL]
---

![세션](https://media.vlpt.us/images/daeseongkim/post/909b572c-b8a8-4366-b6c9-9735de6dc3d7/code.png)

# Session?
- Cookie방식과 다르게 데이터를 서버에 저장하는 방식
- Server가 Client에 유일하고 암호화된 아이디(Session ID)를 부여
> Session은 실제 데이터는 서버에 보관해 두고 각각의 유저들에대한 고유 id 값을 쿠키에 저장하여 기록한다. 
<br>서버는 사용자의 ID값(Cookie에 저장되어 있는)을 비교 및 판단하여 어떤 사용자의 데이터인지와 그 사용자의 접속 여부를 판별할 수 있다.

### Session Store
세션은 유저의 데이터를 서버에 저장하는 방식이다. 세션이 데이터를 저장하는 장소를 Session store라고 한다. 어디에 저장이 되는 것인지 실제로 확인하기 위해 express-session 라이브러리를 통해 실제로 session을 만들어보면서 차례대로 확인해보도록 하자
```js
// app.js
const express = require("express");
const session = require("express-session");
const app = express();

app.use(
  session({
    secret: "keyboard cat",
    resave: false,
    saveUninitialized: true,
  })
);

app.get("/", (req, res) => {
  console.log(req.session);
  if (!req.session.num) {
    req.session.num = 1;
  } else {
    req.session.num += 1;
  }
  res.send(`Views : ${req.session.num}`);
});

app.listen(3000, () => {
  console.log(`server listening on port 3000`);
});
```
express-session을 통해 localhost:3000에 reload를 할 경우 Views를 올려보는 코드를 작성하였다.

`console.log(req.session)`를 통해 나온 값들을 확인해보면
![console.log(req.session)](https://media.vlpt.us/images/daeseongkim/post/6c3b5066-73b1-4aaa-ba0a-fa0e25553f9f/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-06-24%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209.37.44.png)
session이라는 객체에 num이라는 key가 생성이 되고, 코드에 따라 1씩 추가되는 것을 확인할 수 있다.

-> express-session 라이브러리가 session 객체가 생성이 되고, 그 세션에 어떠한 값들이 (어딘가에) 저장이 된다는 사실을 알 수가 있는데, 그 위치가 어디에 저장이 되는 것인지는 알 수가 없어서, express-session의 npm 설명을 찾아보니

default setting
> The default server-side session storage, MemoryStore, is purposely not designed for a production environment. It will leak memory under most conditions, does not scale past a single process, and is meant for debugging and developing.

라고 적혀있다... 조금 더 조사해보니 defalut 저장소는 MemoryStore인데 이것을 사용할 경우 서버나 클라이언트가 꺼졌다가 켜지면 날아가는 휘발성이 된다. 조금만 더 상황을 가정하면 서버가 껐다 켜지면 모든 로그인한 유저가 다 로그아웃이 될 가능성이 있는 것이다. 따라서, production 환경에서 MemoryStore의 사용은 적절하지 않으며, 추가적으로 복수 서버 상에서의 Session data 공유도 MemoryStore에서는 불가능하다고 한다.

### 그럼 Session 객체를 어디에 저장하는 것이 좋을까?
express-session 라이브러리 npm 페이지 를 살펴보면 호환 가능한 session-store 목록이 잘 나와있다. 이 중, 몇가지를 골라 테스트를 해보도록 하겠다.

### session-file-store

```
$ npm install session-file-store
```
을 통해 session-file-store 모듈을 설치해주고
```js
const FileStore = require('session-file-store')(session);
```
해당 모듈을 이런식으로 불러온다음
```
app.use(
  session({
    secret: "keyboard cat",
    resave: false,
    saveUninitialized: true,
    store : new FileStore()
  })
);
```
이렇게 session의 options에 store: new FileStore()를 추가해주면

![session-store](https://media.vlpt.us/images/daeseongkim/post/a57fb678-573c-4456-ae8b-278ed17723c2/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-06-24%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%209.51.23.png)

자동으로 sessions라는 폴더가 추가되고, 그 안에는 아까 console.log로 확인했던 session 객체가
```js
{"cookie":{"originalMaxAge":null,"expires":null,"httpOnly":true,"path":"/"},"num":2,"__lastAccess":1624539059623}
```
저장되는 것을 볼 수 있다.
file로 세션 객체가 저장되기 때문에 이제는 MemoryStore를 사용할 때와 달리 클라이언트와 서버 양측 모두 꺼졌다 켜져도 데이터가 날아가지 않는 것을 볼 수 있다.

### mysql-session
```
$npm install express-mysql-session --save
```
을 통해 express-mysql-session 모듈을 설치해주고
```js
const MySQLStore = require('express-mysql-session')(session);
```
해당 모듈을 이런식으로 불러온다음

```js
var options = {
  host: "localhost",
  user: "root",
  password: "1234",
  database: "session_test",
};

var sessionStore = new MySQLStore(options);
app.use(
  session({
    secret: "keyboard cat",
    resave: false,
    saveUninitialized: false,
    store: sessionStore,
  })
);
```
mySQL Options를 설정해준 뒤, 아까 file-store처럼 store에 new MySQLStore(options)를 정해주면!
![MySQL Session](https://media.vlpt.us/images/daeseongkim/post/d88fbc47-5ced-4110-aba7-d0308d80ac87/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA%202021-06-24%20%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE%2010.07.08.png)
이런식으로 자동으로 session table이 생성되면서 session_id와 data가 mySQL에 저장되는 것을 볼 수 있다!
