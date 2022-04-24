---
layout: post
title: '[Golang] Go Workspace'
categories: [Development]
tags: [golang, go, workspace, modules]
feature_image: 'https://images.unsplash.com/photo-1556761175-4b46a572b786?ixlib=rb-1.2.1&ixid=MnwxMjA3fDB8MHxwaG90by1wYWdlfHx8fGVufDB8fHx8&auto=format&fit=crop&w=3474&q=80'
feature_license: 'Photo by <a href="https://unsplash.com/@austindistel?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Austin Distel</a> on <a href="https://unsplash.com/s/photos/workspace?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>'
use_math: false
published: true
last_modified_at: '2022-04-24 20:36:52'
---

<!-- more -->
# Workspace

Golang 1.18이 릴리즈되면서, workspace 기능이 추가되었다. 언어 자체의 추가 기능이라기보단 go modules 도구의 기능 추가에 가까운듯 하다.

Golang 으로 서버 개발을 하다 보면, 공통으로 사용되는 기능들을 별도의 프로젝트<sub>go modules</sub>로 빼내어 관리하고 싶어지는 경우가 많다. DB query 부분이라던가, remote storage access 등등 microservice 나 또는 데이터 분석 모듈 등에서 공통으로 사용하는 부분을 하나로 별도의 _**library**_ 로 관리/개발하며 서비스 단에서 import 하여 사용하도록 하는 방식이다. 특히 [Protocol Buffers](https://developers.google.com/protocol-buffers) 나 [gRPC](https://grpc.io) 를 사용하는 경우 프로토콜(.proto 파일을 컴파일하여 생성되는 파일들) 자체를 하나의 module 로 사용하는 경우가 다반사이기 때문에 go module 을 활용하여 다수의 프로젝트 디렉토리를 관리해야 한다.

go module 은 별도의 패키지 관리자 혹은 repository 가 없이 git repository 를 기반으로 간단하게 의존성을 관리하는 것까지는 좋은데, 의존하고 있는 module (주로 라이브러리 혹은 프로토콜 module)의 로컬 변경 사항을 서비스 코드에 반영하려면 조금 귀찮은 과정을 거쳐야 한다.

SERVER → LIBRARY 의 의존성을 지닌 관계가 있을 때, LIBRARY 의 코드를 로컬에서 수정하고 이를 SERVER 에 반영하려면  
1. LIBRARY 의 변경 사항을 remote 에 push 하고 새로운 git tag 를 발행한 뒤, SERVER 의 go.mod 에서 version tag 를 update 하여 사용하거나,
2. LIBRARY 의 변경 사항을 remote 에 push 하기 전에 SERVER 의 go.mod 에서 `replace` 구문을 이용하여 local path 를 참조한다.  

의 두 가지 방법 중 하나를 선택해야 하는데, 첫 번째 방법은 변경 사항이 SERVER 에 적용되었을 때 뒤늦게 버그가 발견되거나 하면 다시 commit, push, tag 의 과정을 또 거쳐야 해서 불필요한 태그가 난잡하게 늘어나게 될 소지가 다분하다. 그래서 대부분 두 번째 방법을 택하는데, `replace` 구문을 수동으로 넣고 수정 후 push 할 때 다시 얘를 빼고 버전 태그를 업데이트하고 해줘야 하다 보니, 깜빡 `replace` 구문을 포함한 채로 push 하게 되면 CI 가 깨지거나 동료의 build 환경을 망가트리거나... (보통은 둘 다) 여튼 귀찮다. 🤯

## Workspace!!
설명을 읽어 보자.

> Workspaces in Go 1.18 let you work on multiple modules simultaneously without having to edit go.mod files for each module. Each module within a workspace is treated as a root module when resolving dependencies.

여러개의 go module 들을 일일이 go.mod 파일 수정할 필요 없이 관리할 수 있게 해준다. Workspace 내 모듈들은 각자가 root module 로 취급된다. 라고 한다.

말대로라면 딱 내가 원하던 그 기능이다. 프로토콜도 별도의 모듈로, 서비스의 공통 루틴도 별도의 모듈로 사용하고 있기 때문에, 매번 일일이 "chore: go.mod update" 커밋 메시지 남기기도 너무 번거로웠기 때문이다.

그럼 한번 해보자 일단.

# Ready

디렉토리 구조
```
├── hello
│   ├── go.mod
│   ├── go.sum
│   └── main.go
└── mylib
    ├── go.mod
    └── mylib.go
```

## mylib

`ReverseString` 함수를 제공하는 라이브러리.

```go
module github.com/aiceru/mylib

go 1.18
```
{% include code-caption.html text="mylib/go.mod" %}

```go
package mylib

func ReverseString(str string) string {
  rns := []rune(str)
  for i, j := 0, len(rns)-1; i < j; i, j = i+1, j-1 {
    rns[i], rns[j] = rns[j], rns[i]
  }

  return string(rns)
}
```
{% include code-caption.html text="mylib/mylib.go" %}

```shell
* bcef0af - (HEAD -> master, tag: v0.1.1, origin/master) init ...
```
{% include code-caption.html text="mylib v0.1.1" %}

## Hello

```go
module hello

go 1.18

require github.com/aiceru/mylib v0.1.1
```
{% include code-caption.html text="hello/go.mod" %}

```go
package main

import (
  "fmt"
  "github.com/aiceru/mylib"
)

func main() {
  fmt.Println(mylib.ReverseString("Hello"))
}
```
{% include code-caption.html text="hello/main.go" %}

기존의 방법대로라면 여기서 `mylib` 에 새로운 기능을 추가할 경우, 새 기능을 `hello` 에서 사용하기 위해서는 `mylib` 의 새 버전을 publish 하던가, `hello` 의 go.mod 파일에 `replace` 구문으로 임시로 local path 를 참조하도록 하던가 해야 한다.

# Make new workspace
```shell
go work init ./hello ./mylib
```
`go.work` 파일이 생성된다.
```go
go 1.18

use (
  ./hello
  ./mylib
)
```
{% include code-caption.html text="go.work" %}

## Add a function to mylib
```go
import "unicode"
...
func ToUpper(s string) string {
    r := []rune(s)
    for i := range r {
        r[i] = unicode.ToUpper(r[i])
    }
    return string(r)
}
```
{% include code-caption.html text="mylib/mylib.go" %}

`ToUpper()` 함수를 추가했다.

## Use new function in hello service
```go
package main

import (
  "fmt"
  "github.com/aiceru/mylib"
)

func main() {
  fmt.Println(mylib.ReverseString("Hello"))
  fmt.Println(mylib.ToUpper("Hello"))
}
```
새로운 함수를 hello 에서 사용하도록 추가하고 `go run` 을 실행해 보자.
```shell
go run .
olleH
HELLO
```
Go command 가 상위 디렉토리의 `go.work` 파일에서 mylib 의 경로를 발견하고 자동으로 의존성을 해결해 준다.

시험삼아 상위의 `go.work` 파일을 삭제해봤다.
```shell
go run .
# hello
./main.go:10:20: undefined: mylib.ToUpper
```
역시나 Fail. ❌

그럼, Go 는 `go.work` 파일을 찾기 위해 어디까지 거슬러 올라갈까? `hello` 의 세 단계 상위 디렉토리인 홈 디렉토리에 `go work init ...` 으로 `go.work` 파일을 생성해 봤다.
```shell
go run .
olleH
HELLO
```
잘 된다. ⭕️ 최상위 디렉토리까지 거슬러 올라가며 찾는 것 같다.

# 아쉬운점

> `go work sync` syncs dependencies from the workspace’s build list into each of the workspace modules.

라는 설명이 있기에 혹시 library 에 새로운 버전 태그를 발행하면 의존성이 있는 모듈의 go.mod 도 자동으로 업데이트해 주는가 싶었는데 이건 아닌 듯.

mylib 의 변경사항을 publish, 새로운 버전 태그를 주고
```shell
* 8277eee - (HEAD -> master, tag: v0.1.2, origin/master) add toupper ...
* 15e1398 - (tag: v0.1.1) init ...
```
{% include code-caption.html text="mylib v0.1.2" %}

`go work sync` !!! Magic! 🦄 을 원했으나...  
workspace 에서도 해보고, hello module 에서도 해봤지만 여기까진 욕심인가 보다.

일일이 라이브러리 코드 수정할 때마다 `replace` 손으로 안쳐도 되게 해준 것만으로도 감사하자.

# 주의할점
Workspace 를 사용할 땐 의존성을 가진 모듈 (`hello`, 서비스단) 에서 go.mod 의 버전을 올려주지 않아도 알아서 local directory 를 참조하여 라이브러리의 수정사항이 바로바로 반영되지만 이대로 서비스를 push 하고 랄라라 퇴근했다간 CI 가 깨지고 팀동료의 빌드가 깨진다. **버전 태그 업데이트** 까먹지말자