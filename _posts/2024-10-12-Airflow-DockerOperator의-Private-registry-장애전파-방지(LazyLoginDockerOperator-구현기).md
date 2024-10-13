---
created: 2024-10-12 (토) 22:11:42
modified: 2024-10-13 (일) 23:46:10
title: Airflow DockerOperator의 Private registry 장애전파 방지(LazyLoginDockerOperator 구현기)
categories:
  - Airflow
tags:
  - Airflow
  - Docker
published: true
last_modified_at: 2024-10-13 (일) 23:46:10
---

## 이슈
- Airflow에서 DockerOperator를 이용하여 매번 private registry에서 도커 이미지를 pull해서 수행하는 배치가 있다.
- 그런데 외부 조직에서 관리하는 private registry의 장애가 발생하는 경우 registry의 장애가 서비스의 배치까지 전파되는 문제가 있었다.
- 그리고 private registry의 장애가 빈번해짐에 따라 서비스에 영향을 주지 않게끔 조치가 필요해졌고, registry 장애가 서비스 배치의 장애로 전파되지 않도록 하고자했다.

## 문제 상황 당시 로그

![image](https://github.com/user-attachments/assets/979aeed5-6d43-4432-ab22-9d53d8fc9268)

문제 상황 발생 당시의 로그이다. Docker login 단계에서 private registry에서 500 에러가 발생하면서 해당 DAG가 실패하였다.

## 최초 시도 : force_pull 옵션 변경

서비스에서는 매번 private registry에서 이미지를 pull하도록 하는 DockerOperator의 force_pull 옵션을 True로 설정하여 사용하고 있었다.
force_pull 옵션은 그 이름에서도 쉽게 알 수 있듯이, DockerOperator가 수행될때마다 무조건 이미지를 pull해서 사용하도록 하는 옵션이다.
> **force_pull** ([_bool_](https://docs.python.org/3/library/functions.html#bool "(in Python v3.11)")) – Pull the docker image on every run. Default is False.

True로 설정해두고 사용했던건 아마도 원격의 단일 이미지를 참조하게 함으로써 로컬에 캐싱된 이미지로 인한 불일치나 장애를 막기 위함이 아닐까 추측된다.
따라서 force_pull 옵션을 False로 설정하면 최초 수행 시(이미지 버전 업 등)에만 이미지를 pull하고, 그 이후에는 이미 pull 받은 이미지를 로컬에서 가져올테니,
최초 수행 시에만 registry 의존성이 있고, 그 이후에는 registry 장애에 영향을 받지 않을 것이라고 생각했다.
(물론 이게 유효하려면 매번 같은 worker에서 수행된다는 보장이 되어야하는데, 해당 DAG는 고정된 하나의 장비에서 수행되도록 설정되어 있었다.)

그러나.. force_pull 옵션을 비활성화하고 테스트를 해보았으나, private registry가 올바른 응답을 주지 않는 경우에 여전히 DAG는 실패했다.

## DockerOperator 소스를 살펴보자

force_pull 옵션이 의도대로 동작하지 않고, 문서에도 특별한 설명이 없어서 소스 코드를 살펴보았다.
아래는 DockerOperator의 execute() 함수 전문이다. (서비스에서 사용하고 있던 providers-docker 2.5.0 버전 기준)
참고로, Airflow의 각종 Operator들은 BaseOperator의 execute() 함수를 구현하여 서로 다른 기능을 제공한다.


```python
def execute(self, context: 'Context') -> Optional[str]:
	self.cli = self._get_cli()
	if not self.cli:
		raise Exception("The 'cli' should be initialized before!")

	# Pull the docker image if `force_pull` is set or image does not exist locally
	if self.force_pull or not self.cli.images(name=self.image):
		self.log.info('Pulling docker image %s', self.image)
		latest_status = {}
		for output in self.cli.pull(self.image, stream=True, decode=True):
			if isinstance(output, str):
				self.log.info("%s", output)
				continue
			if isinstance(output, dict) and 'status' in output:
				output_status = output["status"]
				if 'id' not in output:
					self.log.info("%s", output_status)
					continue

				output_id = output["id"]
				if latest_status.get(output_id) != output_status:
					self.log.info("%s: %s", output_id, output_status)
					latest_status[output_id] = output_status
	return self._run_image()

def _get_cli(self) -> APIClient:
	if self.docker_conn_id:
		return self.get_hook().get_conn()
	else:
		tls_config = self.__get_tls_config()
		return APIClient(base_url=self.docker_url, version=self.api_version, tls=tls_config)
```
- [airflow/airflow/providers/docker/operators/docker.py at providers-docker/2.5.0 · apache/airflow · GitHub](https://github.com/apache/airflow/blob/providers-docker/2.5.0/airflow/providers/docker/operators/docker.py#L359-L390)

로직 자체는 이미지를 pull해야하는 경우(force_pull이거나, 이미지가 존재하지 않는 경우) 이미지를 pull하고 run 한다. 정도로 간단하다.

그런데 위 코드에서 주목할만한 점은 execute()을 진입하자마자 docker client(코드에서 `self.cli`)를 `_get_cli()` 함수를 통해 초기화한다.
그리고 `_get_cli()` 함수를 살펴보면, docker_conn_id 가 있는 경우 DockerHook을 이용해서 클라이언트를 초기화한다.
이때 docker_conn_id는 private registry에 접근할때의 인증정보를 포함하고 있는 객체이다.

> If a login to a private registry is required prior to pulling the image, a Docker connection needs to be configured in Airflow and the connection ID be provided with the parameter `docker_conn_id`.

서비스에서는 private registry에 접근하기 때문에 docker_conn_id를 설정하여 사용하고 있었고, 그렇기 때문에 `_get_cli()`에서 해당 분기를 타게 된다.

그렇다면 DockerHook 클래스의 `get_conn()` 함수를 더 살펴보자. 아래는 해당 함수 전문이다.
```python
def get_conn(self) -> APIClient:
	client = APIClient(base_url=self.__base_url, version=self.__version, tls=self.__tls)
	self.__login(client)
	return client


def __login(self, client) -> None:
	self.log.debug('Logging into Docker')
	try:
		client.login(
			username=self.__username,
			password=self.__password,
			registry=self.__registry,
			email=self.__email,
			reauth=self.__reauth,
		)
		self.log.debug('Login successful')
	except APIError as docker_error:
		self.log.error('Docker login failed: %s', str(docker_error))
		raise AirflowException(f'Docker login failed: {docker_error}')
```
- [airflow/airflow/providers/docker/hooks/docker.py at providers-docker/2.5.0 · apache/airflow · GitHub](https://github.com/apache/airflow/blob/providers-docker/2.5.0/airflow/providers/docker/hooks/docker.py#L88-L106)

놀랍게도 클라이언트를 생성한 이후에 즉시 로그인을 하는 것을 확인할 수 있다!
그리고 `__login()` 함수를 따라가다보니 에러발생시 `Docker login failed` 라는 [문제 상황 당시 로그](#문제-상황-당시-로그) 에서 봤던 익숙한 문구가 눈에 띈다.

결국 소스 코드를 살펴봤을때,
**docker client 생성 시점에 로그인을 시도하고, 로그인이 실패하면서, 로컬 이미지의 존재 여부는 조회조차 하지 못한채 실패하는 것이었다.**

## execute() 함수 오버라이딩을 통한 해결

결국 우리가 필요한 것은, 로컬의 이미지를 조회하고, 없는 경우에만 이미지를 pull하면 된다.
따라서 images API 호출 시점의 docker client는 로그인이 필요하지 않고, pull API 호출 시점의 docker client는 로그인이 필요하다.
그런데 위 execute() 함수의 구현 상 docker client(`self.cli`)를 중간에 주입할 수가 없는 구조이기 때문에 아래와 같이 execute() 함수 자체를 오버라이딩하는 방식으로 해결하였다.

1. images API 호출 시점의 docker client는 로그인을 하지 않은 상태의 client 생성
2. images API 호출 이후~ pull API 호출 직전에 client 로그인 수행
3. 그 외의 로직은 동일

위와 같은 요구사항을 반영하여 구현한 전문은 아래와 같다.(DockerOperator를 상속한 LazyLoginDockerOperator)

```python
from typing import Optional
from airflow.providers.docker.operators.docker import DockerOperator
from docker import APIClient, tls

class LazyLoginDockerOperator(DockerOperator):

    def execute(self, context) -> Optional[str]:
        self.cli = self._get_cli()
        if not self.cli:
            raise Exception("The 'cli' should be initialized before!")

        # Pull the docker image if `force_pull` is set or image does not exist locally
        
        if self.force_pull or not self.cli.images(name=self.image):
            if self.docker_conn_id:
                self.cli = self.get_hook().get_conn()

            self.log.info('Pulling docker image %s', self.image)
            latest_status = {}
            
            for output in self.cli.pull(self.image, stream=True, decode=True):
                if isinstance(output, str):
                    self.log.info("%s", output)
                    continue
                if isinstance(output, dict) and 'status' in output:
                    output_status = output["status"]
                    if 'id' not in output:
                        self.log.info("%s", output_status)
                        continue

                    output_id = output["id"]

                    if latest_status.get(output_id) != output_status:
                        self.log.info("%s: %s", output_id, output_status)
                        latest_status[output_id] = output_status
        return self._run_image()

    def _get_cli(self) -> APIClient:
        return APIClient(base_url=self.docker_url, version=self.api_version, tls=self.__get_tls_config())

    def __get_tls_config(self) -> Optional[tls.TLSConfig]:
        return self._DockerOperator__get_tls_config()
```

위와 같이 변경한 LazyLoginDockerOperator를 문제가 된 DAG에 적용하였고, 이후에 동일한 registry 장애가 발생했을때 해당 DAG는 영향을 받지 않고 정상수행됨을 확인했다!
이로써 외부 요인인 registry 장애가 발생했을 때 서비스 배치로의 장애가 전파되는 걸 막을 수 있었고, 장애 대응으로 인한 공수 또한 절감할 수 있었다.

## 이후 버전에서 해결되었을까

사실 문제 발생 당시에 이미지 존재 여부와 무관하게 실패하는 것이 정상동작은 아니라고 생각하여 bug fix가 있었는지를 찾아보았으나, 딱히 올라오진 않았고, 당시 최신버전에서도 문제가 해결되지 않은 것으로 확인해서 위와 같이 어쩔 수 없이 직접 오버라이딩하는 방식으로 해결하였다.
그리고 현재(2024년 10월) 기준 최신 버전인 providers-airflow 3.14.0 버전에서도 여전히 동일한 이슈가 발생하는 것으로 확인된다.

 [make docker operators always use \`DockerHook\` for API calls by Taragolis · Pull Request #28363 · apache/airflow · GitHub](https://github.com/apache/airflow/pull/28363)
위 PR에서 docker client를 획득하는 로직이 변경되긴 했으나, 단순히 생성 시점이 execute() 호출 시점이 아닌 self.cli 최초 호출 시점에 최초 생성된다는 점만 바뀌었을뿐, 여전히 생성 시점에 로그인을 하는 로직은 바뀌지 않은듯하고, 이 로직이 현재 최신 버전에서도 동일하다.

```python
# DockerHook.api_client()의 일부 
if self.docker_conn_id:
	# Obtain connection and try to login to Container Registry only if ``docker_conn_id`` set.
	self.__login(client, self.get_connection(self.docker_conn_id))
```
- [airflow/airflow/providers/docker/hooks/docker.py at providers-docker/3.14.0 · apache/airflow · GitHub](https://github.com/apache/airflow/blob/providers-docker/3.14.0/airflow/providers/docker/hooks/docker.py#L151-L153)

따라서 현재 기준 최신 버전에서도 동일한 이슈는 여전히 발생하기 때문에, 단순 버전업만으로는 위와 같은 문제에 대한 해결은 어려워보인다.

## References

- [airflow/airflow/providers/docker at providers-docker/2.5.0 · apache/airflow · GitHub](https://github.com/apache/airflow/tree/providers-docker/2.5.0/airflow/providers/docker)
- [airflow.providers.docker.operators.docker — apache-airflow-providers-docker Documentation](https://airflow.apache.org/docs/apache-airflow-providers-docker/stable/_api/airflow/providers/docker/operators/docker/index.html)
- [make docker operators always use \`DockerHook\` for API calls by Taragolis · Pull Request #28363 · apache/airflow · GitHub](https://github.com/apache/airflow/pull/28363)
