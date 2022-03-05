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
- Unblocking bench
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
-