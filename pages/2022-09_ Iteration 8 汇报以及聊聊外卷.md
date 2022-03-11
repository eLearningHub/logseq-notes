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
	- 为了解决这个问题，OpenDAL 引入了 [`limited_reader` proposal](https://github.com/datafuselabs/opendal/blob/main/docs/rfcs/0090-limited-reader.md)，通过更明确的语义避免用户多读数据。
- 其次是新增加的 [ObserveReader](https://github.com/datafuselabs/opendal/blob/main/src/readers/observer.rs) 抽象，databend 需要统计每次 read 读取的 size 和花费的时间。size 比较好做，简单的 callback 就能做好，但是花费的时间就稍微麻烦一点，需要能够允许用户在每次读取的前后都能正确的统计时间，还需要能够排除 Pending 上的开销。
	- `ObserveReader` 通过返回 `ReadEvent` 来实现这个功能：
		- ```rust
		  /// ReadEvent will emitted by `ObserveReader`.
		  #[derive(Copy, Clone, Debug)]
		  pub enum ReadEvent {
		      /// `poll_read` has been called.
		      Started,
		      /// `poll_read` returns `Pending`, we are back to the runtime.
		      Pending,
		      /// `poll_read` returns `Ready(Ok(n))`, we have read `n` bytes of data.
		      Read(usize),
		      /// `poll_read` returns `Ready(Err(e))`, we will have an `ErrorKind` here.
		      Error(std::io::ErrorKind),
		  }
		  ```
	- 每当 `ObserveReader` 出现状态切换的时候，就会回调一个用户传入的 callback 函数
		- ```rust
		  impl<R, F> futures::AsyncRead for ObserveReader<R, F>
		  where
		      R: AsyncRead + Send + Unpin,
		      F: FnMut(ReadEvent),
		  {
		      fn poll_read(
		          mut self: Pin<&mut Self>,
		          cx: &mut Context<'_>,
		          buf: &mut [u8],
		      ) -> Poll<std::io::Result<usize>> {
		          (self.f)(ReadEvent::Started);
		  
		          match Pin::new(&mut self.r).poll_read(cx, buf) {
		              Poll::Ready(Ok(n)) => {
		                  (self.f)(ReadEvent::Read(n));
		                  Poll::Ready(Ok(n))
		              }
		              Poll::Ready(Err(e)) => {
		                  (self.f)(ReadEvent::Error(e.kind()));
		                  Poll::Ready(Err(e))
		              }
		              Poll::Pending => {
		                  (self.f)(ReadEvent::Pending);
		                  Poll::Pending
		              }
		          }
		      }
		  }
		  ```
	- 在 PR [dal_context: Use ObserveReader to calculate metrics](https://github.com/datafuselabs/databend/pull/4298) 中，我使用 `ObserveReader` 为 databend 增加了时间统计的支持。
- 最后是 S3 匿名访问的问题。由于 aws-sdk 不支持匿名访问的功能，所以被迫自己搞了一些 Hack，通过修改 AWS SDK 的 Middleware，实现了在没有读取到密钥时直接发送未签名的请求。在这个功能的加持下，databend 能够直接从一个公开的 S3 Bucket 中直接加载数据，非常适合用来做 demo。
-
- 除了 Databend 社区之外，这个周期还给 tikv 旗下的 [minitrace](https://github.com/tikv/minitrace-rust) 和 [minstant](https://github.com/tikv/minstant) 水了一些 PR。
	- minitrace 是一个超快的 tracing 库，从 benchmark 的结果看能比 tokio-tracing 快十倍，在收集的 span 特别多的时候差距能拉大到 100 倍以上。我帮助 [minitrace](https://github.com/tikv/minitrace-rust) 修复了 contributing guide 中的 dead link，增加了简单的开发入门指导，此外还在 PR [deps: Reduce version requirements](https://github.com/tikv/minitrace-rust/pull/108) 中统一了依赖的版本规则，将 `v0.x.y` 统一成了 `v0.x`，放松了一些对版本的要求。
	- minstant 是 minitrace 的依赖，是 `std::time::Instant` 的高性能替代，在支持的平台上会使用 CPU 中的 [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)，比 std 中的实现快一倍。由于这个库功能已经比较成熟，所以相关的维护也比较少，我在 PR [ci: Say goodbye to travis](https://github.com/tikv/minstant/pull/22) 中删掉了已经不再工作的 travis CI 的配置。
-
- 除了