-
-
- 调试工具
	- xfs_io
		- debug the I/O path of an XFS filesystem
			- 可以用来直接测试各种 IO 操作，比如 open，read，pread，seek，readdir 等等
		- 打开文件
			- `xfs_io -c open <options> <path/to/file>`
				- -a  opens append-only (O_APPEND).
				- -d  opens for direct I/O (O_DIRECT).
				- -f  creates the file if it doesn't already exist (O_CREAT).
				- -r  opens read-only (O_RDONLY).
				- -t  truncates on open (O_TRUNC).
				- -n  opens in non-blocking mode if possible (O_NONBLOCK).
			- ```shell
			  :( xfs_io -c open -d ./tests/sql/tpch-full/_q1.slt
			  fd.path = "./tests/sql/tpch-full/_q1.slt"
			  fd.flags = non-sync,direct,read-write
			  stat.ino = 7206412
			  stat.type = regular file
			  stat.size = 1233
			  stat.blocks = 8
			  fsxattr.xflags = 0x80000000 [----------------X]
			  fsxattr.projid = 0
			  fsxattr.extsize = 0
			  fsxattr.cowextsize = 0
			  fsxattr.nextents = 1
			  fsxattr.naextents = 0
			  dioattr.mem = 0x200
			  dioattr.miniosz = 512
			  dioattr.maxiosz = 2147483136
			  ```
		- 参考资料
			- [xfs_io(8) — Linux manual page](https://man7.org/linux/man-pages/man8/xfs_io.8.html)