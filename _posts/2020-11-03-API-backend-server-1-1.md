---
layout: post
title: '[API backend] server 기획부터 배포까지 (1-1) - HTTP API 정의'
category: [Development, API backend server]
tags: [backend, api server, golang, go, server, http api, rest, software design]
feature_image: 'https://images.unsplash.com/photo-1522542550221-31fd19575a2d?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=2100&q=80'
---

<!-- more -->
# REST API? RESTful?
기술스택도 어느 정도 정했겠다, 이제 server 에서 제공할 API 를 정의할 차례다. HTTP 를 이용하여, JSON data 를 주고받는 것으로 client-server 간 communication 방식을 결정하고 나니, 왠지 REST API 여야만 할 것 같은 느낌이 든다. **REST**란 **RE**presentational **S**tate **T**ransfer 의 약자로, *Roy Fielding*이 [Architectural Styles and
the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm) 논문에서 제안한 Architecture 로, 오늘날 social network service 를 비롯, 대부분의 web-based service 에서 사용하고 있다. 하지만 엄밀히 말하면, 실제 대다수의 서비스들은 REST 의 요건을 충족시키지 못하고 있다. 주로,
- uniform interface 요건의
  - self-descriptive'
  - 'HATEOAS<sub>Hypermedia As The Engine Of Application State</sub>'

조건들을 만족하지 못하기 때문인데, 이 두 조건에 대해 조금만 알아보면 그 내용을 충분히 만족하는 서비스를 설계하고 구현한다는 것은 그리 간단한 일이 아님을 알 수 있다. Uniform interface 의 내용에 대한 자세한 설명은 [EungJun Yi](https://slides.com/eungjun) 님의 [그런 REST API로 괜찮은가](https://sookiwi.com/posts/tech/2018/11/11/Is-it-okay-with-such-REST-APIs/) 발표에 아주 쉽고 구체적으로 설명되어 있으니 참고하자.

# 빠른 포기
그래서 어설프게 REST API요! 하다가 참교육을 당하기보다는, 빠른 prototyping 을 위하여 HTTP API 선에서 만족하기로 결정했다. 당장 필요로 하는 API가 그리 방대하지 않아 10여가지 내외의 endpoint 를 제공하는 것만으로 충분하기도 했고.

# Concept 설계와 병행
여기서부터는 어느 정도 domain 개념의 설계와 병행해서 진행해야 한다. 여태까지는 일반적인 방법론, 개발 환경 혹은 어떠한 기술들을 사용할 것인가의 문제였다면 이제부터는 내가 제공하고자 마음먹은 서비스에 어떤 개념들이 들어가야 할 것인가에 대한 내용이 포함된다.

## Domain
Domain 이라 함은, 기술적인 측면이 아닌 실제 가치적인 측면에서 서비스가 제공하는 개념들을 의미한다. (물론, 사전적인 의미에서의 정확한 정의라기보다는, 내가 이해한 바를 글로 표현한 문구이다.) 이를 테면, banking service 를 제공한다고 하면 domain 에는 user, account, deposit 등의 개념이 포함되어야 할 것이고, 게시판<sub>board</sub> service 를 제공한다고 하면 user, post, comment, category 등의 개념이 포함될 것이다.

나는 주식<sub>stock</sub>에 관한 정보를 service 할 것이므로, user, stock, transaction 을 주요 domain 개념으로 설계하였다.

{% include figure.html image="/assets/img/20201103/domain.png" position="center" caption="Domain object design" %}

# API Endpoints
## for 'User' domain

| Method   | URI                           | Description            |
| -------- | ----------------------------- | ---------------------- |
| `POST`   | `/v1/users`                   | Join (Create new user) |
| `POST`   | `/v1/users/auth`              | User login             |
| `DELETE` | `/v1/users/{id}/auth`         | User logout            |
| `GET`    | `/v1/users/{id}/auth-refresh` | Auth token refresh     |
| `GET`    | `/v1/users/{id}`              | Return user info       |
| `Delete` | `/v1/users/{id}`              | Delete user            |

Join, login, logout 등 사용자 관리를 위한 API. Login, logout 등에 HTTP method + URI 의 형태로 어떻게 의미를 부여할 것인지에 대해 고민이 많았다. 검색해보니, 과거에는 '새로운 session을 요청한다', 'session 폐기를 요청한다' 의 의미로  `POST` / `DELETE` method 와 `/user/session/{id}` 형식의 URI를 사용하는 경우도 많았던 모양이다. 나는 인증 방식으로 HTTP header 의 Authorization 필드와 JWT 를 사용하기로 결정하였으므로, auth token 에 대한 생성/삭제/갱신 요청을 사용자 인증에 관련한 URI로 사용하기로 했다.

## for 'Transaction' domain

| Method   | URI                                | Description                                |
| -------- | ---------------------------------- | ------------------------------------------ |
| `GET`    | `/v1/users/{user-id}/transactions` | Return a user's all transaction list       |
| `POST`   | `/v1/transactions`                 | Create new transaction (buying or selling) |
| `PUT`    | `/v1/transactions/{id}`            | Update transaction                         |
| `DELETE` | `/v1/transactions/{id}`            | Delete transaction                         |

Transaction 에 관한 Endpoint 정의는 비교적 수월한 편이었다. Frontend에 직접 전체 transaction의 list를 제공할 경우는 현재 service 구상에서는 없을 것으로 판단하여 transaction 에 대한 `GET` 요청은 user-id를 요청에 포함하여 해당 user의 transaction list만을 filtering하여 제공하는 것으로 설계하였다.

## for 'Stock' domain

| Method | URI                           | Description                              |
| ------ | ----------------------------- | ---------------------------------------- |
| `GET`  | `/v1/stocks/search?q={query}` | Stock name 으로 isin (단축종목코드) 검색 |

현재로서는 종목에 대한 상세 정보를 제공할 계획은 없기 때문에, Stock 에 대한 API는 종목 이름으로 단축종목코드를 검색하여 결과 list를 전달해주는 용도 한 가지면 충분할 것이다.

# Next step

Github repository의 wiki 에 [HTTP API list](https://github.com/aiceru/Gazua/wiki/HTTP-API-%EC%A0%95%EC%9D%98)의 method + URI endpoint 를 정리하였다. 이제 Gitbook의 문서도구를 이용하여 [Gazua HTTP API specification](https://app.gitbook.com/@aiceru/s/gazua-http-api/)을 구체적으로 정의해야 한다. 여기에는 method, URI를 포함하여, reqeust (path parameter, body parameter), response JSON structure 등을 모두 구체화하여 정의한다.