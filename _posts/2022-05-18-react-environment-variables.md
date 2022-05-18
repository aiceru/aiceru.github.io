---
layout: post
title: '[React] Managing Environment Variables with .env file'
categories: [Development]
tags: [react, environment, variables, .env, file, build, staging, production]
feature_image: '/assets/img/posts/20220518/feature.png'
feature_license: ''
use_math: false
published: true
last_modified_at: '2022-05-18 15:53:01'
---

<!-- more -->
CRA(Create React App) project 에서 빌드 환경 별 environment variable 을 .env 파일로 관리하는 법을 알아보자.

CRA 에서 environment variable 은 **build time** 에 결정된다. 단순히 `development` 환경과 `production` 환경만을 구분하는 거라면 CRA 에서 기본적으로 제공하는 기능만으로도 충분하다.

임시로 사용할 환경 변수의 경우 command line 에서 직접 세팅이 가능하다.
```shell
REACT_APP_NAME=myApp yarn start
```

계속해서 사용할 환경 변수라면 .env 파일을 만들어 관리하는 것이 좋다. 프로젝트 루트에 `.env` 파일을 생성하고, 공통 환경 변수를 여기에 넣어 준다.
```
REACT_APP_NAME=myApp
```
> 🖌 Note: 모든 환경 변수의 이름은 `REACT_APP_` 으로 시작해야 하며, 그렇지 않은 경우 무시되는데, 이는 private key 등의 secure information 을 실수로 노출하는 것을 방지하기 위함이라고 한다.

마찬가지로 프로젝트 루트에 `.env.development`, `.env.production` 파일을 생성하고, 각 빌드 환경 별 환경변수를 넣어 준다.
```
REACT_APP_API_SERVER_URL=https://staging.myapp.com
```
{% include code-caption.html text=".env.development" %}
```
REACT_APP_API_SERVER_URL=https://api.myapp.com
```
{% include code-caption.html text=".env.production" %}

# 📌 NODE_ENV
CRA 는 `NODE_ENV` 라는 built-in env variable 을 제공하며, `process.env.NODE_ENV` 로 값을 읽을 수 있다. `npm start` 또는 `yarn start` 로 실행하였을 경우 이 값은 `'development'` 이며, `npm test`, `yarn test` 의 경우 `'test'`, `npm run build` 혹은 `yarn build` 로 production bundle 을 생성하였을 경우 `'production'` 값이 리턴된다. 개발/배포 시 개발자의 실수를 방지하기 위해 이 값은 override 할 수 없도록 되어 있다.

`react-scripts/lib/react-app.d.ts` 파일을 열어 보면, `ProcessEnv` interface 가 정의되어 있다,

```typescript
/// <reference types="node" />
/// <reference types="react" />
/// <reference types="react-dom" />

declare namespace NodeJS {
  interface ProcessEnv {
    readonly NODE_ENV: 'development' | 'production' | 'test';
    // NODE_ENV 는 3가지 값만 가질 수 있다.
    readonly PUBLIC_URL: string;
  }
}
...
```

위 예시에서 추가한 환경 변수들 (`REACT_APP_NAME`, `REACT_APP_API_SERVER_URL`) 역시 소스 코드에서 `process.env.REACT_APP_...` 으로 읽어올 수 있다.

# 📌 .env files
CRA 는 빌드시 자동으로 프로젝트 루트에서 `.env` 파일을 찾아 환경변수를 삽입한다. 이 때 각각의 환경에 따라 어떤 `.env` 파일을 읽을지를 결정한다.

## 📍 .env file names
- `.env`: Default.
- `.env.local`: Local overrides. test 환경을 제외한 모든 환경에서 로딩됨.
- `.env.development`, `.env.test`, `.env.production`: 빌드 환경 별 설정.
- `.env.development.local`, `.env.test.local`, `.env.production.local`: 빌드 환경 별 설정의 local overrides.

## 📍 Priority of .env files
왼쪽에 있는 파일들이 우선순위가 높다. (우측 파일의 내용을 덮어씀)
- `npm start`: `.env.development.local`, `.env.local`, `.env.development`, `.env`
- `npm run build`: `.env.production.local`, `.env.local`, `.env.production`, `.env`
- `npm test`: `.env.test.local`, `.env.test`, `.env`

# 📌 Creating custom build environment
development / production 환경만을 구분하기 위해서는 위 기능만으로도 충분하겠지만, 보통 프로젝트를 진행하면 최소한 development (local), staging, production 의 3가지 환경 구분이 필요하다. `.env.xxx.local` 파일을 잘 활용하면 가능하긴 하겠지만,
- production build 를 로컬에서 진행하는 경우 (`.env.xxx.local` 파일들을 임시 삭제해야 함)
- 모든 개발자들이 CRA 의 .env 파일 규칙(name, priority)을 숙지하고 있어야 함.


등 귀찮은🤦‍♀️ 점들이 많다. 그래서 관리해야 할 빌드 환경이 많아질수록 custom environment 를 만들어 사용하는 것이 편하다.

## 📍 env-cmd
먼저 project 에 [env-cmd](https://github.com/toddbluhm/env-cmd) 를 추가한다. env-cmd 는 node program 실행시 env file 을 지정하여 읽어들일 수 있게 해 주는 도구이다.
```
yarn add env-cmd
```

## 📍 Modify script
`package.json` 파일에서 `"script"` section 을 찾는다. 처음 생성시 아래와 같이 설정되어 있다.
```json
  "scripts": {
    "start": "react-scripts start",
    "build": "react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
```
각 명령어에 맞는 .env 파일을 읽어올 수 있도록 수정해 주고, "build" 는 staging, production 의 2가지 세부 환경으로 다시 나누어 준다.
```json
  "scripts": {
    "start": "env-cmd -f .env.local react-scripts start",
    "build:staging": "env-cmd -f .env.development react-scripts build",
    "build:production": "env-cmd -f .env.production react-scripts build",
    "test": "react-scripts test",
    "eject": "react-scripts eject"
  },
```

# 📌 Auto-completion
여기까지만 해도 환경 변수를 사용하는 데에는 문제가 없지만, typescript 소스 코드 작성시 자동완성이 되지 않는다. 자동완성을 위해서는 `react-app-env.d.ts` 에 `ProcessEnv` interface 를 확장해 주어야 한다. 잘 설명된 블로그가 있으니 [여기](https://velog.io/@dev_space/Typescript-and-React-.env)를 참고하자. 

# 📌 References
- [Adding Custom Environment Variables](https://create-react-app.dev/docs/adding-custom-environment-variables/)
- [env-cmd](https://github.com/toddbluhm/env-cmd)
- [Typescript and React .env](https://velog.io/@dev_space/Typescript-and-React-.env)