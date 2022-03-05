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
-
- 大致的实现
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
-
- Benchmark 测试结果
	- Unblocking bench
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