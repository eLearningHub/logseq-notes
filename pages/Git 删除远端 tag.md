- ```shell
  git tag -d xxx
  git push origin :refs/tags/xxx
  ```
- 首先需要删除本地 tag，然后将空的 tag ref 推送到远端，这样就能够实现删除远端 tag
- 注意这是一个不可逆的危险操作，请二次确认。