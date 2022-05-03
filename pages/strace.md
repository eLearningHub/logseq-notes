- https://strace.io/
- [[linux]] syscall tracer
	- 用来查看所有的 syscall 调用
-
- 指定命令
	- ```shell
	  :) strace pwd
	  execve("/usr/bin/pwd", ["pwd"], 0x7ffed491ab10 /* 52 vars */) = 0
	  ....
	  close(1)                                = 0
	  close(2)                                = 0
	  exit_group(0)                           = ?
	  +++ exited with 0 +++
	  ```
- 指定进程号
	- ```shell
	  :) strace -p 26380
	  strace: Process 26380 attached
	  ...
	  ```
- 过滤指定的路径
	- ```shell
	  :) strace -P /etc/ld.so.cache ls /var/empty
	  open("/etc/ld.so.cache", O_RDONLY) = 3
	  fstat(3, {st_mode=S_IFREG|0644, st_size=22446, ...}) = 0
	  mmap(NULL, 22446, PROT_READ, MAP_PRIVATE, 3, 0) = 0x2b7ac2ba9000
	  close(3) = 0
	  +++ exited with 0 +++
	  ```
-