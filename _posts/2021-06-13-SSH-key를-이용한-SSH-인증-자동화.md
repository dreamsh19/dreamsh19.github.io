---
title: "SSH key를 이용한 SSH 인증 자동화"
categories:
  - ssh
tags:
  - ssh
  - automation
---

## 이슈

ssh를 이용하여 원격 서버(주로 게이트웨이 서버)를 매번 접속하는데 접속할 때마다 계정, 도메인, 비밀번호를 매번 치는 게 번거로웠다. 그래서 이를 자동화하기 위한 스크립트를 만들면 좋을 것 같다는 생각이 들었다. 처음에는 패스워드를 스크립트로 입력하도록 하는 자동화를 생각하며 방법을 찾아보았으나 패스워드 없이 ssh key를 이용하여 로그인하는 방법이 있다는 걸 알게 되었고, 그 방법이 더 좋아보여서 블로그에 남기고자 한다. 추가적으로 ansible에서 활용 가능성까지 다루고자 한다.



## SSH key를 이용한 인증

우선 ssh를 이용하여 로컬 서버에서, 원격의  `remote.com` 서버의 `irteam` 계정으로 접속하고자 하는 상황을 생각해보자. 

접속을 시도하는 쪽이 꼭 로컬 서버일 필요는 없지만(로컬 서버에서 또 다른 로컬 서버로 접속하는 경우) 이 글에서는 편의상 접속을 시도하는 쪽을 로컬 서버, 접속 대상이 되는 쪽을 리모트 서버라고 칭하려고 한다.   

로컬 서버에서

```bash
$ ssh irteam@remote.com
```
의 명령어 입력후 패스워드까지 입력해서 인증 과정을 거쳐 패스워드가 일치하면 원격 접속에 성공하게 된다.

그러나 ssh key를 이용하면 패스워드 대신 ssh key를 이용하여 인증을 진행하여 접속 가능 여부를 판단한다. 



### 1. ssh key 생성

접속을 시도하는 로컬 서버에서 최초(기존에 ssh key를 생성한 적이 없는 경우)에 아래의 명령어를 통해 ssh key를 생성해야한다. 기존에 ssh key가 존재하는 경우 이 과정을 생략가능하다.  일종에 자신을 나타내는 id를 만드는 과정으로 생각할 수 있다. 

```bash
$ ssh-keygen -t rsa -b 2048
```

그럼 아래와 같은 내용이 출력되는데, ssh key를 어디에 저장할지 등등 설정에 관한 내용이다. 

```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/home/username/.ssh/id_rsa): 
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /home/username/.ssh/id_rsa.
Your public key has been saved in /home/username/.ssh/id_rsa.pub.
```

특별히 지정이 필요한 경우가 아니라면, 아무 내용을 입력하지 않고 엔터를 입력하면 디폴트 값이 적용되므로, 엔터를 연타해준다

```bash
$ ls ~/.ssh
authorized_keys id_rsa          id_rsa.pub      known_hosts
```

그러면 홈 디렉토리(~) 아래 .ssh 디렉토리 안에 `id_rsa`, `id_rsa.pub`가 생성된 걸 확인할 수 있다. 각각 private key와 public key를 저장하고 있다.



### 2. ssh key 복사

위에서 생성한 로컬 서버의 ssh key 중 공개키(public key)를 리모트 서버에 전달하는 과정이 필요하다. 앞으로 이 공개키를 이용하여 통신을 할 것이니 리모트 서버에 그 키를 저장하는 과정 정도로 볼 수 있다. 로컬 서버에서 아래와 같은 명령어로 로컬 서버의 공개키를 리모트 서버에 전달할 수 있다.

```bash
$ ssh-copy-id irteam@remote.com
irteam@remote.com's password: 
```

최초 인증이기 때문에 패스워드를 입력하면 공개키 복사가 완료된다.



### 3. ssh 접속

```bash
$ ssh irteam@remote.com
```

공개키를 전달한 후 로컬 서버에서 ssh로 접속시 더 이상 패스워드를 묻지 않고 접속에 성공하게 된다!

추가적으로 리모트 서버에서 아래 명령어를 입력시

```shell
$ cat ~/.ssh/authorized_keys
```

로컬 서버의 공개키(`id_rsa.pub`에 저장되어 있던 키)가 저장되어 있는 것을 확인할 수 있다.  



## Ansible에서의 활용

[Ansible](https://docs.ansible.com/ansible/latest/index.html)은 ssh를 기반으로 여러 호스트에 명령을 하달하는 소프트웨어로, 원격으로 여러 서버에서 배포를 자동화할때 주로 쓰인다. 

이때 여러 호스트에 접속시 패스워드가 필요하기 때문에 플레이북 작성 시 인증 과정을 포함하는 경우가 있다. 그런데 위의 ssh key를 사전에 등록해놓으면 패스워드 입력없이 ssh로 원격 호스트에 접속이 가능하다. 따라서 접속 대상이 되는 호스트들에 대해 ssh key를 사전에 등록함으로써 플레이북에서 인증 과정을 제거할 수 있다. Ansible을 이용해 접속하는 호스트(예를 들어 배포 대상이 되는 서버)들의 경우 일반적으로 그 대상이 결정이 되어있고, 자주 변하는 정보가 아니기 때문에 ssk key를 이용한 인증을 유용하게 사용할 수 있다. 



## References

- [https://serverfault.com/questions/241588/how-to-automate-ssh-login-with-password](https://serverfault.com/questions/241588/how-to-automate-ssh-login-with-password)

