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
	- Future is a Future，
-
- 参考资料
	- [Rust Async & Await RFC](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)
	- [Rust 的 async/await 语法是怎样工作的](https://ipotato.me/article/70)
-