---
layout: post
title: HTTP Authentication methods
tags: [backend, api server, http, auth, authorization, authentication]
feature_image: 'https://images.unsplash.com/photo-1580795478690-5c6afcf4e7c3?ixlib=rb-1.2.1&ixid=eyJhcHBfaWQiOjEyMDd9&auto=format&fit=crop&w=1867&q=80'
---

<!-- more -->
오늘날, 많은 API (REST or not)들이, 사용자의 신원(account)을 확인하거나 제한된 리소스에 대한 접근 허용 여부를 판단하기 위해 HTTP에 기반한 인증 방식을 사용하고 있다. 과거에는 주로 browser cookie 에 기반한 session 관리 방식을 많이 사용했었다. 하지만 이 방식은 사용자 접속시 생성한 session 을 서버의 memory 에 유지하고 있어야 하는 부담이 발생하고, 서버를 여러 대로 확장할 경우 서버 간의 session 정보 공유가 어려워 시스템의 scalability 가 떨어진다. Redis 등의 look-aside session storage 를 따로 두어 확장성을 해결할 수도 있지만, 사용자 session 이 많아짐에 따라 시스템 전체에 session 을 유지하기 위한 부하가 증가한다는 문제는 그대로 남는다. 또한 cookie 는 단일 domain 및 그 하위 domain에서만 작동하도록 설계되어 CORS (Cross Origin Resource Sharing) 의 측면에서도, 쿠키를 기반으로 한 서비스는 구현하기가 까다로우며, 특히 최근 mobile traffic이 급증하면서 Android/iOS 모바일 어플리케이션에서는 별도의 cookie container를 사용해야 하는 등 불편함이 제기되고 있다.

그래서 최근에는, token 에 기반한 인증 방식을 사용하는 경우가 대부분이다. 서버는 클라이언트를 식별하기 위한 고유 string (token)을 발급하고, 클라이언트는 매 요청시 header에 token을 포함하는 방식으로 communication 하는 것이다.

<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-3617657034378751"
     crossorigin="anonymous"></script>
<ins class="adsbygoogle"
     style="display:block; text-align:center;"
     data-ad-layout="in-article"
     data-ad-format="fluid"
     data-ad-client="ca-pub-3617657034378751"
     data-ad-slot="2598977095"></ins>
<script>
     (adsbygoogle = window.adsbygoogle || []).push({});
</script>

### Authentication vs Authorization
깊이 들어가기 전에, Authentication 과 Authorization 이라는 용어의 차이에 대해 짚고 넘어가자. 많은 경우 두 용어는 구체적인 구분 없이 혼용되곤 한다. 두 가지를 구분하는 핵심 질문은

> 실제로 나에 대해서 **무엇**을 증명해 주는가?  
> What do they actually state or prove about me?

이다.

{% include figure.html image="/assets/img/20201104/authentication-authorization.png" position="center" caption="Authorization vs Authentication" %}

**Authentication**은, 내가 '누구'인지를 증명해주는 것이다. 지갑에 들어있는 신분증(주민등록증, 운전면허증, ....)의 역할이 이것이다. 반면 **Authorization**은, 접근 '권한'을 의미한다. 이를 테면, 지문 인식을 통해 스마트폰의 잠금을 해제하는 행동은 authorization이다. 사실상 대부분의 web service 에서는 접속한 사람이 실제로 '누구'인가는 관심이 없다. 정우성이 접속을 했든, 원빈이 접속을 했던 그 사람 자체의 신분에 관해서는 관심이 없다. 다만, 정우성이 이 시스템의 user 인지 admin 인지, 혹은 원빈이 banned user 인지에만 관심을 둘 뿐이다. 다시 말해 실제 web service 에서의 인증은 **authorization**에만 관심이 있다고 봐도 무방할 것이다. (물론 정부 기관이나, 금융 시스템 등에서는 '접속한 사람'이 실제 offline 에서의 '그 사람'이 맞는지에 대해 검증을 거쳐야 하므로 authentication 측면이 강하다고 볼 수 있다. 하지만 그 외 일반적인 회원가입을 거치는 web service에서는 그렇지 않다.)

### HTTP Authentication

HTTP 의 인증 방식에는 여러 가지가 있지만, 대체로 Basic, Bearer, Form-based, Digest 등의 방식들이 많이 사용된다. (실제로는 OAuth2 를 필두로, 대부분 bearer scheme을 사용하는 쪽으로 가는 추세이다.)

#### Basic

{%include figure.html image="/assets/img/20201104/security-httpBasicAuthentication.gif" position="center" caption="HTTP basic authentication" %}

가장 단순한 방식이다. Client는 server 에 요청을 보낼 때 username, password 를 header에 포함하여 보내고, 이를 받은 server는 (User DB등과 비교하여) 사용자 인증 처리를 한다. 이 방식은 계정의 민감한 정보인 password를 단순 base64 encoding을 통해 변환하여 평문<sub>plain text</sub>으로 전송하기 때문에 보안에 취약하다는 단점이 있다. 표준에 반드시 SSL을 이용하여 구현해야 한다고 명시되어 있지는 않지만, 그렇지 않을 경우 보안성을 보장할 수 없다. base64 encoding은 encryption이 아니다!

#### Bearer

Bearer authentication 은 원래 [OAuth 2.0](https://oauth.net/2/)의 일부로 [RFC6750](https://tools.ietf.org/html/rfc6750)표준에 제정된 인증 스킴이다. 한 줄로 축약하자면, "이 token의 bearer에게 접근 권한을 주세요" 라고 표현할 수 있다.

```
Authorization: Bearer <token>
```

위와 같은 형식으로 http header에 기록하여 server로 요청을 보내면, server는 자체 토큰 생성/검증 방식에 의하여 bearer token을 검증하고 그에 알맞는 처리를 하게 된다. bearer token 생성 방식에 대해서는, 표준의 범위에 포함된다/아니다 의견이 분분하지만 현재로서는 JWT(JSON Web Token)가 거의 de facto 표준으로 사용되고 있는 모양새이다.

Basic authentication과 마찬가지로, token을 탈취당할 경우 쉽게 정보가 유출될 수 있으므로 SSL을 이용한 구현이 권장된다. 다만 JWT에서는 이런 경우를 대비하여 access token과 동시에 refresh token을 발행하는 방식으로 보안성의 강화를 꾀하였다.

> JWT에 대해서는 다음 포스팅에서 조금 더 구체적으로 작성할 예정~~이다.~~이었으나, 한글로❤ 너무 잘 설명되어 있는 포스팅을 찾아서 링크로 대신합니다.  
> [JSON Web Token 소개와 구조](https://velopert.com/2389) by Velopert

#### Form-based

{% include figure.html image="/assets/img/20201104/security-formBasedLogin.gif" position="center" caption="Form-based authentication" %}

Server 가 login page (html, jsp 등)을 내려 주고, page 내의 html form을 이용해서 user로부터 username, password를 전송받는다. 전송받은 credential을 user db와 비교하여 일치하는 user data를 판별하여 인증한다. 이 방식은 전통적인 cookie-session 방식에서 많이 사용하는 방식이다. 다만, 개인적인 생각으로는 Bearer나 Basic 방식의 경우에도 최초 client로부터 username, password를 form을 통하여 받아 와야 하므로 이 역시 넓은 범주의 form-based 방식이라고 할 수도 있지 않은가 하는 의문이 드는 중이지만... 구태여 그렇게까지 strict하게 따지려면 밑도 끝도 없기 때문에 모른척🙄 하고 넘어가자.

#### Digest

MD5 hashing 을 이용한 암호화 인증 방식이다.
1. Client가 server에게 request를 보냄
2. Server는 nonce (number), realm (a hash string)을 client에 전달하고 인증을 요청
3. Client는 2에서 받은 realm을 hash key로 username, password를 MD5 hashing하여 nonce와 함께 server로 전송
4. Server는 hashing된 username, password를 비교하여 client를 인증 처리

### Conclusion

[API backend server - Planning to deployment](/series/#api-backend-server-planning-to-deployment) Series를 준비하면서, HTTP 사용자 인증 방식을 결정하며 찾아본 내용들을 정리해 보았다. 여기에 정리한 이외에도 HOBA (HTTP Origin-Bound Authentication) 등 다양한 인증 방식들이 있지만, 시간상 모든 방식에 대해 세세하게 공부하기에는 역부족이라...😫 내 프로젝트에서 사용하기로 결정한 Bearer 방식에 대해 조금 더 깊게 알게 되는 좋은 시간이었다고 생각하자.

### References

1. [4 Most Used REST API Authentication Methods](https://blog.restcase.com/4-most-used-rest-api-authentication-methods/)
2. [Basic, Bearer, Digest Oh MY! So Many Auths!](https://dev.to/caffiendkitten/authentication-types-3984)
3. [Bearer Authentication (from Swagger)](https://swagger.io/docs/specification/authentication/bearer-authentication/)
4. [\[JWT\]토큰(Token)기반 인증에 대한 소개](https://velopert.com/2350)
5. [백엔드가 이정도는 해줘야 함 - 5. 사용자 인증 방식 결정](https://velog.io/@city7310/%EB%B0%B1%EC%97%94%EB%93%9C%EA%B0%80-%EC%9D%B4%EC%A0%95%EB%8F%84%EB%8A%94-%ED%95%B4%EC%A4%98%EC%95%BC-%ED%95%A8-5.-%EC%82%AC%EC%9A%A9%EC%9E%90-%EC%9D%B8%EC%A6%9D-%EB%B0%A9%EC%8B%9D-%EA%B2%B0%EC%A0%95)
6. [JSON Web Token 소개와 구조](https://velopert.com/2389)

---