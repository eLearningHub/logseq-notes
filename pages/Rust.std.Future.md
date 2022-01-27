- [Future](https://doc.rust-lang.org/std/future/trait.Future.html) 是 [[Rust]] 标准库提供的异步计算抽象
- id:: 61ee5b4e-da85-4b59-a318-356e5415afbd
  ```rust
  pub trait Future {
      type Output;
      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
  }
  ```
	- `Output` 是 trait 的关联类型，在此处不再赘述
	- `Pin<&mut Self>` 用于保证 Self 不被 move，详见 [Pin]([[Rust/std/Pin]])
	- `Context` 负责维护异步任务的上下文
		- 目前只提供了 `Waker`，用于通知 executor 当前 future 已经准备好执行了 (ready for wake up)
	- `Poll` 是标准库提供的 enum，有两个状态
		- `Poll::Pending`，表示当前 future 还没有准备好
		- `Poll::Ready(Output)`，表示当前 future 已经准备好返回值
	- `poll` 是 trait 提供的唯一方法，旨在 **尝试** 将某个 future 转换为最终状态
		- 当最终 value 没有准备好的时候，不该 block
			- 应当立刻返回 Pending
		- 当前的任务会被调度，并在未来被唤醒并再度执行
			- poll 可能会被执行多次，需要维护内部状态
		- `poll` 只能返回一次 `Ready`
			- runner 不会再尝试去 poll 这个 future
			- 用户 poll 已完成的 future 会导致未定义行为
-
- 如何实现一个 Future？
	- **Future 内部的状态需要自行维护**，Executor 不会记录函数当前的上下文
	- 比如说我们可以在结构体中维护一个 flag：
		- *来自 [Build a Timer](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html)*
			- ```rust
			  pub struct TimerFuture {
			      shared_state: Arc<Mutex<SharedState>>,
			  }
			  
			  /// Shared state between the future and the waiting thread
			  struct SharedState {
			      /// Whether or not the sleep time has elapsed
			      completed: bool,
			  
			      waker: Option<Waker>,
			  }
			  
			  impl Future for TimerFuture {
			      type Output = ();
			      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
			          // Look at the shared state to see if the timer has already completed.
			          let mut shared_state = self.shared_state.lock().unwrap();
			          if shared_state.completed {
			              Poll::Ready(())
			          } else {
			              shared_state.waker = Some(cx.waker().clone());
			              Poll::Pending
			          }
			      }
			  }
			  
			  impl TimerFuture {
			      /// Create a new `TimerFuture` which will complete after the provided
			      /// timeout.
			      pub fn new(duration: Duration) -> Self {
			          let shared_state = Arc::new(Mutex::new(SharedState {
			              completed: false,
			              waker: None,
			          }));
			  
			          // Spawn the new thread
			          let thread_shared_state = shared_state.clone();
			          thread::spawn(move || {
			              thread::sleep(duration);
			              let mut shared_state = thread_shared_state.lock().unwrap();
			              // Signal that the timer has completed and wake up the last
			              // task on which the future was polled, if one exists.
			              shared_state.completed = true;
			              if let Some(waker) = shared_state.waker.take() {
			                  waker.wake()
			              }
			          });
			  
			          TimerFuture { shared_state }
			      }
			  }
			  ```
	- 更常见的用法是我们会构造出一个状态机，不断的去推进这个 Future 的状态，直到最终返回了 Ready
		- 形如这样的代码就会陷入不断 Pending 的死循环
			- 来自 [dal2: Implement SeekableReader](https://github.com/datafuselabs/databend/pull/3934)
			- ```rust
			  impl<S> AsyncRead for SeekableReader<'_, S>
			  where S: super::Read<S>
			  {
			      fn poll_read(
			          mut self: Pin<&mut Self>,
			          cx: &mut Context<'_>,
			          buf: &mut [u8],
			      ) -> Poll<io::Result<usize>> {
			          let key = self.key.clone();
			          let da = self.da.clone();
			          let pos = self.pos;
			  
			          let n = async {
			              let mut builder = da.read(key.as_str());
			              let r = builder
			                  .offset(pos)
			                  .size(buf.len() as u64)
			                  .run()
			                  .await
			                  .map_err(io::Error::other)?;
			  
			              Pin::new(r).read(buf).await
			          };
			  
			          pin_mut!(n);
			  
			          let n = ready!(n.poll(cx))?;
			  
			          self.pos += n as u64;
			          Poll::Ready(Ok(n))
			      }
			  }
			  ```
		- 因为 Executor 每次来 `poll` 的时候，我们都会生成一个全新的 Future 返回，每次 resolve 得到的状态都是 `Pending`
		- 所以我们需要在这个 Reader 内部维护它的 State，不断的推进状态
			- ```rust
			  enum State<'d> {
			      Idle,
			      Starting(Pin<Box<dyn Future<Output = Result<Reader>> + Send + 'd>>),
			      Reading(Reader),
			  }
			  
			  impl<'d, S> AsyncRead for SeekableReader<'d, S>
			  where S: super::Read<S> + 'd
			  {
			      fn poll_read(
			          mut self: Pin<&mut Self>,
			          cx: &mut Context<'_>,
			          buf: &mut [u8],
			      ) -> Poll<io::Result<usize>> {
			          loop {
			              match self.state {
			                  SeekableReaderState::Idle => {
			                      // build future, bala, bala
			                      self.state = SeekableReaderState::Starting(f.boxed());
			                  }
			                  SeekableReaderState::Starting(ref mut fut) => {
			                      let r = ready!(fut.as_mut().poll(cx)).map_err(io::Error::other)?;
			  
			                      self.state = SeekableReaderState::Reading(r);
			                  }
			                  SeekableReaderState::Reading(ref mut r) => {
			                      pin_mut!(r);
			  
			                      let n = ready!(r.poll_read(cx, buf))?;
			                      self.pos += n as u64;
			                      return Poll::Ready(Ok(n));
			                  }
			              }
			          }
			      }
			  }
			  ```
	- 实际上 `await` 做的事情就是生成了一个 Generators 来维护 Future 的状态
		- 参考伪代码，*来自 [Rust 的 async/await 语法是怎样工作的](https://ipotato.me/article/70)*
			- ```rust
			  struct GeneratorY {
			      state: i32,
			      task_context: Context<'static>,
			      future: dyn Future<Output = Vec<i32>>,
			  }
			  
			  impl Generator for GeneratorY {
			      type Yield = ();
			      type Return = i32;
			  
			      fn resume(mut self: Pin<&mut Self>, resume: ()) -> GeneratorState<Self::Yield, Self::Return> {
			          match self.state {
			              0 => {
			                  self.task_context = Context::new();
			                  self.future = into_future(x());
			                  self.state = 1;
			                  self.resume(resume)
			              }
			              1 => {
			                  let result = loop {
			                      if let Poll::Ready(result) =
			                          Pin::new_unchecked(self.future.get_mut()).poll(self.task_context)
			                      {
			                          break result;
			                      }
			                      return GeneratorState::Yielded(());
			                  };
			                  self.state = 2;
			                  GeneratorState::Complete(result)
			              }
			              _ => panic!("GeneratorY polled with an invalid state"),
			          }
			      }
			  }
			  ```
-
- 常见的技巧
	- 如果需要将 Future 存储起来的话，我们通常会需要 `Pin<Box<dyn Future<Output = T> + Send>>`
		- 特别的，我们有时候还需要附加上生命周期以满足要求 `Pin<Box<dyn Future<Output = T> + Send + 'xxx>>`
-
- 参考资料
	- [Rust futures RFC](https://github.com/rust-lang/rfcs/blob/master/text/2592-futures.md)
	- [Rust Async & Await RFC](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)
	- [Rust 的 async/await 语法是怎样工作的](https://ipotato.me/article/70) *推荐*
	- [Generators and async/await](https://cfsamson.github.io/books-futures-explained/4_generators_async_await.html#generators-and-asyncawait)
	- [Inside Rust's Async Transform](https://blag.nemo157.com/2018/12/09/inside-rusts-async-transform.html)
	- [@sticnarf 关于 Rust Async 的分享](https://docs.google.com/presentation/d/1UYGAAm60-FCudvEmXPV0Ca6REo-nvN_L73ddWIDBuik/edit)
	-