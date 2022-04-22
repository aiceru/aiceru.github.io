---
layout: post
title: '[gRPC] Simple gRPC server with Golang'
categories: [Development]
series: 'Implementing gRPC server with Protobuf, Golang, Flutter'
tags: [grpc, flutter, golang, protobuf]
feature_image: '/assets/img/20220420/20220420.png'
feature_license: ''
use_math: false
published: true
last_modified_at: '2022-04-22 12:41:58'
---

<!-- more -->
{% include series.html %}

# gRPC Server
이전 포스트에서 컴파일된 코드를 이용하여 Golang 으로 gRPC 서버를 만들어보자.
## Go Modules
프로젝트 루트의 go.mod 파일에 아래 내용을 추가한다.
```
require (
  ...
  protocols v1.0.0
)
...

replace (
  protocols => ../protocols
)
```
{% include code-caption.html text="go.mod" %}

아직은 Github 등의 git repository 에 올리지 않고, 로컬에서 임포트할 것이기 때문에 `replace` 구문을 이용하여 상대경로를 지정해준다. 로컬 상대 경로로 replace 할 경우 버전 스트링은 vX.Y.Z 의 포맷만 지켜서 임의로 넣어주면 된다.
> 전체 Project 의 root 디렉토리에 server, client, protocols 의 디렉토리를 만들고 그 안에 각각의 프로젝트를 생성하면 상대 경로로 참조하기가 편하다.<br/>
```
example
├── client
├── protocols
└── server
```


## API server
RPC API 의 로직을 실행하는 부분. 데이터를 저장하고, 찾아오고, 수정하는 등 API 호출에 따른 business logic 이 들어갈 부분이다.
먼저 앞에서 protobuf 를 이용해 생성한 코드를 살펴보면,
```go
...
// UserApiServer is the server API for UserApi service.
// All implementations must embed UnimplementedUserApiServer
// for forward compatibility
type UserApiServer interface {
	GetUser(context.Context, *GetUserRequest) (*GetUserReply, error)
	AddUser(context.Context, *AddUserRequest) (*AddUserReply, error)
	mustEmbedUnimplementedUserApiServer()
}

// UnimplementedUserApiServer must be embedded to have forward compatible implementations.
type UnimplementedUserApiServer struct {
}

func (UnimplementedUserApiServer) GetUser(context.Context, *GetUserRequest) (*GetUserReply, error) {
	return nil, status.Errorf(codes.Unimplemented, "method GetUser not implemented")
}
func (UnimplementedUserApiServer) AddUser(context.Context, *AddUserRequest) (*AddUserReply, error) {
	return nil, status.Errorf(codes.Unimplemented, "method AddUser not implemented")
}
func (UnimplementedUserApiServer) mustEmbedUnimplementedUserApiServer() {}
...
```
{% include code-caption.html text="api_user_grpc.pb.go" %}

'`service UserApi`' 로 설정한 rpc 호출을 받을 수 있는 interface 가 정의되어 있다. 주석의 설명대로 `UnimplementedUserApiServer` interface 를 임베드하여 API server 구조체를 정의하고, 필요한 API handler function 들을 구현한다.

```go
import(
  ...
  "google.golang.org/grpc/codes"
  "google.golang.org/grpc/status"
  "protocols/goapi"
)

// API server struct.
type UserApi struct {
  goapi.UnimplementedUserApiServer
}

func (u *UserApi) GetUser(ctx context.Context, request *goapi.GetUserRequest) (*goapi.GetUserReply, error) {
  // rpc GetUser(GetUserRequest) returns (GetUserReply) {}
  ...
  // example success return
  return &goapi.GetUserReply{
    Users: []*goapi.User{user},
  }, nil

  // example error return
  return nil, status.Errorf(codes.NotFound, "email: %s", email)
}

func (u *UserApi) AddUser(ctx context.Context, request *goapi.AddUserRequest) (*goapi.AddUserReply, error) {
  // rpc AddUser(AddUserRequest) returns (AddUserReply) {}
}
```
{% include code-caption.html text="user_api.go" %}

## grpc Server
API server 를 정의하고 구현한 뒤, 객체를 생성하여 grpc server 객체에 등록해주면 해당하는 rpc call 이 라우팅되어 handler function 이 실행된다.
```go
import (
	...
	"google.golang.org/grpc"
	"protocols/goapi"
)

func main() {
  // grpc server 객체를 생성
  grpcServer := grpc.NewServer()
  // UserApi 객체를 생성하여 grpc server 에 등록
  goapi.RegisterUserApiServer(grpcServer, &UserApi{})

  lis, err := net.Listen("tcp", ":8080")
  if err != nil {
    log.Fatal(err)
  }
  if err := grpcServer.Serve(lis); err != nil {
    log.Fatal(err)
  }
}
```
{% include code-caption.html text="main.go" %}

이것으로 서버의 준비는 완료. 다음 편에서는 Flutter client 의 구현을 설명한다.