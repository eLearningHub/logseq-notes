type:: [[Product]]

- Smart Sync 实现
	- 所有的文件会有两种状态: online-only vs local
	- Smart Sync 会维护完整的 metadata 数据，用于展示
	- 当用户尝试访问某个文件时候，smart sync 自动地将这个文件设置为 local
	- 然后 smart sync 会有类似生命周期的概念，把很久没有打开过的数据再变成 online-only
	- 看起来要做的话需要从 fuse 层入手？
-
- 忽略特定文件
	- ```shell
	  attr -s com.dropbox.ignored -V 1 /path
	  ```
	- dropbox 会根据文件的 attr 来忽略同步
	- 参考资料
		- [How to set a Dropbox file or folder to be ignored](https://help.dropbox.com/files-folders/restore-delete/ignored-files)