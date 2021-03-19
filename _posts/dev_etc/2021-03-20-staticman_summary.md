---
title: staticman을 이용한 답글 기능 추가
layout: single
categories:
  - dev_etc
tags:
  - docker
  - heroku
  - staticman
  - jekyll
language: KR
---

## 왜 Staticman?

'jekyll 댓글'로 구글링해보면, 대부분이 Disqus를 이용하고 있다.  
`<script>`하나면 설치가 가능해서 라는데....

흔히 보이는 디자인이 뭔가 맘에 안든다.  
그보다도, 이 답글들이 어디로 저장될지 모른다는 점이 맘에 안든다.

그래서 선택한게 staticman. 그 이유로는
1. 답글 내용이 전부 블로그의 github 리포지토리에 저장된다. 말그대로 static.
2. Docker를 이용한다. Docker 한번 써보고 싶었는데 잘됐다.
3. 남들이 안쓴다.

## staticman을 Heroku에 올리기
staticman의 문서를 보면, heroku내의 staticman을 fork해서, 바로 쓰는방법도 있지만,  
heroku client를 설치해야한다.

docker를 써보고 싶은 것 뿐이지 heroku를 딱히 공부 할 생각도 없으므로,  
heroku에 github연동 후 docker 이미지를 올리는 방법을 선택했다.

일단 [staticman github repo](https://github.com/eduardoboucas/staticman)를 clone
```
$ git clone git@github.com:eduardoboucas/staticman.git
```
대충 훑어보고 지울거 지우고 새 repo를 만들고 push한 뒤,  
heroku에서 새 app을 만들고, github에 연동.
![heroku settings]({{ site.url }}{{ site.baseurl }}/assets/images/heroku_github_staticman.png)

아직은 필수 세팅이 안되있으므로 안돌아가는 상태.

## github에 app추가
staticman은 heroku에서 돌아가면서 내 블로그의 repo에 답글 내용을 직접 추가시킨다.  
이 과정에서 push/commit을 위해서 github쪽에다가 app으로써 추가해줘야 한다.

app 추가방법은 github문서와 staticman공식 문서를 참고.  
[GitHub Docs](https://docs.github.com/en/developers/apps/creating-a-github-app)
[Staticman Getting Started](https://staticman.net/docs/getting-started.html)

> Homepage: "https://staticman.net/"
> Webhook URL: "{STATICMAN_BASE_URL}/v1/webhook" - e.x. "https://mystaticmaninstance.herokuapp.com/v1/webhook"
> Contents: Read & Write - Necessary to read the Staticman site config
> Pull Requests: Read & Write - Necessary to merge pull requests
> Subscribe to Pull request events

앱 생성 후 Edit창에서 Private Key를 생성하면, pem파일을 다운로드 한다.

## heroku의 환경변수 세팅
staticman을 쓰기 위해서는, 이하의 값을 세팅해줘야 한다.
* 전송 내용을 암호화 할 RSA Private key(필수)
* Github에 등록한 App하고 연동 할 값

블로그와 어떻게 연동하는가에 따라 달라지지만, Github app으로써 연동시켰으므로  
Github App Id, App의 Private Key를 등록하면 된다.

이 값들은 staticman의 config file에 적어도 되지만,  
private key를 github에 올리는건 거부감이 있다.

staticman은 환경변수로 설정값을 읽어오게끔 되어있으므로, heroku의 환경변수로 세팅하는 쪽으로.

[staticman api](https://staticman.net/docs/api)
![heroku env settings]({{ site.url }}{{ site.baseurl }}/assets/images/heroku_github_staticman.png)

* GITHUB_APP_ID: github에 등록한 app의 app id
* GITHUB_PRIVATE_KEY: github app에서 private key생성할 때 받은 pem파일의 내용
* RSA_PRIVATE_KEY: staticman이 내용을 암호화 할 때 쓰는 키. openssl로 생성한 rsa를 이용

세팅이 제대로 되면, heroku의 open app을 클릭하면 다음과 같은 메시지가 나온다.
```
Hello from Staticman version 3.0.0!
```

## staticman세팅
blog에서 staticman을 추가시켜야 한다.  
staticman.yml파일을 blog의 root디렉토리에 넣고,  
_config.yml에서 staticman을 쓰도록 설정해줌.  

설정방법은 사용하는 jekyll의 테마 등에 따라 다르지만,  
이 블로그에서 사용하고 있는 minimal mistakes는 이미 staticman과 연동하도록 되어있으므로, 조금 건드려준다.  
현재 설정은 다음과 같다.

```yml
# staticman.yml
comments:
  allowedFields: ["name", "email", "message"]
  allowedOrigins: ["localhost", "lastruggling.github.io"]
  branch: "master"
  commitMessage: "Add new comment via staticman"
  filename: "comment-{@timestamp}"
  format: "yml"
  generatedFields:
    date:
      type: date
      options:
        format: "iso8601"
  moderation: false
  name: "lastruggling.github.io"
  path: "_data/comments/{options.slug}"
  requiredFields: ["name", "message"]
  transforms:
    email: md5

# _config.yml
...
repository               : "lastruggling/lastruggling.github.io"
...
comments:
  provider               : "staticman_v2"
  staticman:
    branch               : "master"
    endpoint             : "https://lastruggling-staticman.herokuapp.com/v3/entry/github/"
...
defaults:
  # _posts
  - scope:
      path: ""
      type: posts
    values:
      ...
      comments: true
      ...
```

## 디자인 변경
답글 창이 영 맘에 안든다.  
메일 주소나 url주소를 받을 의향도 없고, 이미지를 띄울 의향도 없으므로  
jekyll의 프론트 쪽(minimal mistakes의 comment layout파일)을 대충 수정해준다.

수정 전
![before modify]({{ site.url }}{{ site.baseurl }}/assets/images/comment_before.png)

수정 후
![after modify]({{ site.url }}{{ site.baseurl }}/assets/images/comment_modified.png)

## 끝
이런 단점이 존재한다.
* heroku가 무료이므로, idle상태에선 시간이 조금 걸린다
* heroku에서 블로그 repo로 push해도, github내부에서 jekyll을 빌드하므로 시간이 조금 걸린다

대충 반영까지 1분 정도 잡야아 하는듯 하다.
어차피 답글 쓸 사람도 없을테니 신경 쓰지 말아야지.

각 단계의 상세는 기분 내키면 써야지.(귀찮아)