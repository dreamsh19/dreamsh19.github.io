---
categories:
  - Nginx
tags:
  - Nginx
  - '414'
  - http status code
---



성능 테스트를 위해 go 기반의 http 로드 테스팅 툴인 [vegata](https://github.com/tsenart/vegeta) 를 이용하여 진행하던 중 이상한 점을 발견했다. 



## 이슈

- vegeta를 이용하여 테스트를 진행했을 때는 모두 200 OK를 반환
- 그런데 동일한 요청을 로컬에서 브라우저 또는 Postman으로 테스트했을 때는 414 Request URI Too Large가 반환
- 상태 코드의 이름에서 알 수 있듯이, Request URI가 너무 길어서 발생한 것임(실제로 요청의 쿼리 파라미터가 매우 긴 형태였다)을 유추할 수는 있었지만 그렇다면 vegeta로 요청을 보냈을때도 마찬가지로 414가 반환되어야 하는데, 왜 200이 반환되었는가?



같은 요청에 대해 다른 결과가 나오는 것이 의아하기도 했고, http status code 중 414는 처음 접하는 에러인지라 이 에러에 대해 찾아보게 되었다. 





## Http status code 414

Http status code 414는 URI Too Long에 해당하는 status code로, 서버가 정한 요청 uri의 최대길이보다 더 큰 요청을 보냈을 때 반환하는 상태 코드이다. 

주로 발생하는 경우는 아래와 같다.

- post 요청을 get 요청으로 전환하여 쿼리 파라미터가 아주 길어지는 경우
- redirection loop에 빠지는 경우(redirection uri가 다시 자신을 가리키는 경우 등)
- 혹은 악의적으로 uri를 길게 구성하여 보안 취약점을 이용하려는 경우

사용하는 웹서버는 nginx로 구성했기 때문에 nginx에서 해당 에러와 관련된 설정을 찾아보았다.



## Nginx client request header buffer

Nginx의 경우에는 client request header buffer라는 것이 존재하고, 클라이언트에서 요청을 보내면, 해당 요청의 request header를 버퍼에 저장하는 방식으로 작동한다. 그런데 그 버퍼의 종류에는 두가지가 있는데, 일반적인 요청의 경우에 사용하는 버퍼와 request header가 큰 요청의 경우이다.

### 1. 일반적인 요청

Nginx의 경우 `client_header_buffer_size` 라는 변수로 일반적인 요청에 대한 헤더 버퍼 크기를 지정한다. 디폴트 값은 1KB이다. 대부분의 경우는 1KB를 넘지 않기 때문에 이 버퍼를 거쳐간다. 그리고 이 버퍼의 사이즈를 초과하는 요청이 들어왔을 경우에는 아래에서 다룰, 조금 더 큰 버퍼로 처리하게 된다. 

![image-20210509172245182](https://raw.githubusercontent.com/dreamsh19/dreamsh19.github.io/master/assets/image/image-20210509172245182.png)



### 2. Header size가 큰 요청

요청 header size가 위에서 지정한 `client_header_buffer_size` 를 초과하는 경우 보다 큰 새로운 버퍼(large buffer라고 칭하겠다)에 요청 헤더를 저장하게 된다. 이때, 그 크기는 `large_client_header_buffers` 값을 통해 지정하게 된다. 여기서는 large buffer의 개수와 크기를 같이 지정해주게 되는데 디폴트 값은 개수 4개, 크기는 8KB이다. 즉, 헤더 크기가 `client_header_buffer_size` 를 넘는 요청은 최대 4개까지 동시에 처리가 가능함을 의미한다. 

![image-20210509172318697](https://raw.githubusercontent.com/dreamsh19/dreamsh19.github.io/master/assets/image/image-20210509172318697.png)

이때 large buffer의 경우에는 항상 떠있는 건 아니고 요청이 있을 때 할당하는 것이라고 한다. 애초에 목적 자체가 일반적으로 사용되는 목적보다는 헤더 크기가 큰 특수한 요청만 처리하는 목적이기 때문인 것으로 생각된다. 그리고 커넥션이 keep-alive 상태가 되면 할당을 해제한다. 

그리고 버퍼 개수보다 많은 요청이 들어오는 경우에는 새로운 요청을 큐에 저장하여 나중에 처리하는지 혹은 애초에 요청 자체를 거부하는지는 확인이 필요하다. 



결론적으로 저 large buffer의 크기**(`large_client_header_buffers `)보다 큰 헤더를 가진 요청이 들어오는 경우에 414 status code를 리턴하게 된다.** 



## Client에 따른 헤더 크기 차이

그렇다면 동일 요청이라면 결과가 일관되게 나와야 할 것 같은데 vegeta로 요청 시 200이 나오고 postman으로 요청 시 414가 나오는 이유는 무엇일까?

결론부터 말하자면, **동일 요청이어도 클라이언트에 따라 request header 크기가 달라져서 결과가 달랐던 것**이다. Request header의 경우 url을 제외하고도 기타 정보(User-agent) 등을 추가적으로 포함하는데 이러한 기타 정보는 요청을 구성하는 클라이언트에 따라 달라지게 된다. 그래서 실제로 동일 요청에 대해 클라이언트별로 Request header 크기를 확인해보았다.



### 1. curl

curl의 경우 vegeta와 마찬가지로 414가 나오지 않고 200이 반환되었다. 그래서 이때의 request header 크기를 확인해보았다. curl의 경우 -w 옵션을 이용하여 아래와 같은 명령어로 request size를 알 수 있다.(실제 반환되는 결과는 request 전체의 크기, 즉 header+body의 크기이지만, get 요청이기 때문에 request body가 존재하지 않아 request 전체의 크기=header의 크기가 된다)

```shell
curl -o /dev/null -w "size_request:%{size_request}" {url}
```

해당 요청의 결과 사이즈는 **5,575B로, 8K(nginx 디폴트 설정)를 넘지 않아** 200을 리턴하게 된다.



### 2. Postman

Postman의 경우 414가 반환되었다. 

![image-20210509190409942](https://raw.githubusercontent.com/dreamsh19/dreamsh19.github.io/master/assets/image/image-20210509190409942.png)

이때 request size는 아래와 같은 size 탭을 통해 확인할 수  있으며, request header 크기가 **9.29KB로 8K를 초과**하여 414를 리턴하게 된다. 

추가적으로 url 길이를 줄여가며, 요청을 보냈을 때 크기가 8.18KB일 때까지 200이 반환되다가 8K(8,192B)를 넘어가는 순간 414가 반환되었다.



### 3. Chrome browser

크롬 브라우저의 경우 414가 반환되었다.

![image-20210509190750887](https://raw.githubusercontent.com/dreamsh19/dreamsh19.github.io/master/assets/image/image-20210509190750887.png)

크롬 브라우저의 request header size를 확인하는 방법은 조금 까다로웠다. 위의 그림에서 처럼 크롬 개발자도구 > Network 탭에서 해당 요청을 선택한후 Copy as cURL을 선택하여 복사한 후 위의 1번 curl의 방법으로 확인했다.

확인 결과 크기가 **9,797B로 8K를 넘어** 414를 리턴하게 된다.





## 해결 방법

보통 이처럼 GET 방식의 요청에 쿼리 파라미터로 데이터로 전달하는 과정에서 414 에러가 발생하는 경우에 POST 방식으로 전환하는 것이 정석이라고 한다. 하지만 부득이하게 get 요청을 유지해야하는 경우가 있을 수도 있다. API 스펙을 변경하는 것은 해당 API를 호출하는 다른 모든 서비스들의 변경이 필요하기 때문이다. 따라서 불가피하게 GET 요청 방식을 유지해야하는 경우,  nginx의 `large_client_header_buffers` 의 크기를 더 큰 값으로 수정함으로써 문제를 해결할 수 있다. (일반 요청 버퍼 사이즈인 `client_header_buffer_size` 를 늘려도 되지만, 일반 요청에 대해서도 지나치게 큰 버퍼를 할당하는 것은 메모리 낭비이다.)  하지만 이러한 해결방식이 갖는 문제점이 있는데

1. 버퍼의 개수 제한이 있기 때문에 헤더 사이즈가 큰 요청이 빈번하게 들어오는 경우 병목이 발생할 수 있다.
2. 최대 길이의 상한을 예측하기 어려운 경우, 즉 해당 버퍼 크기를 넘는 요청이 들어오지 않는다는 보장을 할 수 없는 경우 결국 같은 문제에 봉착하게 된다. 

따라서, 헤더 크기가 큰 요청이 빈번하지 않고, 요청 헤더 크기의 상한을 예측가능한 경우에는 유용한 방법으로 생각된다.





## References

- <https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/414>

- <http://nginx.org/en/docs/http/ngx_http_core_module.html#large_client_header_buffers>

- <https://sub0709.tistory.com/175>

- <https://serverfault.com/questions/892006/chrome-devtools-request-header-size>