---
categories:
  - HTTPS
tags:
  - SSL
  - HTTPS
  - dev
published: true
created: 2024-03-31 (일) 03:39:06
modified: 2024-03-31 (일) 23:57:29
---

## 이슈

- 광고 마크업을 테스트하기 위한 환경 구축이 필요했다.
	- 광고 마크업은 그 특성상, first party 사이트에 들어가지 않고 third party로 매체 사이트에 들어가게 된다.
	- 그렇기 때문에 테스트를 하기 위해서 실제 광고 마크업이 들어가게 될 first party 사이트, 즉 매체 사이트를 구축할 필요가 있었다.
- 이때 단순 로컬 html 파일로 테스트하지 않은 이유는, 브라우저 입장에서 실제 매체 사이트와 구분이 불가능해야했다.
	- 실제로 로컬 html 파일에서 문제가 없더라도, 인증서 이슈나 도메인으로 인한 CORS 이슈 등의 문제가 브라우저 상에서 발생할 수 있기 때문이다.
- 물론 직접 도메인 및 인증서를 발급받아서 매체 사이트를 구축하는 것도 가능하기는 하나
	- 비용 문제가 있기도 하고,
	- 수정 -> 브라우저 렌더링 테스트 의 과정이 빈번하게 반복되어야하는데, 로컬 파일에서 수정한 것을 즉시 확인할 수 있는 환경이 필요했다.
- 위와 같은 니즈로 인해 로컬에 HTTPS 서버를 도메인과 함께 구동할 수 있는 방법에 대해 살펴보고 정리하였다.

## 로컬 환경

아래에서 기술할 로컬 환경은 macOS m1 및 python3을 기준으로 작성하였다.
그리고 아래에서 기술할 시나리오는 `my-little-domain.com` 도메인으로 HTTPS 서버를 띄우는 것을 목표로 작성한 시나리오이다.
아래 시나리오 상에서 `my-little-domain.com` 대신, 필요한 도메인으로 대체하면 된다.

## 로컬 HTTP 서버 구동

인증서가 필요없는 http 서버는 아래와 같은 한 줄짜리 파이썬 코드를 통해 구동할 수 있다. (8000번 포트)
```bash
python3 -m http.server 8000 # python3 기준
```

```bash
python -m SimpleHTTPServer 8000 # python2 기준
```

위 명령어를 실행하게 되면, 해당 명령이 실행된 디렉토리의 파일들을 http://localhost:8000 에 접근하여 브라우저에서 볼 수 있다.
하지만, http로 서비스되는 매체 사이트는 이제 거의 없다고 봐야하기 때문에 같은 기능을 하는 https 매체 사이트가 필요했다.

## SSL 인증서 발급

HTTPS 서버를 띄우기 위해서는 인증서 발급이 선행되어야한다.

### openssl
이때 openssl을 통해 아래와 같이 간단하게 인증서를 발급할 수 있다.

```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365
```

하지만 위 인증서로 웹서버를 구동하는 경우 브라우저에서는 아래와 같이 워닝을 띄운다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/c8b9a20a-744e-4259-a329-cbd532ed085a)

그럴 수 밖에 없는 것이 openssl을 통해 발급한 인증서는 공인 기관(CA, Certificate Authority)으로부터 검증된 인증서가 아니기 때문이다.
그래서 우리에게는 CA로부터 공인된 인증서가 필요하다.

### mkcert

이걸 해주는 오픈소스 프로젝트가 바로 mkcert 이다.
mkcert는 프로젝트에 적혀있는 것처럼 "locally trusted development certificates", 즉 로컬 한정 공인된 인증서를 만들어주는 오픈소스이다.
> A simple zero-config tool to make locally trusted development certificates with any names you'd like.

mkcert를 우선 설치해주자.
os별 설치 방법은 readme에 자세히 나와있으니 참고하면 된다. 나는 brew를 통해 설치했다.

```bash
brew install mkcert
```

설치 후에 아래와 같은 명령어를 입력하여 로컬에 CA로 mkcert를 등록해준다.

```
mkcert -install
```

이때, 시스템 내 인증기관을 등록하는 일이기 때문에 비밀번호 입력이 필요하다.

위 명령어 실행 후 macOS 키체인 접근을 통해 확인해보면, 아래와 같이 루트 인증기관으로 mkcert가 등록되어 있음을 확인할 수 있다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/b736715c-7c42-4fac-82f7-b38dfab57efd)

이제 mkcert가 발급한 인증서는 내 로컬 pc에서 유효한 인증서가 되는 것이다.
그럼 로컬에서 CA가 된 mkcert로부터 "localhost" 도메인과 "my-little-domain.com" 도메인을 위한 인증서를 발급받아보자.

```bash
mkcert -key-file key.pem -cert-file cert.pem localhost my-little-domain.com
```

- private key 파일(`key.pem`)과 인증서 파일(`cert.pem`) 경로 및 파일명은 후에 웹서버를 구동할 때 필요하므로, 기억해두어야한다.

private key와 인증서가 준비되었으니 이제 HTTPS 웹서버를 구동해보자.

## 웹서버 구동

웹서버를 구동하는 방법은 apache, nginx 등등 정말 다양하다.
그러나 apache, nginx 등을 사용하는 건 일단 설치가 필요하고, 설치 후에도 conf 설정 방법에 대한 이해도 필요하다.
하지만 지금 당장 필요한건 단순히 로컬의 정적인 파일들을 브라우저에서 보기만 하면 된다.
그리고 다른 로컬 환경에서도 별도 의존성 없이 실행할 수 있는 방법 좋을 것이다.
그래서 위와 같은 요구사항을 위해 macOS에서 별도 의존성 없이 실행할수 있는 python `http.server` 사용하기로 했다.

아래와 같은 간단한 파이썬 코드(web_server.py)를 작성하고,

```python
# web_server.py
from http.server import HTTPServer, SimpleHTTPRequestHandler
import ssl

# Specify the path to your SSL certificate and key
ssl_certificate = 'cert.pem'
ssl_key = 'key.pem'

# Create an HTTP server with SSL support
httpd = HTTPServer(('localhost', 443), SimpleHTTPRequestHandler)
httpd.socket = ssl.wrap_socket(httpd.socket, certfile=ssl_certificate, keyfile=ssl_key, server_side=True)

# Start the server
print("Server started at https://localhost:443")
httpd.serve_forever()
```
(위 샘플 코드는 key 파일과 cert 파일과 같은 디렉토리에 작성하였으므로, 상대경로로만 파일들의 경로를 명시하였고, 별도 디렉토리에 작성하는 경우 그에 해당하는 절대 또는 상대 경로로 입력해주어야한다. )

아래 명령어를 통해 간단하게 localhost 443 포트에 HTTPS 웹서버를 구동할 수 있다.
```bash
sudo python3 web_server.py
```

이때, sudo가 필요한 이유는, HTTPS용 포트인 443 포트는 1024 이하의 [Privileged Ports](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html) 이기 때문에 루트 권한이 필요하다 .

이렇게 끝인줄로만 알았는데... 다른 pc에서 동일하게 실행하다가
`AttributeError: module 'ssl' has no attribute 'wrap_socket'` 에러와 함께 실패하는 것을 발견했다.

그래서 찾아보니, ssl 패키지의 socket을 생성하는 `wrap_socket()` 함수에 대한 API가 python 3.7 버전에서 deprecate 되었고, python 3.12 버전에서는 완전히 삭제되었다. (정확히는 사용 방법이 바뀐 것이다.)

>  - Remove the `ssl.wrap_socket()` function, deprecated in Python 3.7: instead, create a [`ssl.SSLContext`](https://docs.python.org/3.12/library/ssl.html#ssl.SSLContext "ssl.SSLContext") object and call its [`ssl.SSLContext.wrap_socket`](https://docs.python.org/3.12/library/ssl.html#ssl.SSLContext.wrap_socket "ssl.SSLContext.wrap_socket") method. Any package that still uses `ssl.wrap_socket()` is broken and insecure. The function neither sends a SNI TLS extension nor validates server hostname. Code is subject to [CWE-295](https://cwe.mitre.org/data/definitions/295.html): Improper Certificate Validation. (Contributed by Victor Stinner in [gh-94199](https://github.com/python/cpython/issues/94199).)

기존에 테스트하던 pc 환경은 python 3.6 버전이어서 위 코드가 문제 없이 동작했던 것이고, 새로운 환경은 python 3.12 버전이어서 에러와 함께 실패한 것이다.

그래서 파이썬 버전에 따른 분기처리를 추가하여 수정한 최종 코드는 아래와 같다.

```python
# web_server.py
import sys

from http.server import HTTPServer, SimpleHTTPRequestHandler
import ssl

# Specify the path to your SSL certificate and key
ssl_certificate = 'cert.pem'
ssl_key = 'key.pem'

# Create an HTTP server with SSL support    
server_address = ('localhost', 443)
httpd = HTTPServer(server_address, SimpleHTTPRequestHandler)

if sys.version_info >= (3, 7): # For Python 3.7 and later

    # Wrap the socket with SSL
    context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
    context.load_cert_chain(certfile=ssl_certificate, keyfile=ssl_key)
    httpd.socket = context.wrap_socket(httpd.socket, server_side=True)

else: # For Python versions earlier than 3.7
    
    # Wrap the socket with SSL
    httpd.socket = ssl.wrap_socket(httpd.socket, certfile=ssl_certificate, keyfile=ssl_key, server_side=True)

# Start the server
print("Server started at https://localhost:443")
httpd.serve_forever()
```

그래서 위 코드는 파이썬 3점대 버전에서 마이너 버전과 무관하게 동작하게 되고
다시 아래와 같은 명령어를 통해 웹 서버를 구동할 수 있고,
```bash
sudo python3 web_server.py
```
https://localhost 에 접근하여 브라우저에서 해당 디렉토리의 파일들을 볼 수 있다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/30f15903-1f85-4c98-a7d7-c72a1ec473c1)

## 호스트 파일(/etc/hosts) 수정

- 사실 위의 과정까지만 하면 필요한 것은 모두 된 것이다. localhost 라는 도메인의 HTTPS 웹서버를 띄운 셈이기 때문이다.
- 하지만 좀 더 매체 환경과 비슷하게 만들어주기 위해 localhost 외의 도메인을 localhost에 바인딩하여, localhost 외의 도메인(여기서는 my-little-domain.com)으로 접근할 수 있도록 변경해보자.
- 이를 위해서 로컬 DNS에 해당하는 호스트 파일 /etc/hosts를 수정한다.

```bash
echo '127.0.0.1 my-little-domain.com' | sudo tee -a /etc/hosts
```

- 이때 /etc/hosts 파일의 권한 문제로, output redirection(>>)으로 수정할 수 없기 때문에 `sudo tee -a` 로 추가해주어야 한다.
- 혹은 직접 파일을 에디터로 열어 수정해도 된다

이제 https://my-little-domain.com/ 에 접근하여 로컬 파일들을 볼 수 있다!

## References
- [GitHub - FiloSottile/mkcert: A simple zero-config tool to make locally trusted development certificates with any names you'd like.](https://github.com/FiloSottile/mkcert)
- [Privileged Ports](https://www.w3.org/Daemon/User/Installation/PrivilegedPorts.html)
- [What’s New In Python 3.12 — Python 3.12.2 documentation](https://docs.python.org/3.12/whatsnew/3.12.html#ssl)
