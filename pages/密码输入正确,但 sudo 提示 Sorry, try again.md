- 有时候 [[sudo]] 输错次数多了，[[faillock]] 会自动锁定帐户，这时候不管密码输的对不对都会报错
- ```shell
  :( sudo ls
  [sudo] password for xuanwo:
  Sorry, try again.
  ```
- 此时用 [[faillock]] 查看帐号锁定状态
- ```shell
  > faillock
  xuanwo:
  When                Type  Source                                           Valid
  2022-01-07 15:50:40 TTY   /dev/pts/4                                           V
  2022-01-07 15:50:43 TTY   /dev/pts/4                                           V
  2022-01-07 15:50:47 TTY   /dev/pts/4                                           V
  ```
- 可以使用 `--reset` 参数来尝试重置
- ```shell
  faillock --user xuanwo --reset
  ```
-