# SSH key
## 키 생성 타입
- 키 생성 타입을 정해줄 수 있다. `ssh-keygen -t rsa` 로 선택하면 rsa 타입으로 만들어준다. 가끔 SHA 1 방식이 기본값인 경우는 옛날 방식이라고 256이나 rsa 사용하라고 말하는데 그럴때 이용하면 될 것 같다.
## 원하는 키 이름
- ssh-keygen 을 사용하면 기본으로 ~/.ssh/id_rsa(.pub) 키 쌍을 만들어 준다. 원하는 다른 키 파일 이름이 있으면 명령어를 사용하고 단계에 맞춰서 적어주면 된다.(해보면 안다 ssh-keygen을 실행하자!)
### 그런데
- 임의로 생성한 키 값은 자동으로 등록이 안된다. ~/.ssh/config 에서 github.com 에서는 이걸 쓰라고 등록해줬는데 source 실행시켰어야 하나??
### 디버깅
```sh
GIT_SSH_COMMAND="ssh -v" git push
```
- 이렇게 실행시켰더니 무슨 키 파일을 사용하는지 다 보여준다. ssh의 디버깅 옵션이라 생각하면 될듯.
- 그랬더니 id_rsa만 참고하길래, 아래 명령어로 키를 등록했다.
    ```sh
    ssh-add ~/.ssh/github_rsa
    ```
- 정상 작동한다. 아래는 등록 전 후의 디버깅 비교이다. Will attempt key가 추가된 것을 볼 수 있다.

```
debug1: Will attempt key: /Users/dongbin/.ssh/id_rsa RSA SHA256:{secret}
debug1: Will attempt key: /Users/dongbin/.ssh/id_ecdsa 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ecdsa_sk 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ed25519 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ed25519_sk 
debug1: Will attempt key: /Users/dongbin/.ssh/id_xmss 
debug1: Will attempt key: /Users/dongbin/.ssh/id_dsa 
```
```
debug1: Will attempt key: dongbin@{secret} RSA SHA256:{secret} agent
debug1: Will attempt key: /Users/dongbin/.ssh/id_rsa RSA SHA256:{secret}
debug1: Will attempt key: /Users/dongbin/.ssh/id_ecdsa 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ecdsa_sk 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ed25519 
debug1: Will attempt key: /Users/dongbin/.ssh/id_ed25519_sk 
debug1: Will attempt key: /Users/dongbin/.ssh/id_xmss 
debug1: Will attempt key: /Users/dongbin/.ssh/id_dsa 
```