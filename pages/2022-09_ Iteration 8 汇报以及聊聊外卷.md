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
	- 这通常被叫做观察者模式，非常适合这种场景。在 PR [dal_context: Use ObserveReader to calculate metrics](https://github.com/datafuselabs/databend/pull/4298) 中，我为 databend 增加了时间统计的支持。
		- ```rust
		  async fn read(&self, args: &OpRead) -> DalResult<BoxedAsyncReader> {
		    let metric = self.metrics.clone();
		  
		    self.inner.as_ref().unwrap().read(args).await.map(|reader| {
		      let mut last_pending = None;
		      let r = ObserveReader::new(reader, move |e| {
		        let start = match last_pending {
		          None => Instant::now(),
		          Some(t) => t,
		        };
		        match e {
		          ReadEvent::Pending => last_pending = Some(start),
		          ReadEvent::Read(n) => {
		            last_pending = None;
		            metric.inc_read_bytes(n);
		          }
		          ReadEvent::Error(_) => last_pending = None,
		          _ => {}
		        }
		        metric.inc_read_bytes_cost(start.elapsed().as_millis() as u64);
		      });
		  
		      Box::new(r) as BoxedAsyncReader
		    })
		  }
		  ```
-