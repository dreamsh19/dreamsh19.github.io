---
title: Git commit hash는 어떻게 만들어질까
categories:
  - Git
tags:
  - Git
  - Hash
published: true
created: 2024-10-26 (토) 20:48:53
modified: 2024-10-27 (일) 22:44:31
---

## 이슈
- 개발을 하다보면 git rebase를 자주 사용하는데, rebase 할때 실제 반영되는 내용이 동일한데도 commit hash가 바뀌는 것을 보고, 커밋 시간이나 부모 커밋 등의 정보가 달라졌으니 hash값이 달라지는구나 정도로 어렴풋이 추측만 했다.
- 그렇다면 서로 다른 커밋을 "다르다"라고 판단하는 기준이 무엇일까가 궁금해졌고,
- 어렴풋이 추측만 하던, git에서의 id 체계인 commit hash가 어떻게 만들어지는지에 대해 살펴보았다.

## 결론

결론부터 얘기하자면, 임의의 git repository에 진입해서 아래 명령어를 수행하면, 현재 위치한 HEAD의 commit hash를 재현해낼 수 있다.

```bash
(printf "commit %s\0" $(git --no-replace-objects cat-file commit HEAD | wc -c); git cat-file commit HEAD) | sha1sum
# 출처 : https://gist.github.com/masak/2415865#file-explanation-md
```

위 명령어를 수행하고, `git show -s` 명령어를 통해 현재 commit hash를 확인해보면 동일한 것을 확인 할 수 있다.

아래는 직접 확인해본 결과이다. 역시나 동일함을 확인할 수 있다.
![image](https://github.com/user-attachments/assets/e1ccf805-c939-4292-8e05-0e09548fd7f2)

그렇다면, 위 명령어에 대해 하나씩 살펴보자.
우선 마지막의 sha1sum 명령어는 단순 해싱 함수이고, 본 글에서는 hash 함수의 input이 궁금한 것이므로 미뤄두자.

눈에 띄는 것은 `git cat-file commit HEAD` 명령어가 결국 주요한 input으로 보인다. (앞의 printf 파트는 해당 명령어의 바이트수를 포맷팅하여 출력하는 정도이므로)

그렇다면 `git cat-file commit HEAD` 명령어는 대체 어떤 내용을 담고 있는지 살펴보자.

## 확인을 위한 간단한 구성

살펴보기 전에 간단한 git repository를 아래와 같이 생성하여, 해당 repository 하에서 내용을 확인하였다.

```bash
mkdir git-hash && cd git-hash
git init
touch README.md
git add . && git commit -m "Intial commit"
echo "Hello World" >> README.md
git add . && git commit -m "Add Hello World to README"
```

## git cat-file commit 상세

우선 git cat-file 명령어는 git의 object에 대한 세부정보를 출력해주는 명령어이다.
여기서 object란 git에서 내부적으로 리소스를 관리할때 사용하는 객체이며, commit도 object의 한 종류이다.
결국 `git cat-file commit HEAD` 명령어는 현재(HEAD) 커밋에 대한 세부정보를 출력해줘 정도의 명령어이다.

실제로 명령어를 통해 출력한 결과는 아래와 같다.

```bash
[user@server ~/sources/git-hash]$ git cat-file commit HEAD
tree db78f3594ec0683f5d857ef731df0d860f14f2b2
parent c612c9316c74c3f7489135e18545a2082e5ebd0e
author Seung-Hun Han <dreamsh19@gmail.com> 1730020485 +0900
committer Seung-Hun Han <dreamsh19@gmail.com> 1730020485 +0900

Add Hello World to README
```

크게 5개(tree, parent, author, committer, commit message)로 구성된 것을 확인할 수 있다. 각각에 대해 살펴보자.

### tree

```
tree db78f3594ec0683f5d857ef731df0d860f14f2b2
```

해당 커밋 상태에서의 모든 파일에 대한 스냅샷(으로 부터 추출된 hash값)이다.
참고로, git에서의 tree는 git 내부적으로 사용하는 자체 파일 시스템 정도로 이해할 수 있다. git에서 자체 파일 시스템이 필요한 이유는 버저닝에 특화되어 있어야하고, 특정 파일 시스템에 종속되지 않아야하는 등의 요구사항이 필요하기 때문이다.

결국 파일의 내용(blob)이 변경되면 hash값이 바뀐다. 정도로 이해하면 될 것 같고, 커밋의 고유성을 결정할때 가장 직관적인 기준이다. 그리고 tree hash 외의 다른 4개의 항목들은 모두 commit 자체에 대한 메타데이터인 반면, 파일의 내용에 대한 항목은 tree hash가 유일하다.

하지만 예상하겠지만, 파일의 내용만으로는 커밋의 고유성을 보장할 수 없다. (나머지 4개 항목이 필요한 이유이기도 하다.)

간단한 예로, 커밋 A를 수행하고 바로 A를 revert하는 경우를 떠올려보면, 파일의 내용은 달라진게 없지만 2개의 추가적인 커밋이 발생했고, git에서는 이를 구별해야한다.
실제로 확인해보자. README에 "Mistakes" 문구를 추가했다가 곧바로 revert하고 git cat-file commit 명령어를 수행해보았다.

```bash
[user@server ~/sources/git-hash]$ echo "Mistakes" >> README.md
[user@server ~/sources/git-hash]$ git add . && git commit -m "Made a mistake"
[main 1f002c2] Made a mistake
 1 file changed, 1 insertion(+)
[user@server ~/sources/git-hash]$ git revert HEAD
[main 3b9a0c2] Revert "Made a mistake"
 1 file changed, 1 deletion(-)
Revert "Made a mistake"

This reverts commit 1f002c2bf3f8f779f73db79a88cb7575a77eab91.
[user@server ~/sources/git-hash]$ git cat-file commit HEAD
tree db78f3594ec0683f5d857ef731df0d860f14f2b2
parent 1f002c2bf3f8f779f73db79a88cb7575a77eab91
author Seung-Hun Han <dreamsh19@gmail.com> 1730022719 +0900
committer Seung-Hun Han <dreamsh19@gmail.com> 1730022719 +0900

Revert "Made a mistake"

This reverts commit 1f002c2bf3f8f779f73db79a88cb7575a77eab91.
```

여기서 확인할 수 있는 것은, 2개의 커밋이 발생했음에도 마지막 명령어 결과에서 tree hash값은 `db78f35...`으로 동일한 것을 확인할 수 있다.

### parent

```
parent c612c9316c74c3f7489135e18545a2082e5ebd0e
```

부모 커밋의 hash값이다.
커밋의 parent 정보는 다른 커밋과의 관계를 정의하는 유일한 정보이기 때문에, 커밋의 고유성을 결정하기에 필수적인 정보이다.
실제로 같은 tree hash를 가지는 서로 다른 커밋이 있다고 하더라도, 부모 커밋이 다르다면 전혀 다른 커밋이 된다. 당장 위의 revert 예시에서도 동일한 tree hash지만 parent 정보가 다름으로 인해 구별할 수 있다.

### author, committer

```
author Seung-Hun Han <dreamsh19@gmail.com> 1730020485 +0900
committer Seung-Hun Han <dreamsh19@gmail.com> 1730020485 +0900
```

author와 committer의 정보와 시간 정보가 담겨 있다.
(author와 committer의 차이에 대해서는 자세히 다루지 않겠다. 궁금하다면 [Git - Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History) 를 참고하면 된다.)
git은 기본적으로 협업을 위한 프로그램이기 때문에 author, committer 정보가 중요할 수 밖에 없다. 같은 부모 커밋에서 같은 내용을 커밋하더라도 수행하는 사람이 다르다면, 서로 다른 커밋으로 취급한다.

개인적으로 author와 commiter 정보에서 특이하다고 느낀 점은 두가지인데, 첫번째는 시간의 정밀도가 초단위까지밖에 없다는 것이고, 두번째는 타임존 정보를 포함하고 있다는 것이다.

#### 1. 초단위 정밀도
시간 정보는 Unix timestamp로 표현되어있는데, `1730020485`로 초단위 정밀도이다.
그러면 자연스럽게 "1초 내에 동일한 커밋을 두 번하면 어떻게 되는거지?" 라는 생각이 든다.
그래서 직접 해보았다. main을 부모 커밋으로 가지는 서로 다른 두개의 브랜치를 만들고 각 브랜치에서 똑같은 커밋을 1초 내로 수행해보았다.

```bash
for i in {1..2}; do 
	git checkout main;
	git checkout -b branch-${i};
	touch file.tmp;
	git add . && git commit -m "Create file.tmp"; 
done
```

결과는 놀랍게도 동일한 커밋으로 취급되었다. (실제로 커밋은 두 번 발생했음에도)

![image](https://github.com/user-attachments/assets/4a9a110b-1bc8-4368-b6a0-60d1a106755c)

그리고 위 명령어 마지막에 `sleep 1;` 을 추가하여 확인해보니 서로 다른 커밋으로 취급되었다.

그리고 초단위 정밀도를 가지는 걸 이상하게 생각한 것은 나 뿐만이 아닌듯 하다.
[What is the resolution of Git's commit-date or author-date timestamps? - Stack Overflow](https://stackoverflow.com/a/28238046)
위 링크에서 보면, 실제로 같은 커밋을 1초 내에 두번하면 하나의 커밋으로 취급된다는 내용이 있다.

#### 2. 타임존 정보

위에서 살펴보았듯이 시간 정보는 Unix timestamp로 저장되고, Unix timestamp는 타임존에 독립적인데, 타임존 정보가 포함된 것이 의아했다.
처음에는 "타임존 정보가 굳이 포함될 필요가 있을까?"라는 관점에서 의아했으나, 애초에 커밋 정보에 타임존 정보를 포함하고 있으니, 굳이 제외하지 않았다 정도의 관점으로 받아들이면 이상할 것도 없을 듯하다.
물론, 동일한 커밋을 같은 순간에(1초 내) 서로 다른 타임존에서 수행하면 다른 커밋 hash가 도출되고, 서로 다른 커밋으로 간주된다.(아래는 실험 결과)

```bash
for i in {3..4}; do 
	git checkout main;
	git checkout -b branch-${i};
	touch file.tmp;
	git add .;
	GIT_COMMITTER_DATE="$(TZ=UTC+${i} date -R)" git commit -m "Create file.tmp";
done
```

![image](https://github.com/user-attachments/assets/99329a35-b19b-451e-8a7e-488e436a785a)

### commit message

```
Add Hello World to README
```

커밋의 "이름" 정도라고 보면, 직관적으로는 커밋의 고유성을 위한 key로 채택함에 있어서 큰 무리가 없을 듯 하다.

다만, commit message를 포함하는데 있어서 개인적인 생각을 덧붙여보자면,
위의 네 가지 항목들은 커밋을 했을때 git에 의해 자동으로 결정되는 것들이지, "개발자가 명시적으로" 커밋의 고유성을 위해 지정할 수 있는 항목들은 아니다. (커밋 시간 등을 조정하면 할 수야 있지만 시간을 조정해서 고유성을 확보하는 것이 자연스럽진 않다고 생각이 된다.)
따라서 commit message는 개발자가 명시적으로 커밋의 고유성을 부여하기 위한 수단으로 볼 수 있다.
정확한 비유는 아니지만 일란성 쌍둥이지만 이름으로 구별할수 있게 하겠다.. 정도로 생각했다. (일란성 쌍둥이지만 태어난 시간으로 구별하는 건 자연스럽지 않으니..)

## 헤더 정보

위에서 `git cat-file commit HEAD` 명령어에 대해서는 자세히 살펴보았고, 이제 commit의 고유성을 결정하는 5가지 요소에 대해서는 모두 알게 되었다.
그렇다면 글 도입부 `printf` 파트에서 보면 "commit 바이트수\\0"를 포함하고 있는데, 이 정보는 무엇일까?

commit도 object 중 하나라는 점에서 눈치챘을 수도 있지만,
사실 git은 모든 것을 object 라는 개념으로 관리하고, 이 object의 id 체계로 hash를 활용하고 있다. commit hash도 이 중 하나일 뿐이다. (위에서 tree가 hash 값을 가지는 것도 같은 이유이다.)
그리고 object의 id를 생성할때 다음과 같이 일관된 형태로 해싱 함수의 input을 구성한다.

```
<type> <size>\0<content>
```

이때 `<content>`를 제외한 `<type> <size>\0` 부분을 git에서는 헤더라고 한다.

실제 git의 소스 코드 [git/object-file.c at v2.47.0 · git/git · GitHub](https://github.com/git/git/blob/v2.47.0/object-file.c#L1140-L1155)를 살펴보면 아래와 같은 코드로 헤더를 생성함을 확인할 수 있다.
(아래는 2.47.0 버전 상의 소스 코드이나, 이전 버전에서도 코드의 형태는 조금씩 다르지만 동일한 로직을 가진다.)
![image](https://github.com/user-attachments/assets/fd51472a-20d5-46a6-b032-e6271480e061)

결국 `<type> <size>`의 형태로 포맷팅하고, 마지막의 +1을 통해 null character를 할당하는 로직이다.
여기서의 type은 object의 타입이며 글의 도입부에 `commit`이라는 고정 문자열이 입력된 것도 `commit` 타입임을 명시하는 것이다.
그리고 size는 content의 바이트 수이다. 사실 content에 종속된 정보라 불필요한 정보라고 생각될 수 있지만, 파일 시스템 상의 오류로 content 상의 불일치가 발생한 경우, 이를 탐지하기 위한 장치이다.

## SHA1 해싱

여기까지 하면, hash 함수의 input에 대해서는 모두 살펴보았고, 결국 해당 input에 SHA1 해싱을 통해 최종적인 hash값을 도출해낸다.
그런데 SHA1 알고리즘은 컴퓨팅 파워의 증가로, 보안 취약점을 갖고 있는 알고리즘이 되어버렸고, 해시 충돌로 인한 위험성이 있는 것으로 밝혀졌다.

Git에서도 이를 인지하고 있고, 더 안전한 SHA256으로의 전환을 점진적으로 추진한다고 한다.
그리고 기존 SHA1 시스템에서의 해시 충돌 문제를 방지하기 위해 Github에서는 해시 충돌이 발생한 경우 이를 탐지하고 실패처리하는 것을 이미 적용하고 있다. 실제로 Git 프로젝트를 보면 해시 충돌 방지에 대한 [소스](https://github.com/cr-marcstevens/sha1collisiondetection/)를 submodule로 관리하고 있음을 확인할 수 있다.

### Hash 충돌이 문제가 되는 이유

앞서 언급했듯이, git에서 hash값은 곧 id이다. id 체계는 유니크함을 보장하는 것이 필수적이나, 해시 충돌이 발생한다면 유니크함이 보장되지 않게 된다.

단적인 예로, 파일 A의 해시값이 abcde라고 하고. 악의적인 사용자가 abcde의 해시값을 도출해내는 다른 파일 B를 찾아낸 상황을 가정하자. 악의적인 사용자는 B 기반의 코드로 문제가 없음을 증명한 후 main 브랜치에 머지를 하려고 하면, git은 A와 B가 다르다는 걸 인지하지 못하게 된다. 결국 별 문제 없이 코드가 머지될 수 있고, 최종적으로 저장소에는 머지가 됐음에도 여전히 파일 A로 남아있게 되고, 동작하지 않는 전혀 다른 코드가 되어버린다.

물론, 아주 단적인 예일 뿐이고, 애초에 id 체계의 근간을 흔들 수 있기 때문에 많은 문제가 발생할 수 있다.

## 요약

내용이 길었지만 결국 commit hash 만드는 과정을 요약하면 아래와 같다.

- git의 commit의 hash는 5가지 요소(파일 내용의 스냅샷, 부모 커밋, author, committer 정보, 커밋 메시지)에 의해 결정된다.
- 그리고 추가적인 헤더 정보(object 타입, 바이트 수)를 추가하며, 이는 commit에 한정된 것은 아니고 object 모두에 해당된다.
- 위의 결과에 SHA1 해싱을 적용하여 최종적인 값으로 채택한다.

## References

- [How is git commit sha1 formed · GitHub](https://gist.github.com/masak/2415865)
- [Git - git-cat-file Documentation](https://git-scm.com/docs/git-cat-file)
- [GitHub - git/git: Git Source Code Mirror - This is a publish-only repository but pull requests can be turned into patches to the mailing list via GitGitGadget (https://gitgitgadget.github.io/). Please follow Documentation/SubmittingPatches procedure for any of your improvements.](https://github.com/git/git)
- [Git - Viewing the Commit History](https://git-scm.com/book/en/v2/Git-Basics-Viewing-the-Commit-History)
- [What is the resolution of Git's commit-date or author-date timestamps? - Stack Overflow](https://stackoverflow.com/a/28238046)
- [Why do git objects include a length and a delimiter as metadata? - Stack Overflow](https://stackoverflow.com/a/70266188)
- [Git의 commit id는 어떻게 생성될까?](https://noah-ios.dev/how-to-generate-commit-id/)
- [SHA-1 collision detection on GitHub.com - The GitHub Blog](https://github.blog/news-insights/company-news/sha-1-collision-detection-on-github-com/)
- [Git Transitioning Away from the Aging SHA-1 Hash - The New Stack](https://thenewstack.io/git-transitioning-away-from-the-aging-sha-1-hash/)
