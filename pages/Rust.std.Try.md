title:: Rust/std/Try

- [Try](https://doc.rust-lang.org/std/ops/trait.Try.html) 是 [[Rust]] 在 `try_trait_v2` feature 中引入的新 ops，旨在支持 `?` 操作符和 `try {}` block。
	- Tracking Issue: [Tracking Issue for try_trait_v2, A new design for the ? desugaring](https://github.com/rust-lang/rust/issues/84277)
-
- 现在 Rust 中支持通过 `?` 来自动返回 `Result<T, E>` 中的 `Err` 分支，但是 `Ok` 分支还是处理的不好。
-
- 过去 `x?` 等价于
- ```rust
  match Try::into_result(x) {
      Ok(v) => v,
      Err(e) => return Try::from_error(From::from(e)),
  }
  ```
- 在引入了 `try_trait_v2` 之后会变成
- ```rust
  match Try::branch(x) {
      ControlFlow::Continue(v) => v,
      ControlFlow::Break(r) => return FromResidual::from_residual(r),
  }
  ```
-
- 用户可以在实现 `Try` 的时候控制 `ControlFlow::Continue` 中到底是什么。
-
- `Option` 和 `Rusult` 的实现都比较显然，可以以更有用的 `std::task::Ready` 为例：
-
-
- ```rust
  #[unstable(feature = "poll_ready", issue = "89780")]
  pub struct Ready<T>(pub(crate) Poll<T>);
  
  #[unstable(feature = "poll_ready", issue = "89780")]
  impl<T> Try for Ready<T> {
      type Output = T;
      type Residual = Ready<convert::Infallible>;
  
      #[inline]
      fn from_output(output: Self::Output) -> Self {
          Ready(Poll::Ready(output))
      }
  
      #[inline]
      fn branch(self) -> ControlFlow<Self::Residual, Self::Output> {
          match self.0 {
              Poll::Ready(v) => ControlFlow::Continue(v),
              Poll::Pending => ControlFlow::Break(Ready(Poll::Pending)),
          }
      }
  }
  
  #[unstable(feature = "poll_ready", issue = "89780")]
  impl<T> FromResidual for Ready<T> {
      #[inline]
      fn from_residual(residual: Ready<convert::Infallible>) -> Self {
          match residual.0 {
              Poll::Pending => Ready(Poll::Pending),
          }
      }
  }
  ```
- 通过 `try_trait_v2` 的支持，我们可以构造一个 `Ready<T>` 并实现 `Try`，从而实现自动提取 `Poll<T>` 中的 `T`。过去需要使用一个 macro `ready!(fut.poll(cx))` ，现在可以很舒服地改写成 `fut.poll(cx).ready()?`
-
-
- 参考资料
	- [Trait std::ops::Try](https://doc.rust-lang.org/std/ops/trait.Try.html)