---
layout: post
title: '[Golang] Development Environment Setup with Vim'
categories: [Development]
tags: [go, vim, environment, setup]
feature_image: https://blog.golang.org/gopher/header.jpg
last_modified_at: '2022-04-09 23:50:01'
---

<!-- more -->

# The Go Project

> Go is an open source project developed by a team at [Google](https://www.google.com) and many [contributors](https://golang.org/CONTRIBUTORS) from the open source community.

Go 언어는 Google 이 주도하고, 여러 오픈 소스 커뮤니티 컨트리뷰터들의 기여로 만들어진 언어이다. 요즘 대세가 되고 있는 스크립트 언어들과 달리, native machine code 로 수행되는 compile-언어이며 자체적으로 garbage collection 을 지원한다. 가볍다, 빠르다 등등 좋은 점들에 대한 말들도 많이 들리는 반면 아직 언어 자체가 완성단계가 아니라 빠르게 발전해 나가는 개발 단계이다보니 지원되지 않는 기능들도 꽤 많고, 이래저래 단점들도 많다고 한다.

어쨌든 배워둬서 손해볼 건 없을 것 같아 요즘 남는 시간에는 Go 언어를 공부중.

헌데... IDE 가 없다. LiteIDE 나 JetBrain 의 Go plugin 등 나름 Go 언어를 지원하는 IDE 가 있는 것 같긴 한데, 새로운 language 를 학습하는 초기에는 IDE 의 도움 없이 소스코드 작성부터 빌드까지 손으로 직접 해보는 게 도움이 많이 되는 터라, vim 과 console terminal 로 개발환경을 세팅하기로 결정.

(참고 : [The Go Programming Language](https://golang.org/) : 뭐든, 공홈만큼 좋은 step by step guide 는 없다)

# Installing Go

[Getting Started](https://golang.org/doc/install) page 에 가서 Go binary 를 다운로드한다.

- Linux / Mac OS X, FreeBSD tarball 설치 (여기에서 설치 경로는 `/usr/local`)
  - 설치 경로에 tarball 압축 해제
  ```bash
  $ tar -C /usr/local -xzf go$VERSION.$OS-$ARCH.tar.gz
  ```
  - System 환경변수 `PATH` 에 `/usr/local/go/bin` 을 추가한다.
  ```bash
  $ export PATH=$PATH:/usr/local/go/bin
  ```
  `/usr/local` 대신 다른 경로에 설치할 수 있지만, 다른 경로에 설치할 경우 반드시 `GOROOT` 환경변수를 설정해 주어야 한다.
  ```bash
  $ export GOROOT=$HOME/go
  ```

## Test Installation

우선, workspace 를 하나 생성하고, (ex. `$HOME/work`) go environment variable `GOPATH` 에 workspace 위치를 지정해준다. Go 의 workspace 에 관한 자세한 설명은 [여기](https://golang.org/doc/code.html#Workspaces)로.

```bash
$ export GOPATH=$HOME/work
```

다음으로 workspace 에 `src/github.com/user/hello` 디렉토리를 만들고, `hello` 디렉토리에 아래 내용으로 hello.go 파일을 작성한다.

```go
package main

import "fmt"

func main() {
    fmt.Printf("hello, world\n")
}
```

go 의 컴파일 명령은 아래와 같다.

```bash
$ go install github.com/user/hello
```

컴파일, 인스톨이 정상적으로 수행된 경우, `$GOPATH/bin` 디렉토리에 executable 파일이 생성된다.

```bash
$ $GOPATH/bin/hello
hello, world
```

무사히 Go 를 설치하고 테스트까지 마쳤다면, 본격적으로 Go 언어 개발을 위한 vim 환경 세팅에 들어가보자.

# vim-go plugin

Vundle, Pathogen, vim-plug, NeoBundle 등 다양한 플러그인 매니저를 이용하여 vim-go 를 설치할 수 있다. 자세한 내용은 [vim-go @ github](https://github.com/faith/vim-go) 페이지를 참고하자.

Go language 의 syntax highlighting 및 파일 저장(save)시 자동으로 `gofmt` 실행 (.go 소스 코드의 들여쓰기, 띄어쓰기 등 code style 을 표준 권장 사항에 맞게 자동으로 수정해 주는 도구이다.) 이나, vim 내에서 `:GoBuild`, `:GoInstall`, `:GoRun` 등의 명령으로 빌드, 인스톨, 실행 기능 지원 등등 강력한 기능들이 많다. Full Feature 는 github 페이지의 Features 섹션을 읽어보자. (나도 아직 다 안읽어본 게 함정...)

위에서 언급한 기능들만 사용해도, 언어를 학습하는 입장에서는 차고 넘칠 정도. 아직 디버깅이나 gotags 를 이용한 복잡한 코드 분석 정도까지는 할 일이 없으니... 나중에 필요한 일이 생기면 차차 배우도록 하...겠지...? -\_-)

## 기본 setting

기본적으로, Functions, Methods, Structs, Interfaces, Operators 등에 대해 syntax highlighting 기능을 on.

```vim
let g:go_highlight_functions = 1
let g:go_highlight_methods = 1
let g:go_highlight_structs = 1
let g:go_highlight_interfaces = 1
let g:go_highlight_operators = 1
let g:go_highlight_build_constraints = 1
```

`gofmt` 대신 `goimport` 가 import 문을 자동으로 삽입하도록 설정 (무슨말인지 잘 모르겠다-\_-)

```vim
let g:go_fmt_command = "goimports"
```

`gofmt` 실패시 에러메세지 출력하지 않음, 파일 저장시 자동으로 gofmt 실행

```vim
let g:go_fmt_fail_silently = 1
let g:go_fmt_autosave = 1
```

`:GoInstallBinary` (vim-go 의 다양한 기능들을 활용하려면, 먼저 이 명령으로 필요한 binary 파일들을 설치해야 한다) 로 설치되는 binary 파일들의 설치 경로는 기본적으로 `$GOBIN` 이나 `$GOPATH/bin` 으로 지정되어 있는데, 아래 설정으로 원하는 설치 경로를 지정할 수 있다. 내 경우는 `~/.gotools` 로 지정해 놓았다.)

```vim
let g:go_bin_path=expand("~/.gotools")
```

유용한 key mapping

```vim
au FileType go nmap <leader>r <Plug>(go-run)
au FileType go nmap <leader>b <Plug>(go-build)
au FileType go nmap <leader>i <Plug>(go-install)
```

이 3가지만 키매핑을 해놔도 미친듯이 편하다. 학습용 코드를 작성하고 나서,

save -> `ctrl-z` 를 눌러 백그라운드로 -> `go build (install) source.go` -> `$GOPATH/bin/executable` 실행 해야 하던 것이...

save -> `\b (\i)` -> `\r` 로 끝.

이 정도만 설정을 해 주어도 기본적인 문법을 학습하는 정도의 소스코드 작성과 실행에는 전혀 불편함이 없을 것이다.
