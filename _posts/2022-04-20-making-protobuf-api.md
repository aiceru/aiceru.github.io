---
layout: post
title: '[gRPC] Making gRPC API with Protocol buffers'
categories: [Development]
series: 'Implementing gRPC server with Protobuf, Golang, Flutter'
tags: [protobuf, backend, api, server, client, grpc]
feature_image: '/assets/img/20220420/20220420.png'
feature_license: ''
use_math: false
published: true
last_modified_at: '2022-04-22 11:53:00'
---

<!-- more -->
{% include series.html %}

# Protocol Buffers 이용하여 gRPC API 만들기
[Protocol Buffers](https://developers.google.com/protocol-buffers) 를 사용하여 간단한 [gRPC](https://grpc.io/about/) 명세를 작성하고, protobuf compiler 를 통해 생성된 코드를 dart(flutter), golang 에서 이용하는 방법을 설명한다.

# Backgrounds
## gRPC
> gRPC is a modern open source high performance Remote Procedure Call (RPC) framework that can run in any environment. It can efficiently connect services in and across data centers with pluggable support for load balancing, tracing, health checking and authentication. It is also applicable in last mile of distributed computing to connect devices, mobile applications and browsers to backend services.

Google 이 개발한 Remote Procedure Call (RPC) framework. 서버-클라이언트 간 인터페이스를 구축할 때, HTTP REST + json 방식과 함께 가장 많이 사용되는 방식이다.

## Protocol Buffers
> Protocol buffers provide a language-neutral, platform-neutral, extensible mechanism for serializing structured data in a forward-compatible and backward-compatible way. It’s like JSON, except it's smaller and faster, and it generates native language bindings.
> &nbsp;  
> &nbsp;  
> Protocol buffers are a combination of the definition language (created in .proto files), the code that the proto compiler generates to interface with data, language-specific runtime libraries, and the serialization format for data that is written to a file (or sent across a network connection).

역시 Google 이 개발한 통신 규격이다. 직렬화/역직렬화로 실제 network layer 에서는 byte format 으로 통신하며, 사용법 자체는 JSON 과 거의 유사하지만 자체 언어로 message 규격을 정의하고 protocol buffer compiler 로 사용하려는 language 에 대한 binding code 를 생성, 임포트하여 사용한다. JSON 에 비해 빠르고 가벼워서 gRPC 와 함께 주로 microservice 의 구현에 많이 사용하는 편이다.

## Exp.
내 경우에는 회사에서는 HTTP + protobuf, HTTP + json 방식을 사용하고 있고, 개인 프로젝트에서는 gRPC + protobuf 를 사용한다. 각각 장단점은 뚜렷이 나타나는 반면, 종합적으로는 어떤 방식이 더 낫다를 판가름하기는 힘든 것 같다.

- HTTP + json
  - 준비하는 과정이 거의 없이 바로 구현에 들어갈 수 있다.
  - HTTP 서버와 json body parsing 부분을 별도로 구현해야 한다.
  - body 가 human-readable → 가장 큰 장점! 눈으로 디버깅 가능
  - 다양한 HTTP API test tool ([postman](https://www.postman.com/), [Advanced REST client](https://chrome.google.com/webstore/detail/advanced-rest-client/hgmloofddffdnphfgcellkdfbfbjeloo) 등) 을 사용 가능, 서버 사이드만 개발한다면 별도의 테스트용 클라이언트를 만들지 않아도 된다.

- gRPC + protobuf
  - 구현이 곧 문서화. .proto 파일을 이용하여 rpc 를 정의하여 이 파일이 곧 그대로 API 문서가 된다.
  - 각 개발 파트 (프론트엔드 / 백엔드) 간 API spec 의 공유가 쉽다.
  - API 명세를 위한 준비 작업이 필요하다.
  - byte-oriented 이기 때문에 unmarshal 과정을 거쳐야 사람이 이해할 수 있는 형태로 변환된다.
  - Network 부하가 적다.

- HTTP + protobuf
  - 각각의 단점만 합쳐 놓은 것 같은 조합 🤯
  - 기존 protobuf 를 사용하던 프로젝트를 포팅해야 하는데 client 나 server side 의 언어가 gRPC 를 지원하지 않는 경우 어쩔 수 없이 사용한다던가 하는 외에는 딱히 쓸모가 잘 떠오르지 않는다.

# API specification with Protocol Buffers
## Defining APIs/messages with proto3 language
먼저 proto3 language 를 이용하여 통신에 사용할 message 들을 정의한다. proto3 language 의 구체적인 사용 방법은 [공식 문서](https://developers.google.com/protocol-buffers/docs/proto3) 를 참조하자.
```
syntax = "proto3";
package protocols;

option go_package = "protocols/goapi";

message User {
  string id = 1;
  string email = 2;
  string name = 3;
}

service UserApi {
  rpc GetUser(GetUserRequest) returns (GetUserReply) {}
  rpc AddUser(AddUserRequest) returns (AddUserReply) {}
}

message GetUserRequest {
  string email = 1;
}
message GetUserReply {
  repeated User users = 1;
}

message AddUserRequest {
  string email = 1;
  string name = 2;
}
message AddUserReply {
  User user = 1;
}
```
{% include code-caption.html text="api_user.proto" %}

- `option go_package` : golang 에서 go modules 를 통해 임포트하여 사용하기 위한 옵션. golang binding code 의 package name 을 지정해준다.
- `service` 정의에서 rpc 호출 명세를 작성하고
- rpc 호출시 사용될 각각의 `message` 명세를 정의해준다.

> 같은 message format 을 여러 군데서 사용할 경우 하나의 common message format 을 재사용하지 않고, 각각의 용도에 따라 message name 을 다르게 하여 전부 정의해 주는 것이 좋다. 그 편이 나중에 API 요구사항의 변화에 따라 message 구조를 수정하더라도 다른 API 에 영향을 미치지 않고 필요한 부분만 수정할 수 있기 때문이다.

## Compile to generate language-bindings
이제 서버와 클라이언트를 구현하기 위해 사용될 binding code 를 생성한다. 

### Installing protoc compiler and plugins
.proto 파일을 컴파일하려면 `protoc` protobuf compiler 를 설치해야 한다.  또한 사용하려는 프로그래밍 언어별 plugin 도 설치되어 있어야 한다.
- [Protocol Buffer compiler](https://developers.google.com/protocol-buffers/docs/downloads)
- [Dart Protocol Buffer plugin](https://github.com/google/protobuf.dart/tree/master/protoc_plugin#how-to-build)  
  ```
  dart pub global activate protoc_plugin
  ```
- [Golang Protocol Buffer plugin](https://developers.google.com/protocol-buffers/docs/gotutorial#compiling-your-protocol-buffers)  
  ```
  go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
  ```

### Compiling
프로젝트를 진행하면서, API 의 수정 및 추가 등으로 인해 반복적으로 사용될 것이므로 shell script 를 작성하는 것을 추천한다.
```shell
#!/bin/bash

SRC_DIR=$(pwd)
DART_OUT=$SRC_DIR/dartapi/lib
GO_DIR=$SRC_DIR/goapi

rm -rf "$GO_DIR"
rm -rf "$DART_OUT"
mkdir -p "$GO_DIR"
mkdir -p "$DART_OUT"

protoc \
-I="$SRC_DIR" \
--dart_out=grpc:"$DART_OUT" \
--go_out="$SRC_DIR" \
--go_opt=module=protocols \
--go-grpc_out="$SRC_DIR" \
--go-grpc_opt=module=protocols \
"$SRC_DIR"/*.proto
```
{% include code-caption.html text="gen.sh" %}

golang 관련 옵션이 좀 많다. golang binding code 의 output 위치를 현재 디렉토리 `SRC_DIR` 로 지정해 주고, option 으로 module prefix ([설명 참고](https://developers.google.com/protocol-buffers/docs/reference/go-generated#invocation))를 설정해준다. 이렇게 설정하면 아래와 같은 구조로 output 이 생성된다.
```
├── dartapi
│   └── lib
│       ├── api_user.pb.dart
│       ├── api_user.pbenum.dart
│       ├── api_user.pbgrpc.dart
│       └── api_user.pbjson.dart
└── goapi
│   ├── api_user.pb.go
│   └── api_user_grpc.pb.go
├── api_user.proto
└── gen.sh
```
server side 에서 go modules 를 통해 임포트할 수 있도록 go.mod 파일을 추가한다. 프로젝트 루트 디렉토리에서,
```
go mod init protocols && go mod tidy
```
하면 아래와 같이 go.mod 파일이 생성된다.
```
module protocols

go 1.17

require (
	google.golang.org/grpc v1.45.0
	google.golang.org/protobuf v1.28.0
)

require (
	github.com/golang/protobuf v1.5.2 // indirect
	golang.org/x/net v0.0.0-20200822124328-c89045814202 // indirect
	golang.org/x/sys v0.0.0-20200323222414-85ca7c5b95cd // indirect
	golang.org/x/text v0.3.0 // indirect
	google.golang.org/genproto v0.0.0-20200526211855-cb27e3aa2013 // indirect
)
```
{% include code-caption.html text="go.mod" %}

마찬가지로, flutter(dart) 에서 pubspec 으로 임포트가 가능하도록 /dartapi 디렉토리에 pubspec.yaml 파일을 추가해준다.
```
name: dartapi

publish_to: 'none' # Remove this line if you wish to publish to pub.dev

environment:
  sdk: ">=2.15.0 <3.0.0"

dependencies:
  protobuf: ^2.0.1
```
{% include code-caption.html text="pubspec.yaml" %}

이제 서버와 클라이언트를 각각의 언어로 구현할 **준비**가 끝났다. 다음 편에서는 이렇게 생성한 binding code 를 어떻게 임포트하고 사용하는지 알아보자.
