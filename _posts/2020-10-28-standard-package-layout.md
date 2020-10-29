---
layout: post
title: Standard Package Layout in Go
tags: [dev, golang, package, design]
feature_image: "/assets/img/20201028/gopher_package.png"
feature_license: "Title picture by Ashley McNamara CC BY-NC-SA 4.0"
---

<!-- more -->
최근 여러 군데 인터뷰를 보다 보니, 2차 이상의 technical interview 에서 system design 능력을 보는 경우가 많네요. 지난 주에 인터뷰를 진행했던 곳은 아직 국내에 크게 알려져 있진 않지만, 하고 싶었던 일이기도 했고 미래에 대한 가능성도 꽤 높은 것 같아 꼭 가고 싶었는데 아쉽게도 탈락했습니다. 개인적인 회고이지만, microservice 설계에 해당하는 질문이 나왔었는데 충분한 사전지식 없이 기존에 해오던 대로 program logic 만을 바탕으로 설계를 했었던 점이 문제가 아니었을까 싶어 조금 체계적으로 DDD(Domain Driven Design)에 대해 공부하고 있습니다.

조금 내용 자체가 추상적인 내용이라 어렵긴 하지만, [WTF Dial](https://medium.com/wtf-dial) series 에 Go 와 DDD 를 기반으로 한 설계/구현 내용이 비교적 이해하기 쉽게 나와있어서 참고하던 중, 해당 포스트 저자인 [Ben Johnson](https://medium.com/@benbjohnson?source=post_page-----9655cd523182--------------------------------) 이 제안한 ['standard package layout'](https://medium.com/@benbjohnson/standard-package-layout-7cdbc8391fc1) 포스트의 내용이 좋아 보여 번역을 해 보았습니다. 의역이 많이 섞여 있으니, 가급적 이 포스트를 보신 다음 원문도 함께 읽어 보시길 권합니다.

### Standard Package Layout

Vendoring, Generics 등은 Go community 에서 흔히 볼 수 있는 큰 이슈들이다. 하지만 거의 언급되지 않는 또 다른 (중요한) 이슈가 하나 있다. 바로 Application package layout 에 관련한 내용이다.

*어떤 구조로 코드를 만들 것인가?* 라는 질문에 대한 답은 모든 프로젝트 혹은 application 에서 제각각 다르다. 하나의 package 에 모든 코드를 작성하기도 하고, type 혹은 module 을 기준으로 package 를 나누기도 한다. 적어도 하나의 **team**으로 일할 때에는 package 구조에 대한 구체적인 (좋은) 전략이 필요하다. 그렇지 않으면 code 는 정돈되지 않고 제각각의 package 에 흩어진 채로 존재하게 될 것이며 이는 결국 유지보수의 비용 증가 등으로 이어지게 된다. → Go application design 의 **표준**을 만들고 싶다!!

#### Common flawed approaches

##### Approach #1: Monolithic package

모든 code 를 하나의 package 에 작성하는 방식이다. 단순하고, 작은 application 의 경우 잘 동작하며, dependency 자체가 없기 때문에 circular dependency 가 발생할 걱정이 없다.

경험상, 10,000 SLOC*<font size="1rem">source lines of code</font>* 이하의 규모에서는 큰 문제가 발생하지 않았다. 하지만 그 이상의 규모에서는 code 를 탐색하거나, 다른 code 들로부터 분리해 내기가 굉장히 어렵다.

##### Approach #2: Rails-style layout

또 다른 접근법은, functional type 에 따라 package 를 구분하는 것이다. 예를 들면, handlers, controllers, models 와 같은 식으로 나눈다. 주로 (나를 포함한) [Rails](http://rubyonrails.org/) 개발자들이 이런 식으로 package 를 나누는 편이다.

이 접근법에는 두 가지 issue 가 존재한다. 첫 번째로 naming 이 극악<font size="1rem">atrocious</font>하다. ```controller.UserController``` 처럼, type name 과 package name 에 같은 단어가 중복된다. 나는 naming 에 민감한 편이고, name 은 가장 훌륭한 documentation 방법이라고 생각하기 때문에 (누군가 당신의 code 를 볼 때, 가장 먼저 접하게 되는 부분은 name 이다!) 이러한 중복은 문제라고 생각한다.

더 큰 문제는, circular dependency 의 문제이다. 다양한 functional type 들은 서로를 reference 하게 될 확률이 매우 높다. Rails-style 의 접근법은 오로지 one-way dependency 만 존재할 때 잘 작동하는데, 대부분 우리가 구현하고자 하는 application 들은 그리 단순하지가 않다.

##### Approach #3: Group by module

이 접근법은 function 이 아니라 module 을 기준으로 구분한다는 것 외에는 위 2번째의 접근법과 비슷하다. 예를 들면, ```user``` package 와 ```accounts``` package 로 나뉘는 경우 등을 볼 수 있다.

이 역시 마찬가지로 naming issue 가 존재한다. (이를 테면, ```users.User``` 라던가...) 거기다 두 번째 issue 인 circular dependency 의 issue 역시 해결되지 않는다. - ```accounts.Controller``` 와 ```users.Controller``` 가 서로 참조하며 interact 하는 상황을 생각해 보라!

#### A Better Approach

내가 프로젝트를 진행할 때 사용하는 규칙은 아래 4가지이다.
1. Root package is for domain types
2. Group subpackages by dependency
3. Use a shared *mock* subpackage
4. *Main* pakcage ties together dependencies

이 규칙들은 각각의 package 들을 완전히 독립적으로 설계하고, application 전체에 걸쳐 명확한 domain language 를 정의하는 데 유용하다. 이 규칙들이 어떻게 적용되는지 실제 예를 들어 보도록 하자.

##### 1. Root package is for domain types

Domain 이란, data 와 process 들이 어떻게 상호작용하는지를 서술하는 논리적인 하나의 high-level language 이다. E-commerce application 에서는 customers, accounts, charging credit cards, handling inventory 등이 있을 수 있고, Facebook 이라면 users, likes, relationships 등이 있을 수 있다. 이러한 domain 의 정의는 실제 application 을 구현하기 위한 특정 technology 들과는 전혀 상관없이 정의된다.

나는 이러한 domain type 들을 root package 에 정의한다. 이 root package 에는 user 의 정보를 저장하는 ```User``` struct 나, user 정보를 가져오거나 저장하는 ```UserService``` interface 와 같이 단순한 data type 들만 정의한다.

<script src="https://gist.github.com/aiceru/a9c4ec2a56302faa7191c05a1761669e.js"></script>

이 규칙은 root package 를 극히 simple 하게 유지할 수 있도록 해 준다. 물론 root package 에 어떠한 action 을 포함하는 data type 들 (역주: Go 에서, method 를 포함하는 struct 를 의미하는 듯.) 역시도 포함될 수 있지만, 그 data type 이 오로지<font size="1rem">if-and-only-if</font> 다른 domain type 에만 의존성을 가질 때에만 root package 에 포함시켜야 한다. 예를 들면, ```UserService``` 를 주기적으로 polling 하는 data type 을 root package 에 정의한다고 했을 때, 그 data type 은 다른 external service 를 호출한다거나, database 에 접근한다거나 하는 외부 의존성을 가져서는 안 된다.

> *The root package should not depend on any other package in your application!*

##### 2. Group subpackages by dependency

Root domain 에 외부 의존성<font size="1rem">external dependency</font>을 허용하지 않았기 때문에, 이러한 의존성이 필요한 부분들은 subpackage 에 때려넣어야 한다. 이 approach 에서, subpackage 는 실제 구현과 domain 간의 연결을 위한 adapter 역할을 한다.

예를 들면, `UserService` 는 PostgreSQL 을 이용하여 구현<font size="1rem">backed by</font>될 수 있고 우리는 ```postgres.UserService``` 구현체<font size="1rem">implementation</font> 를 제공하는 ```postgres``` package 를 만들어 사용할 수 있다.

<script src="https://gist.github.com/aiceru/a50ada9d64fbbb98e58fc81c5105989c.js"></script>

PostgreSQL 에 대한 dependency 를 완전히 분리시킴으로써 testing 을 단순화할 수 있으며, 나중에 다른 database system 으로 이전할 경우에도 (변경할 database 에 해당하는 package 만 추가함으로써) 쉽게 code 를 수정할 수 있도록 해 준다. [BoltDB](https://github.com/boltdb/bolt) 등의 다른 database 에 대한 지원을 추가할 때에도 이러한 구조를 pluggable architecture 방식으로 사용할 수 있다. (역자 주: 원 post 의 저자가 BoltDB 의 제작자이다. 🤷‍♂️)

이러한 구조는 또한 layer implementation 을 이용하는 데에도 도움이 된다. PostgreSQL 의 앞 단에 in-memory, LRU cache 를 두고 싶어졌다고 생각해 보자. ```UserService``` interface 를 구현하고, 동시에 PostgreSQL 의 ```UserService``` struct 를 wrapping 하는 ```UserCache``` structure 를 추가함으로써 간단히 해결할 수 있다.

(역자 주: 여기서부터 ```UserService``` 가 의미하는 것이 무엇인지 헷갈리기 시작합니다.🤦‍♂️ 때로는 ```myapp.UserService``` interface 를 의미하기도 하고, 때로는 구현 package 의 ```xxx.UserService``` struct 구현체를 의미하기도 한다. 번역하는 입장에서 최대한 혼동되지 않도록 노력하겠지만 읽을 때에 주의를 기울이기 바랍니다. 가능한 한 'interface' 와 'struct' 또는 '구현체' 등의 설명을 붙여 이해를 도울 수 있도록 하겠습니다.)

<script src="https://gist.github.com/aiceru/c5317fe2a188c5ec2656bee64edcf93e.js"></script>

이와 같은 approach 는 Go 의 standard library 에서도 찾아볼 수 있다. [io.Reader](https://golang.org/pkg/io/#Reader) 는 byte 들을 읽어들이기 위한 domain type 으로, 구체적인 구현체들은 `tar.Reader`, `gzip.Reader`, `multipart.Reader` 등과 같이 실제 구현체의 dependency 를 기준으로 분류되어 있다. 위에서 예를 든 `UserCache` 와 마찬가지로, `os.File` → `bufio.Reader` → `gzip.Reader` → `tar.Reader` 순으로 wrapping 되어 파일의 압축과 묶음 작업을 layer 구조로 수행할 수 있다.

###### Dependencies between dependencies

실제 상황에서, dependency 는 항상 복잡한 양상으로 나타난다. `User` data 를 PostgreSQL 에 저장하기로 결정했지만, `User` 의 financial transaction 은 [Stripe](https://stripe.com/) 같은 third party library 에 저장되어 있을 수도 있다. 이런 경우, 우리는 Stripe 에 대한 dependency 를 wrapping 하는 logical domain type 을 추가할 수 있다. 새로 추가한 domain type 을 `TransactionService` 라 하자. (역자 주: `myapp.UserService` 처럼 myapp package 에 interface 로 정의하며, stripe package 에 이 interface 를 구현한 `stripe.TransactionService` struct 를 정의한다.)

```go
type UserService struct {
  DB *sql.DB

  TransactionService myapp.TransactionService
}
```

이제 PostgreSQL, Stripe 에 대한 2개의 dependency 는 서로에 대한 dependency 는 전혀 없으며, 각각 우리가 만든 공통된 domain language 와만 직접적인 의존성을 가지고 동작하게 된다. 따라서 우리는 아무 때나 Stripe 에 대한 의존성에 영향을 끼치지 않으면서 PostgreSQL 을 MySQL 로 대체할 수도 있고, 또는 PostgreSQL 에는 영향 없이 Stripe 를 다른 payment processor 로 교체할 수 있게 되었다.

###### Don't limit this to third party dependencies

이상하게 들릴 수도 있지만, third party library 에 대해서 뿐만이 아니라 standard library 들에 대해서도 위와 같은 규칙을 적용하여 독립성을 보장해야 한다. 이를 테면, `net/http` package 에 대한 dependency 역시 마찬가지로 `http` subpackage 를 정의하여 이 안에 넣어버린다.

`http` 라는 같은 name 을 다시한번 이용하여 subpackage 를 작성하는 것에 대해 의아하게 생각될 수도 있다. 하지만 이는 전적으로 의도한 사항이다. `http` 라는 이름의 subpackage 를 정의했을 때, 만일 application 의 다른 부분에서 `net/http` 를 이용하는 code 가 존재할 경우 name conflict 가 일어난다. 따라서 `net/http` 에 dependency 를 가지는 code 들을 새로 정의한 `http` package 에만 몰아넣도록 강제할 수 있게 된다.

<script src="https://gist.github.com/aiceru/91e8f93f0486171226b5da5d34b2f566.js"></script>

위 code 에서 `http.Handler` 는 우리의 domain 과 HTTP protocol (`net/http` 가 제공하는) 사이에서 adapter 의 역할을 하게 된다.

##### 3. Use a shared mock subpackage

Domain interface 를 통해서, 모든 external dependency 들을 고립⛔<font size="1rem">isolated</font>시키는 데에 성공했다. 이제 이 domain interface 와의 connecting point 를 통해 mock implementation 을 주입<font size="1rem">injection</font>할 수 있다!

[GoMock](https://github.com/golang/mock) 과 같은 다양한 mocking library 들이 존재하지만, 개인적으로는 직접 mock 을 작성하는 편을 선호한다. 경험상 대부분의 mocking tool 들은 필요 이상으로 복잡했기 때문이다.

작성할 mock 은 아주 심플하다. `UserService` interface 를 통해 주입할 mock 은 아래와 같이 만들 수 있다.

<script src="https://gist.github.com/aiceru/05f918eb4e467330ebfe29eeaa0c8f3c.js"></script>

이 mock 은 `myapp.UserService` interface 를 사용하는 것이라면 어떤 것에든지 쉽게 function 을 주입하여 argument validation 이나, return 값의 검사, 혹은 inject failure 등을 테스트할 수 있게 해 준다.

바로 위에서 만들었던, `http.Handler` 를 테스트해 보자.

<script src="https://gist.github.com/aiceru/1f4fd5a8be86e4d3cf1e2696f1d0e3e3.js"></script>

`mock.UserService` struct 의 `UserFn` 을 test 내부에서 직접 구현하여 원하는 mock 동작을 하도록 만들었다. 여기서 우리의 관심사는 `http.Handler` 가 HTTP request (method 와 URI path) 를 정확히 구분하여 의도한 대로 `Handler` 내부의 `UserService` 를 통해 `User` 함수를 호출하는지 까지이다. 이후의 `User` 함수 내부의 동작이 제대로 이루어지는지는 여기에서는 관심사가 아니다. `Handler` 가 `User` 함수에 parameter (id - 100) 를 제대로 전달하였는지, 그리고 실제로 `User()` 함수의 호출이 이루어졌는지를 `UserInvoked` 를 이용하여 체크하는 것으로 테스트의 목적을 달성할 수 있다.