- 读取一个 buffer 而不是完整的 size
-
- 可能的实现
	- 记录一下 reader 消费了多少数据
	- 在消费全部完毕后，去尝试开启一个新的 read 而不是返回 0
		- 同时也不要每次都去尝试开新的 read
-
- 其他的考虑
	- 有没有可能跟现有的实现共存呢？
		- 什么时候 read current size 更好，什么时候读一个 buf 更好呢？
		- 加一个 mode？
			- 用户需要表明他计划顺序读完还是只读其中的一部分
-
- fs
	- 优化前
		- ```rust
		  fs/read_exact/e2454b3b-b1e5-4ff3-9af6-f00d828488bd
		                          time:   [861.60 us 870.29 us 879.62 us]
		                          thrpt:  [4.4408 GiB/s 4.4885 GiB/s 4.5337 GiB/s]
		  
		  ```
	- 优化后
		- ```rust
		  fs/read_exact/783083d3-1fed-42f1-be06-cebdc3b9fcc9
		                          time:   [757.74 us 803.31 us 846.02 us]
		                          thrpt:  [4.6172 GiB/s 4.8627 GiB/s 5.1552 GiB/s]
		  
		  ```
- s3 (local minio)
	- 优化前
		- ```rust
		  s3/read_exact/3b9c4e39-8e2d-4531-b1c4-9e7290e26a4c
		                          time:   [1.8896 ms 1.9370 ms 1.9880 ms]
		                          thrpt:  [1.9649 GiB/s 2.0167 GiB/s 2.0673 GiB/s]
		  
		  ```
	- 优化后
		- ```rust
		  s3/read_exact/f34bf150-0a5c-45e6-886c-9e76b21dd91f
		                          time:   [1.7521 ms 1.7995 ms 1.8513 ms]
		                          thrpt:  [2.1100 GiB/s 2.1707 GiB/s 2.2294 GiB/s]
		  ```
-
- 看起来优化效果不是很明显，看一下真实的环境