title:: 2022-09: Iteration 8 汇报以及聊聊外卷

- [[Iteration/8]]
-
- 这个 Iteration 的主要工作仍然是围绕着 opendal 展开，有三件事情比较有意思。
- 首先是多读了部分数据导致读取性能下降的 BUG: [Improvement: Avoid reading unnecessary data](https://github.com/datafuselabs/opendal/issues/86)。
	- Databend 在 benchmark 的过程中发现 opendal 会读取比预想的要更多的数据
	- ```rust
	  let r = o.reader();
	  let buf = vec[0;4*1024*1024];
	  r.read_exact(buf).await;
	  ```
	- 理论上我们应该只从 S3 获取 4MB 的数据，然后这个链接被销毁。但是 opendal 在实现的时候总是使用当前 reader 的 size 来发送请求：
		- ```rust
		  ```