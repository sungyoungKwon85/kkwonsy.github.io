### 출처 
- https://velog.io/@creco/ssh-git-remote


#### 키젠 생성하기
```
$ ssh-keygen -t rsa -b 4096 -C "이메일" -f ~/.ssh/id_rsa-계정
```
> 그냥 enter 만 2번   

#### 키 복사 
```
$ cat ~/.ssh/id_rsa-계정.pub
```
> github.com -> settings -> ssh 추가  


#### 로컬 컴퓨터에 ssh 커스텀 Host 추가
```
# .ssh/config
Host customgit
  Hostname github.com
  IdentityFile ~/.ssh/id_rsa-계정
  IdentitiesOnly yes
```

#### clone 할 때
```
$ git clone git@customgit:계정/레포.git
```