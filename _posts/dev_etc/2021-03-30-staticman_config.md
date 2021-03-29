---
title: staticman의 설정 방법
layout: single
categories:
  - dev_etc
tags:
  - docker
  - heroku
  - staticman
  - jekyll
language: KR
excerpt: staticman 설정 과정 작업 로그
---

### 현재의 staticman 설정

결론부터 말하면, 현재 heroku에서 돌아가는 staticman의 리포지토리에  
config.production.json은 존재하지 않으며,  
인증을 위한 몇 개의 설정만 환경변수로 설정 하고 있다.


### 공식 문서에 의하면...

[Staticman Getting Started](https://staticman.net/docs/getting-started.html)에 의하면

> Read through the Staticman API config values and note the config values you wish to use.  
> At a minimum, you must include a way for Staticman to auth with a git provider, as well as an RSA private key.

production에서 쓸 config파일을 생성하고
```bash
$ cp config.test.json config.production.json
```
자기한테 맞춰서 설정하라고 하는데......🤔


### 결국에는 비어있는 파일

[Staticman Github의 샘플파일](https://github.com/eduardoboucas/staticman/blob/master/config.sample.json)을 열어보면
```json
{
  "gitlabToken": "YOUR_GITLAB_TOKEN",
  "githubToken": "YOUR_GITHUB_TOKEN",
  "rsaPrivateKey": "-----BEGIN RSA PRIVATE KEY-----YOUR_KEY-----END RSA PRIVATE KEY-----",
  "port": 80
}
```
일단 블로그가 올라가있는 리포지토리와의 연동을 위한 인증 관련 설정, 내용 암호화하는 rsa private key는 필수라는 것 같은데,  
github에 token이나 private key를 리포에 올리는 대가리 깨진짓은 하고싶지도 않고,  
직접 굴리는 서버도 아니니 포트넘버를 설정하는것도 그닥 의미가 없다.

결국 config.production.json의 내용은 아무것도 없어질 뿐 더러,  
[staticman의 API Configuration](https://staticman.net/docs/api)을 참고해도 앤간한건 환경변수로 커버가 가능하다.  
모처럼 heroku쓰는데 굳이 config에다가 적을 필요가 있을까?

필요없는 파일 그냥 지워버리고 싶어서 config.js를 뜯어보니
```javascript
try {
  config = convict(schema)

  const fileName = 'config.' + config.get('env') + '.json'

  config.loadFile(path.join(__dirname, fileName))
  config.validate()

  console.log('(*) Local config file loaded')
} catch (e) {

}
```
convict에서 file not exist가 떠도 그냥 무시하게 되어있다. json파일도 삭제.


### rsaPrivateKey가 뭔데?
공식문서에서도 밝혔듯이, rsaPrivateKey는 필수이다.  
이 값을 설정 안하면 heroku에서도 에러를 뱉어낸다.

github/gitlab이랑 연동시키는데 쓰는 것도 아니고, 대체 이게 뭔가 싶어서 뒤져보니

```javascript
// lib/RSA.js
key.importKey(config.get('rsaPrivateKey'), 'private')

module.exports.encrypt = text => {
  try {
    const encryptedText = key.encrypt(text, 'base64')

    return encryptedText
  } catch (err) {
    return null
  }
}
```

encrypt하는데 쓰이는건 알겠는데..

```javascript
// contollers/encrypt.js
module.exports = (req, res) => {
  const encryptedText = RSA.encrypt(req.params.text)
...

// controllers/auth.js
module.exports = async (req, res) => {
  ...
  return getUser
        .then((user) => {
          res.send({
            accessToken: RSA.encrypt(accessToken),
            user
          })
        })
```

text랑 accessToken 암호화..

```javascript
// server.js
this.server.get(
  '/v:version/encrypt/:text',
  this.bruteforce.prevent,
  this.requireApiVersion([2, 3]),
  this.controllers.encrypt
)
...
this.server.get(
  '/v:version/auth/:service/:username/:repository/:branch/:property',
  this.bruteforce.prevent,
  this.requireApiVersion([2, 3]),
  this.requireService(['github', 'gitlab']),
  this.controllers.auth
)
```
결국에는 staticman 혼자 뭐 할 때 encrypt하는데만 쓰인다.  
어디랑 페어를 맞출 필요는 없고, 차후에 어딘가에 공개키를 요구하는 일도 없어보이므로 대충 설정해도 될듯.


### github과의 연동?

staticman은 내 블로그의 소스코드에 답글을 직접 쓰는 방식이다.  
내 github 블로그용 repo에 수정가능하도록 연동을 시켜야 한다.

1. github application으로써 등록
2. staticman용 봇 계정을 생성해서 그 봇을 collaborator로 등록
3. staticman이 내 계정 쓸수있게 인증정보 넘기기

세가지 선택지가 있다.  
staticman에서도 1번 방법을 추천하고 있으며,  
개인적으로도 귀찮게 계정을 새로 파고싶지도 않으며  
(heroku client 인스톨도 싫은데 이거땜에 계정을 파?)  
내 계정 허벌창 만들고 싶지도 않다.


1번 방식으로 하기로 하고, 인증을 어떻게 할지 github토큰관련 코드를 뜯어보니...
```javascript
  const isAppAuth = config.get('githubAppID') &&
    config.get('githubPrivateKey')
  const isLegacyAuth = config.get('githubToken') &&
    ['1', '2'].includes(options.version)

  let authToken

  if (options.oauthToken) {
    authToken = options.oauthToken
  } else if (isLegacyAuth) {
    authToken = config.get('githubToken')
  } else if (isAppAuth) {
    authToken = await this._authenticate(options.username, options.repository)
  } else {
    throw new Error('Require an `oauthToken` or `token` option')
  }
```

githubtoken은 이름부터 legacy라 왠지 싫고, 저게 뭔 원리인지도 잘 모르겠다.

appId하고 privatekey를 주는게 어디에 키를 저장해둘 필요도 없고, 제일 귀찮은일 없어보인다.  


### config는 heroku의 환경변수로 끝

heroku의 앱 설정에서, 환경변수에 `GITHUB_APP_ID`,`GITHUB_PRIVATE_KEY`,`RSA_PRIVATE_KEY`  
이 세가지만 등록하면 설정은 끝.  

reCapcha같은 추가 기능과 연동도, [staticman의 API Configuration](https://staticman.net/docs/api)를 보면 환경변수만으로 떡을 치고도 남는다.
