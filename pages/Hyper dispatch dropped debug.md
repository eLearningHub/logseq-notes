- 复现方式
	- https://github.com/TCeason/test
	- 使用如下语句建表
	  collapsed:: true
		- ```sql
		  create database db;
		  
		  create table db.t (
		  id1 int,
		  id2 int,
		  id3 int,
		  id4 int,
		  id5 int,
		  id6 int,
		  id7 int,
		  id8 int,
		  id9 int,
		  id10 int,
		  id11 int,
		  id12 int,
		  id13 int,
		  id14 int,
		  id15 int,
		  id16 int,
		  str1 String,
		  str2 String,
		  str3 String,
		  dt timestamp
		  );
		  ```
	- 然后加大并发量尝试插入
	  collapsed:: true
		- ```shell
		  ./bench8028 --thread-nums=10 --host=127.0.0.1 --user=root --password=root --port=3307 --batch-rows=1000 --total-rows=6000
		  ```
	- 一般会出现如下错误
		- ```rust
		  java.sql.BatchUpdateException: Code: 1068, displayText = Cannot join handle from context's runtime, cause: panic.
		          at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method)
		          at java.base/jdk.internal.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:77)
		          at java.base/jdk.internal.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
		          at java.base/java.lang.reflect.Constructor.newInstanceWithCaller(Constructor.java:499)
		          at java.base/java.lang.reflect.Constructor.newInstance(Constructor.java:480)
		          at com.mysql.cj.util.Util.handleNewInstance(Util.java:192)
		          at com.mysql.cj.util.Util.getInstance(Util.java:167)
		          at com.mysql.cj.util.Util.getInstance(Util.java:174)
		          at com.mysql.cj.jdbc.exceptions.SQLError.createBatchUpdateException(SQLError.java:224)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeBatchedInserts(ClientPreparedStatement.java:755)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeBatchInternal(ClientPreparedStatement.java:426)
		          at com.mysql.cj.jdbc.StatementImpl.executeBatch(StatementImpl.java:795)
		          at demo.lambda$multiThreadImport$0(demo.java:67)
		          at java.base/java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1136)
		          at java.base/java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:635)
		          at java.base/java.lang.Thread.run(Thread.java:833)
		  Caused by: java.sql.SQLException: Code: 1068, displayText = Cannot join handle from context's runtime, cause: panic.
		          at com.mysql.cj.jdbc.exceptions.SQLError.createSQLException(SQLError.java:129)
		          at com.mysql.cj.jdbc.exceptions.SQLExceptionsMapping.translateException(SQLExceptionsMapping.java:122)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeInternal(ClientPreparedStatement.java:953)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1098)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeUpdateInternal(ClientPreparedStatement.java:1046)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeLargeUpdate(ClientPreparedStatement.java:1371)
		          at com.mysql.cj.jdbc.ClientPreparedStatement.executeBatchedInserts(ClientPreparedStatement.java:716)
		          ... 6 more
		  
		  ```
- 初步分析
	- hyper 跨 runtime 运行时会出现 dispatch dropped 这样的问题
	- ```rust
	      pub(crate) fn try_send(&mut self, val: T) -> Result<RetryPromise<T, U>, T> {
	          if !self.can_send() {
	              return Err(val);
	          }
	          let (tx, rx) = oneshot::channel();
	          self.inner
	              .send(Envelope(Some((val, Callback::Retry(tx)))))
	              .map(move |_| rx)
	              .map_err(|mut e| (e.0).0.take().expect("envelope not dropped").0)
	      }
	  ```
	- 每次 req 发送的时候会创建一个 oneshot::channel，如果发送端被 drop 了，recv 的时候就会直接报错
	- backtrace
	  collapsed:: true
		- ```rust
		  2022-03-17T06:40:27.511419Z DEBUG hyper::client::pool: pooling idle connection for ("http", 127.0.0.1:9900)
		  2022-03-17T06:40:27.511221Z ERROR databend_query::servers::mysql::writers::query_result_writer: OnQuery Error: Code: 1068, displayText = Cannot join handle from context's runtime, cause: panic.
		  
		     0: common_exception::exception_code::<impl common_exception::exception::ErrorCode>::TokioError
		               at /home/xuanwo/Code/datafuselabs/databend/common/exception/src/exception_code.rs:36:66
		     1: core::ops::function::FnOnce::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/ops/function.rs:227:5
		     2: <core::result::Result<T,E> as common_exception::exception::ToErrorCode<T,E,CtxFn>>::map_err_to_code::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/common/exception/src/exception.rs:196:13
		     3: core::result::Result<T,E>::map_err
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/result.rs:842:27
		     4: <core::result::Result<T,E> as common_exception::exception::ToErrorCode<T,E,CtxFn>>::map_err_to_code
		               at /home/xuanwo/Code/datafuselabs/databend/common/exception/src/exception.rs:194:9
		     5: databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorkerBase<W>::exec_query::{{closure}}::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:342:28
		     6: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		     7: databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorkerBase<W>::exec_query::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:308:5
		     8: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		     9: databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorkerBase<W>::do_query::{{closure}}::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:286:57
		    10: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    11: databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorkerBase<W>::do_query::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:270:5
		    12: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    13: <databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorker<W> as opensrv_mysql::AsyncMysqlShim<W>>::on_query::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:176:47
		    14: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    15: <core::pin::Pin<P> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
		    16: opensrv_mysql::AsyncMysqlIntermediary<B,S>::run::{{closure}}
		               at /home/xuanwo/.cargo/git/checkouts/opensrv-2d23bfb068524349/9690be9/mysql/src/lib.rs:1083:30
		    17: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    18: opensrv_mysql::AsyncMysqlIntermediary<B,S>::run_with_options::{{closure}}
		               at /home/xuanwo/.cargo/git/checkouts/opensrv-2d23bfb068524349/9690be9/mysql/src/lib.rs:835:17
		    19: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    20: opensrv_mysql::AsyncMysqlIntermediary<B,S>::run_on::{{closure}}
		               at /home/xuanwo/.cargo/git/checkouts/opensrv-2d23bfb068524349/9690be9/mysql/src/lib.rs:815:66
		    21: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    22: databend_query::servers::mysql::mysql_session::MySQLConnection::run_on_stream::{{closure}}::{{closure}}
		               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_session.rs:44:88
		    23: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
		    24: tokio::runtime::task::core::CoreStage<T>::poll::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:161:17
		    25: tokio::loom::std::unsafe_cell::UnsafeCell<T>::with_mut
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/loom/std/unsafe_cell.rs:14:9
		    26: tokio::runtime::task::core::CoreStage<T>::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:151:13
		    27: tokio::runtime::task::harness::poll_future::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:467:19
		    28: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
		    29: std::panicking::try::do_call
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
		    30: __rust_try
		    31: std::panicking::try
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
		    32: std::panic::catch_unwind
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
		    33: tokio::runtime::task::harness::poll_future
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:455:18
		    34: tokio::runtime::task::harness::Harness<T,S>::poll_inner
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:103:27
		    35: tokio::runtime::task::harness::Harness<T,S>::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:57:15
		    36: tokio::runtime::task::raw::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:128:5
		    37: tokio::runtime::task::raw::RawTask::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:80:18
		    38: tokio::runtime::task::LocalNotified<S>::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/mod.rs:347:9
		    39: tokio::runtime::thread_pool::worker::Context::run_task::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:425:13
		    40: tokio::coop::with_budget::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:102:9
		    41: std::thread::local::LocalKey<T>::try_with
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/local.rs:413:16
		    42: std::thread::local::LocalKey<T>::with
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/local.rs:389:9
		    43: tokio::coop::with_budget
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:95:5
		        tokio::coop::budget
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:72:5
		        tokio::runtime::thread_pool::worker::Context::run_task
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:424:9
		    44: tokio::runtime::thread_pool::worker::Context::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:391:24
		    45: tokio::runtime::thread_pool::worker::run::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:376:17
		    46: tokio::macros::scoped_tls::ScopedKey<T>::set
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/macros/scoped_tls.rs:61:9
		    47: tokio::runtime::thread_pool::worker::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:373:5
		    48: tokio::runtime::thread_pool::worker::Launch::launch::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:352:45
		    49: <tokio::runtime::blocking::task::BlockingTask<T> as core::future::future::Future>::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/task.rs:42:21
		    50: tokio::runtime::task::core::CoreStage<T>::poll::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:161:17
		    51: tokio::loom::std::unsafe_cell::UnsafeCell<T>::with_mut
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/loom/std/unsafe_cell.rs:14:9
		    52: tokio::runtime::task::core::CoreStage<T>::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:151:13
		    53: tokio::runtime::task::harness::poll_future::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:467:19
		    54: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
		    55: std::panicking::try::do_call
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
		    56: __rust_try
		    57: std::panicking::try
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
		    58: std::panic::catch_unwind
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
		    59: tokio::runtime::task::harness::poll_future
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:455:18
		    60: tokio::runtime::task::harness::Harness<T,S>::poll_inner
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:103:27
		    61: tokio::runtime::task::harness::Harness<T,S>::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:57:15
		    62: tokio::runtime::task::raw::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:128:5
		    63: tokio::runtime::task::raw::RawTask::poll
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:80:18
		    64: tokio::runtime::task::UnownedTask<S>::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/mod.rs:384:9
		    65: tokio::runtime::blocking::pool::Task::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:91:9
		    66: tokio::runtime::blocking::pool::Inner::run
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:308:17
		    67: tokio::runtime::blocking::pool::Spawner::spawn_thread::{{closure}}
		               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:288:17
		    68: std::sys_common::backtrace::__rust_begin_short_backtrace
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/sys_common/backtrace.rs:122:18
		    69: std::thread::Builder::spawn_unchecked_::{{closure}}::{{closure}}
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/mod.rs:498:17
		    70: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
		    71: std::panicking::try::do_call
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
		    72: __rust_try
		    73: std::panicking::try
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
		    74: std::panic::catch_unwind
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
		    75: std::thread::Builder::spawn_unchecked_::{{closure}}
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/mod.rs:497:30
		    76: core::ops::function::FnOnce::call_once{{vtable.shim}}
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/ops/function.rs:227:5
		    77: <alloc::boxed::Box<F,A> as core::ops::function::FnOnce<Args>>::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/alloc/src/boxed.rs:1854:9
		        <alloc::boxed::Box<F,A> as core::ops::function::FnOnce<Args>>::call_once
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/alloc/src/boxed.rs:1854:9
		        std::sys::unix::thread::Thread::new::thread_start
		               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/sys/unix/thread.rs:108:17
		    78: start_thread
		    79: __clone
		  
		  ```
- 一些尝试
	- 将 pool_max_idle_per_host 设置为 0，即关闭连接池，这个错误就不会再复现
		- 接下来重点看一下连接池相关的逻辑
		- 一个猜想是如果某个 conn 创建于 runtime A，在 runtime B 中尝试发送请求就会出现该问题
		- 创建 conn 的逻辑如下
			- ```rust
			  let (tx, conn) = conn_builder.handshake(io).await?;
			  
			  trace!("handshake complete, spawning background dispatcher task");
			  executor.execute(
			    conn.map_err(|e| debug!("client connection error: {}", e))
			    .map(|_| ()),
			  );
			  
			  // Wait for 'conn' to ready up before we
			  // declare this tx as usable
			  let tx = tx.when_ready().await?;
			  
			  let tx = {
			    #[cfg(feature = "http2")]
			    {
			      if is_h2 {
			        PoolTx::Http2(tx.into_http2())
			      } else {
			        PoolTx::Http1(tx)
			      }
			    }
			    #[cfg(not(feature = "http2"))]
			    PoolTx::Http1(tx)
			  };
			  
			  Ok(pool.pooled(
			    connecting,
			    PoolClient {
			      conn_info: connected,
			      tx,
			    },
			  ))
			  ```
- 根本原因
	- hyper 每次创建一个新 conn 的时候，都会创建一对 channel，其中 tx 交给用户发送请求，rx 包装进一个 future，并提交给 runtime 执行。
	- 然后 hyper 默认启用了 http keep alive，所有有新的请求进来时会首先从 pool 中寻找可用的链接，并尝试通过 tx 来 send req，如果此时 rx (即之前被提交给 runtime 的 future) 已经被 drop了，就会直接 panic
	- 可能的原因有两种
		- 保存这个 future 的 runtime 已经被 drop (目前来看更有可能是这个)
		- 这个 future 在执行的时候出现了 panic
	- 相关 issue： https://github.com/hyperium/hyper/issues/2649
- 解决方案
	- 每次 query 都创建一个新 client
		- 开销很大，根据 benchmark，每次 `reqwest::new()` 平均需要 2ms
		- ```rust
		  pub fn bench_reqwest(c: &mut Criterion) {
		      c.bench_function("reqwest_new", |b| {
		          b.iter(|| {
		              let _ = reqwest::Client::new();
		          })
		      });
		  }
		  ```
	- 开销较大的一个版本
		- 关闭 keepalive，每次都创建新链接
	- 更有效的版本
		- 把 hyper 的连接池独立出来执行
			- https://docs.rs/hyper/latest/hyper/client/struct.Builder.html#method.executor
- For Databed
	- 现在 databend 已经有了一个独立的 IO 线程池，或许可以直接使用它？
	- reqwest 不支持 set 底层的 runtime，所以需要改造一下 opendal，直接操作 hyper client
	- 然后 databend 的 runtime 需要改造一下，支持暴露底层的 handle
	- 都做好了，让我们来测试一下吧！
		- DONE！Fixed.
- 新的问题
  collapsed:: true
	- 使用了 external runtime 之后会出现来自 tokio 的报错
	- ```rust
	  2022-03-17T12:17:51.646580Z ERROR opendal::services::s3::backend: object 2/1/_ss/1c99cd32e61d4eeb8df4243492dd290b head_object: hyper::Error(Io, Custom { kind: Other, error: "IO driver has terminated" })
	  2022-03-17T12:17:51.664635Z  WARN common_base::runtime: runtime "query-ctx" dropped
	  2022-03-17T12:17:51.664648Z ERROR databend_query::servers::mysql::writers::query_result_writer: OnQuery Error: Code: 3003, displayText = unexpected: (source: connection error: IO driver has terminated).
	  
	     0: common_exception::exception_code::<impl common_exception::exception::ErrorCode>::DalTransportError
	               at /home/xuanwo/Code/datafuselabs/databend/common/exception/src/exception_code.rs:36:66
	     1: <&databend_query::sessions::query_ctx::QueryContext as databend_query::storages::fuse::io::meta_readers::BufReaderProvider>::buf_reader::{{closure}}::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/io/meta_readers.rs:176:26
	     2: core::result::Result<T,E>::map_err
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/result.rs:842:27
	     3: <&databend_query::sessions::query_ctx::QueryContext as databend_query::storages::fuse::io::meta_readers::BufReaderProvider>::buf_reader::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/io/meta_readers.rs:174:28
	     4: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	     5: <core::pin::Pin<P> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
	     6: databend_query::storages::fuse::io::meta_readers::<impl databend_query::storages::fuse::cache::local_cache::Loader<V> for T>::load::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/io/meta_readers.rs:134:59
	     7: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	     8: <core::pin::Pin<P> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
	     9: databend_query::storages::fuse::cache::local_cache::CachedReader<V,L>::load::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/cache/local_cache.rs:103:46
	    10: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    11: databend_query::storages::fuse::cache::local_cache::CachedReader<V,L>::read::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/cache/local_cache.rs:67:49
	    12: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    13: databend_query::storages::fuse::fuse_table::FuseTable::read_table_snapshot::{{closure}}::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/fuse_table.rs:185:38
	    14: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    15: databend_query::storages::fuse::fuse_table::FuseTable::read_table_snapshot::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/fuse_table.rs:178:5
	    16: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    17: databend_query::storages::fuse::operations::commit::<impl databend_query::storages::fuse::fuse_table::FuseTable>::try_commit::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/operations/commit.rs:137:49
	    18: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    19: databend_query::storages::fuse::operations::commit::<impl databend_query::storages::fuse::fuse_table::FuseTable>::do_commit::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/operations/commit.rs:85:69
	    20: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    21: <databend_query::storages::fuse::fuse_table::FuseTable as databend_query::storages::storage_table::Table>::commit_insertion::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/storages/fuse/fuse_table.rs:154:59
	    22: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    23: <core::pin::Pin<P> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
	    24: <databend_query::interpreters::interpreter_insert::InsertInterpreter as databend_query::interpreters::interpreter::Interpreter>::execute::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/interpreters/interpreter_insert.rs:118:14
	    25: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    26: <core::pin::Pin<P> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
	    27: <databend_query::interpreters::interpreter_factory_interceptor::InterceptorInterpreter as databend_query::interpreters::interpreter::Interpreter>::execute::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/interpreters/interpreter_factory_interceptor.rs:62:61
	    28: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    29: <core::pin::Pin<P> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/future.rs:124:9
	    30: databend_query::servers::mysql::mysql_interactive_worker::InteractiveWorkerBase<W>::exec_query::{{closure}}::{{closure}}::{{closure}}
	               at /home/xuanwo/Code/datafuselabs/databend/query/src/servers/mysql/mysql_interactive_worker.rs:323:60
	    31: <core::future::from_generator::GenFuture<T> as core::future::future::Future>::poll
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/future/mod.rs:91:19
	    32: <tracing::instrument::Instrumented<T> as core::future::future::Future>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tracing-0.1.31/src/instrument.rs:272:9
	    33: tokio::runtime::task::core::CoreStage<T>::poll::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:161:17
	    34: tokio::loom::std::unsafe_cell::UnsafeCell<T>::with_mut
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/loom/std/unsafe_cell.rs:14:9
	    35: tokio::runtime::task::core::CoreStage<T>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:151:13
	    36: tokio::runtime::task::harness::poll_future::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:467:19
	    37: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
	    38: std::panicking::try::do_call
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
	    39: __rust_try
	    40: std::panicking::try
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
	    41: std::panic::catch_unwind
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
	    42: tokio::runtime::task::harness::poll_future
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:455:18
	    43: tokio::runtime::task::harness::Harness<T,S>::poll_inner
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:103:27
	    44: tokio::runtime::task::harness::Harness<T,S>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:57:15
	    45: tokio::runtime::task::raw::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:128:5
	    46: tokio::runtime::task::raw::RawTask::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:80:18
	    47: tokio::runtime::task::LocalNotified<S>::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/mod.rs:347:9
	    48: tokio::runtime::thread_pool::worker::Context::run_task::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:425:13
	    49: tokio::coop::with_budget::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:102:9
	    50: std::thread::local::LocalKey<T>::try_with
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/local.rs:413:16
	    51: std::thread::local::LocalKey<T>::with
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/local.rs:389:9
	    52: tokio::coop::with_budget
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:95:5
	        tokio::coop::budget
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/coop.rs:72:5
	        tokio::runtime::thread_pool::worker::Context::run_task
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:424:9
	    53: tokio::runtime::thread_pool::worker::Context::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:391:24
	    54: tokio::runtime::thread_pool::worker::run::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:376:17
	    55: tokio::macros::scoped_tls::ScopedKey<T>::set
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/macros/scoped_tls.rs:61:9
	    56: tokio::runtime::thread_pool::worker::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:373:5
	    57: tokio::runtime::thread_pool::worker::Launch::launch::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/thread_pool/worker.rs:352:45
	    58: <tokio::runtime::blocking::task::BlockingTask<T> as core::future::future::Future>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/task.rs:42:21
	    59: tokio::runtime::task::core::CoreStage<T>::poll::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:161:17
	    60: tokio::loom::std::unsafe_cell::UnsafeCell<T>::with_mut
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/loom/std/unsafe_cell.rs:14:9
	    61: tokio::runtime::task::core::CoreStage<T>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/core.rs:151:13
	    62: tokio::runtime::task::harness::poll_future::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:467:19
	    63: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
	    64: std::panicking::try::do_call
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
	    65: __rust_try
	    66: std::panicking::try
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
	    67: std::panic::catch_unwind
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
	    68: tokio::runtime::task::harness::poll_future
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:455:18
	    69: tokio::runtime::task::harness::Harness<T,S>::poll_inner
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:103:27
	    70: tokio::runtime::task::harness::Harness<T,S>::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/harness.rs:57:15
	    71: tokio::runtime::task::raw::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:128:5
	    72: tokio::runtime::task::raw::RawTask::poll
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/raw.rs:80:18
	    73: tokio::runtime::task::UnownedTask<S>::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/task/mod.rs:384:9
	    74: tokio::runtime::blocking::pool::Task::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:91:9
	    75: tokio::runtime::blocking::pool::Inner::run
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:308:17
	    76: tokio::runtime::blocking::pool::Spawner::spawn_thread::{{closure}}
	               at /home/xuanwo/.cargo/registry/src/github.com-1ecc6299db9ec823/tokio-1.17.0/src/runtime/blocking/pool.rs:288:17
	    77: std::sys_common::backtrace::__rust_begin_short_backtrace
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/sys_common/backtrace.rs:122:18
	    78: std::thread::Builder::spawn_unchecked_::{{closure}}::{{closure}}
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/mod.rs:498:17
	    79: <core::panic::unwind_safe::AssertUnwindSafe<F> as core::ops::function::FnOnce<()>>::call_once
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/panic/unwind_safe.rs:271:9
	    80: std::panicking::try::do_call
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:492:40
	    81: __rust_try
	    82: std::panicking::try
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panicking.rs:456:19
	    83: std::panic::catch_unwind
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/panic.rs:137:14
	    84: std::thread::Builder::spawn_unchecked_::{{closure}}
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/thread/mod.rs:497:30
	    85: core::ops::function::FnOnce::call_once{{vtable.shim}}
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/core/src/ops/function.rs:227:5
	    86: <alloc::boxed::Box<F,A> as core::ops::function::FnOnce<Args>>::call_once
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/alloc/src/boxed.rs:1854:9
	        <alloc::boxed::Box<F,A> as core::ops::function::FnOnce<Args>>::call_once
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/alloc/src/boxed.rs:1854:9
	        std::sys::unix::thread::Thread::new::thread_start
	               at /rustc/8769f4ef2fe1efddd1f072485f97f568e7328f79/library/std/src/sys/unix/thread.rs:108:17
	    87: start_thread
	    88: __clone
	  
	  ```
	- 这是一个可以忽略的错误吗？
		- 不是的， 最后推出的时候有一些相关的报错可以理解，但是在运行过程中不应该出现
		- 或许能允许 retry？
	- 强行手动指定成同一个 runtime 是没有问题的
		- 说明目前的 IO 逻辑没有问题，接下来要看 processor 这边
	- 有没有可能是返回的 Reader 之类的结构体离开了 runtime ？
		- 从现象上看确实是在其他的 runtime 关闭之后就开始出错了，但是不一定是返回的 Reader，write  请求也有报错
			- ```rust
			  2022-03-17T13:38:18.204372Z  WARN common_base::runtime: runtime "query-ctx" dropped
			  2022-03-17T13:38:18.204856Z DEBUG hyper::client::pool: reuse idle connection for ("http", 127.0.0.1:9900)
			  2022-03-17T13:38:18.205316Z DEBUG hyper::proto::h1::io: flushed 65951 bytes
			  2022-03-17T13:38:18.205577Z DEBUG hyper::proto::h1::io: flushed 29849 bytes
			  2022-03-17T13:38:18.205850Z DEBUG hyper::proto::h1::conn: parse error (connection error: IO driver has terminated) with 0 bytes
			  2022-03-17T13:38:18.206000Z DEBUG hyper::proto::h1::dispatch: read_head error: connection error: IO driver has terminated
			  2022-03-17T13:38:18.206215Z ERROR opendal::services::s3::backend: object 2/1/_ss/251de0a2c81f4884ae5f8259a2ee562a put_object: hyper::Error(Io, Custom { kind: Other, error: "IO driver has terminated" })
			  ```
	- 创建一个全局的 runtime 给 IO 使用确认不行
	- 一个可能有问题的地方
		- hyper 创建 timer 的时候使用的是 tokio::time::sleep，有没有可能这个 timer 被绑定在了当前 runtime？
			- ```rust
			  let interval = IdleTask {
			    interval: tokio::time::interval(dur),
			    pool: WeakOpt::downgrade(pool_ref),
			    pool_drop_notifier: rx,
			  };
			  
			  self.exec.execute(interval);
			  ```
			- 看起来并非如此
	- 去掉 QueryContrxt 的 drop 调用也 OK
		- 不过这样肯定不对，有内存泄漏
		- Runtime 没有释放，所以肯定就没问题了
		- 还是需要把所有的 IO 请求都调度到统一的 runtime 上
	- 卧槽，我是天才
		- OpenDAL 的 Layer 设计可以非常轻松的调度所有的 IO 请求
		- ```rust
		  async fn read(&self, args: &OpRead) -> DalResult<BoxedAsyncReader> {
		    let op = self.inner.as_ref().unwrap().clone();
		    let args = args.clone();
		    self.runtime
		    .spawn(async move { op.read(&args).await })
		    .await
		    .expect("join must success")
		  }
		  ```
			- 需要评估一下这里的额外开销有多大
-
- 感觉可以整理整理写成文章: [[Hyper Debug 之旅]]