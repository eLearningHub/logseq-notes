type:: [[Product]]
features:: [[Object Storage]]

-
- 使用技巧
  id:: 624195bf-947a-4bc1-821b-1b2ce9c3381d
	- 启动并指定端口
		- ```shell
		  minio server . --address ":9900"
		  ```
		- 当前目录下所有的目录都会视为一个 bucket，`mkdir` 即可创建
	- 配置本地的 alias
		- minio 的命令行工具为了支持管理多个 profile，增加了 alias 的支持
		- 可以通过如下参数将本地的 minio alias 为 local
			- ```shell
			  mcli alias set local http://127.0.0.1:9900 minioadmin minioadmin
			  ```
	- 查看 minio server 端的 tracing
		- 开发的时候经常需要查看服务器端收到了怎样的请求
		- 可以通过 minio 的命令行工具查看
			- ```shell
			  mcli admin trace local
			  ```
			- 增加 debug 参数可以看到每个请求的详细信息
				- ```shell
				  mcli admin trace local --debug -v
				  ```
	- 将 bucket 配置为 anonymous 可读
		- minio 对 AWS S3 的 acl & policy 的实现不是很完整
		- 将指定 bucket 设置为匿名可读需要使用内部的 API
		- ```shell
		  mcli anonymous set public local/opendal
		  ```
-