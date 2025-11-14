# git push

```
Pushing to ...
To ...
! [rejected]        lg-master -> lg-master (non-fast-forward)
error: failed to push some refs to 'http://soominlee@smartdev.lgcns.com/bitbucket/scm/paas/git-test.git'
hint: Updates were rejected because the tip of your current branch is behind
hint: its remote counterpart. Integrate the remote changes (e.g.
hint: 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.
```
당황하지 말고 자세히 봅시다.

error: failed to push 

으응... push 에 실패했네.

hint: Updates were rejected because the tip of your current branch is behind its remote conterpart. 

너의 현재 브렌치가 원격의 다른 사람보다 뒤에 있기 때문에 실패한거야... 으응?

hint: Integrate the remote changes (e.g. 'git pull ...') before pushing again.

원격 변경 사항을 통합하고 다시 해 보시라 (예를 들어 git pull 로)

```
hint: Updates were rejected because the remote contains work that you do
hint: not have locally. This is usually caused by another repository pushing
hint: to the same ref. You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.
hint: See the 'Note about fast-forwards' in 'git push --help' for details.

error: You have not concluded your merge (MERGE_HEAD exists).
hint: Please, commit your changes before merging.
fatal: Exiting because of unfinished merge.
```