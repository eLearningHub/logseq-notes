- [Future](https://doc.rust-lang.org/std/future/trait.Future.html) 是 [[Rust]] 标准库提供的异步计算抽象
- id:: 61ee5b4e-da85-4b59-a318-356e5415afbd
  ```rust
  pub trait Future {
      type Output;
      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
  }
  ```
	- `Output` 是 trait 的关联类型，在此处不再赘述
	- `Pin<&mut Self>` 用于保证 Self 不被 move，详见 [Pin]([[Rust/std Pin]])
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
	- 实际上 `await` 做的事情就是生成了一个 Generators 来维护 Future 的状态
		- 来自 []
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
- 参考资料
	- [Rust Async & Await RFC](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)
	- [Rust 的 async/await 语法是怎样工作的](https://ipotato.me/article/70) *推荐*
-