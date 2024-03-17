---
categories:
  - Airflow
tags:
  - Airflow
  - Python
published: true
created: 2024-03-17 (일) 17:05:55
modified: 2024-03-17 (일) 20:03:22
---

## 이슈
- Airflow 상에서 매시간(hourly) 도는 DAG가 어느 순간부터 00시대에만 항상 실패하는 이슈가 있었다.
- 해당 태스크는 실패하더라도, 플랜 B가 동작하기 때문에 서비스에 지장은 없었지만, 반복적으로 00시에 실패하는 것이 우연이 아닐 것이라 생각하여 자세히 살펴보기 시작했다.

## 원인 파악
- 문제가 된 DAG 내 태스크는 하둡 특정 시간대의 디렉토리의 파일 존재여부를 체크하는 PythonSensor 오퍼레이터로 작성된 태스크였다.
- 구체적으로는, 아래와 같은 형태였다.
	- 이때 날짜(date)와 시간(hour)은 배치 수행시간(`data_interval_start`)에서 파싱해서 가져온다.

```python
def check_flag_file(date, hour):
	file_path=f'{conf.hdfs}/dt={date}/hr={hour}/_*'
	// 후략  
  
check_success = PythonSensor(
	task_id='check_success',
	python_callable=check_flag_file,
	op_kwargs={"date": date, "hour": hour},
	poke_interval=10,
	// 후략
)
```

- 그래서 실패한 자정 시간대의 태스크 기록을 보니, 아래와 같이 '00'의 형태가 아닌 '0'으로 들어가고 있었다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/4a50124f-a837-4681-8ead-68f8a6b4ece5)

- 타겟 디렉토리명은 시간이 두자리로 포맷팅된 형태였고, `hr=00` 이 아닌 `hr=0` 디렉토리를 조회하니, 항상 존재하지 않아 태스크가 실패했던 것이다.
- 그럼 직접적인 원인은 확인했으니, 왜 '00'이 아닌 '0'으로 들어가고 있는지 더 자세히 알아보기 시작했다.

## int 타입의 포맷팅 이슈가 아닐까?
00이 0으로 바뀌었으니, 처음으로 든 생각은 int 타입의 변수가 어떠한 이유로 한자리로 포맷팅되면서 발생한 이슈가 아닐까?라는 생각이었다.
그러나, 단순 포맷팅 문제라면, 00시 뿐만 아니라, 01시, 02시부터 09시까지 모두 문제가 됐어야했다.
하지만 실제로 문제가 된건 00시 뿐이었고, 01시~09시까지는 의도대로 두자리수로 처리되어 문제가 없었다.
그래서 단순 포맷팅 이슈는 아닐꺼라고 판단하여 구체적으로 변수가 변환되는 과정을 추적하기 시작했다.

## Airflow 템플릿 내부 동작

위의 태스크 기록을 봤을때 kwargs로 들어갈때 이미 '0'이 들어갔다는 것은 곧 템플릿이 렌더링된 시점에 이미 '0'이 되어버린 것이다. 그래서 airflow의 템플릿의 동작 방식을 알아보기 시작했다.
airflow는 템플릿 문법을 지원하며, 내부적으로 템플릿 엔진으로 jinja를 채택하고 있다.
- [https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#jinja-templating](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/operators.html#jinja-templating)

그래서 문서를 읽던 중, `render_template_as_native_obj` 옵션에 대한 이야기가 있었고, 이 내용이 눈에 띄었다. 이 옵션이 눈에 띈 이유는 최근에 해당 DAG에 `render_template_as_native_obj=True` 옵션을 적용하는 배포가 있었기 때문이다.
해당 옵션은 템플릿화된 변수를 렌더링할때, 단순문자열(디폴트 설정, `render_template_as_native_obj=False`)로 렌더링할 것이냐, 아니면 문자열을 파싱해서 파이썬 내장 객체로 렌더링할 것이냐를 지정하는 옵션이다.
공식 문서에서 설명하고 있는 사용 예시는, dict 형태의 input을 전달받고 싶을때 해당 옵션을 켜줌으로써 `string -> dict`로의 변환을 개발자가 아닌 airflow 설정만으로 가능하게 한다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/cb310bce-69dd-4378-8259-c85baa3f541f)


그래서 최근에 해당 옵션이 적용된 만큼, 실제 설정을 적용했을때의 내부 동작을 좀더 확인해보기 시작했다.

`render_template_as_native_obj` 옵션을 켜면, 내부적으로 jinja2의 [NativeEnvironment](https://jinja.palletsprojects.com/en/2.11.x/nativetypes/)의 render() 함수를 활용하여 파이썬 내장객체를 리턴한다.
그래서 NativeEnvironment의 render() 함수의 소스코드를 살펴보았고, 일부를 발췌했다.

```python
// 전략
try:
	return literal_eval(raw)
except (ValueError, SyntaxError, MemoryError):
	return raw
```
- [jinja/src/jinja2/nativetypes.py at aa3d688a15aece0a0de0b59f94dda870c724bc87 · pallets/jinja · GitHub](https://github.com/pallets/jinja/blob/aa3d688a15aece0a0de0b59f94dda870c724bc87/src/jinja2/nativetypes.py#L32-L35)

결국 내부적으로 ast.literal_eval() 함수를 호출하여 문자열을 내장객체로 파싱하고, 예외가 발생(파싱에 실패)하는 경우 raw, 즉 문자열 원본을 그대로 리턴하는 로직이다.
(ast는 Abstract Syntax Trees를 의미하는 파이썬 내장 패키지이며, 일종의 파이썬 문법을 파싱하는 패키지정도로 이해했다.)

그렇다면, ast.literal_eval() 함수의 리턴 결과를 직접 확인해보기로 했다. (현재 팀에서 airflow를 구성하는데 사용한 python 3.6.5 버전으로 확인했다)

```bash
[root@server ~]$ python3
Python 3.6.5 (default, May  8 2018, 12:10:43)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-18)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import ast
>>> res = ast.literal_eval('00')
>>> res
0
>>> type(res)
<class 'int'>
```

확인 결과, 문제가 됐던 상황처럼 문자열 '00'이 int 0으로 파싱되었다!

그렇다면 '01'은 int로 파싱이 안되는 것일까?

```bash
>>> res = ast.literal_eval('01')
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "/usr/local/lib/python3.6/ast.py", line 48, in literal_eval
    node_or_string = parse(node_or_string, mode='eval')
  File "/usr/local/lib/python3.6/ast.py", line 35, in parse
    return compile(source, filename, mode, PyCF_ONLY_AST)
  File "<unknown>", line 1
    01
     ^
SyntaxError: invalid token
```

'01'은 파싱에 실패하여 SyntaxError가 발생하였다.
그리고 위에 발췌한 NativeEnvironment의 render()의 구현에 따르면, SyntaxError가 발생한 경우 원본 문자열인 '01'을 리턴하게 된다.
이것을 통해 '00'과 '01'의 파싱 결과가 다르다는 사실을 확인했다.

결국 이것을 확인함으로써 모든 것이 명확해졌다.
`render_template_as_native_obj=True` 옵션을 적용하면서 템플릿을 렌더링하는 로직이 달라졌고,
해당 옵션을 적용했을때의 템플릿 엔진이 문자열 '00'을 int 0으로 파싱하게 되면서 00시에는 오류가 발생했고,
문자열 '01'은 파싱에 실패하여 문자열 원본 '01'이 그대로 리턴되면서 그외의 시간에는 문제가 발생하지 않았던 것이다.

## 해결

`render_template_as_native_obj` 옵션이 명확하게 원인인 것을 알았으니, 다시 사용하지 않는게 명확한 해결책이지만, 해당 옵션은 다른 태스크에서 활용하고 있었기 때문에 이 해결책은 기각하였다.

결국 우리에게 필요한 것은 아래와 같으므로,
1. int 0이 들어왔을때 00을 리턴한다.
2. string '01'~'23'이 들어왔을때 01~23을 리턴한다.

아래와 같이 명시적인 "형변환 후 포맷팅" 로직을 적용하여 해결하였다.

```python
def check_flag_file(date, hour):
	file_path=f'{conf.hdfs}/dt={date}/hr={int(hour):02}/_*'
	// 후략  
```

결국 해결은 `render_template_as_native_obj` 옵션을 활용하진 않았지만, 해당 옵션에 대한 이해가 부족했다면, 적용하지 못했을 해결 방법이다.

## 추가적인 궁금증

위 내용을 통해 ast.literal_eval() 이 이 문제의 범인임은 알았다. 그렇다면 '00'은 int 0으로 파싱하면서 '01'은 파싱을 못하는 건 ast의 스펙인걸까?
그래서 ast.literal_eval() 소스코드를 좀 더 살펴보기 시작했다.
결국 ast 역시 파이썬 내장 compile() 함수에 파싱을 위임하고 있었다.
![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/6c612f4d-41cf-4730-adf5-fb0f5d72b0ae)
- [cpython/Lib/ast.py at v3.6.5 · python/cpython · GitHub](https://github.com/python/cpython/blob/v3.6.5/Lib/ast.py#L35)


그럼 결국 이건 ast의 스펙이 아니고, 파이썬의 스펙이라는 것인데, 그래서 아래와 같이 확인해봤다.
```bash
[root@server ~]$ python3
Python 3.6.5 (default, May  8 2018, 12:10:43)
[GCC 4.4.7 20120313 (Red Hat 4.4.7-18)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> a=00
>>> a
0
>>> type(a)
<class 'int'>
>>> a=01
  File "<stdin>", line 1
    a=01
       ^
SyntaxError: invalid token
```

확인결과, 파이썬의 스펙인듯하다. 파이썬 문법상 리터럴 00은 int 0으로 파싱되지만, 리터럴 01은 파싱에 실패한다.

이걸 이상하게 생각하는 건 나뿐만이 아니었던 듯하다. 관련해서 구글링을 해보니 아래와 같은 글을 찾을 수 있었다.
[Why does Python 3 allow "00" as a literal for 0 but not allow "01" as a literal for 1? - Stack Overflow](https://stackoverflow.com/questions/31447694/why-does-python-3-allow-00-as-a-literal-for-0-but-not-allow-01-as-a-literal)

요약하자면, `"0"+`이 스페셜 케이스이고, 이걸 도입했을 당시의 명확한 이유가 기억이 나지 않는다고 한다;;
그래서 많은 사람들이 이러한 스페셜 케이스를 없애자고 제안차 버그 리포트를 올렸으나, 반영이 되지 않아 현재까지 `"0"+` 리터럴은 0으로 파싱되고 있다고 한다.

![image](https://github.com/dreamsh19/dreamsh19.github.io/assets/47855638/d09100cc-6ea4-45d8-b3f9-99ede7820f31)

## References

- [Operators — Airflow Documentation](https://airflow.apache.org/docs/apache-airflow/2.2.3/concepts/operators.html#jinja-templating)
- [Native Python Types — Jinja Documentation (2.11.x)](https://jinja.palletsprojects.com/en/2.11.x/nativetypes/)
- [jinja/src/jinja2/nativetypes.py at aa3d688a15aece0a0de0b59f94dda870c724bc87 · pallets/jinja · GitHub](https://github.com/pallets/jinja/blob/aa3d688a15aece0a0de0b59f94dda870c724bc87/src/jinja2/nativetypes.py)
- [cpython/Lib/ast.py at v3.6.5 · python/cpython · GitHub](https://github.com/python/cpython/blob/v3.6.5/Lib/ast.py)
- [ast — Abstract Syntax Trees — Python 3.12.2 documentation](https://docs.python.org/3/library/ast.html#ast.literal_eval)
- [Why does Python 3 allow "00" as a literal for 0 but not allow "01" as a literal for 1? - Stack Overflow](https://stackoverflow.com/questions/31447694/why-does-python-3-allow-00-as-a-literal-for-0-but-not-allow-01-as-a-literal)
