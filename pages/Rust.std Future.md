- ```rust
  pub trait Future {
      type Output;
      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
  }
  ```
-
-
-
- 如何实现一个 Future？
	-
-
- 参考资料
	- [Rust Async & Await RFC](https://github.com/rust-lang/rfcs/blob/master/text/2394-async_await.md)
	- [Rust 的 async/await 语法是怎样工作的](https://ipotato.me/article/70)
-