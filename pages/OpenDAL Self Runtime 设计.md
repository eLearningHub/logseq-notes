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
	-