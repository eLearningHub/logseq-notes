- 调试工具
	- xfs_io
		- 使用 O_DIRECT 打开文件
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