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
	- 理论上应该只从 S3 获取 4MB 的数据，但实际上会下载将近 4.5MB 的数据，其中多余的部分数据会在这个请求销毁的时候被一同丢弃掉。Databend 平均每次请求的大小大约为 256KB，opendal 多读取的数据经常会超出一倍左右。
	- 背后的原因是 opendal 并不知道用户会如何使用这个 reader ，所以在实现的时候总是使用当前 reader 的 size 来发送请求：
		- ```rust
		  let op = OpRead {
		    path: self.path.to_string(),
		    offset: Some(self.current_offset()),
		    size: self.current_size(),
		  };
		  ```
	- 这就使得 reader 会从服务器端获取比用户预期更多的数据。
- 一种可能的解决方式是 reader 每次读取的时候都按照用户传入的 buf length 来读取。