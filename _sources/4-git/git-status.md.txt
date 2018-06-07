# git status

## Your branch is behind 'origin/master' by 5 commits, and can be fast-forwarded.

```
$ git fetch
$ git status
On branch master
Your branch is behind 'origin/master' by 5 commits, and can be fast-forwarded.
  (use "git pull" to update your local branch)
nothing to commit, working tree clean
```

내 브랜치 master 가 원격지 저장소인 origin/master 보다 5 커밋 뒤쳐저있다 (is behind by). 

내가 마지막 push 한 이후 원격 저장소를 다른 개발자가 새로운 push 를 날린 경우이다. git pull 을 하면 내 로컬 워크스페이스가 최신 commit 으로 대체된다 (fast-forward). 
