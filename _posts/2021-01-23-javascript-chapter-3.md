---
title: 3장 자바스크립트 개발 환경과 실행 방법
categories:
  - Javascript
tags:
  - Javascript
  - 브라우저
  - Node.js
  - VS Code

---



## 자바스크립트 실행 환경 - 브라우저 vs Node.js

- 모든 브라우저 및 Node.js는 자바스크립트 엔진을 내장하고 있다
  
  - 자바스크립트 엔진 : 자바스크립트 해석 및 실행 -> ECMAScipt 지원
  
- 그러나 용도가 다르다

  | 브라우저                                                     | Node.js                                                      |
  | ------------------------------------------------------------ | ------------------------------------------------------------ |
  | HTML, CSS, 자바스크립트를 실행해 웹페이지를 브라우저에 **렌더링**하기 위함 | 렌더링과 관계없이 **브라우저 외부**에서 자바스크립트 실행 환경을 제공하기 위함 |
  | DOM API를 제공 ( HTML 요소를 조작하기 위해 )                 | DOM API 제공 x (브라우저와 관련된 DOM을 굳이 갖고 있을 이유 없다) |
  | 파일 시스템 제공 x(보안상의 이유)                            | 파일 시스템 제공 o                                           |
  | 클라이언트 사이드 웹 API 지원                                | Node.js 고유 API 지원                                        |

  - DOM API : HTML을 파싱하여 객체화한 DOM을 선택 및 조작하는 기능의 집합

  - 브라우저에서 실행되는 자바스크립트가 사용자 컴퓨터의 로컬에서 CRUD 가능한 것은 위험하다.



## 웹 브라우저

- 크롬
  - 자바스크립트 엔진 : V8
    - Node.js에서도 사용
  - 개발자 도구(Command + option + i)
    - Elements(최종 렌더링된 뷰), Console(에러 및 console.log 결과), Sources(자바스크립트 코드 디버깅), Network(네트워크 요청 정보 및 성능), Application(웹 스토리지, 세션, 쿠키) 등을 제공
    - Sources에서 breakpoint 설정하여 디버깅 가능

- 브라우저는 HTML 파일의 `<script>` 태그에 포함된 자바스크립트 코드 실행



## Node.js

프로젝트 규모가 커짐에 따라 브라우저만으로 개발 어려워짐

Node.js와 npm 필요성 대두

Node.js 

- 크롬 V8 자바스크립트 엔진으로 빌드된 자바스크립트 런타임 환경
- 브라우저 이외의 환경에서 자바스크립트를 동작시킬 수 있도록 함

npm(Node Package Manager)

- Node.js에서 사용할 수 있는 모듈을 모아둔 저장소
- 각 모듈의 설치 및 관리를 위한 CLI 제공



### Node.js 설치

- <https://nodejs.org/> 에서 설치

- macOS는 `/usr/local/bin/node` 에 설치

- 설치 후  
  
  ```shell
  $ node -v // v14.15.4 (21.01.31 기준 LTS)
$ npm -v // 6.14.10
  ```
  
  로 정상 설치 확인



### Node.js 실행

```shell
$ node
```

로 REPL(Read Eval Print Loop) 또는

```shell
$ node index.js
```

로 js 파일 실행



## Visual Studio Code

설치 : <https://code.visualstudio.com/Download>

확장 플러그인 설치

- Code Runner : 에디터에서 자바스크립트 파일 실행 시켜줌(그냥 `$ node 파일.js` 해주는 것)
- Live Server : 가상 서버 기동(port 5500)하여 브라우저 환경에서 HTML 파일 자동 로딩해주는 확장 플러그인
  - 소스코드 수정 시 자동 반영
  - 클라이언트 사이드 웹 API가 포함된 자바스크립트 코드 실행을 위해 필요. (아래 예시 참고)

cf) 아래 코드를 VS Code에서 실행하면?

```javascript
const arr = [1, 2];
arr.forEach(alert);
```

결과는 ReferenceError

- 브라우저 환경 : alert 함수는 클라이언트 사이드 웹 API로, 브라우저 환경에서는 정상 작동
- Node.js : 클라이언트 사이드 웹 API를 지원하지 않으므로 alert 함수를 알 수 없음.



## Reference

본 포스트는 [이웅모, 모던자바스크립트 Deep Dive](https://wikibook.co.kr/mjs/)의 3장 내용을 요약 및 재구성한 것입니다.

