- OpenDAL 是一个重 IO 的库，可以考虑提供维护自己的 runtime
-
- 几个设计考虑
	- 独立的 runtime，同时允许用户使用自己的 runtime？
		- 考虑到现状，可能必须要自己抽象 trait 或者绑定在 tokio 上
	- API 是否需要变化？
		- 从 async xx 改成返回一个 Future？
			- 有生命周期的要求
	- 能够实现 thread local 的分配？
		- 比如 s3 的 client 每个 thread 分配一个而不是共享同一个
	- 使用同一个 runtime 还是每个 backend 分配一个自己的？
		- 默认可以使用外部的 executor
		- 然后提供一个 global 的 executor
		- 最后每个 backend 可以选择再自己创建 executor？
			- 放在 backend 里面？
-
- 可以参考的项目
	- influxdata DedicatedExecutor
		- https://github.com/influxdata/influxdb_iox/blob/main/executor/src/lib.rs
	- hyper 支持外部的 executor
collapsed:: true
		- ```rust
		  #[derive(Clone)]
		  pub enum Exec {
		      Default,
		      Executor(Arc<dyn Executor<BoxSendFuture> + Send + Sync>),
		  }
		  
		  // ===== impl Exec =====
		  
		  impl Exec {
		      pub(crate) fn execute<F>(&self, fut: F)
		      where
		          F: Future<Output = ()> + Send + 'static,
		      {
		          match *self {
		              Exec::Default => {
		                  #[cfg(feature = "tcp")]
		                  {
		                      tokio::task::spawn(fut);
		                  }
		                  #[cfg(not(feature = "tcp"))]
		                  {
		                      // If no runtime, we need an executor!
		                      panic!("executor must be set")
		                  }
		              }
		              Exec::Executor(ref e) => {
		                  e.execute(Box::pin(fut));
		              }
		          }
		      }
		  }
		  ```
	- 跨线程 IO 投递任务
collapsed:: true
		- ```rust
		  for field in &fields {
		    if let Some(meta) = col_map.get(field.name.as_str()) {
		      let (start, len) = meta.byte_range();
		      let mut reader = data_accessor.object(path.as_str()).range_reader(start, len);
		      let mut chunk = vec![0; len as usize];
		      debug!("read_exact, offset {}, len {}", start, len);
		      let current = Span::current();
		      let fut = async move {
		        reader
		        .read_exact(&mut chunk)
		        .instrument(
		          debug_span!(parent: current, "read_exact_col_chunk").or_current(),
		        )
		        .await?;
		        Ok::<_, ErrorCode>(chunk)
		      };
		      // spawn io tasks
		      let fut = exec.spawn(fut);
		      futs.push(fut);
		      col_meta.push(meta);
		    }
		  }
		  ```
-
- 大致的实现
collapsed:: true
	- ```rust
	  pub static GLOBAL_EXECUTOR: Lazy<tokio::runtime::Runtime> =
	      Lazy::new(|| tokio::runtime::Runtime::new().unwrap());
	  
	  pub trait Spawner {
	      fn spawn<T>(&self, future: T) -> JoinHandle<T::Output>
	      where
	          T: Future + Send + 'static,
	          T::Output: Send + 'static,
	      {
	          let _ = future;
	          unimplemented!()
	      }
	  }
	  
	  #[derive(Clone, Debug)]
	  pub enum Executor {
	      External,
	      Global,
	      Internal(),
	  }
	  
	  impl Spawner for Executor {
	      fn spawn<T>(&self, future: T) -> JoinHandle<T::Output>
	      where
	          T: Future + Send + 'static,
	          T::Output: Send + 'static,
	      {
	          match self {
	              Executor::External => tokio::spawn(future),
	              Executor::Global => GLOBAL_EXECUTOR.spawn(future),
	              Executor::Internal() => unimplemented!(),
	          }
	      }
	  }
	  ```
-
- 目前存在的问题
	- 只是在 accessor 中使用 `self.exec.spawn` 的话好像并不会保证返回的 Reader 也运行在这个 runtime 中
collapsed:: true
		- 返回的 Reader 需要额外包装一下，确保所有的 poll_read call 都在自己的 runtime 上运行？
			- 这样是不是有额外的开销？
				- 每次 poll_read 下去都会创建一个新的 future，然后要再等待
			- 考虑把 API 改成接收一个 `BoxedWriter` 吗？
				- 这样可以不用去改造 BoxedReader
				- 这样的话需要维护一个 channel，连接 Reader 和 Writer
					- ```rust
					  let (pr, pw) = pipe();
					  op.object().reader();
					  
					  r.read()
					  ```
					- 还有一种可能是直接传入 `&mut buf` 但是这样是不是有生命周期的问题
						- 修改 API，要求传入一个 owned buf，最后返回？
							- 没法用啊，怎么实现 `futures::AsyncRead` 呢？
								- 在内存里面多复制一遍(性能开销大)
									- 内存中 coyp 4MB bytes 需要 1.2 ms 左右，吞吐大约为 3GB/s
										- ```rust
										  copy/memory_copy        time:   [1.2547 ms 1.2613 ms 1.2674 ms]
										                          thrpt:  [3.0820 GiB/s 3.0971 GiB/s 3.1133 GiB/s]
										                   change:
										                          time:   [+3.6469% +4.6892% +5.5711%] (p = 0.00 < 0.05)
										                          thrpt:  [-5.2771% -4.4792% -3.5186%]
										  
										  ```
									- reuse 同一个 buf？
								- ```rust
								      fn read2(mut self) -> JoinHandle<Result<usize>> {
								          self.backend.exec.spawn(async move {
								              let mut f = self.inner.lock().expect("file mutex poisoned");
								              let n = f
								                  .read(self.buf.as_mut())
								                  .map_err(|e| parse_io_error(e, "read", "x"))?;
								              Ok(n)
								          })
								      }
								  ```
								- 不只是性能问题，这个 Object 只能 read 一次 - -
							- [[bytedance/monoio]] 的设计是返回 owned T
								- ```rust
								  pub trait AsyncReadRent {
								      type ReadFuture: Future<Output = BufResult<usize, T>>;
								      type ReadvFuture: Future<Output = BufResult<usize, T>>;
								      fn read<T: IoBufMut>(&self, buf: T) -> Self::ReadFuture;
								      fn readv<T: IoVecBufMut>(&self, buf: T) -> Self::ReadvFuture;
								  }
								  
								  type ReadFuture: Future<Output = BufResult<usize, T>>
								  ```
							- 实现 `futures::AsyncWrite` 的时候也会有问题
								- 需要接受一个 owned slice
						- 还有 file 没法修改的问题
							- MutexGuard 不是 Send 的
								- `Arc<Mutex<std::fs::File>>` 是 Send，在线程内部加锁即可
						- 如果接收 `&mut buf` 作为参数，出现  invalid range 怎么办？
							- 比如 s3 在一个 1MB 的文件上尝试读取 4MB 的 range
							- 直接返回错误可以吗？
								- 在 Object 的层面维护好 remaining size？
								- 可以解析 get_object 时返回 content_range?
								- 其实可以自己尝试 retry 一下
									- 如果 range 不对就直接读取后面的全部 size
										- 挺离谱的，rust sdk 甚至没有返回 invalid range 错误的支持。。
									- 如果 range 不对就 stat 一下？
									- > Amazon S3 doesn't support retrieving multiple ranges of data per GET request.
							- 把 object 变成 trait？
				- 下推到 object 一层，在 object 上直接实现 reader 和 writer
					- 能不能保持 fd 开着呢？避免重复的 open/seek/close
						- 只有 fs 需要做这个，其他的服务可以直接发新的请求
						- 啊，想到了 s3 不能这么做的原因
							- s3 不支持 pwrite，所以只能支持顺序写
								- 可以考虑再追加一个 ReadFrom，WriteTo
					- 可以在 futures::AsyncRead 之外，再提供一套自己的底层 API
						- Readiness + Read + Write
					- 返回一个 Object？
						- 在 Object 里塞一个 inner 结构体？
							- 比如说 `std::any`，然后做运行时反射
							- `Box<dyn Any>`
		- 除了性能考虑之外，更严重的是会 block 当前的 runtime
			- 比如 sync io 泄漏给了外面的 runtime
	- 每个 Accessor 都要这样实现一遍好像没有必要，应该可以在最外面套一个？
		- 跟异步的 API 可以结合吗？
	- 为什么 blocking 这么快呢？
		- 需要学习借鉴一下
-
- Benchmark 测试结果
	- blocking bench
collapsed:: true
		- ```rust
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 8.0s, enable flat sampling, or reduce sample count to 50.
		  fs/read                 time:   [2.1549 ms 2.2474 ms 2.3194 ms]
		                          thrpt:  [6.7366 GiB/s 6.9525 GiB/s 7.2508 GiB/s]
		  Benchmarking fs/buf_read: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 9.8s, enable flat sampling, or reduce sample count to 50.
		  fs/buf_read             time:   [2.2682 ms 2.4028 ms 2.5224 ms]
		                          thrpt:  [6.1944 GiB/s 6.5028 GiB/s 6.8887 GiB/s]
		  Benchmarking fs/range_read: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 5.1s, enable flat sampling, or reduce sample count to 70.
		  fs/range_read           time:   [1.3729 ms 1.4400 ms 1.5597 ms]
		                          thrpt:  [5.0088 GiB/s 5.4252 GiB/s 5.6905 GiB/s]
		  Found 9 outliers among 100 measurements (9.00%)
		    7 (7.00%) low severe
		    1 (1.00%) low mild
		    1 (1.00%) high severe
		  Benchmarking fs/read_half: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 6.8s, enable flat sampling, or reduce sample count to 60.
		  fs/read_half            time:   [1.4088 ms 1.4251 ms 1.4414 ms]
		                          thrpt:  [5.4202 GiB/s 5.4820 GiB/s 5.5456 GiB/s]
		  Found 8 outliers among 100 measurements (8.00%)
		    6 (6.00%) low mild
		    2 (2.00%) high mild
		  fs/write                time:   [7.2716 ms 7.3625 ms 7.4675 ms]
		                          thrpt:  [2.0924 GiB/s 2.1223 GiB/s 2.1488 GiB/s]
		  Found 3 outliers among 100 measurements (3.00%)
		    2 (2.00%) high mild
		    1 (1.00%) high severe
		  
		  s3 not set, ignore
		  fs_parallel/parallel_range_read_2
		                          time:   [1.6653 ms 1.7779 ms 1.9013 ms]
		                          thrpt:  [8.2182 GiB/s 8.7882 GiB/s 9.3827 GiB/s]
		                   change:
		                          time:   [+3.0814% +11.651% +21.259%] (p = 0.01 < 0.05)
		                          thrpt:  [-17.532% -10.435% -2.9893%]
		                          Performance has regressed.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high severe
		  fs_parallel/parallel_range_read_4
		                          time:   [3.4346 ms 3.4913 ms 3.5616 ms]
		                          thrpt:  [8.7743 GiB/s 8.9509 GiB/s 9.0986 GiB/s]
		                   change:
		                          time:   [-2.4457% +0.2222% +2.8301%] (p = 0.87 > 0.05)
		                          thrpt:  [-2.7522% -0.2217% +2.5070%]
		                          No change in performance detected.
		  Found 3 outliers among 100 measurements (3.00%)
		    3 (3.00%) high severe
		  fs_parallel/parallel_range_read_8
		                          time:   [8.2057 ms 8.4363 ms 8.6722 ms]
		                          thrpt:  [7.2069 GiB/s 7.4085 GiB/s 7.6166 GiB/s]
		                   change:
		                          time:   [-19.837% -17.064% -14.218%] (p = 0.00 < 0.05)
		                          thrpt:  [+16.574% +20.575% +24.746%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  fs_parallel/parallel_range_read_16
		                          time:   [20.859 ms 21.349 ms 21.847 ms]
		                          thrpt:  [5.7216 GiB/s 5.8551 GiB/s 5.9927 GiB/s]
		                   change:
		                          time:   [-9.9573% -6.9863% -4.0236%] (p = 0.00 < 0.05)
		                          thrpt:  [+4.1923% +7.5110% +11.058%]
		                          Performance has improved.
		  
		  ```
	- tokio 跟测试共享 同一个 tokio runtime
collapsed:: true
		- ```rust
		  fs/read                 time:   [1.8692 ms 1.9521 ms 2.0349 ms]
		                          thrpt:  [7.6785 GiB/s 8.0044 GiB/s 8.3591 GiB/s]
		                   change:
		                          time:   [-2.2447% +4.8898% +12.552%] (p = 0.19 > 0.05)
		                          thrpt:  [-11.152% -4.6619% +2.2962%]
		                          No change in performance detected.
		  fs/buf_read             time:   [2.0496 ms 2.1506 ms 2.2522 ms]
		                          thrpt:  [6.9377 GiB/s 7.2655 GiB/s 7.6233 GiB/s]
		                   change:
		                          time:   [-21.682% -16.311% -10.640%] (p = 0.00 < 0.05)
		                          thrpt:  [+11.907% +19.490% +27.685%]
		                          Performance has improved.
		  fs/range_read           time:   [959.04 us 1.0468 ms 1.1413 ms]
		                          thrpt:  [6.8456 GiB/s 7.4630 GiB/s 8.1462 GiB/s]
		                   change:
		                          time:   [-35.457% -28.056% -19.372%] (p = 0.00 < 0.05)
		                          thrpt:  [+24.026% +38.996% +54.935%]
		                          Performance has improved.
		  Found 2 outliers among 100 measurements (2.00%)
		    2 (2.00%) high severe
		  fs/read_half            time:   [671.11 us 729.57 us 797.96 us]
		                          thrpt:  [9.7906 GiB/s 10.708 GiB/s 11.641 GiB/s]
		                   change:
		                          time:   [-36.996% -32.532% -27.240%] (p = 0.00 < 0.05)
		                          thrpt:  [+37.438% +48.218% +58.720%]
		                          Performance has improved.
		  
		  fs_parallel/parallel_range_read_2
		                          time:   [1.5739 ms 1.6297 ms 1.6815 ms]
		                          thrpt:  [9.2923 GiB/s 9.5879 GiB/s 9.9274 GiB/s]
		                   change:
		                          time:   [-30.258% -23.227% -15.368%] (p = 0.00 < 0.05)
		                          thrpt:  [+18.159% +30.254% +43.386%]
		                          Performance has improved.
		  Found 2 outliers among 100 measurements (2.00%)
		    1 (1.00%) high mild
		    1 (1.00%) high severe
		  fs_parallel/parallel_range_read_4
		                          time:   [2.8989 ms 2.9747 ms 3.0536 ms]
		                          thrpt:  [10.234 GiB/s 10.505 GiB/s 10.780 GiB/s]
		                   change:
		                          time:   [-17.760% -14.797% -11.948%] (p = 0.00 < 0.05)
		                          thrpt:  [+13.570% +17.366% +21.595%]
		                          Performance has improved.
		  Found 10 outliers among 100 measurements (10.00%)
		    6 (6.00%) low mild
		    2 (2.00%) high mild
		    2 (2.00%) high severe
		  fs_parallel/parallel_range_read_8
		                          time:   [8.3861 ms 8.6552 ms 8.9230 ms]
		                          thrpt:  [7.0043 GiB/s 7.2211 GiB/s 7.4528 GiB/s]
		                   change:
		                          time:   [-1.6689% +2.5945% +7.0404%] (p = 0.23 > 0.05)
		                          thrpt:  [-6.5773% -2.5289% +1.6973%]
		                          No change in performance detected.
		  fs_parallel/parallel_range_read_16
		                          time:   [20.320 ms 20.739 ms 21.153 ms]
		                          thrpt:  [5.9094 GiB/s 6.0274 GiB/s 6.1515 GiB/s]
		                   change:
		                          time:   [-5.8135% -2.8594% +0.1498%] (p = 0.07 > 0.05)
		                          thrpt:  [-0.1496% +2.9436% +6.1724%]
		                          No change in performance detected.
		  
		  
		  ```
	- pure sync io on tokio runtime
collapsed:: true
		- 这个结果应该是有问题的，没有把数据完整的读完
		- ```rust
		  fs/read                 time:   [833.42 us 845.03 us 858.12 us]
		                          thrpt:  [18.208 GiB/s 18.491 GiB/s 18.748 GiB/s]
		                   change:
		                          time:   [-61.254% -59.267% -56.968%] (p = 0.00 < 0.05)
		                          thrpt:  [+132.38% +145.50% +158.09%]
		                          Performance has improved.
		  Benchmarking fs/buf_read: Warming up for 3.0000 s
		  Warning: Unable to complete 100 samples in 5.0s. You may wish to increase target time to 7.2s, enable flat sampling, or reduce sample count to 50.
		  fs/buf_read             time:   [1.3497 ms 1.3610 ms 1.3742 ms]
		                          thrpt:  [11.371 GiB/s 11.480 GiB/s 11.577 GiB/s]
		                   change:
		                          time:   [-47.808% -45.890% -43.745%] (p = 0.00 < 0.05)
		                          thrpt:  [+77.763% +84.807% +91.600%]
		                          Performance has improved.
		  Found 2 outliers among 100 measurements (2.00%)
		    2 (2.00%) high mild
		  fs/range_read           time:   [465.86 us 468.30 us 470.79 us]
		                          thrpt:  [16.594 GiB/s 16.683 GiB/s 16.770 GiB/s]
		                   change:
		                          time:   [-31.179% -27.721% -24.533%] (p = 0.00 < 0.05)
		                          thrpt:  [+32.508% +38.353% +45.304%]
		                          Performance has improved.
		  Found 3 outliers among 100 measurements (3.00%)
		    2 (2.00%) high mild
		    1 (1.00%) high severe
		  fs/read_half            time:   [397.28 us 399.32 us 401.04 us]
		                          thrpt:  [19.481 GiB/s 19.565 GiB/s 19.665 GiB/s]
		                   change:
		                          time:   [-68.208% -66.587% -64.760%] (p = 0.00 < 0.05)
		                          thrpt:  [+183.77% +199.29% +214.54%]
		                          Performance has improved.
		  
		  fs_parallel/parallel_range_read_2
		                          time:   [886.09 us 891.16 us 896.53 us]
		                          thrpt:  [17.428 GiB/s 17.533 GiB/s 17.634 GiB/s]
		                   change:
		                          time:   [-20.953% -12.822% -4.9245%] (p = 0.00 < 0.05)
		                          thrpt:  [+5.1796% +14.708% +26.507%]
		                          Performance has improved.
		  fs_parallel/parallel_range_read_4
		                          time:   [1.8112 ms 1.8191 ms 1.8270 ms]
		                          thrpt:  [17.104 GiB/s 17.179 GiB/s 17.254 GiB/s]
		                   change:
		                          time:   [-47.654% -46.728% -45.885%] (p = 0.00 < 0.05)
		                          thrpt:  [+84.791% +87.714% +91.037%]
		                          Performance has improved.
		  Found 2 outliers among 100 measurements (2.00%)
		    1 (1.00%) high mild
		    1 (1.00%) high severe
		  fs_parallel/parallel_range_read_8
		                          time:   [3.7666 ms 3.9386 ms 4.1412 ms]
		                          thrpt:  [15.092 GiB/s 15.869 GiB/s 16.593 GiB/s]
		                   change:
		                          time:   [-58.093% -55.961% -53.260%] (p = 0.00 < 0.05)
		                          thrpt:  [+113.95% +127.07% +138.63%]
		                          Performance has improved.
		  Found 8 outliers among 100 measurements (8.00%)
		    1 (1.00%) high mild
		    7 (7.00%) high severe
		  fs_parallel/parallel_range_read_16
		                          time:   [7.3261 ms 7.5424 ms 7.7859 ms]
		                          thrpt:  [16.055 GiB/s 16.573 GiB/s 17.062 GiB/s]
		                   change:
		                          time:   [-66.569% -65.263% -64.040%] (p = 0.00 < 0.05)
		                          thrpt:  [+178.09% +187.88% +199.13%]
		  
		  ```
		-
	-
	- unsafe async read
collapsed:: true
		- ```rust
		  fs/read                 time:   [17.743 ms 17.951 ms 18.136 ms]
		                          thrpt:  [882.23 MiB/s 891.32 MiB/s 901.78 MiB/s]
		                   change:
		                          time:   [+0.6244% +1.9695% +3.2652%] (p = 0.00 < 0.05)
		                          thrpt:  [-3.1620% -1.9315% -0.6206%]
		                          Change within noise threshold.
		  Found 11 outliers among 100 measurements (11.00%)
		    5 (5.00%) low severe
		    2 (2.00%) low mild
		    4 (4.00%) high mild
		  fs/buf_read             time:   [2.3751 ms 2.4501 ms 2.5227 ms]
		                          thrpt:  [6.1938 GiB/s 6.3773 GiB/s 6.5788 GiB/s]
		                   change:
		                          time:   [-3.4575% -0.3920% +2.4060%] (p = 0.80 > 0.05)
		                          thrpt:  [-2.3495% +0.3935% +3.5814%]
		                          No change in performance detected.
		  Found 21 outliers among 100 measurements (21.00%)
		    2 (2.00%) low severe
		    15 (15.00%) low mild
		    4 (4.00%) high mild
		  fs/range_read           time:   [4.0747 ms 4.2163 ms 4.3980 ms]
		                          thrpt:  [1.7764 GiB/s 1.8529 GiB/s 1.9173 GiB/s]
		                   change:
		                          time:   [-48.636% -45.543% -41.999%] (p = 0.00 < 0.05)
		                          thrpt:  [+72.412% +83.632% +94.690%]
		                          Performance has improved.
		  Found 6 outliers among 100 measurements (6.00%)
		    6 (6.00%) high severe
		  fs/read_half            time:   [8.6497 ms 9.1379 ms 9.7009 ms]
		                          thrpt:  [824.66 MiB/s 875.48 MiB/s 924.88 MiB/s]
		                   change:
		                          time:   [+8.2479% +13.362% +20.585%] (p = 0.00 < 0.05)
		                          thrpt:  [-17.071% -11.787% -7.6195%]
		                          Performance has regressed.
		  Found 13 outliers among 100 measurements (13.00%)
		    1 (1.00%) high mild
		    12 (12.00%) high severe
		  fs/write                time:   [6.8004 ms 6.8491 ms 6.9082 ms]
		                          thrpt:  [2.2618 GiB/s 2.2813 GiB/s 2.2977 GiB/s]
		                   change:
		                          time:   [-1.8862% -1.0006% -0.0722%] (p = 0.02 < 0.05)
		                          thrpt:  [+0.0722% +1.0107% +1.9225%]
		                          Change within noise threshold.
		  Found 6 outliers among 100 measurements (6.00%)
		    4 (4.00%) low mild
		    1 (1.00%) high mild
		    1 (1.00%) high severe
		  
		  ```
		- ```rust
		  fs/read                 time:   [18.298 ms 18.423 ms 18.573 ms]
		                          thrpt:  [861.45 MiB/s 868.48 MiB/s 874.43 MiB/s]
		                   change:
		                          time:   [+1.3287% +2.6302% +4.0890%] (p = 0.00 < 0.05)
		                          thrpt:  [-3.9284% -2.5628% -1.3113%]
		                          Performance has regressed.
		  Found 14 outliers among 100 measurements (14.00%)
		    1 (1.00%) low severe
		    4 (4.00%) high mild
		    9 (9.00%) high severe
		  fs/buf_read             time:   [3.9463 ms 3.9623 ms 3.9799 ms]
		                          thrpt:  [3.9260 GiB/s 3.9434 GiB/s 3.9594 GiB/s]
		                   change:
		                          time:   [+57.084% +61.721% +66.768%] (p = 0.00 < 0.05)
		                          thrpt:  [-40.036% -38.165% -36.340%]
		                          Performance has regressed.
		  Found 16 outliers among 100 measurements (16.00%)
		    14 (14.00%) high mild
		    2 (2.00%) high severe
		  fs/range_read           time:   [5.7431 ms 6.2377 ms 6.7498 ms]
		                          thrpt:  [1.1574 GiB/s 1.2525 GiB/s 1.3603 GiB/s]
		                   change:
		                          time:   [+34.514% +47.944% +60.830%] (p = 0.00 < 0.05)
		                          thrpt:  [-37.823% -32.407% -25.658%]
		                          Performance has regressed.
		  fs/read_half            time:   [8.8356 ms 8.8871 ms 8.9467 ms]
		                          thrpt:  [894.18 MiB/s 900.18 MiB/s 905.43 MiB/s]
		                   change:
		                          time:   [-8.3907% -2.7437% +2.7720%] (p = 0.36 > 0.05)
		                          thrpt:  [-2.6972% +2.8211% +9.1593%]
		                          No change in performance detected.
		  Found 9 outliers among 100 measurements (9.00%)
		    2 (2.00%) low mild
		    2 (2.00%) high mild
		    5 (5.00%) high severe
		  fs/write                time:   [6.7729 ms 6.7921 ms 6.8116 ms]
		                          thrpt:  [2.2939 GiB/s 2.3005 GiB/s 2.3070 GiB/s]
		                   change:
		                          time:   [-1.7090% -0.8331% -0.0568%] (p = 0.05 < 0.05)
		                          thrpt:  [+0.0569% +0.8401% +1.7387%]
		                          Change within noise threshold.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  ```
	- 稍微做了一些优化
collapsed:: true
		- ```rust
		  fs/read                 time:   [15.221 ms 15.288 ms 15.362 ms]
		                          thrpt:  [1.0171 GiB/s 1.0220 GiB/s 1.0266 GiB/s]
		                   change:
		                          time:   [-17.783% -17.016% -16.317%] (p = 0.00 < 0.05)
		                          thrpt:  [+19.499% +20.505% +21.629%]
		                          Performance has improved.
		  Found 14 outliers among 100 measurements (14.00%)
		    3 (3.00%) high mild
		    11 (11.00%) high severe
		  fs/buf_read             time:   [3.2227 ms 3.2292 ms 3.2367 ms]
		                          thrpt:  [4.8274 GiB/s 4.8387 GiB/s 4.8484 GiB/s]
		                   change:
		                          time:   [-18.895% -18.503% -18.114%] (p = 0.00 < 0.05)
		                          thrpt:  [+22.121% +22.703% +23.297%]
		                          Performance has improved.
		  Found 4 outliers among 100 measurements (4.00%)
		    2 (2.00%) high mild
		    2 (2.00%) high severe
		  fs/range_read           time:   [7.5952 ms 7.8492 ms 8.1078 ms]
		                          thrpt:  [986.71 MiB/s 1019.2 MiB/s 1.0286 GiB/s]
		                   change:
		                          time:   [+15.553% +25.835% +37.336%] (p = 0.00 < 0.05)
		                          thrpt:  [-27.186% -20.531% -13.459%]
		                          Performance has regressed.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) low mild
		  fs/read_half            time:   [11.430 ms 12.233 ms 13.086 ms]
		                          thrpt:  [611.35 MiB/s 653.95 MiB/s 699.90 MiB/s]
		                   change:
		                          time:   [+28.928% +37.652% +47.005%] (p = 0.00 < 0.05)
		                          thrpt:  [-31.975% -27.353% -22.437%]
		                          Performance has regressed.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  fs/write                time:   [6.5152 ms 6.5787 ms 6.6451 ms]
		                          thrpt:  [2.3514 GiB/s 2.3751 GiB/s 2.3982 GiB/s]
		                   change:
		                          time:   [-4.1086% -3.1418% -2.1582%] (p = 0.00 < 0.05)
		                          thrpt:  [+2.2059% +3.2438% +4.2846%]
		                          Performance has improved.
		  Found 1 outliers among 100 measurements (1.00%)
		    1 (1.00%) high mild
		  
		  ```
	- 使用 unsafe set_len (on global)
collapsed:: true
		- ```rust
		  fs/read                 time:   [8.3752 ms 8.4022 ms 8.4306 ms]
		                          thrpt:  [1.8534 GiB/s 1.8596 GiB/s 1.8656 GiB/s]
		                   change:
		                          time:   [-40.470% -38.895% -37.025%] (p = 0.00 < 0.05)
		                          thrpt:  [+58.792% +63.652% +67.981%]
		                          Performance has improved.
		  Found 9 outliers among 100 measurements (9.00%)
		    7 (7.00%) low mild
		    1 (1.00%) high mild
		    1 (1.00%) high severe
		  fs/buf_read             time:   [2.5840 ms 2.5872 ms 2.5906 ms]
		                          thrpt:  [6.0315 GiB/s 6.0394 GiB/s 6.0469 GiB/s]
		                   change:
		                          time:   [-34.940% -32.911% -30.682%] (p = 0.00 < 0.05)
		                          thrpt:  [+44.263% +49.055% +53.704%]
		                          Performance has improved.
		  Found 12 outliers among 100 measurements (12.00%)
		    10 (10.00%) high mild
		    2 (2.00%) high severe
		  fs/range_read           time:   [5.3462 ms 5.6247 ms 5.9101 ms]
		                          thrpt:  [1.3219 GiB/s 1.3890 GiB/s 1.4613 GiB/s]
		                   change:
		                          time:   [-27.026% -21.966% -15.952%] (p = 0.00 < 0.05)
		                          thrpt:  [+18.980% +28.150% +37.036%]
		                          Performance has improved.
		  fs/read_half            time:   [8.8952 ms 9.0929 ms 9.3406 ms]
		                          thrpt:  [856.48 MiB/s 879.81 MiB/s 899.36 MiB/s]
		                   change:
		                          time:   [-42.275% -37.479% -32.056%] (p = 0.00 < 0.05)
		                          thrpt:  [+47.179% +59.945% +73.234%]
		                          Performance has improved.
		  Found 6 outliers among 100 measurements (6.00%)
		    1 (1.00%) high mild
		    5 (5.00%) high severe
		  fs/write                time:   [6.6648 ms 6.6897 ms 6.7149 ms]
		                          thrpt:  [2.3269 GiB/s 2.3357 GiB/s 2.3444 GiB/s]
		                   change:
		                          time:   [-0.4973% +0.7196% +1.8254%] (p = 0.24 > 0.05)
		                          thrpt:  [-1.7927% -0.7145% +0.4997%]
		                          No change in performance detected.
		  
		  
		  ```
	- 使用 unsafe set_len (on external)
collapsed:: true
		- ```rust
		  fs/read                 time:   [13.979 ms 15.384 ms 16.768 ms]
		                          thrpt:  [954.18 MiB/s 1.0157 GiB/s 1.1178 GiB/s]
		                   change:
		                          time:   [+67.725% +83.093% +102.74%] (p = 0.00 < 0.05)
		                          thrpt:  [-50.675% -45.383% -40.378%]
		                          Performance has regressed.
		  fs/buf_read             time:   [4.4740 ms 4.4907 ms 4.5085 ms]
		                          thrpt:  [3.4657 GiB/s 3.4794 GiB/s 3.4924 GiB/s]
		                   change:
		                          time:   [+72.937% +73.576% +74.309%] (p = 0.00 < 0.05)
		                          thrpt:  [-42.631% -42.388% -42.176%]
		                          Performance has regressed.
		  Found 2 outliers among 100 measurements (2.00%)
		    1 (1.00%) high mild
		    1 (1.00%) high severe
		  fs/range_read           time:   [11.678 ms 11.734 ms 11.792 ms]
		                          thrpt:  [678.41 MiB/s 681.79 MiB/s 685.07 MiB/s]
		                   change:
		                          time:   [+98.524% +108.61% +119.56%] (p = 0.00 < 0.05)
		                          thrpt:  [-54.455% -52.064% -49.628%]
		                          Performance has regressed.
		  Found 21 outliers among 100 measurements (21.00%)
		    1 (1.00%) low severe
		    7 (7.00%) low mild
		    7 (7.00%) high mild
		    6 (6.00%) high severe
		  fs/read_half            time:   [8.5076 ms 8.7344 ms 9.0195 ms]
		                          thrpt:  [886.96 MiB/s 915.92 MiB/s 940.34 MiB/s]
		                   change:
		                          time:   [-7.2484% -3.9428% -0.1972%] (p = 0.04 < 0.05)
		                          thrpt:  [+0.1976% +4.1047% +7.8149%]
		                          Change within noise threshold.
		  Found 7 outliers among 100 measurements (7.00%)
		    3 (3.00%) high mild
		    4 (4.00%) high severe
		  fs/write                time:   [6.4632 ms 6.5024 ms 6.5422 ms]
		                          thrpt:  [2.3883 GiB/s 2.4030 GiB/s 2.4175 GiB/s]
		                   change:
		                          time:   [-3.4676% -2.8008% -2.0884%] (p = 0.00 < 0.05)
		                          thrpt:  [+2.1330% +2.8816% +3.5922%]
		                          Performance has improved.
		  Found 3 outliers among 100 measurements (3.00%)
		    3 (3.00%) high mild
		  
		  ```
-
- 感觉效果其实还可以，可以认真实现一把看看
-
- 实现需要考虑的问题
collapsed:: true
	- 可以去掉 `async_trait` 了？
		- 其实没啥变化，所有的 future 还是需要包上 Box？
	- 对用户的 API 可以尽可能不变化吗？
		- 现在的 API
			- op.object(path).reader()
		- 想象中的 API
			- 向标准库看齐？
			- `op.remove_file(path)`
			- `op.remove_dir(path)`
			- `op.create_dir(path)`
			- `op.create_dir_all(path)`
			- `op.read_dir(path)`
			-
			- `op.write(path, Vec<u8>)`
			- `op.read(path) -> Vec<u8>`
			- `op.open(path) -> Result<File>`
			- `op.create(path) -> Result<File>`
			-
			- 本质上还是 fs 和 对象的抽象很难统一到一起
collapsed:: true
				- 要不要放弃对 file 侧的优化？
					- 每次 read 都重新打开文件和 take offset？
					- 打开一个文件和进行 offset 的开销其实很小，本地这边测试都在 ns 级别
collapsed:: true
						- 跟 IO 相比不算是大头，感觉没有必要针对文件进行优化？
						- ```rust
						  file/open_file          time:   [804.53 ns 806.59 ns 808.92 ns]
						                          change: [+5.3797% +5.8694% +6.3125%] (p = 0.00 < 0.05)
						                          Performance has regressed.
						  Found 10 outliers among 100 measurements (10.00%)
						    1 (1.00%) low severe
						    3 (3.00%) low mild
						    4 (4.00%) high mild
						    2 (2.00%) high severe
						  file/seek_file          time:   [890.87 ns 891.81 ns 892.90 ns]
						                          change: [+4.1270% +4.5609% +5.0086%] (p = 0.00 < 0.05)
						                          Performance has regressed.
						  Found 16 outliers among 100 measurements (16.00%)
						    5 (5.00%) low severe
						    1 (1.00%) low mild
						    7 (7.00%) high mild
						    3 (3.00%) high severe
						  file/read_file_1k       time:   [1.0367 us 1.0375 us 1.0383 us]
						                          change: [+3.6974% +4.5057% +5.2159%] (p = 0.00 < 0.05)
						                          Performance has regressed.
						  Found 20 outliers among 100 measurements (20.00%)
						    2 (2.00%) low severe
						    3 (3.00%) low mild
						    6 (6.00%) high mild
						    9 (9.00%) high severe
						  file/read_file_4k       time:   [1.1051 us 1.1071 us 1.1092 us]
						                          change: [-0.6052% -0.1509% +0.2611%] (p = 0.50 > 0.05)
						                          No change in performance detected.
						  Found 10 outliers among 100 measurements (10.00%)
						    2 (2.00%) low severe
						    1 (1.00%) low mild
						    4 (4.00%) high mild
						    3 (3.00%) high severe
						  file/read_file_1MB      time:   [31.355 us 31.422 us 31.484 us]
						                          change: [-5.9701% -5.6834% -5.3993%] (p = 0.00 < 0.05)
						                          Performance has improved.
						  Found 4 outliers among 100 measurements (4.00%)
						    3 (3.00%) high mild
						    1 (1.00%) high severe
						  file/read_file_4MB      time:   [127.47 us 127.77 us 128.03 us]
						                          change: [-9.0247% -8.3413% -7.7052%] (p = 0.00 < 0.05)
						                          Performance has improved.
						  
						  ```
						- 读取 1KB -> 78%
						- 读取 4KB -> 73%
						- 读取 1MB -> 2.6%
						- 读取 4MB -> 0.6%
						-
						- 随着读取数据的变多会占的越来越少，所以实际上可以不用考虑？
							- 而且需要考虑 OpenDAL 的定位
			-
			- op.object(path).reader()
			-
			-
-
- 当面 main 分支 benchmark (参考值)
	- ```rust
	  s3/read                 time:   [4.8642 ms 4.9794 ms 5.0968 ms]
	                          thrpt:  [3.0656 GiB/s 3.1379 GiB/s 3.2123 GiB/s]
	                   change:
	                          time:   [-14.139% -11.915% -9.7364%] (p = 0.00 < 0.05)
	                          thrpt:  [+10.787% +13.526% +16.467%]
	                          Performance has improved.
	  s3/buf_read             time:   [4.9669 ms 5.0360 ms 5.1018 ms]
	                          thrpt:  [3.0626 GiB/s 3.1027 GiB/s 3.1458 GiB/s]
	                   change:
	                          time:   [-51.868% -50.818% -49.625%] (p = 0.00 < 0.05)
	                          thrpt:  [+98.511% +103.33% +107.76%]
	                          Performance has improved.
	  Found 4 outliers among 100 measurements (4.00%)
	    4 (4.00%) low mild
	  s3/range_read           time:   [3.0333 ms 3.1433 ms 3.2663 ms]
	                          thrpt:  [2.3919 GiB/s 2.4854 GiB/s 2.5756 GiB/s]
	                   change:
	                          time:   [+10.974% +16.301% +22.100%] (p = 0.00 < 0.05)
	                          thrpt:  [-18.100% -14.016% -9.8889%]
	                          Performance has regressed.
	  Found 4 outliers among 100 measurements (4.00%)
	    1 (1.00%) high mild
	    3 (3.00%) high severe
	  s3/read_half            time:   [2.9057 ms 2.9813 ms 3.0648 ms]
	                          thrpt:  [2.5491 GiB/s 2.6205 GiB/s 2.6887 GiB/s]
	                   change:
	                          time:   [-8.2502% -4.9245% -1.0213%] (p = 0.01 < 0.05)
	                          thrpt:  [+1.0319% +5.1795% +8.9921%]
	                          Performance has improved.
	  Found 5 outliers among 100 measurements (5.00%)
	    4 (4.00%) high mild
	    1 (1.00%) high severe
	  s3/write                time:   [39.665 ms 40.472 ms 41.275 ms]
	                          thrpt:  [387.64 MiB/s 395.34 MiB/s 403.38 MiB/s]
	                   change:
	                          time:   [-61.179% -60.259% -59.242%] (p = 0.00 < 0.05)
	                          thrpt:  [+145.35% +151.63% +157.59%]
	                          Performance has improved.
	  
	  fs not set, ignore
	  s3_parallel/parallel_range_read_2
	                          time:   [3.5926 ms 3.6604 ms 3.7435 ms]
	                          thrpt:  [4.1739 GiB/s 4.2687 GiB/s 4.3493 GiB/s]
	                   change:
	                          time:   [+5.5733% +8.5830% +11.664%] (p = 0.00 < 0.05)
	                          thrpt:  [-10.445% -7.9046% -5.2791%]
	                          Performance has regressed.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high severe
	  s3_parallel/parallel_range_read_4
	                          time:   [4.9526 ms 5.0581 ms 5.1640 ms]
	                          thrpt:  [6.0515 GiB/s 6.1783 GiB/s 6.3099 GiB/s]
	                   change:
	                          time:   [+4.4256% +8.1345% +11.594%] (p = 0.00 < 0.05)
	                          thrpt:  [-10.390% -7.5225% -4.2381%]
	                          Performance has regressed.
	  Found 3 outliers among 100 measurements (3.00%)
	    3 (3.00%) low mild
	  s3_parallel/parallel_range_read_8
	                          time:   [8.1868 ms 8.4811 ms 8.7875 ms]
	                          thrpt:  [7.1124 GiB/s 7.3693 GiB/s 7.6342 GiB/s]
	                   change:
	                          time:   [-13.658% -9.0803% -3.9414%] (p = 0.00 < 0.05)
	                          thrpt:  [+4.1032% +9.9872% +15.819%]
	                          Performance has improved.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high mild
	  s3_parallel/parallel_range_read_16
	                          time:   [19.370 ms 19.851 ms 20.352 ms]
	                          thrpt:  [6.1420 GiB/s 6.2970 GiB/s 6.4533 GiB/s]
	                   change:
	                          time:   [-4.1929% -0.7665% +2.8570%] (p = 0.67 > 0.05)
	                          thrpt:  [-2.7777% +0.7724% +4.3764%]
	                          No change in performance detected.
	  Found 5 outliers among 100 measurements (5.00%)
	    5 (5.00%) high mild
	  ```
- 目前实现的 benchmark
	- s3 buf read
		- ```rust
		  s3/buf_read             time:   [10.047 ms 10.240 ms 10.438 ms]
		                          thrpt:  [1.4969 GiB/s 1.5259 GiB/s 1.5552 GiB/s]
		                   change:
		                          time:   [+4.1896% +6.4811% +9.0741%] (p = 0.00 < 0.05)
		                          thrpt:  [-8.3192% -6.0866% -4.0211%]
		  ```
-
- memory 的 benchmark
	- ```rust
	  copy/memory_copy_128B   time:   [2.0103 ns 2.0107 ns 2.0111 ns]
	                          thrpt:  [59.277 GiB/s 59.289 GiB/s 59.299 GiB/s]
	                   change:
	                          time:   [-7.5861% -7.4435% -7.3184%] (p = 0.00 < 0.05)
	                          thrpt:  [+7.8962% +8.0422% +8.2088%]
	                          Performance has improved.
	  Found 19 outliers among 100 measurements (19.00%)
	    6 (6.00%) high mild
	    13 (13.00%) high severe
	  Benchmarking copy/memory_copy_256B: Collecting 100 samples in estimated 5.0000 s (1.6B                                                                                      copy/memory_copy_256B   time:   [3.0379 ns 3.0394 ns 3.0411 ns]
	                          thrpt:  [39.199 GiB/s 39.222 GiB/s 39.240 GiB/s]
	                   change:
	                          time:   [+59.261% +59.620% +60.108%] (p = 0.00 < 0.05)
	                          thrpt:  [-37.542% -37.351% -37.210%]
	                          Performance has regressed.
	  Found 16 outliers among 100 measurements (16.00%)
	    4 (4.00%) high mild
	    12 (12.00%) high severe
	  Benchmarking copy/memory_copy_512B: Collecting 100 samples in estimated 5.0000 s (1.3B                                                                                      copy/memory_copy_512B   time:   [3.7197 ns 3.7227 ns 3.7280 ns]
	                          thrpt:  [127.91 GiB/s 128.09 GiB/s 128.19 GiB/s]
	                   change:
	                          time:   [-6.0335% -5.8034% -5.6017%] (p = 0.00 < 0.05)
	                          thrpt:  [+5.9341% +6.1610% +6.4209%]
	                          Performance has improved.
	  Found 13 outliers among 100 measurements (13.00%)
	    3 (3.00%) high mild
	    10 (10.00%) high severe
	  Benchmarking copy/memory_copy_1k: Collecting 100 samples in estimated 5.0000 s (649M i                                                                                      copy/memory_copy_1k     time:   [7.9070 ns 8.0311 ns 8.1559 ns]
	                          thrpt:  [116.93 GiB/s 118.75 GiB/s 120.61 GiB/s]
	                   change:
	                          time:   [-6.3639% -4.8805% -3.1718%] (p = 0.00 < 0.05)
	                          thrpt:  [+3.2758% +5.1309% +6.7964%]
	                          Performance has improved.
	  Found 1 outliers among 100 measurements (1.00%)
	    1 (1.00%) high severe
	  Benchmarking copy/memory_copy_4k: Collecting 100 samples in estimated 5.0003 s (5.4M i                                                                                      copy/memory_copy_4k     time:   [918.03 ns 918.77 ns 919.62 ns]
	                          thrpt:  [4.1481 GiB/s 4.1520 GiB/s 4.1553 GiB/s]
	  Found 5 outliers among 100 measurements (5.00%)
	    4 (4.00%) high mild
	    1 (1.00%) high severe
	  Benchmarking copy/memory_copy_8k: Collecting 100 samples in estimated 5.0074 s (2.8M i                                                                                      copy/memory_copy_8k     time:   [1.8074 us 1.8096 us 1.8121 us]
	                          thrpt:  [4.2103 GiB/s 4.2162 GiB/s 4.2213 GiB/s]
	                   change:
	                          time:   [-1.5153% -1.0090% -0.5109%] (p = 0.00 < 0.05)
	                          thrpt:  [+0.5135% +1.0193% +1.5386%]
	                          Change within noise threshold.
	  Found 14 outliers among 100 measurements (14.00%)
	    4 (4.00%) high mild
	    10 (10.00%) high severe
	  Benchmarking copy/memory_copy_1M: Collecting 100 samples in estimated 5.6357 s (35k it                                                                                      copy/memory_copy_1M     time:   [155.66 us 157.12 us 158.59 us]
	                          thrpt:  [6.1579 GiB/s 6.2154 GiB/s 6.2737 GiB/s]
	                   change:
	                          time:   [-5.5206% -3.8590% -2.2796%] (p = 0.00 < 0.05)
	                          thrpt:  [+2.3327% +4.0139% +5.8432%]
	                          Performance has improved.
	  Found 7 outliers among 100 measurements (7.00%)
	    1 (1.00%) low mild
	    5 (5.00%) high mild
	    1 (1.00%) high severe
	  Benchmarking copy/memory_copy_4M: Collecting 100 samples in estimated 9.3087 s (10k it                                                                                      copy/memory_copy_4M     time:   [919.06 us 921.67 us 924.41 us]
	                          thrpt:  [4.2257 GiB/s 4.2382 GiB/s 4.2503 GiB/s]
	                   change:
	                          time:   [-0.5356% +1.3699% +3.2749%] (p = 0.15 > 0.05)
	                          thrpt:  [-3.1711% -1.3514% +0.5385%]
	                          No change in performance detected.
	  
	  ```
	- 过了 4KB 之后速度断崖式下降
-
- 感觉不太行，性能不是很好