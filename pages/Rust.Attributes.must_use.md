- `must_use` 用来标记某个值，让它在没有被使用就会触发一个警告。
- 这个属性能够被用在结构体，方法和 trait 里面，用来确保 caller 调用正确。
-
- 比如在 [[Rust/std Future]] 中，源码中会有这样的一行：
	- ```rust
	  #[must_use = "futures do nothing unless you `.await` or poll them"]
	  pub trait Future {
	      type Output;
	      fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
	  }
	  ```
	- 如果我们构造了一个 future，但是没有被调用 `.await` / `poll` ，编译期就会给出报错
- 还有一个特别典型的案例就是 [[Rust/std/Result]] 中通过 `must_use` 来确保 result 被显式的处理过
	- ```rust
	  #[must_use = "this `Result` may be an `Err` variant, which should be handled"]
	  pub enum Result<T, E> {
	      Ok(T),
	  
	      Err(E),
	  }
	  ```
- 实现原理
	-
- 参考资料
	- [Must Use Functions](https://rust-lang.github.io/rfcs/1940-must-use-functions.html)
	- [The must_use attribute](https://doc.rust-lang.org/reference/attributes/diagnostics.html#the-must_use-attribute)
	- [When to add #[must_use]](https://std-dev-guide.rust-lang.org/code-considerations/design/must-use.html)